---
layout: post
title: "Cuckoo Hashing: Elegant Collision Resolution"
date: 2023-04-05
categories: [algorithms, data-structures, hashing]
tags: [algorithms]
---

# Cuckoo Hashing: Elegant Collision Resolution

Recently I came across Monolith paper by ByteDance about their recommendation system[1]
Here they talk about cuckoo hashing as a way to manage dictionaries aka associate arrays.
The benefit of using Cuckoo hashing is that it guarantees O(1) worst-case lookup time.
Named after the cuckoo bird's nest-parasitic behavior, it uses a strategic eviction mechanism to maintain consistent performance where traditional hash tables might degrade to O(n) lookup times.

## Core Mechanism

The algorithm maintains multiple hash tables (typically two) with different hash functions. When inserting an element:

1. Calculate its position in each table using the respective hash functions
2. If any position is empty, place the element there
3. If all positions are occupied, evict an existing element and recursively find a new home for it

This creates a displacement chain - each element has multiple potential locations, and insertions may trigger cascading relocations. To limit a cascading effect of evications, usually the evictions are limited to a max number, if the evictions breach this threshold, then the whole table is rehashed - however this is the worst case.
<br>
<img src="/assets/post_images/cuckoo-hashing.png" alt="Cuckoo hashing" width="500" height="300">

## Performance Characteristics

* **Lookup**: O(1) worst-case
* **Insert**: O(1) amortized, with occasional longer chains
* **Delete**: O(1)
* **Load factor**: Typically 40-50%

The algorithm represents a practical balance between theoretical elegance and real-world performance needs, making it particularly suitable for systems where consistent performance guarantees matter more than raw average-case speed.

## Reference

1. ByteDance Monolith paper [https://arxiv.org/pdf/2209.07663](https://arxiv.org/pdf/2209.07663)
2. Cuckoo hashing - [https://cs.stanford.edu/~rishig/courses/ref/l13a.pdf](https://cs.stanford.edu/~rishig/courses/ref/l13a.pdf)
