# System Design Components

A deep-dive into system design concepts — exploring how real systems behave under load, where they break, and how to reason about them from first principles.

---

## Topics

[**Topic #1 — An Open Question**](/topics/topic-1-an-open-question.md)
A deceptively simple question — *what breaks first at 1M RPS?* — used to unpack the interplay between network, CPU, and RAM under extreme load.

[**Topic #2 — CPU Cache**](/topics/topic-2-cpu-cache.md)
How L1/L2/L3 cache hierarchies work, why cache misses are so costly, and how cache-aware design decisions can dramatically change performance. We have attached a guide on [**Latency Numbers**](/resources/latency-number-every-programmer-should-know.md) in resource.

[**Topic #3 — Core Pinning**](/topics/topic-3-core-pinning.md)
What CPU affinity and core pinning are, why high-throughput systems use them, and how they reduce context-switching overhead and NUMA-related latency. [**Process VS Thread**](/resources/process-vs-thread.md) and [**Concurrency VS Parallelism**](/resources/concurrency-vs-parallelism.md) can help to understand core pinning deeply. We have added [**Event Loop**](/resources/event-loop.md) as an additional reading.

[**Topic #4 — CAP Theorem**](/topics/topic-4-cap-theorem.md)
A breakdown of Consistency, Availability, and Partition Tolerance — what the theorem actually guarantees (and what it doesn't), with real-world database examples.

[**Topic #5 — Consistent Hashing**](/topics/topic-5-consistent-hashing.md)
How consistent hashing solves the node addition/removal problem in distributed systems, and why it's a cornerstone of scalable caching and sharding strategies.

[**Topic #6 — Latency**](/topics/topic-6-latency.md)
How latency originates, compounds, and is measured across software systems — from I/O and queuing delays to tail latency amplification and distributed tracing.

[**Topic #7 — Throughput**](/topics/topic-7-throughput.md)
How throughput is defined, measured, and maximized across hardware and software layers — from CPU pipelines and memory bandwidth to application-level RPS, Little's Law, and JVM-specific tuning.



---

## Additional Resources

* [**Streaming a Million Likes/Second: Real-Time Interactions on Live Video**](/resources/linkedins-real-time-platform.md)
A talk given by [Akhilesh Gupta](https://www.linkedin.com/in/guptaakhilesh/) on how Linkedin uses the Play/Akka Framework and a scalable distributed system to enable live interactions like likes/comments at massive scale at extremely low costs across multiple data centers.

* [**Use Back-of-the-envelope-calculations to Choose the Best Design**](/resources/back-of-the-envelope-calculations.md)
Jeff Dean, Head of Google's School of Infrastructure Wizardry—instrumental in many of Google's key systems: ad serving, BigTable; search, MapReduce, ProtocolBuffers—advocates evaluating different designs using back-of-the-envelope calculations. He gives the full story in a Stanford video presentation.