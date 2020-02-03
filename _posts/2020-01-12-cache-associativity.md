---
layout: default
title: Cache Associativity
---

# Cache Associativity

Optimizing applications requires us to understand the small details of our hardware. Understanding the memory hierarchy is especially important. In this tutorial, we will be exploring the effects of cache associativity on performance using a simple power-of-two acceess pattern, and different sizes of arrays. A link to the source code used in this tutorial can be found here:

- [Source Code](https://github.com/CoffeeBeforeArch/spring_2020_tutorial/tree/master/associativity)

## Background

Before we dig into our benchmarks, we must first talk about the structure of modern caches. Caches typically come in three varieties, each with their own set of pros and cons. While we could discuss the organization and design tradeoffs of caches in detail, I'll limit our discussion to only what is necessary to understand our benchmarks.

### Direct Mapped Caches

Direct mapped caches are the most basic form a cache. In a direct mapped cache, each cache block has single location where it can be stored. That means that when we look up an address in our cache, we only need to check a single location. Furthermore, replacing cache blocks is simple. If we load cache block _B_, and cache block _A_, is mapped to the same location and already in that slot, we simply kick _A_ out the cache.

However, this simplicity has some significant disadvantages in terms of performance. If we repeatadly access cache blocks that are mapped to the same location, we end up kicking out data we will access in the near future. These pathological access patterns are common enough where we don't see direct mapped caches anymore in the memory hierarchy.

### Fully Associative Caches

At the other extreme we have fully associative caches. With a fully associative cache, a cache block can sit in any location. This makes our cache extremely flexible, and immune to the pathological access patterns that make direct mapped caches unnatractive. However, there's no such thing as a free lunch. With a fully associative cache, we now have to check the _entire_ cache when looking up an address (our cache block could be in anywhere in the cache!). This makes the hardware incredibly expensive, and not feasible for large caches.

### Set-Associative Caches

Set-associative caches represent a compromise between direct mapped and fully associative. In a set-associative cache, each cache block can be placed in one of _M_ ways in the set it has been mapped to. While not as flexible as a fully-associative cache, a set-associative cache can avoid many of the pathological access patterns of a direct mapped cache while still being practical to implement. These are the most common types of caches in modern architectures.

### Problematic Access Patterns for Set-Associative Caches

While a set-associative cache is good, there are still access patterns that can lead to poor performance. If we access cache blocks that all happen to map to the same set, there are still on _M_ places to put them, even if the rest of our cache is empty. We tend to see problems with associativity when we have accesses in a stride length that is a power-of-two. This is because our cache sizes and associativity tend to also be powers of two. We'll look more closely as to why this happens with some calculations later on. 

### The Four Types of Cache Misses (The Four Cs)
Before we continue our discussion on associativity, we should discuss how cache misses are classified. This is typically done in four categories:

- Cold-start
- Capacity
- Coherence
- Conflict

Cold-start misses are intuitive. If we've never accessed a piece of data before, it's unlikely to be in our cache. However, hardware and software prefetching mechanisms help programmers avoid some of these misses.

Capacity misses occur because the data we are accessing is simply larger than what will fit inside of our cache. If our cache is only 32KB, it's not going to be able to hold a 1GB array.

Coherence misses are slightly more complicated, as they are the result of multiple threads writing data to the same cache blocks. For more information about this, check out my [previous blog post](https://coffeebeforearch.github.io/2019/12/28/false-sharing-tutorial.html) on false sharing.

Conflict misses deal with the structure of our caches, and will be the primary subject of this blog post. Conflict misses can occur even when the majority of our cache is empty. The problem is that modern caches are set-associative. If we access too many cache blocks that are mapped to the same set, they end up kicking each other out.

## Benchmarking and Profiling

The following results were collected using [Google Benchmark](https://github.com/google/benchmark) and the [perf Linux profiler](https://perf.wiki.kernel.org/index.php/Main_Page). Each of our benchmarks follows the following form, and was compiled using:

```bash
g++ l1_bench.cpp -lbenchmark -lpthread -O3 -march=native
```

The benchmarks also follow the following structure:

```cpp
static void L1_Bench(benchmark::State &s) {
  // Const step size (4kB)
  const int step = 1 << 10;

  // Use a variable array size
  int size = 1 << s.range(0);
  vector<int> v(size);

  // Number of accesses
  const int MAX_ITER = 1 << 20;

  // Profile the runtime of different step sizes
  while (s.KeepRunning()) {
    int i = 0;
    for (int iter = 0; iter < MAX_ITER; iter++) {
      v[i]++;
      // Reset if we go off the end of the array
      i += step;
      if (i >= size) i = 0;
    }
  }
}
BENCHMARK(L1_Bench)->DenseRange(13, 16)->Unit(benchmark::kMillisecond);
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

On Linux machines, you can get this information with the following command:

```bash
getconf -a | grep CACHE
```

### Our Benchmarks at the Assembly Level

Before we take a look at the results, it's important to understand what our code is doing at the lowest level. The following pieces of assembly were taken from perf reports, where the column labled "Percent" corresponds to where the profiler is saying our program is spending most time. As the only operation we are doing is incrementing integers along a stride, our memory access takes most of the time.

```assembly
Percent│const int step = 1 << 10;
  8.76 │68:┌─→ movslq %eax,%rcx
  0.04 │   │  add    0x400,%eax
 90.77 │   │  incl   0x0(%rbp,%rcx,4)
  0.42 │   │  cmp    %eax,%ebx
  0.01 │   │  cmovge %r12d,%eax
       │   ├──dec    %edx
  0.01 │   └──jne    90
```

### Predicting Pathological Access Patterns

Now that we understand the basics of cache associativity, and the platform we are working on, we can use some faily basic calculation to predict which accesss patterns will lead to bad performance. For this experiment, we will focus on the L1 cache, as it is the closest to the core, and easiest to predict the behavior of.

Our L1 cache is 32KB, which we can re-write as 2^15B. Power-of-two exponents are much easier to deal with in these calculations. We also know that it is 8-way set associative (2^3 ways). From there, we can figure out the size the each way in our cache, which is 2^15B / 2^3 ways = 2^12B/way.

Ok, so we've found the B/way of our L1 cache. Why do we care? This also corresponds to the stride of our pathological access pattern! But how? More simple calculations to the rescue! Cache blocks are mapped to sets in a cache.  A set are just group of _M_ ways where a cache block can sit. We can find the number of sets in our cache by doing some more division. The number of sets is the number of bytes per way divided by the cache block size (typically 64B).

For our processor, the number of sets is equal to 2^12 / 2^6 which gives us 2^6 (64) sets in our cache. So how do we interpret these results? Our addresses are first split into cache blocks every 64B. For example, addresses 0x0 -> 0x3f get mapped to cache block 0, and address 0x40 -> 0x7f get mapped to cache block 1. These cache blcok are then mapped to a set.

The mapping of addresses to sets in the L1 cache is simple. We can find which set a cache block is mapped do by taking the cache block number and doing the modulo by the number of sets (Cache block # % # of sets). For example, cache block 0 gets mapped to set 0 (0 % 64), cache block 1 to set 1 (1 % 64), and cache block 64 back to set 0 (64 % 64). Because I have an 8-way set associative L1 cache in my processor, there are 8 places for a cache block to go in each set. If we accessed cache block numbers 0, 64, 128, ..., 448 (eight cache blocks with 64 cache blocks between each one), all 8 could fit into a single set of our cache.

Pathological access patterns for associativity depend on two parameters; the stride of the accesses, and the number of cache blocks accessed. If we do a non-power-of-two stride, our cache blocks will be mapped across multiple sets. If we access too few cache blocks, there won't be enough contention. Luckily for us, we only need to extend our previous access pattern example slightly to show how the L1 cache can be rendered completely ineffective.

If we keep our stride of 64 cache blocks (4096B), we can ensure each cache block we access is mapped to the same set in the L1 cache. If we double the number of cache blocks we access from the previous example (16, instead of 8), we will be accessing twice as many cache blocks as could fit into a single set of our cache. This means that we will bring the first 8 cache blocks, then immediately kick them out when we access the next 8, and vice versa when we access the first 8 cache blocks again. This should give use an approximately 100% miss-rate. 

## Execution Time and Miss-Rate Results

In this section we'll look at some execution time and miss-rate numbers. These will be used to prove out our calculations from the previous section.

### Notes on the benchmark
Note, our benchmarks use integers (4B) instead of characters (1B). This is because we often work with larger data types (ints, floats, doubles), so I wanted to provide a more realistic example. As a result, we must convert things, like our stride, from 4096B to 1024 integers. Furthermore, when you see the benchmark output:

```
L1_Bench/13 ...
```

The number 13 corresponds to the power-of-two number of integers in the array (2^13 = 8192 integers = 32kB). We will be using our calculated problematic stride (4096B = 1024 integers). An interesting aspect of this benchmark you can also explore is the impact of different power-of-two strides. For the sake of brevity, that data is omitted from this post. 

### No Contention (~100% Hit-Rate)

Let's first take a look at the case where we access only 8 cache blocks that map to a single set.

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

That is just about as close to 100% as we're ever going to get!

### Full Contention (~100% Miss-Rate)

Now we can take a look at increasing the length of our array by 2x. Now we're accessing 16 cache blocks that all map to a single set.

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

### A Non-Power-of-Two Stride

For some fun, we can play around with our stride to make sure this is an associativity problem, not just a capacity one. Let's add a number like 17 to our stride, and see how the results differ.

Our only change to the benchmark:

```cpp
const int step = (1 << 10) + 17;
```

The benchmark results:

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

Notice how it takes us significantly longer to see any increase in execution time? This is because the cache blocks we're accessing are mapped to more than one set! Eventually we see an increase in execution time, but we can attribute this to the L1 cache only being 32kB, and not being able to hold an entire multi-megabyte vector (capacity misses).

### Beyond the L1 Cache
We know that our L2 and L3 caches are set-associative as well. Let's take a look at some results from a slightly different benchmark where we increase our stride to 512kB:

```
-------------------------------------------------------
Benchmark             Time             CPU   Iterations
-------------------------------------------------------
LLC_Bench/20       1.76 ms         1.76 ms          397
LLC_Bench/21       4.45 ms         4.44 ms          159
LLC_Bench/22       5.05 ms         5.05 ms          139
LLC_Bench/23       5.94 ms         5.93 ms          116
LLC_Bench/24       5.75 ms         5.75 ms          124
LLC_Bench/25       5.84 ms         5.83 ms          120
LLC_Bench/26       6.93 ms         6.92 ms           83
LLC_Bench/27       11.6 ms         11.6 ms           54
LLC_Bench/28       23.9 ms         23.9 ms           30
LLC_Bench/29       38.7 ms         38.7 ms           17
LLC_Bench/30       45.2 ms         45.2 ms           13
```

Unsurprisingly the execution time increases as we get more conflict misses at lower levels of the cache hierarchy. For comparison, let's do the same test we did with the L1 cache, where we add 17 to the stride. Here are the results:

```
-------------------------------------------------------
Benchmark             Time             CPU   Iterations
-------------------------------------------------------
LLC_Bench/20       1.28 ms         1.27 ms          552
LLC_Bench/21       1.32 ms         1.32 ms          513
LLC_Bench/22       1.32 ms         1.32 ms          527
LLC_Bench/23       1.34 ms         1.34 ms          531
LLC_Bench/24       1.35 ms         1.34 ms          524
LLC_Bench/25       1.37 ms         1.37 ms          515
LLC_Bench/26       1.51 ms         1.51 ms          465
LLC_Bench/27       1.70 ms         1.70 ms          427
LLC_Bench/28       13.3 ms         13.3 ms           52
LLC_Bench/29       15.9 ms         15.9 ms           47
LLC_Bench/30       21.9 ms         21.6 ms           35
```

A huge difference in performance! Even at our largest array that we iterate over (2^30 integers), we're still 2x faster than the power of two stride! Many of the misses we see in this case are capacity, rather than conflict misses.

## Concluding remarks

If you care about your performance of your software, you should also care about what hardware that software is running on. Seemingly innocuous access patterns can lead to significant drops in performance when they stress things like the organization of our caches. While we intentionally exposed the limitations of our cache associativity through microbenchmarks, this can also happen in real-world applications. Examples of this would be column-major accesses of a matrix with a power-of-two dimension, or striding through an array of objects, updating a fields that are a power-of-two number of bytes away from each other.

Thanks for reading,

--Nick

### Links

[Source Code](https://github.com/CoffeeBeforeArch/spring_2020_tutorial/tree/master/associativity)
