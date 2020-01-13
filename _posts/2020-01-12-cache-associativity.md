---
layout: default
title: Cache Associativity
---

# Cache Associativity

Understanding why different access patterns lead to different performance results requires us to understand the underlying hardware of our processors. In this tutorial, we will be exploring the effects of cache associativity on performance using a simple power of two acceess pattern. A link to the source code used in this tutorial can be found here:
- [Source Code](https://github.com/CoffeeBeforeArch/uarch_benchmarks/tree/master/associativity)

## Background

Before we dig into our benchmarks, we must first talk about the structure of modern caches. Caches typically come in three varieties, each with their own set of pros and cons. While we could discuss, the organization and design tradeoffs of caches in length, we'll limit our discussion to only what is necessary to understand the observed effects.

### Direct Mapped Caches

Direct mapped caches are the most basic form a cache. In a direct mapped cache, each cache block has single location where it can be stored. This design has a number of benefits. For example, when we look up an address in our cache, we only need to check a single location. Furthermore, replacing cache blocks becomes simple. If we load cache block _B_, and cache block _A_, is mapped to the same location and already there, we simply kick out _A_ from the cache.

However, this simplicity has some significant disadvantages in terms of performance. If we repeatadly access cache lines that are mapped to the same location, we end up kicking out data we will access in the near future. These pathological access patterns are common enough where we don't see many direct mapped caches anymore.

### Fully Associative Caches

At the other extreme we have fully associative caches. With a fully associative cache, a cache block can sit in any location. This makes our cache extremely flexible, and immune to the pathological access patterns that make direct mapped caches infeasable. However, this flexibility has a different disadvantage. With a fully associative cache, we now have to check the _entire_ cache when looking up an address. This makes the hardware significantly more expensive than the direct mapped cache.

### Set-Associative Caches

Set-associative caches represent a comprimise between direct mapped, and fully associative caches. In a set-associative cache, each cache line can be placed in one of _M_ ways in the set it has been mapped to. This compromise allows us to avoid many of the pathological access patterns of a direct mapped cache, while avoiding the significant hardware cost of fully associative caches. These are the most common types of caches in modern architectures.

## Problematic Access Patterns for Set-Associative Caches

While a set-associative cache is good, there are still access patterns that can lead to poor performance. If we access cache lines that all happen to map to the same set, there are still on _M_ places to put them. This tends to happen with power of 2 stride access patterns. The mappings of cache blocks to sets is fairly simple. When we increase the power of two of our stride, the cache lines we access are mapped to fewer and fewer sets, leading to greater contention.

## Benchmarking and Profiling

The following results were collected using [Google Benchmark](https://github.com/google/benchmark) and the [perf Linux profiler](https://perf.wiki.kernel.org/index.php/Main_Page). Each of our benchmarks follows the following form, and was compiled using:

```bash
g++ associativity.cpp -lbenchmark -lpthread -O3 -march=native
```

The benchmarks also follow the following structure:

```cpp
static void assocBench(benchmark::State &s) {
  // Unpack the power of 2 stride
  int step = 1 << s.range(0);

  // Create a large vector
  const int N = 1 << 23;
  vector<int> v(N);
 
  // Number of accesses
  const int MAX_ITER = 1 << 20;
 
  while (s.KeepRunning()) {
    // Profile 2^20 accesses at this stride
    int i = 0;
    for (int iter = 0; iter < MAX_ITER; iter++) {
      v[i]++;
 
      // Reset if we go off the end of the array
      i += step;
      if (i >= N) i = 0;
    }
  }
}
BENCHMARK(assocBench)->DenseRange(0, 9)->Unit(benchmark::kMillisecond);
```

The caches on the Skylake processor this was run on are organized as follows:
```bash
LEVEL1_ICACHE_SIZE                 32768
LEVEL1_ICACHE_ASSOC                8
LEVEL1_ICACHE_LINESIZE             64
LEVEL1_DCACHE_SIZE                 32768
LEVEL1_DCACHE_ASSOC                8
LEVEL1_DCACHE_LINESIZE             64
LEVEL2_CACHE_SIZE                  262144
LEVEL2_CACHE_ASSOC                 4
LEVEL2_CACHE_LINESIZE              64
LEVEL3_CACHE_SIZE                  8388608
LEVEL3_CACHE_ASSOC                 16
LEVEL3_CACHE_LINESIZE              64
```

### Our benchmarks at the assembly level

Before we take a look at the results, it's important to understand what our code is doing at the lowest level. The following pieces of assembly were taken from perf reports, where the column labled "Percent" corresponds to where the profiler is saying our program is spending most time. As the only operation we are doing is incrementing along a stride, our memory access takes most of the time.

```assembly
Percent│
  8.76 │68:┌─→ movslq %eax,%rcx
  0.04 │   │  add    %ebx,%eax
 90.77 │   │  incl   0x0(%rbp,%rcx,4)
  0.42 │   │  cmp    $0x800000,%eax
  0.01 │   │  cmovge %r12d,%eax
       │   ├──dec    %edx
  0.01 │   └──jne    68
```

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
