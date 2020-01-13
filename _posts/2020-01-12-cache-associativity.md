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

Execution time results for our four microbenchmarks can be found below.

```
--------------------------------------------------------
Benchmark              Time             CPU   Iterations
--------------------------------------------------------
assocBench/0       0.907 ms        0.907 ms          779
assocBench/1        1.05 ms         1.05 ms          672
assocBench/2        1.60 ms         1.60 ms          430
assocBench/3        3.24 ms         3.24 ms          215
assocBench/4        6.42 ms         6.42 ms          107
assocBench/5        8.91 ms         8.91 ms           74
assocBench/6        10.0 ms         10.0 ms           69
assocBench/7        11.6 ms         11.6 ms           60
assocBench/8        12.2 ms         12.2 ms           58
assocBench/9        16.1 ms         16.1 ms           43
```
As predicted, our execution time steadily increases as the stride increases.

If we knew nothing about associativity, we may have predicted that our execution time would increase until 2^4, and stayed constant afterwards. At a stride of 2^4 (16) elements, each element we access belongs to a different cache line. This is also true for any power of two greater that 2^4. However, our execution time continues to increase. This is because the cache lines we access are mapped to fewer and fewer sets.

### L1 cache hit rate

Another statistic we can look at to confirm our increase in cache pressure is our cache miss-rate. Below are the miss-rates for each of the benchmarks:

```
assocBench/0   L1-dcache-load-misses   #   6.28%  of all L1-dcache hits
assocBench/1   L1-dcache-load-misses   #   12.53% of all L1-dcache hits
assocBench/2   L1-dcache-load-misses   #   24.95% of all L1-dcache hits
assocBench/3   L1-dcache-load-misses   #   48.89% of all L1-dcache hits
assocBench/4   L1-dcache-load-misses   #   97.42% of all L1-dcache hits
assocBench/5   L1-dcache-load-misses   #   95.35% of all L1-dcache hits
assocBench/6   L1-dcache-load-misses   #   97.73% of all L1-dcache hits
assocBench/7   L1-dcache-load-misses   #   96.70% of all L1-dcache hits
assocBench/8   L1-dcache-load-misses   #   98.54% of all L1-dcache hits
assocBench/9   L1-dcache-load-misses   #  172.16% of all L1-dcache hits
```

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
