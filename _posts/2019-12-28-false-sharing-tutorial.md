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

As a well-intentioned programmer, we may try to solve this problem by giving each thread its own atomic integer to increment. This is actually a common optimization strategy in parallel programming. We can reduce the number of shared writes to a single shared location by having threads calculate partial results independently, then merging these partial results. That could look something like this:
```cpp
void falseSharing() {
  // Create a single atomic integer
  std::atomic<int> a;
  a = 0;
  std::atomic<int> b;
  b = 0;
  std::atomic<int> c;
  c = 0;
  std::atomic<int> d;
  d = 0;

  // Four threads each with their own atomic<int>
  std::thread t1([&]() {work(a)});
  std::thread t2([&]() {work(b)});
  std::thread t3([&]() {work(c)});
  std::thread t4([&]() {work(d)});

  // Join the 4 threads
  t1.join();
  t2.join();
  t3.join();
  t4.join();
} 
```
For simplicity, we ignore the merging of the partial results, as the 3 additional add operations does not significantly change the results.

While it looks like we solved our sharing problem by creating multiple atomic integers, we'll see this has almost the exact same performance as the directSharing benchmark. Why is that? Let's think harder about where those four atomic integers sit in memory. We can print out their addresses like so:
```cpp
void print() {
  std::atomic<int> a;
  std::atomic<int> b;
  std::atomic<int> c;
  std::atomic<int> d;
 
  // Print out the addresses
  std::cout << "Address of atomic<int> a - " << &a << '\n';
  std::cout << "Address of atomic<int> b - " << &b << '\n';
  std::cout << "Address of atomic<int> c - " << &c << '\n';
  std::cout << "Address of atomic<int> d - " << &d << '\n';
}
```

Compiling with gcc-10, I got the following prints:
```
Address of atomic<int> a - 0x7ffe5dd94f6c
Address of atomic<int> b - 0x7ffe5dd94f68
Address of atomic<int> c - 0x7ffe5dd94f64
Address of atomic<int> d - 0x7ffe5dd94f60
```

Notice that all of our atomics are four bytes away from each other. We already discussed that cache coherence is typically at the cache-line/block granularity, and cache lines in modern processors are typically 64 bytes. Therefore, our 4 atomics allocated like this wound up on the same cache line. When each thread grabs its dedicated atomic integer, it grabs all four. There's our false sharing!

All hope is not lost. We just need to space the atomic integers out so that they are not sitting next to each other in memory. We can do this easily by placing them in a struct, and setting the alignment to 64 bytes:
```cpp
struct alignas(64) AlignedType {
  AlignedType() { val = 0; }
  std::atomic<int> val;
};
```

Let's print out the addresses of four of our *AlignedType* structs:
```cpp
void print() {
  AlignedType a{};
  AlignedType b{};
  AlignedType c{};
  AlignedType d{};
 
  // Print out the addresses
  std::cout << "Address of AlignedType a - " << &a << '\n';
  std::cout << "Address of AlignedType b - " << &b << '\n';
  std::cout << "Address of AlignedType c - " << &c << '\n';
  std::cout << "Address of AlignedType d - " << &d << '\n';
}
```
Compiling with gcc-10, I got the following prints:
```
Address of AlignedType a - 0x7ffd7dc7a300
Address of AlignedType b - 0x7ffd7dc7a2c0
Address of AlignedType c - 0x7ffd7dc7a280
Address of AlignedType d - 0x7ffd7dc7a240
```
Now we have 64 bytes between each of our objects. This means that none of our atomics will be on the same cache line! Our modified code using this *AlginedType* looks like this:
```cpp
void diff_line() {
  AlignedType a{};
  AlignedType b{};
  AlignedType c{};
  AlignedType d{};
 
  // Launch the four threads now using our aligned data
  std::thread t1([&]() { work(a.val); });
  std::thread t2([&]() { work(b.val); });
  std::thread t3([&]() { work(c.val); });
  std::thread t4([&]() { work(d.val); });
 
  // Join the threads
  t1.join();
  t2.join();
  t3.join();
  t4.join();
 }
```

## Benchmarking and Profiling
To benchmark the previously listed code, I used [Google Benchmark](https://github.com/google/benchmark). I used perf Linux profiler for profiling.

### Execution time results

