---
layout: default
title: Performance Implications of False Sharing
---

# Performance Implications of False Sharing

One of the most important considerations when writing parallel applications is how different threads or processes will share data. In some circumstances, sharing is unavoidable. However, our data layout and architecture may introcude unintentional sharing, known as false sharing. This tutorial provides a basic introduction to false sharing through some simple benchmarking and profiling.

## Background
Why is sharing data so bad? It's not! If we have many threads that are only reading some shared piece of memory, our performance shouldn't suffer. The problem comes when multiple threads want to write to that piece of memory, and this goes back to cache coherence.

### Cache Coherence

Cache coherence is often defined using two invarients, as taken from [A Primer on Memory Consistency and Cache Coherence](https://www.morganclaypool.com/doi/abs/10.2200/S00346ED1V01Y201104CAC016):
1. Single-Writer, Multiple Reader Invariant: For a memory location *A*, at any given logical time, there exists only a single core that may write to *A* (and read from *A*), or some number of cores (maybe 0) that may only read *A*.
2. Data-Value Invariant - The value at any given memory location *A* at the start of an epoch is the same as the value of the memory location at the end of its last read-write epoch.

Invariant 1 shows why sharing read-only data is OK, while sharing writable data can cause performance problems. When one core wants to write to a memory location, access must be taken away from other cores (to avoid them reading an old/stale value) if they have that memory location in their cache(s). This is done through invalidations. Once the core trying to write has gained exclusive access, it can perform its write operation.

But what exactly is getting invalidated? Typically, it is at the cache-line/block granularity. Cache-lines/blocks strike a good balance between control and overhead. Finer-grained coherence (say at the byte-level) would require us to maintain a coherence state for each byte of memory in our caches. Coarser-grained coherence (say at the page-level) can lead to the needless invalidation of large pieces of memory.
We can abstract the range of data for which a coherence state is maintained using by referring to it as a *coherence block* (a page or cache-line/block in practice).

### Where does false sharing come from?

False sharing occurs when data from multiple threads that was not meant to be shared gets mapped to the same coherence block. When this happens, cores accessing seeming unrelated data send invalidations to each other. This can lead to performance that looks like two threads are directly sharing data. Two shows is off, let's consider a simple optimization case-study where a best-intentions programmer optimizes code without thinking about the underlying architectures.

## Optimization Case Study
Consider the following simple function in C++ that we want to run 4 times:
```cpp
void work(std::atomic<int>& a) {
  for(int i = 0; i < 100000; ++i) {
    a++;
  }
}
```
All we are doing is atomically incrementing some atomic integer _a_ 100k times. Let's say that we want to implement this in a single thread, and call our work function 4 times. We could do something like this:
```cpp
void singleThread() {  
  // Create a single atomic integer
  std::atomic<int> a;
  a = 0;

  // Call the work function 4 times
  work(a);
  work(a);
  work(a);
  work(a);
}
```

Note, the only reason we are using an atomic integer for a single-threaded implementation is to better isolate the overhead of sharing. You could always replace it with a regular integer if you are only using a single thread.

A terrible way to speed up this process is to *only* throw threads at the problem. That could look something like this:
```cpp
void directSharing() {
  // Create a single atomic integer
  std::atomic<int> a;
  a = 0;
  
  // Four threads all sharing one atomic<int>
  std::thread t1([&]() {work(a)});
  std::thread t2([&]() {work(a)});
  std::thread t3([&]() {work(a)});
  std::thread t4([&]() {work(a)});

  // Join the 4 threads
  t1.join();
  t2.join();
  t3.join();
  t4.join();
}
```

This isn't false sharing. This is *direct* sharing. All four threads are incrementing the same atomic integer *a*. The cache line that *a* sits on will bounce between the different cores the threads are scheduled to, leading to significantly worse performance than if we did all the work in a single thread.

As a best-intentions programmer, we 
## Benchmarks

Now that we've motivated why we should care about sharing, let's look at some benchmarks that show off false-sharing.
