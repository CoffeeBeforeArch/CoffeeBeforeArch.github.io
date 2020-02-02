---
layout: default
title: Cache Associativity
---

# Cache Associativity

Understanding why different access patterns lead to different performance results requires us to understand the underlying hardware of our processors. In this tutorial, we will be exploring the effects of cache associativity on performance using a simple power-of-two acceess pattern, and different sizes of arrays. A link to the source code used in this tutorial can be found here:

- [Source Code](https://github.com/CoffeeBeforeArch/uarch_benchmarks/tree/master/associativity)

## Background

Before we dig into our benchmarks, we must first talk about the structure of modern caches. Caches typically come in three varieties, each with their own set of pros and cons. While we could discuss, the organization and design tradeoffs of caches in length, we'll limit our discussion to only what is necessary to understand what we are measuring.

### Direct Mapped Caches

Direct mapped caches are the most basic form a cache. In a direct mapped cache, each cache block has single location where it can be stored. This design has a number of benefits. For example, when we look up an address in our cache, we only need to check a single location. Furthermore, replacing cache blocks becomes simple. If we load cache block _B_, and cache block _A_, is mapped to the same location and already there, we simply kick out _A_ from the cache.

However, this simplicity has some significant disadvantages in terms of performance. If we repeatadly access cache lines that are mapped to the same location, we end up kicking out data we will access in the near future. These pathological access patterns are commaon enough where we don't see many direct mapped caches anymore.

### Fully Associative Caches

At the other extreme we have fully associative caches. With a fully associative cache, a cache block can sit in any location. This makes our cache extremely flexible, and immune to the pathological access patterns that make direct mapped caches infeasable. However, this flexibility has a different disadvantage. With a fully associative cache, we now have to check the _entire_ cache when looking up an address. This makes the hardware significantly more expensive than the direct mapped cache.

### Set-Associative Caches

Set-associative caches represent a comprimise between direct mapped, and fully associative caches. In a set-associative cache, each cache line can be placed in one of _M_ ways in the set it has been mapped to. This compromise allows us to avoid many of the pathological access patterns of a direct mapped cache, while avoiding the significant hardware cost of fully associative caches. These are the most common types of caches in modern architectures.

### Problematic Access Patterns for Set-Associative Caches

While a set-associative cache is good, there are still access patterns that can lead to poor performance. If we access cache lines that all happen to map to the same set, there are still on _M_ places to put them. This tends to happen with power of 2 stride access patterns. The mappings of cache blocks to sets is fairly simple. When we increase the power of two of our stride, the cache lines we access are mapped to fewer and fewer sets, leading to greater contention.

### The Four Types of Cache Misses

Cache misses are typically grouped into 4 categories:
- Cold-start
- Capacity
- Coherence
- Conflict

Cold-start misses are intuitive. If we've never accessed a piece of data before, it's unlikely to be in our cache. However, hardware and software prefetching mechanisms help programmers avoid some of these misses.

Capacity misses occur because the data we are accessing is simply larger than what will fit inside of our cache. If our cache is only 32kB, it's not going to be able to hold a 1GB array.

Coherence misses are slightly more complicated, as they are the result of multiple threads writing data to the same cache lines. For more information about this, check out my previous blog post on false sharing.

Conflict misses deal with the structure of our caches, and will be the primary subject of this post. Conflict misses occur even when there is unused space in the cache. The problem is that modern caches are set-associative. Cache lines may end up kicking each other out of the cache when they get mapped to the same set.

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

The caches on the Kaby Lake processor this was run on are organized as follows:

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
LEVEL3_CACHE_SIZE                  3145728
LEVEL3_CACHE_ASSOC                 12
LEVEL3_CACHE_LINESIZE              64
```

On Linux machines, you can get this information from the following command:

```bash
getconf -a | grep CACHE
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

### Predicting Pathological Access Patterns

Now that we understand the basics of cache associativity, and the platform we are working on, we can use some faily basic calculation to predict which accesss patterns will lead to bad performance. For this experiment, we will focus on the L1 cache, as it is the closest to the core, and easiest to predict the behavior of.

Our L1 cache is 32 kB, which we can re-write as 2^15 B. Power-of-two exponents are much easier to deal with in these calculations. We also know that it is 8-way set associative. From there, we can figure out the size the each way in our cache. This can be done with a simple calculation of 2^15 B / 2^3 ways = 2^12 B/way.

Ok, so we've found the B/way of our L1 cache. Why do we care? This also corresponds to the stride of our pathological access pattern! But how? More simple calculations to the rescue! Cache lines are mapped to sets in a cache. We can find the number of sets in our cache by doing some more division. We can find the number of sets in our cache by dividing the number of bytes in each way by the cache lines size (typically 64 B). These represents all of the "slots" to which a cache line can be mapped.

For our values, we find that the number of sets in our cache is equal to 2^12 B/way / 2^6 B/cache line which gives us 2^6 (64) sets in our cache. So how do we interpret these results? Our addresses are first split into cache lines every 64 B. For example, addresses 0x0 -> 0x3f get mapped to cache lines 0, and address 0x40 -> 0x7f get mapped to cache line 1. These cache lines are in turn mapped to sets in our cache.

The mapping of addresses to sets in the L1 cache is simple. If we have 64 sets, cache lines get mapped to these sets using a simple modulo operation (Cache Line # % # of sets). This means cache line 0 gets mapped to set 0 (0 % 64), cache line 1 to set 1 (1 % 64), and cache line 64 back to set 0 (64 % 64). Because we have an 8-way set associative cache, there are 8 places for a cache line to go in each set. That means that if we happened to access cache lines 0, 64, 128, ..., 448 (eight cache lines with 64 cache lines in between each line), all 8 could fit into our L1 cache.

Pathological access patterns for associativity depend on two parameters; the stride of the access pattern, and the number of cache lines accessed. If we do a non-power-of-two stride, our cache lines will be mapped across multiple sets. If we access too few cache lines, we will just experience hits in the cache. Luckily for us, we only need to extend our previous example slightly to render our L1 cache completely useless.

If we keep our stride of 64 cache lines (4096 B), we can ensure each cache line we access is mapped to the same set in the L1 cache. If we double the number of cache lines we access from the previous example (16, instead of 8), we will be accessing twice as many cache lines as could fit into a single set of our cache. This means that we will bring the first 8 cache lines, then immediately kick them out when we access the next 8, and vice versa when we access the first 8 cache lines again. This should give use an approximately 100% miss-rate. 

## Execution time and miss-rate results

Let's first take a look at the case where we access only 8 cache lines that map to a single set.

```
------------------------------------------------------
Benchmark            Time             CPU   Iterations
------------------------------------------------------
L1_Bench/13       1.52 ms         1.52 ms         2775
```

We can't tell much with a single measurement, but we can still see if our prediction about having a 100% hit rate ended up being true.

```
4,081,018,718    L1-dcache-loads         693.978 M/sec
733,294          L1-dcache-load-misses   0.02% of all L1-dcache hits

```

That is just about as close to 100% as we're ever going to get! Now we can take a look at increasing the length of our array by 2x. Now we're accessing 16 cache lines that all map to a single set.

```
------------------------------------------------------
Benchmark            Time             CPU   Iterations
------------------------------------------------------
L1_Bench/14       4.19 ms         4.19 ms         1004
```

Our execution time increased by over 2x (from 1.53ms to 4.19ms)! Now let's look at our L1 cache hit rate.

```
1,173,287,652   L1-dcache-loads         249.982 M/sec
1,328,613,357   L1-dcache-load-misses   113.24% of all L1-dcache hits
```

As expected, our L1 cache miss-rate went through the roof!

For some fun, we can play around with our stride to make sure this is an associativity problem, not just a capacity one. Let's add a number like 17 to our stride, and see how the results differ.

```
Benchmark            Time             CPU   Iterations
------------------------------------------------------
L1_Bench/13       1.14 ms         1.14 ms          606
L1_Bench/14       1.14 ms         1.14 ms          614
L1_Bench/15       1.14 ms         1.14 ms          613
L1_Bench/16       1.14 ms         1.14 ms          611
L1_Bench/17       1.24 ms         1.24 ms          550
L1_Bench/18       1.24 ms         1.24 ms          561
L1_Bench/19       1.26 ms         1.26 ms          552
L1_Bench/20       1.33 ms         1.33 ms          526
L1_Bench/21       5.02 ms         5.02 ms          137
```

Notice how it takes us significantly longer to see any increase in execution time? This is because the cache lines we're accessing are mapped to more than one set! Eventually we see an increase in execution time, but we can attribute this to the L1 cache only being 32kB, and not being able to hold an entire multi-megabyte vector (capacity misses).

## Concluding remarks

If you care about your performance of your software, you should also care about what hardware that software is running on. Seemingly innocuous access patterns can lead to significant drops in performance.

Thanks for reading,

--Nick

### Links to source and YouTube Tutorial

- [Source Code](https://github.com/CoffeeBeforeArch/uarch_benchmarks/tree/master/associativity)

### Additional discussion points

- Where does this happen in reality?
  - If you have a matrix with power-of-two elements in each row, a column major access pattern could lead to these conflict misses
  - If you have an array of objects to update a field in each one, this could lead to these conflict misses.
