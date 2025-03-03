---
layout: post
title: "Consistent Hashing: Balancing Distribution in Dynamic Systems"
date: 2021-12-02
categories: [distributed-systems, algorithms, system-design]
tags: [distributed-systems, algorithms, system-design]
---

# Consistent Hashing: Balancing Distribution in Dynamic Systems

Distributing data across different worker nodes is a very important problem in distributed systems.
A simple modulo(%) no of nodes operation does not work here because as the no of nodes change ie a new node is added or removed it disrupts the whole data distribution, this means we need a redistribution even when a single new node is added or an existing node is removed.
Adding or removing a node is a very common operation in a scalable system where it needs to respond to the load.
This is where consistent hashing comes very handy.

## Core Mechanism

Consistent hashing maps both data keys and nodes to the same circular keyspace (typically 0 to 2^n-1). The algorithm works as follows:

1. Hash each node to a position on the circular keyspace
2. For each data key, hash it to the keyspace and assign it to the next node encountered when moving clockwise
3. When nodes join or leave, only keys between the affected node and its predecessor need remapping

<br>
<img src="/assets/post_images/consistent-hashing-ring.png" alt="Consisent hashing ring" width="500" height="300">

This approach ensures that adding or removing a node impacts only a small fraction of keys, approximately 1/n of the total.

## Performance Characteristics

* **Lookup**: O(log n) with a binary search tree, O(1) with additional structures
* **Node addition/removal**: Only affects K/n keys on average (where K is total keys)
* **Distribution quality**: Can be uneven with few nodes
* **Memory overhead**: proportional to number of nodes

## Addressing Imbalance

Basic consistent hashing can result in uneven distribution. Common solutions include:

* Virtual nodes (vnodes): Each physical node is represented as multiple points on the keyspace, improving balance at the cost of increased memory
* Weighted distribution: Assigning high-capacity servers more virtual nodes

<br>
<img src="/assets/post_images/consistent-hashing-ring-with-vnodes.png" alt="Consistent hashing with vnode" width="500" height="300">

## Production Applications

Consistent hashing powers many distributed systems:

* Amazon's Dynamo database and derivatives (Cassandra, Riak)
* Load balancers distributing traffic across backend servers
* Distributed caches like Memcached and Redis clusters

Consistent hashing comes handy when the workers are stateful and requests need to be routed to a specific worker.
