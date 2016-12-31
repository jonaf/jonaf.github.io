---
title: The Java Memory Model
updated: 2016-12-18 1:32
---

This is just a draft, but I have a lot to say about the Java memory model and what I've learned about it.

Highlights:

- there's two sides to this thing. I used to think there was only one: syntax. Turns out, the memory model is the biggest and most important part, and it's practically invisible to a newcomer.
- all programs require memory access, and memory access requires hardware, and hardware has *physical* limitations, like how many bytes you can read at most (or at minimum!)
- concurrency is hard. It's hard to reason about. Java's latest JSR for the JMM does a decent job.
- I wrote a load balancing algorithm for Ostrich in Java and learned a lot about concurrency.
- I found some useful resources, which were pretty hard to dig up, and even harder to understand. [1][2]
- even in Java, when dealing with concurrency, you need to have basic understanding of CPU architecture and memory access -- fences or barriers, caches (L1..3), "main memory," scheduling, how threading *actually* works
- cores/CPU's actually matter. Spinning up 1,000 threads on a 4-core laptop to test concurrency may work fine, but on a 16-core server everything breaks down. Whoops!
- architecture really matters. The effect of a synchronized block may be different, in terms of performance and sometimes even correctness, depending on architecture. The JMM makes *no guarantees whatsoever* regarding performance, and *aims* to provide guarantees on correctness.
- facets of concurrency: order, atomicity, (others?).

[^1] https://shipilev.net/blog/2014/jmm-pragmatics/
[^2] https://docs.oracle.com/javase/specs/jls/se8/html/jls-17.html