---
layout: post
title: "Resilient Systems: Managing Retries and Traffic Patterns"
date: 2024-07-02
categories: [distributed-systems, system-design]
tags: [system-design, latency, resilience, performance, reliability, python]
---

# Resilient Systems: Managing Retries and Traffic Patterns

## Retry Mechanisms

At their core, retry mechanisms are simple: when an operation fails, try again. However, implementing them correctly is crucial for system stability.

This naive approach has a critical flaw: all clients will retry at the same intervals, potentially creating synchronized waves of traffic.

## Retry with Jitter

Adding randomness (jitter) to retry intervals prevents the above problem:

```python
def retry_with_jitter(operation, max_attempts=3, base_delay=1):
    attempts = 0
    while attempts < max_attempts:
        try:
            return operation()
        except Exception as e:
            attempts += 1
            if attempts == max_attempts:
                raise e

            # Exponential backoff with jitter
            delay = base_delay * (2 ** attempts)
            jitter = random.uniform(0, delay / 2)
            time.sleep(delay + jitter)
```

Jitter disperses retry attempts across time, preventing traffic spikes and reducing system strain.

## The Thundering Herd Problem

The thundering herd occurs when many processes or clients simultaneously react to a single event:

1. A popular cache key expires
2. Thousands of requests miss the cache simultaneously
3. All requests hit the database at once
4. The database becomes overwhelmed

This pattern also appears when a service restarts and all waiting clients attempt to reconnect at once.

## Retry Storms

Retry storms are cascading failures triggered by retries:

1. A service slows down due to increased load
2. Clients experience timeouts and begin retrying
3. Retries further increase load on the struggling service
4. More clients experience failures and retry
5. The system becomes completely overwhelmed

This destructive feedback loop can take down otherwise healthy systems.

## Load Shedding

Load shedding is the deliberate dropping of work when a system approaches capacity.

More load overall increases system load and after some point, it will impact every request degrading the system performance overall (Amdahl's Law).

Hence we need to identify a proper strategy to reject the incoming request so that at least the requests currently in flight can finish without issues.

Effective load shedding requires:

* Accurate measurement of system capacity
* Prioritization of requests (critical vs. non-critical)
* Clear failure signals to clients
* Graceful degradation of non-essential functionality

## How These Patterns Work Together

These techniques complement each other to create resilient systems:

1. **Rate limiting** prevents clients from overwhelming services
2. **Jittered retries** disperse traffic after failures
3. **Load shedding** protects services during peak demand
4. **Thundering herd prevention** protects shared resources

Combining all these can help effectively handling burst requests while maintaining service SLA requirements under given constraints.
