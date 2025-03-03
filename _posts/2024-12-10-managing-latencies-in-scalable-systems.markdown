---
layout: post
title: "Taming Latency: Strategies to Handle Latencies in Scalable Systems"
date: 2024-12-10
categories: [engineering, distributed-systems]
tags: [performance, latency, scalability]
---

# Taming Latency: Strategies to Handle Latencies in Scalable Systems

The most difficult bugs I have solved are always about latencies.

In production systems, based on different factors like traffic patterns, cache invalidations, resource allocations can cause unpredictable latency distributions.

With strict SLA agreements around latencies for critical systems, it becomes very important to be able to address the issues. Over period of time, I have found some very simple but very useful patterns for managing latencies as systems grow.

## 1. Do Small Work

One of the most effective ways to reduce latency is simply to do less work per operation.

While it may seem obvious, implementing it is not always that straight forward.

Doing small work can mean either you work on small data, or you increase the workers.

This is basically a divide and conquer approach.

The divide step can be parallelized and the conquer needs special handling so that it does not become a bottleneck.

Different systems handle this differently but this has been the most impactful solution in my experience.

### Parallelize Work

When tasks can be performed independently, don't process them sequentially:

* Use thread pools, worker processes, or asynchronous programming patterns
* Take advantage of multi-core processors with parallel algorithms
* Consider techniques like scatter-gather for distributing work across multiple services

For example, instead of fetching user data, recommendations, and notifications sequentially, send all three requests simultaneously and compose the results when they return.

### Chunk the Work

Break large operations into smaller pieces:

* Process data in batches rather than all at once
* Implement pagination for large result sets
* Use incremental loading techniques for large files or datasets
* Consider window functions for processing streams of data

This approach not only reduces individual response times but also provides better resource utilization across the system.

## 2. Precompute When Possible

Why calculate something on-demand when you can have it ready in advance?

* Generate reports during off-peak hours instead of on-request
* Maintain materialized views for complex database queries
* Use background jobs to prepare data that will likely be needed
* Implement multi level caching (application, database, CDN)

Netflix's recommendation system, for instance, doesn't calculate your personalized recommendations when you log in—they're precomputed and updated periodically, ready to be served instantly.

## 3. Separate Reads and Writes

Read and write operations have fundamentally different characteristics and hence different SLA requirements. Depending on the problem and the domain, sometimes reads needs to be very fast but writes can afford to be relatively slower. This kind of takes us back to PACELC theorem[^1] where we need to balance latency vs consistency.

* Identify and separate read and write patterns 
* Use specialized data stores optimized for either reads or writes
* Consider eventual consistency models where appropriate
* Scale read replicas independently from write masters

This separation allows you to optimize each path independently and often leads to more scalable architectures.

## 4. Hide Latency When Possible

Sometimes, the best approach is not to eliminate latency but to make it less noticeable:

### Stream Responses

Instead of waiting for complete results:

* Use streaming APIs to send partial results as they become available
* Implement server-sent events or WebSockets for real-time updates
* Consider progressive rendering techniques for web interfaces
* Use chunked transfer encoding for HTTP responses

### UX Changes

Make latency less frustrating through design:

* Show meaningful loading states that indicate progress
* Use skeleton screens instead of spinner icons
* Provide immediate feedback for user actions, even if the backend processing takes time

Google Search starts showing results as they're found, rather than waiting until all results are compiled—making the experience feel faster even when the complete operation takes time.

## 5. Do Asynchronous Work

Decouple request handling from processing by using asynchronous patterns:

* Accept requests quickly and return a reference ID or token
* Move heavy processing to background workers or queues
* Use job schedulers to manage workloads across the system
* Implement notification mechanisms (webhooks, push notifications, emails) to alert users when processing completes

This strategy is particularly effective for resource-intensive operations like report generation, or batch imports. For instance, when a user uploads a profile picture that needs resizing and optimization, you can acknowledge receipt immediately and process the image asynchronously while the user continues with other tasks.

What latency challenges are you facing in your systems? Which of these strategies might help you address them?

[^1]: [PACELC theorem](https://en.wikipedia.org/wiki/PACELC_theorem)