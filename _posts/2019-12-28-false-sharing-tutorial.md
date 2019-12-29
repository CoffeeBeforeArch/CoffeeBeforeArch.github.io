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
void noSharing() {
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
The following results were collected using [Google Benchmark](https://github.com/google/benchmark) and the [perf Linux profiler](https://perf.wiki.kernel.org/index.php/Main_Page). Each of our benchmarks follow the following form, and was compiled using:

```bash
g++ false_sharing.cpp -lbenchmark -lpthread -O3 -march=native
```

The benchmarks also follow the following structure, where *benchName* and *funcName* are placeholder names.

```cpp
static void benchName(benchmark::State& s) {
  while (s.KeepRunning()) {
    funcName();    
  }
 }
BENCHMARK(noSharing)->UseRealTime()->Unit(benchmark::kMillisecond);
```

### Our benchmarks at the assembly level
Before we take a look at the results, it's important we understand what our code is doing at the lowest level. For our *singleThread* benchmark, four calls to the *work()* function are inlined, leading to four tight loops that do an atomic increment:

```assembly
Percent│
       │    0000000000406e50 <single_thread()>:
       │    _Z13single_threadv():
       │      xor    %eax,%eax
       │      xchg   %eax,-0x4(%rsp)
       │      mov    $0x186a0,%eax
       │      nop
 25.16 │10:   lock   incl   -0x4(%rsp)
       │      dec    %eax
       │    ↑ jne    10
       │      mov    $0x186a0,%eax
       │      xchg   %ax,%ax
 24.84 │20:   lock   incl   -0x4(%rsp)
       │      dec    %eax
       │    ↑ jne    20
       │      mov    $0x186a0,%eax
       │      xchg   %ax,%ax
 24.95 │30:   lock   incl   -0x4(%rsp)
       │      dec    %eax
       │    ↑ jne    30
       │      mov    $0x186a0,%eax
       │      xchg   %ax,%ax
 25.05 │40:   lock   incl   -0x4(%rsp)
       │      dec    %eax
       │    ↑ jne    40
       │    ← retq
```

0x186a0 translates to 100k decimal (the number of iterations in our *work()* loop. The loop corresponds to only three instructions:

```assembly
10: lock   incl   -0x4(%rsp)
    dec    %eax
    jne    10
```
If you have not compiled with -march=native, the incl instruction may be replaced with and addl instruction:

```assembly
10:   lock   addl   $0x1,(%rdx)
```

The code for our multi-threaded benchmarks all look identical. Instead of all four calls to our work function being inlined in the benchmark loop, each thread runs the inlined version of the function.

```assembly
Percent│
       │    0000000000406b20 <std::thread::_State_impl<std::thread::_Invoker<std::tuple<diff_var()::{lambda()#2}> > >::_M_run()>:
       │    _ZNSt6thread11_State_implINS_8_InvokerISt5tupleIJZ8diff_varvEUlvE0_EEEEE6_M_runEv():
  0.29 │      mov    0x8(%rdi),%rdx
       │      mov    $0x186a0,%eax
       │      nop
 99.59 │10:   lock   incl   (%rdx)
       │      dec    %eax
  0.12 │    ↑ jne    10
       │    ← retq

```
As is the case for the *singleThread* benchmark, all three of the multithreaded benchmarks spend all their time on the atomic increment.

### Execution time results
Execution time results for our four microbenchmarks can be found below:
```
------------------------------------------------------------------
Benchmark                        Time             CPU   Iterations
------------------------------------------------------------------
singleThread                  2.35 ms         2.35 ms          300
directSharing/real_time       7.69 ms        0.084 ms           95
falseSharing/real_time        7.78 ms        0.083 ms           92
noSharing/real_time           1.75 ms        0.083 ms          386
```

Unsurprisingly, our *directSharing* and *falseSharing* benchmarks take roughly the same execution time. In fact, they are over three times slower than the *singleThread* benchmark. Our *noSharing* benchmark had the best performance, about 25% faster than the single threaded baseline. Why is it not 4x faster? Creating and joining threads isn't free! If we made our *work()* function do only 10k increments, our *singleThread* and *noSharing* take roughly the same time:
```
------------------------------------------------------------------
Benchmark                        Time             CPU   Iterations
------------------------------------------------------------------
singleThread                 0.233 ms        0.233 ms         3001
directSharing/real_time      0.668 ms        0.059 ms          762
falseSharing/real_time       0.710 ms        0.053 ms          998
noSharing/real_time          0.245 ms        0.044 ms         2614
```

How does the performance look when we scale the number of threads? More threads means more contention on a single memory location. Below are the results for our *falseSharing* benchmark using 2, 4, and 8 threads (with the ammount of work appropriately scaled as well):
```
-----------------------------------------------------------------
Benchmark                       Time             CPU   Iterations
-----------------------------------------------------------------
twoThreads/real_time         6.49 ms        0.050 ms          106
fourThreads/real_time        7.95 ms        0.077 ms           90
eightThreads/real_time       9.24 ms        0.125 ms           77
```

### L1 cache hit rate
Our perf report gives us a decent idea about where our time is being spent in our benchmarks (unsurprisingly, waiting for our atomic increments). However, if we want to know why the performance of these applications differ so greatly, we must look at another metric. Specifically, we will be looking at the L1 cache hit rate using perf stat.

We are looking at cache hit rate because the claim we made about direct and false sharing was that invalidations on cache-lines/blocks was going on. As a result, we'd expect both both the *directSharing* and *falseSharing* benchmarks to have a low L1 hit rate as the cache-line/block bounces between cores, and the *singleThread* and *noSharing* benchmarks we should expect very high hit rates, as each thread should have exclusive access to the atomic integers.
  
For the *singleThread* benchmark, we get the expected result, as shown below:
```
1,158,985,491   L1-dcache-loads         #   170.514 M/sec 
319,420         L1-dcache-load-misses   #   0.03% of all L1-dcache hits
```
A very high hit rate! If we accesses the same variable 400k times from the same thread, it's probably going to stick around in our L1 cache. But what about two benchmarks with sharing? Our *directSharing* results:

```
318,186,290     L1-dcache-loads         #   16.242 M/sec
124,027,868     L1-dcache-load-misses   #   38.98% of all L1-dcache hits
```
Look incredibly similar to our *falseSharing* results:
```
456,782,494     L1-dcache-loads         #   30.275 M/sec
183,018,811     L1-dcache-load-misses   #   40.07% of all L1-dcache hits
```
But this isn't incredibly surprising. At the lowest level, these benchmarks behave almost identically. All four of our threads are competing for the same cache-line/block in both benchmarks. It does not matter if they are accessing the same or different parts of that cache-line/block, because both will cause the same invalidation.

Our *noSharing* benchmark looks much likes the first:
```
1,641,417,172   L1-dcache-loads         #   106.878 M/sec
54,749,621      L1-dcache-load-misses   #   3.34% of all L1-dcache hits
``` 

## Concluding remarks



