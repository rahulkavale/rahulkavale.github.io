---
layout: post
title: "Amdahl's Law & Little's Law: Foundational Principles for System Design"
date: 2023-10-04
categories: [system-design, scalability, performance]
---

When designing scalable systems, it's a very important factor to implement stateless services to leverage horizontal scaling.
However, based on the nature of the problem, adding multiple workers does not always mean full parallelism.
To predictably model this behaviour to identify the constraints, there are two simple laws that come very handy -  Amdahl's Law and Little's Law. 
While they address different aspects of system behavior, together they provide critical insights for designing high-performance systems.

## Amdahl's Law: The Limits of Parallelization[1]

Amdahl's Law, formulated by Gene Amdahl in 1967, quantifies the maximum theoretical speedup when parallelizing a system. It's expressed as:

**Speedup = 1 / [(1 - p) + p/s]**

Where:
- p is the proportion of execution that can be parallelized
- s is the speedup of the parallelized portion
- (1 - p) represents the serial fraction

### Key Insights

**Diminishing Returns of added worker pool/resources**: As s approaches infinity, speedup is bounded by 1/(1-p). Even with infinite resources, a system with 10% serial work cannot exceed 10× speedup.

**Serial work items cause bottlenecks**: Small serial parts in the system can significantly constrain maximum speedup we can get even with parallelism. A system with 5% serial work can never exceed 20× speedup, irrespective of how many processing units are added in the system

<br>
<img src="https://upload.wikimedia.org/wikipedia/commons/thumb/e/ea/AmdahlsLaw.svg/1920px-AmdahlsLaw.svg.png" alt="Amhdal's Law" width="500" height="300">

### System Design Applications

**Identifying Critical Paths**: Scaling by adding multiple workers does not translate parallelization of tasks. Identifying the serial parts and mitigating them is equally important to make use of added worker pool.

**Capacity planning or Resource Allocation**: Use this law to identify the needed worker pool size considering the serial work in the system.

**System design consideration**: To make use of multiple workers to parallelize the system execution, having small serial parts is better.

## Little's Law: Understanding Throughput Relationships[2]

Little's Law, developed by John Little in 1961, describes the relationship between concurrent items in a system, arrival rate, and processing time:

**L = λW**

Where:
- L is the average number of items in the system ie queue size
- λ is the average arrival rate
- W is the average time an item spends in the system

### Key Insights

**Throughput Limits**: For a given processing rate, there is maximum limit of items in the queue after which the queue will get overflowed potentially causing data loss.

**Latency vs. Throughput**: Reducing processing time (W) directly increases potential throughput (λ) provided they can be processed in parallel.

### System Design Applications

**Capacity Planning**: Calculate required concurrency to handle peak loads given known processing times.

**Queue Sizing**: Identify appropriate queue sizes to accommodate traffic spikes without excessive resource consumption or rejecting incoming requests.

**SLA Management**: This law helps in identifying system load and response time to set realistic SLA.

## Practical Application Example: API Gateway Design

Consider designing an API gateway with a target throughput of 10,000 RPS and maximum latency of 50ms.
We can use Little's Law to determine minimum concurrency:

L = λW = 10,000 * 0.05 = 500 concurrent connections

Apply Amdahl's Law to the request processing pipeline:

If some preprocessing in requests(eg authentication) is a serial bottleneck taking 20% of processing time, maximum theoretical speedup from parallelizing everything else is 5×.
This reveals that such serial parts should be optimized first, perhaps through caching or other alternatives.

Use both laws to guide horizontal vs. vertical scaling decisions:

- If most processing is parallelizable, horizontal scaling makes sense
- If serial bottlenecks dominate, vertical scaling (faster CPUs) could also be considered
- Serial parts could further be broken into smaller parts for avoiding wasted work if certain requests are invalid using fail fast approach
- Multiple smaller parts could be batched together for better efficiency

## Conclusion

Amdahl's Law and Little's Law provide mathematical frameworks for understanding fundamental system performance limits. Together, these two laws enable data-driven decisions about architecture, resource allocation, preventing issues like over-provisioning or misplaced optimization efforts.

## References

- [1] Amdahl's Law - [https://en.wikipedia.org/wiki/Amdahl%27s_law](https://en.wikipedia.org/wiki/Amdahl%27s_law)
- [2] Little's Law [https://en.wikipedia.org/wiki/Little%27s_law](https://en.wikipedia.org/wiki/Little%27s_law)