---
layout: default
title: Performance Implications of False Sharing
---

# Performance Implications of False Sharing

One of the most important considerations when writing parallel applications is how different threads or processes will share data. In some circumstances, sharing is unavoidable. However, our data layout and architecture may introcude unintentional sharing, known as false sharing. This tutorial provides a basic introduction to false sharing through some simple benchmarking and profiling.

## Background
Why is sharing data so bad? It's not! If we have many threads that are only reading some shared piece of memory, our performance shouldn't suffer. The problem comes when multiple threads want to write to that piece of memory, and this goes back to cache coherence.

Cache coherence is often defined using two invarients, as taken from [A Primer on Memory Consistency and Cache Coherence](https://www.morganclaypool.com/doi/abs/10.2200/S00346ED1V01Y201104CAC016):
1. Single-Writer, Multiple Reader Invariant: For a memory location *A*, at any given logical time, there exists only a single core that may write to *A* (and read from *A*), or some number of cores (maybe 0) that may only read *A*.
2. Data-Value Invariant - The value at any given memory location *A* at the start of an epoch is the same as the value of the memory location at the end of its last read-write epoch.

Invariant 1 shows why sharing read-only data is OK, while sharing writable data can cause performance problems. When one core wants to write to a memory location, access must be taken away from other cores (to avoid them reading an old/stale value) if they have that memory location in their cache(s). This is done through invalidations. Once the core trying to write has gained exclusive access, it can perform its write operation.

But what exactly is getting invalidated? Typically, it is cache-lines/blocks. Cache-lines/blocks strike a good balance between control and overhead. Finer-grained coherence (say at the byte-level) would require us to maintain a coherence state for each byte of memory in our caches. Coarser-grained coherence (say at the page-level) leads to large invalidations of data that will never be used.

