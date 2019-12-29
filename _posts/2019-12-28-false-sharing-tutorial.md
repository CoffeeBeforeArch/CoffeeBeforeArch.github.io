---
layout: default
title: Performance Implications of False Sharing
---

# The Performance Implications of False Sharing

One of the most important considerations when writing parallel applications is how different threads or processes share data. In some circumstances, sharing is unavoidable. However, our data layout and architecture may introduce unintentional sharing, known as false sharing. This tutorial provides a basic introduction to false sharing through some simple benchmarking and profiling.

The links below are to the source code used and video version of this tutorial:
- [Source Code](https://github.com/CoffeeBeforeArch/uarch_benchmarks/tree/master/false_sharing)
- [YouTube Video](https://youtu.be/FygXDrRsaU8)

This tutorial was inspired by the following talk on performance by Timur Doumler:
- [Want fast C++? Know your hardware!](https://youtu.be/a12jYibw0vs)

## Background

Why is sharing data so bad? It's not! If we have many threads that are only reading some shared pieces of memory, our performance shouldn't suffer. The problem comes when multiple threads want to write to that piece of memory, and this goes back to cache coherence.

### Cache Coherence

Cache coherence is often defined using two invariants, as taken from [A Primer on Memory Consistency and Cache Coherence](https://www.morganclaypool.com/doi/abs/10.2200/S00346ED1V01Y201104CAC016):

1. Single-Writer, Multiple Reader Invariant: For a memory location *A*, at any given logical time, there exists only a single core that may write to *A* (and read from *A*), or some number of cores (maybe 0) that may only read *A*.

2. Data-Value Invariant - The value at any given memory location *A* at the start of an epoch is the same as the value of the memory location at the end of its last read-write epoch.

Invariant 1 shows why sharing read-only data is OK while sharing writable data can cause performance problems. When one core wants to write to a memory location, access to that location must be taken away from other cores (to keep them from reading old/stale values). Access is taken away by invalidating copies of the data other cores have in their cache hierarchy. Once the core trying to perform a write has gained exclusive access, it can perform its write operation.

But what exactly is getting invalidated? Typically, it is a cache-line/block. Cache-lines/blocks strike a good balance between control and overhead. Finer-grained coherence (say at the byte-level) would require us to maintain a coherence state for each byte of memory in our caches. Coarser-grained coherence (say at the page-level) can lead to the unnecessary invalidation of large pieces of memory.

To simplify our discussion, we will simply refer to the range of data for which a single coherence state is maintained as a *coherence block* (a page or cache-line/block in practice).

### Where does false sharing come from?

False sharing occurs when data from multiple threads that was not meant to be shared gets mapped to the same coherence block. When this happens, cores accessing seeming unrelated data still send invalidations to each other when performing writes. This can lead to performance that looks like multiple threads are fighting over the same data. Let's consider a simple optimization case-study where a well-intentioned programmer optimizes code without thinking about the underlying architectures.

## Optimization Case Study

Consider the following simple function in C++ that we want to run 4 times:

```cpp
void work(std::atomic<int>& a) {
  for(int i = 0; i < 100000; ++i) {
    a++;
  }
}

```

All we are doing is atomically incrementing an atomic integer _a_ 100k times. Let's say that we want to implement this in a single thread. All we need to do is call our work function 4 times. We implement this like so:

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

A terrible way to speed up this computation is to *only* throw threads at the problem, and have each theads handle one of the function calls. That could look something like this:

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

This is *direct* sharing because all four threads are incrementing the same atomic integer *a*. The cache-line/block that *a* sits on bounces between the different cores the threads are scheduled to, leading to significantly worse performance than if we did all the work in a single thread.

As a well-intentioned programmer, we may try to solve this problem by giving each thread a unique atomic integer to increment. This is a common optimization strategy in parallel programming. We can reduce the number of shared writes to a single shared location by having threads calculate partial results independently, then merging these partial results. That could be implemented as follows:

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

For simplicity, we ignore the merging of the partial results, as the 3 additional add operations do not significantly change the results.

While it looks like we solved our sharing problem by creating multiple atomic integers, we'll see this has almost the same performance as the *directSharing* benchmark. Why is that? Let's think harder about where those four atomic integers sit in memory. We can print out their addresses like so:

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

Notice that all of our atomics are four bytes away from each other. We already discussed that cache coherence is typically at the cache-line/block granularity, and cache lines in modern processors are typically 64 bytes. Therefore, our 4 atomics allocated like this will typically end up on the same cache line. When each thread grabs its dedicated atomic integer, it grabs all four. There's our false sharing!

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

Now we have 64 bytes between each of our objects. Our atomics are now guaranteed to be on different cache-lines/blocks! Our modified code using this *AlginedType* looks like this:

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

The following results were collected using [Google Benchmark](https://github.com/google/benchmark) and the [perf Linux profiler](https://perf.wiki.kernel.org/index.php/Main_Page). Each of our benchmarks follows the following form, and was compiled using:

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

Before we take a look at the results, it's important to understand what our code is doing at the lowest level. The following pieces of assembly were taken from perf reports, where the column labled "Percent" corresponds to where the profiler is saying our program is spending most time.

For our *singleThread* benchmark, four calls to the *work()* function are inlined, leading to four tight loops that do an atomic increment:

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

If you have not compiled with -march=native, the incl (increment) instruction may be replaced with and addl instruction:

```assembly
10:   lock   addl   $0x1,(%rdx)
```

The code for our multi-threaded benchmarks all look identical. Each thread simply runs a single inlined version of the *work()* function.

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
As is the case for the *singleThread* benchmark, all three of the multithreaded benchmarks spend most of their time on the atomic increment.

### Execution time results

For each of the benchmarks we will be focusing on the wall clock time (column labled "Time"). This is because the CPU time (column labled CPU) only measures the time spent by the main thread, which is not helpful for the multi-threaded benchmarks. 

Execution time results for our four microbenchmarks can be found below.

```
------------------------------------------------------------------
Benchmark                        Time             CPU   Iterations
------------------------------------------------------------------
singleThread                  2.35 ms         2.35 ms          300
directSharing/real_time       7.69 ms        0.084 ms           95
falseSharing/real_time        7.78 ms        0.083 ms           92
noSharing/real_time           1.75 ms        0.083 ms          386
```

Unsurprisingly, our *directSharing* and *falseSharing* benchmarks take roughly the same execution time. They are over three times slower than the *singleThread* benchmark. Our *noSharing* benchmark had the best performance, about 25% faster than the single-threaded baseline. Why is it not 4x faster? Creating and joining threads isn't free! If we made our *work()* function do only 10k increments, our *singleThread* and *noSharing* take roughly the same time:

```
------------------------------------------------------------------
Benchmark                        Time             CPU   Iterations
------------------------------------------------------------------
singleThread                 0.233 ms        0.233 ms         3001
directSharing/real_time      0.668 ms        0.059 ms          762
falseSharing/real_time       0.710 ms        0.053 ms          998
noSharing/real_time          0.245 ms        0.044 ms         2614
```

How does the performance look when we scale the number of threads? More threads lead to more contention on a single memory location. Below are the results for our *falseSharing* benchmark using 2, 4, and 8 threads (with the amount of work per thread appropriately scaled):

```
-----------------------------------------------------------------
Benchmark                       Time             CPU   Iterations
-----------------------------------------------------------------
twoThreads/real_time         6.49 ms        0.050 ms          106
fourThreads/real_time        7.95 ms        0.077 ms           90
eightThreads/real_time       9.24 ms        0.125 ms           77
```

### L1 cache hit rate

Our perf report gives us a decent idea about where our time is being spent in our benchmarks (unsurprisingly, waiting for our atomic increments). However, if we want to know why the performance of these applications differs so significantly, we must look at another metric. The L1 data cache hit rate is an excellent place to start, and we can be access it using perf stat.

For our direct and false sharing benchmarks, we claimed that invalidation requests were being sent back and forth between the cores. As a result, we'd expect both *directSharing* and *falseSharing* to have a low L1 hit rate as the cache-line/block with the atomic integer(s) bounces between cores. Furthermore, we should expect *singleThread* and *noSharing* to have very high hit rates.
  
For the *singleThread* benchmark, we get the expected result, as shown below:

```
1,158,985,491   L1-dcache-loads         #   170.514 M/sec 
319,420         L1-dcache-load-misses   #   0.03% of all L1-dcache hits
```

If we access the same variable 400k times from the same thread, it's probably going to stick around in our L1 cache. We should expect our benchmarks with sharing to look similar. Our *directSharing* results are shown below:

```
318,186,290     L1-dcache-loads         #   16.242 M/sec
124,027,868     L1-dcache-load-misses   #   38.98% of all L1-dcache hits
```
These look incredibly similar to our *falseSharing* results:

```
456,782,494     L1-dcache-loads         #   30.275 M/sec
183,018,811     L1-dcache-load-misses   #   40.07% of all L1-dcache hits
```

But this isn't incredibly surprising. At the lowest level, these benchmarks behave almost identically. In each of these benchmarks, all four of the threads compete for the same cache-line/block. It does not matter if they are accessing the same or different parts of that cache-line/block, because both cause the same invalidation request.

The hit rate from our *noSharing* benchmark, however, is closer to that of the *singleThread* benchmark:

```
1,641,417,172   L1-dcache-loads         #   106.878 M/sec
54,749,621      L1-dcache-load-misses   #   3.34% of all L1-dcache hits
``` 

This was because we ensured our atomics could never be mapped to the same cache line!

## Concluding remarks

Understanding how your underlying hardware works is incredibly important when you're trying to squeeze performance out of your code. False sharing is an example of how an obvious solution to a problem (like using multiple atomic integers) may not be a solution. We'll cover more optimization case studies like this in later tutorials.

Thanks for reading,

--Nick

### Links to source and YouTube Tutorial

- [Source Code](https://github.com/CoffeeBeforeArch/uarch_benchmarks/tree/master/false_sharing)
- [YouTube Video](https://youtu.be/FygXDrRsaU8)

### Additional discussion points

- Our benchmarks could be rewritten using atomic references that have been introduced in C++20. These provide an excellent way to have selective atomicity.
- Real workloads typically do more that have 4 threads only compete for a single cache-line. However, there may be regions of execution where this pathological case exists, and still causes a significant performance impact.
- Why do you need to use an atomic integer at all? [Here's](https://stackoverflow.com/a/39396999/12482128) a Stack Overflow response answering that question.
