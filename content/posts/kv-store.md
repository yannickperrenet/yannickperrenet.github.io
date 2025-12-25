---
title: "Thread-safe (in-memory) key-value store with revisions and TTL"
date: 2025-12-25
lastmod: 2025-12-25
---

As a fun exercise I tried to code up a thread-safe key-value store with revisions and TTL, meaning that values can time out and past value can be accessed through a full revision log. There are a bunch of open-source key-value store out there to learn and draw inspiration, such as [Redis](https://github.com/redis/redis) and [etcd](https://github.com/etcd-io/etcd).

## First try (no threading)

You'll need at least Python3.12 and [`sortedcontainers`](https://grantjenks.com/docs/sortedcontainers/) to run this.

```python
from collections import defaultdict
from sortedcontainers import SortedList
from typing import NamedTuple
import bisect
import operator

type _Timestamp = int

class _LogEntry[V](NamedTuple):
    timestamp: _Timestamp
    value: V | None

class Store[K: str, V: int]:
    """Key-value store with TTL.

    It is required for methods to be called with increasing
    `timestamp`.
    """
    def __init__(self):
        self.log: defaultdict[K, list[_LogEntry[V]]] = defaultdict(list)

        self.ttl_map: dict[K, _Timestamp] = {}
        # https://peps.python.org/pep-0257/
        """Key `K` will timeout at `_Timestamp`

        If `K` is not present, then `K` will have an infinite TTL.
        """
        self.ttls = SortedList()
        "`SortedList[tuple[_Timestamp, K]]`"

    def _honor_ttl(self, timestamp: _Timestamp) -> None:
        while self.ttls and self.ttls[0][0] <= timestamp:
            ttl, key = self.ttls.pop(0)
            if self.ttl_map.get(key, -1) == ttl:
                self.ttl_map.pop(key)

            self.log[key].append(_LogEntry(ttl, None))

    def get(self, timestamp: _Timestamp, key: K) -> V | None:
        self._honor_ttl(timestamp)

        if (log := self.log.get(key, [])):
            return log[-1].value

    def set(self, timestamp: _Timestamp, key: K, value: V) -> None:
        """Set key-value pair without TTL.

        Meaning that the pair will never time out.
        """
        self._honor_ttl(timestamp)

        self.log[key].append(_LogEntry(timestamp, value))
        try:
            ttl = self.ttl_map.pop(key)
            self.ttls.remove((ttl, key))
        except KeyError:
            ...

    def set_with_ttl(self, timestamp: _Timestamp, key: K, value: V, ttl: int) -> None:
        if ttl <= 0:
            raise ValueError("TTL should be at least 1.")

        self._honor_ttl(timestamp)

        self.log[key].append(_LogEntry(timestamp, value))
        try:
            prev_ttl = self.ttl_map.pop(key)
            self.ttls.remove((prev_ttl, key))
        except KeyError:
            ...

        self.ttl_map[key] = timestamp + ttl
        self.ttls.add((timestamp + ttl, key))

    def scan(self, timestamp: _Timestamp) -> list[str]:
        """Get all key-value pairs.

        Format for a pair is: `"<key>(<value>)"`
        """
        self._honor_ttl(timestamp)

        ans = []
        for key in self.log:
            value = self.get(timestamp, key)
            if (value := self.get(timestamp, key)) is not None:
                ans.append(f"{key}({value})")
        return ans

    def get_at(
        self,
        timestamp: _Timestamp,
        key: K,
        at_timestamp: _Timestamp,
    ) -> V | None:
        if at_timestamp > timestamp:
            raise ValueError("It should hold that: at_timestamp <= timestamp.")

        self._honor_ttl(timestamp)

        if (log := self.log.get(key)) is None:
            return

        i = -1 + bisect.bisect_right(
                log,
                at_timestamp,
                key=operator.attrgetter("timestamp")
        )
        return log[i].value if i >= 0 else None
```

Some notes about the implementation:

-   Implementing revisions was as simple as storing a `list` of entries together with their timestamp. When updating the value of a key, a simply `.append()` call does the trick.

-   Since we store a full revision log there is no need to remove timed out values as soon as possible. We don't free up any memory. Instead every call that accesses the store will first have to update all TTL'd values through `_honor_ttl()`.

-   Tracking TTL required a `self.ttl_map: dict[K, _Timestamp] = {}` to find the timestamp of a key in order to remove the `(_Timestamp, K)` entry from the `SortedList` whenever the TTL value was updated.

    Now we need to keep two data structures consistent and we are relying on a third-party package (don't get me wrong, `sortedcontainers` is an absolute killer package!). Unlike the [`heap.go`](https://cs.opensource.google/go/go/+/refs/tags/go1.25.5:src/container/heap/heap.go) implementation that [etcd uses](https://github.com/etcd-io/etcd/blob/88feab3eecbeabd018876e86a9b9e448a26cd610/server/etcdserver/api/v2store/ttl_key_heap.go#L20), Python's [`heapq` module](https://docs.python.org/3/library/heapq.html) does [not have](https://github.com/python/cpython/issues/112498) a `.heapremove()` method that runs in `O(log n)` to allow performant updates. All heap modifications would have to be tracked so that given a key its index in the heap can be found and a corresponding sift down operation can move the entry to the end of the list for a `.pop()` call.

    This can not be implemented on top of `heapq` as heap modifications as part of its methods can't be tracked like they can through the `heap.go` interface.

-   Adding new methods or new features to the store will be hard as they all have to make sure to
    correctly update the existing structures. As you can see, currently each method is updating the
    `self.ttl_map` and `self.ttls`, this is bound to introduce issues down the road.

## Second try

```python
from collections import defaultdict
from collections.abc import Hashable
from typing import NamedTuple
import bisect
import functools
import operator
import random
import threading
import time

type _Timestamp = int
type _Immutable = int | str | bytes

MAX_LOG_SIZE: int = 5

class _LogEntry[V: _Immutable](NamedTuple):
    timestamp: _Timestamp
    value: V | None
    """Immutable value.

    If the value were mutable, then an entry could be changed
    someplace else and the entry would no longer be the entry
    at time of insertion. So not a true snapshot.
    """

def _behind_mutex(meth):
    @functools.wraps(meth)
    def wrapper(self, *args, **kwargs):
        with self._mutex:
            return meth(self, *args, **kwargs)
    return wrapper

class _Entry[V: _Immutable]:
    def __init__(self, expires_at: _Timestamp | None = None):
        self.expires_at: _Timestamp | None = expires_at
        """Timestamp at which the current value expires."""
        self._log: list[_LogEntry[V]] = []
        """Revision log of values."""

        self._mutex = threading.Lock()

    @_behind_mutex
    def set_value(
        self,
        timestamp: _Timestamp,
        value: V,
        *,
        # Has to be passed as keyword argument.
        ttl: int | None,
    ) -> None:
        """Set value for `T >= timestamp` to `value`."""
        if self._log and self._log[-1].timestamp > timestamp:
            raise RuntimeError("Timestamp values are not increasing.")

        if self.expires_at is not None and self.expires_at <= timestamp:
            self._log.append(_LogEntry(self.expires_at, None))

        if ttl is None:
            self.expires_at = None
        else:
            self.expires_at = timestamp + ttl

        self._log.append(_LogEntry(timestamp, value))

    @_behind_mutex
    def get_value_at(self, timestamp: _Timestamp) -> V | None:
        """Get value at `timestamp`."""
        if not self._log:
            return None
        elif self.expires_at is not None and self.expires_at <= timestamp:
            return None
        # Quick path for regular get().
        elif self._log[-1].timestamp <= timestamp:
            return self._log[-1].value
        # Revision path.
        else:
            i = -1 + bisect.bisect_right(
                        self._log,
                        timestamp,
                        key=operator.attrgetter("timestamp"))
            return self._log[i].value if i >= 0 else None

    @_behind_mutex
    def trim_log(self) -> None:
        del self._log[:-MAX_LOG_SIZE]

class Store[K: Hashable, V: _Immutable]:
    """Key-value store with TTL.

    It is required for methods to be called with increasing
    `timestamp`.
    """
    def __init__(self, trim: bool = False):
        self.map: defaultdict[K, _Entry[V]] = defaultdict(_Entry)
        # Duplicate keys so we can pick a number of them at random
        # in O(1).
        self.keys: list[K] = []
        self._mutex = threading.Lock()
        """Mutex to protect adding and removing keys."""

        # https://github.com/etcd-io/etcd/blob/main/server/lease/lessor.go#L245
        if trim:
            self._trimmer_thread = threading.Thread(
                target=self._run_trimmer,
                daemon=True
            )
            self._trimmer_thread.start()

    def _run_trimmer(self):
        """Periodically trim the revision log of entries.

        Entries are selected randomly.
        """
        while True:
            time.sleep(0.5)

            with self._mutex:
                to_trim = random.sample(self.keys, k=min(len(self.keys), 3))

            for key in to_trim:
                with self._mutex:
                    self.map[key].trim_log()

    def get(self, timestamp: _Timestamp, key: K) -> V | None:
        # Check __contains__ first to prevent increasing defaultdict
        # size.
        if key not in self.map:
            return None
        else:
            return self.map[key].get_value_at(timestamp)

    def set(self, timestamp: _Timestamp, key: K, value: V) -> None:
        """Set key-value pair without TTL.

        Meaning that the value will never time out.
        """
        if key in self.map:
            # Use lock of underlying _Entry.
            self.map[key].set_value(timestamp, value, ttl=None)
        else:
            with self._mutex:
                self.map[key].set_value(timestamp, value, ttl=None)
                self.keys.append(key)

    def set_with_ttl(self, timestamp: _Timestamp, key: K, value: V, ttl: int) -> None:
        if ttl <= 0:
            raise ValueError("TTL should be at least 1.")

        if key in self.map:
            self.map[key].set_value(timestamp, value, ttl=ttl)
        else:
            with self._mutex:
                self.map[key].set_value(timestamp, value, ttl=ttl)
                self.keys.append(key)

    def scan(self, timestamp: _Timestamp) -> list[str]:
        """Get all key-value pairs.

        Format for a pair is: `"<key>(<value>)"`
        """
        ans = []
        for key, entry in self.map.items():
            if (value := entry.get_value_at(timestamp)) is not None:
                ans.append(f"{key}({value})")
        return ans

    def get_at(
        self,
        timestamp: _Timestamp,
        key: K,
        at_timestamp: _Timestamp,
    ) -> V | None:
        if at_timestamp > timestamp:
            raise ValueError("It should hold that: at_timestamp <= timestamp.")

        if key not in self.map:
            return None
        else:
            return self.map[key].get_value_at(at_timestamp)
```

Some notes about the implementation:

-   Introduced a long-running thread that trims revision to `MAX_LOG_SIZE`. Trimming is done by randomly selecting keys to trim as Redis [does the same](https://redis.io/docs/latest/commands/expire/#how-redis-expires-keys) for removing TTL'd values. Of course, both are very different things but as an exercise I thought it was good practice.

-   Easier to read and maintain version as TTL adherence is offloaded to the `_Entry` class. This
    allowed for easily adding locking mechanisms to make the store thread-safe.

-   The `trim_log()` methods uses `del log[:-N]` syntax to trim older entries, which resolves to the exact same function call in CPython as `log[:-N] = []` when following the bytecodes [`DELETE_SUBSCR`](https://docs.python.org/3/library/dis.html#opcode-DELETE_SUBSCR) and [`STORE_SLICE`](https://docs.python.org/3/library/dis.html#opcode-STORE_SLICE) respectively.

    [`DELETE_SUBSCR`](https://github.com/python/cpython/blob/86d904588e8c84c7fccb8faf84b343f03461970d/Python/bytecodes.c#L1143) calls `PyObject_DelItem` and [`STORE_SLICE`](https://github.com/python/cpython/blob/86d904588e8c84c7fccb8faf84b343f03461970d/Python/bytecodes.c#L871) calls `PyObject_SetItem`. Which can both be followed further using something like GitHub's code search.

    One way to make the trimming faster is by using a FIFO queue (like [`collections.deque`](https://docs.python.org/3/library/collections.html#collections.deque)) so items can be removed from the front in O(1) instead of O(n).

## Test cases

Besides the below unittests a great way to test the thread-safety of the store would be using the new [`InterpreterPoolExecutor`](https://docs.python.org/3/library/concurrent.futures.html#interpreterpoolexecutor) introduced in Python3.14 that has true multi-core parallelism by spawning an interpreter for each thread (so each has its own GIL).

```python
import unittest

class TestStore(unittest.TestCase):

    def test_without_ttl(self):
        store = Store[str, int]()
        T = 0

        store.set(T + 1, "1", 1)
        store.set(T + 2, "2", 2)
        self.assertEqual(store.scan(T + 3), ["1(1)", "2(2)"])

        store.set(T + 4, "1", 3)
        self.assertEqual(store.scan(T + 5), ["1(3)", "2(2)"])
        self.assertEqual(store.get_at(T + 6, "1", T + 3), 1)
        self.assertEqual(store.get_at(T + 7, "1", T + 5), 3)

    def test_with_ttl(self):
        store = Store[str, int]()
        T = 0

        store.set(T + 1, "1", 1)
        store.set_with_ttl(T + 2, "2", 2, ttl=1)
        self.assertEqual(store.scan(T + 3), ["1(1)"])
        self.assertEqual(store.get_at(T + 4, "2", T + 1), None)
        self.assertEqual(store.get_at(T + 5, "2", T + 2), 2)
        self.assertEqual(store.get_at(T + 6, "2", T + 3), None)

    def test_mix(self):
        store = Store[str, int]()
        T = 0

        store.set(T + 1, "1", 1)
        store.set_with_ttl(T+2, "1", 2, 10)
        store.set(T+20, "1", 3)

        self.assertEqual(store.get_at(T + 27, "1", T + 3), 2)
        self.assertEqual(store.get_at(T + 28, "1", T + 11), 2)
        self.assertEqual(store.get_at(T + 28, "1", T + 12), None)
        self.assertEqual(store.get_at(T + 29, "1", T + 20), 3)
        self.assertEqual(store.get_at(T + 50, "1", T + 40), 3)

    def test_reduced_ttl(self):
        store = Store[str, int]()
        T = 0

        store.set_with_ttl(T+1, "1", 1, 10)
        store.set_with_ttl(T+2, "1", 2, 5)
        self.assertEqual(store.get_at(T+3, "1", T+2), 2)
        self.assertEqual(store.get_at(T+7, "1", T+7), None)
        self.assertEqual(store.get_at(T+11, "1", T+11), None)

    def test_trimming(self):
        global MAX_LOG_SIZE
        MAX_LOG_SIZE = 5

        key = "1"
        store = Store[str, int](trim=True)
        for i in range(2 * MAX_LOG_SIZE):
            store.set(0, key, i)
        time.sleep(0.7)

        self.assertLessEqual(len(store.map[key]._log), MAX_LOG_SIZE)

if __name__ == "__main__":
    unittest.main()
```
