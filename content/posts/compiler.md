---
title: "Compiler, interpreter and machine code"
date: 2025-03-11
lastmod: 2025-03-11
---

In this post, we'll be looking at the benefits of the different types of compilers:

-   ahead-of-time (AOT)
-   just-in-time (JIT)
-   interpreted

    I call this a compiler as well, since the initial source code is compiled into bytecode that is understood by the interpreter virtual machine (VM), instead of the source code being compiled into machine code directly.


## Introduction to compiler design

Before we can compare the different types of compilation we need a common ground against which we can make the comparison.

This section is basically a summary of "The architecture of open source applications"[^1] [LLVM chapter](https://aosabook.org/en/v1/llvm.html). I highly recommend reading it to get more details.

In the chapter's [Figure 11.3](https://aosabook.org/en/v1/llvm.html#fig.llvm.lcom) we see that most compilers follow a three-phase design:

-   A language front-end, e.g. for Python and Rust, that is compiled into an internal representation (IR).

    This is done using a lexer and parser to build an AST, which is then converted to the IR. More details and exceptional hands-on tutorial can be found in the book [Writing an interpreter in Go](https://interpreterbook.com/).

-   An optimizer that optimizes the input IR, e.g. through function inlining and reducing instructions like `X-0` to `X`, and outputs an optimized version of the IR.

    Without an optimizer AOT compilation simply creates an executable without any additional performance gains as the source code is directly converted into machine. Thus the performance gain from statically compiled languages such as C++ come largely from the optimizer (C++ is not machine code and needs to be translated to it, an alternative would be to write something closer to machine language such as Assembly and write optimal code).

    By using one IR for different front-ends the optimizer functions only need to understand one representation instead of one for each front-end.

-   A back-end for specific architectures that take the (optimized) IR to produce native machine code.

    Note, different architectures, e.g. ARM and X86, have different instruction sets and thus need different machine code to produce the same result. For example, running an ARM binary for Firefox on an X86 machine doesn't work.


### Machine code

In simplistic terms, all a (AOT) compiler does is taking source code and producing an executable that contains the native machine code for the target machine.

To really drive this point home, we'll take a quick look at the [Assembly language](https://en.wikipedia.org/wiki/Assembly_language) (ASM). The beauty of ASM is that the instructions in the source code strongly correspond to the machine code instructions (whereas in high-level languages such as Python it is unclear what exactly the CPU is doing for particular lines of code), which explains why different architectures with different instruction sets need different machine code to run.

For this section, we again take another great source and cherry-pick from it what we need. For more details on ASM for X86 (64-bit), take a closer look at the tutorial [Assembly programming](https://github.com/0xAX/asm).

A hello-world program in ASM[^2]:

```asm
;; Definition of the `data` section
section .data
    ;; String `msg` variable with the value `hello world!`
    msg db      "hello, world!"

;; Definition of the text section
section .text
    ;; Reference to the entry point of our program
    global _start

;; Entry point
_start:
    ;; Specify the number of the system call (1 is `sys_write`).
    ;; https://github.com/torvalds/linux/blob/master/arch/x86/entry/syscalls/syscall_64.tbl
    mov     rax, 1
    ;; Set the first argument of `sys_write` to 1 (`stdout`).
    mov     rdi, 1
    ;; Set the second argument of `sys_write` to the reference of the `msg` variable.
    mov     rsi, msg
    ;; Set the third argument to the length of the `msg` variable's value (13 bytes).
    mov     rdx, 13

    ;; Call the `sys_write` system call.
    syscall
    ;; Specify the number of the system call (60 is `sys_exit`).
    mov    rax, 60
    ;; Set the first argument of `sys_exit` to 0. The 0 status code is success.
    mov    rdi, 0
    ;; Call the `sys_exit` system call.
    syscall
```

Note how a specific (CPU) instruction `mov` takes in a register and value, e.g. `mov rax 1`, moving the value `1` into the register `rax`. This can already be a point of difference between architectures, as the (CPU) registers can have different names and calling conventions (e.g. `syscall` expecting the system call number to be in the register `a2`[^3] instead of `rax`).

Even though ASM is not the output of a compiler, it clearly shows how different architectures need different machine code.


## AOT compilation

It should now become clear what ahead-of-time compilation is: (1) take in source code (2) try to optimize it (3) output machine code.

These steps mean that the compiler needs to know "everything" upfront (before ever running the actual program) as the program itself can't change dynamically based on the inputs it is receiving. For example, all types have to be declared (or inferred) upfront to know exactly what operation the CPU has to execute -- are we adding integers or strings (i.e. concatenation)?

After exploring interpreters by looking at Python, we'll explain why the above makes it hard for Python to be compiled AOT.


## Interpreted languages

Similar to the three-phase design of a compiler, interpreted languages "compile" the source code (often through an AST) to an internal representation called bytecode. Another process, e.g. the Java virtual machine, then processes this bytecode to execute the program. To better understand how this works, let's take a look at [writing a Python interpreter in Python](https://aosabook.org/en/500L/a-python-interpreter-written-in-python.html) (again a read I can highly recommend).

Python bytecode can be inspected (disassembled) using the `dis` module:
```python
import dis

def foo(x):
    y = x + 5
    return y

# Code object that contains everything the Python interpreter needs
# to execute the function
obj = foo.__code__
# The actual bytecode
bytecode = obj.co_code
# bytecode = b'\x97\x00|\x00d\x01z\x00\x00\x00}\x01|\x01S\x00'
l = list(bytecode)
# l = [151, 0, 124, 0, 100, 1, 122, 0, 0, 0, 125, 1, 124, 1, 83, 0]
opname = dis.opname[151]
# opname = 'RESUME'

# Disassemble the function to see the instructions
dis.dis(foo)
# Column 1: line number in Python file
# Column 2: index in bytecode
# Column 3: opname
# Column 4: arguments to the operation
# Column 5: helper to show what the argument means in the source
#
# 1           0 RESUME                   0

# 2           2 LOAD_FAST                0 (x)
#             4 LOAD_CONST               1 (5)
#             6 BINARY_OP                0 (+)
#            10 STORE_FAST               1 (y)

# 3          12 LOAD_FAST                1 (y)
#            14 RETURN_VALUE
```

In the case of CPython, this bytecode is processed in C at runtime. Meaning that invoking `python <source>` starts the CPython VM, converts the source file to bytecode, feeds the bytecode to the VM and has the VM execute the bytecode step-by-step. For example, when the `BINARY_OP` instruction is encountered it is matched and the arguments `5` and `x` are added in C.[^4][^5]

Then why is a Python program slower than a program directly written in C? Well, it skips the optimizer of a C compiler, as everything is decided at runtime instead of ahead-of-time. Moreover, additional work has to be done to decide what C code to invoke (i.e. parsing the bytecode).


### AOT compilation and Python

>   The compiler's ignorance is one of the challenges to optimizing Python or analyzing it statically, just looking at the bytecode, without actually running the code, you don't know what each instruction will do! In fact, you could define a class that implements the `__mod__` method, and Python would invoke that method if you use `%` on your objects. So `BINARY_MODULO` could run any code at all! [^4]

Think of it like this: an AOT compiler has to generate fully correct code before the application even runs, which is hard if you can redefine everything (e.g. redefining the module operation `%` through `__mod__`).

Combined with other aspects of the language make it hard to compile Python AOT, although still possible as the [codon project by Exaloop](https://github.com/exaloop/codon) shows. Codon allows static compilation of (some but not all) Python programs.

Moreover, there is another form of compilation called JIT that can work well in the scenarios where AOT wouldn't work. Which is what the [Pypy](https://pypy.org/) project does.


## JIT (for Python)

Just-in-time compilation only compiles particular parts of the source code if needed. For example, a Python function that is continuously executed and makes up 90% of the total runtime of a program can be the only part of the program that is compiled, yet still lead to a significant performance gain.

There are different types of JIT compilers, but here we are just interested in the broad strokes of how a JIT compiler could work. In short, "hot" code (i.e. code that is executed many times) is compiled at runtime to machine code and this machine code is written to memory to be executed each time the hot code would've been invoked. Because JIT causes compilation to happen at the target computer (the one running the actual program) the compilation can use all details about the computer's architecture without having to worry about compatibility issues with other architectures.

For Python, a great example of JIT compilation is [numba](https://numba.pydata.org/). Note that not all Python code can be JIT compiled (for the same reasons that AOT compilation of Python code is hard). Even Python itself has a JIT compiler since Python 3.13.

To get more details on what a JIT is and how Python implements it, I recommend reading [Python 3.13 gets a JIT](https://tonybaloney.github.io/posts/python-gets-a-jit.html) by Anthony Shaw.


## Conclusion

If you want run your code written in an interpreted language, e.g. a Python program, then Python needs to be installed since the Python interpreter is needed and isn't contained in the program itself. Thus shipping a Python program requires installation of Python at all target computers.

AOT compilation creates an executable that is generic for a specific architecture. However, JIT can compile very specifically for a particular chipset and can therefore execute programs faster than AOT would (in certain cases). JVM shows that JIT compilation can be competitive with AOT compilation in terms of performance thanks to exactly this reason. This makes JIT compilation a good candidate for interpreted languages to increase performance.

So while AOT doesn't require a VM (e.g. the Python VM), which has its benefits, it isn't always the best solution (as it is also hard to implement for dynamic programming languages).


[^1]: [The Architecture of Open Source Applications](https://aosabook.org/en/)
[^2]: [ASM hello-world examle](https://github.com/0xAX/asm/blob/master/content/asm_1.md#hello-world-example)
[^3]: [Syscall(2) manpage](https://man7.org/linux/man-pages/man2/syscall.2.html)
[^4]: [A Python Interpreter Written in Python](https://aosabook.org/en/500L/a-python-interpreter-written-in-python.html)
[^5]: [CPython VM `ceval.c` source code](https://github.com/python/cpython/blob/92e5f826ace1eab29920a0a3487a3cac7eeab925/Python/ceval.c#L427)

