---
title: "Privacy in the digital age"
date: 2024-12-21
lastmod: 2024-12-21
---

> This will be the future: a world of people too busy playing with their phones to even notice that someone else controls them.
> -- [Edward Snowden](https://edwardsnowden.substack.com/p/ns-oh-god-how-is-this-legal)

**Privacy**: they know who you are, but can't see what you're doing.

Funnily, when talking about privacy in the context of (offline) real life then in most cases it is clear when you are invading someone's privacy. However, in the context of digital life we as humans don't automatically grasp the concept of privacy.

Say you are texting a loved one, then you'd rather not have that creepy person in the train constantly looking over your shoulder reading along with your messages. But that same creepy person might as well be working for some big corporation that has access to those exact messages. How is that different? And why can we instantly feel someone is invading our privacy in the train, but not online?

In this post I won't be writing about the importance of privacy as you can easily find other convincing thought pieces online e.g. [Why privacy matters!](https://mullvad.net/en/why-privacy-matters). I also won't be giving my elaborate take on the aforementioned questions besides stating that I think it has to do with our general lack of understanding of digital privacy and its implications. I'll let this statement serve as the introduction to what you can do and have to be aware of to (somewhat) safe your digital privacy.

-   Be sure to check out the business model of a company before using any of their services.
-   Companies silently thinking: "**With AI on the rise all data is a prize!**"
-   Know your goals when it comes to privacy. Different goals ask for different measures.
-   Blend in with the crowd.
-   Clear cookies and cache when you end your browsing session to make it harder to be tracked
    across sessions. Yes, this requires you to log-in again.


## Technologies

### VPN

-   Essentially just moves your trust from your ISP to your VPN provider.
-   Encrypts traffic from your PC, not only browser traffic.
-   Masks your IP.

### DNS

-   Changing your DNS provider or hosting your own recursive resolver prevents from sharing your DNS queries with your ISP or other large parties, e.g. Google DNS. Since every website you visit will result in a DNS query, your DNS server knows a great deal about your internet behavior.

    Note, DNS queries are only hidden from your ISP if you encrypt your DNS queries, e.g. with DoT or DoH. Most DNS servers support this, but be sure to check!

-   If you're using a VPN, then you'll likely want to opt for using the DNS they've configured. Otherwise you might have a [DNS leak](https://dnsleaktest.com/).

-   Can be useful to filter websites, for example for the purpose of ad blocking.

### Browser

-   Choice of browser

    Note, make sure not to use a browser like [Google Chrome](https://www.wired.co.uk/article/google-chrome-browser-data) as it tracks everything you do (check out this [fun comic](https://contrachrome.com/) to drive the point home).

-   Cookies
-   Fingerprinting: [What is fingerprinting?](https://blog.torproject.org/browser-fingerprinting-introduction-and-challenges-ahead/)

    [Test out fingerprinting](https://abrahamjuliot.github.io/creepjs/) by revisiting the site with different browser settings etc. and seeing it is still counting your number of unique visits. Similar website by one of the Tor maintainers: [Am I Unique?](https://amiunique.org/) which is great to see what exactly constitutes towards a fingerprint.

    Even when disabling Javascript (which is an anomaly in itself) there are techniques such as [If-None-Match](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/If-None-Match), see it in action [here](https://privacycheck.sec.lrz.de/passive/fp_etag/fp_etag.php). It is essentially a value that is passed from the server to your client, which your client caches. So every time you visit the website the server will ask for your value if you have it.

### Mobile phone

-   A mobile phone uses a shared IP of the network provider. Furthermore, carrying a phone with you will reveal your location due to cellular tower triangulation.

    Against this you can keep your phone in a faraday bag or leave airplane mode enabled.

-   Use end-to-end encrypted messaging applications.

    Though, keep in mind, metadata is gold to big tech companies so think twice before using their service regardless of E2EE on the messages.

### Search engine

-   Use a privacy respecting search engine.

    Similarly to a DNS, a search engine knows a great deal about your browsing behavior.
