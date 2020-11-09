---
layout: default
title: Spinlocks
---

# Spinlocks

One of the my important parts of parallel programming is synchronization and coordinating access to shared data. An important tool in that domain is the spinlock. Spinlocks provide a way for threads to busy-wait for a lock instead of yielding their remaining time to the O.S.'s thread scheduler. This can be increadibly useful when you know your threads will not be waiting very long for access to the lock.

In this blog post, we'll look at how spinlocks and their optimizations can be implemented in C++.

### Link to the source code

- [Source Code: ](https://github.com/CoffeeBeforeArch/spinlocks)
- [My YouTube Channel: ](https://www.youtube.com/channel/UCsi5-meDM5Q5NE93n_Ya7GA?view_as=subscriber)
- [My GitHub Account: ](https://github.com/CoffeeBeforeArch)
- My Email: CoffeeBeforeArch@gmail.com

## Our Benchmark

We will evaluate our spinlock implementations using [Google Benchmark](https://github.com/google/benchmark). The structure of the benchmarks can be found below:

```cpp
 // Small Benchmark
 static void bench(benchmark::State &s) {
   // Sweep over a range of threads
   auto num_threads = s.range(0);
 
   // Value we will increment
   std::int64_t val = 0;
 
   // Allocate a vector of threads
   std::vector<std::thread> threads;
   threads.reserve(num_threads);
 
   Spinlock sl;
 
   // Timing loop
   for (auto _ : s) {
     for (auto i = 0u; i < num_threads; i++) {
       threads.emplace_back([&] { inc(sl, val); });
     }
     // Join threads
     for (auto &thread : threads) thread.join();
     threads.clear();
   }
 }
 BENCHMARK(bench)
     ->RangeMultiplier(2)
     ->Range(1, std::thread::hardware_concurrency())
     ->UseRealTime()
     ->Unit(benchmark::kMillisecond);
```

We'll be evaluating our benchmark for a power-of-two number of threads (1, 2, 4, and 8 on my machine), where each thread runs a function called `inc`. This function can be found below:

```cpp
 // Increment val once each time the lock is acquired
 void inc(Spinlock &s, std::int64_t &val) {
   for (int i = 0; i < 100000; i++) {
     s.lock();
     val++;
     s.unlock();
   }
 }
```

For 100k iterations, our threads try and lock our spinlock, increment a shared value, then release the lock.

## A Naive Spinlock

## A Spinlock with Locally Spinning

## A Spinlock with Active Backoff

## A Spinlock with Passive Backoff

## A Spinlock with Non-Constant Backoff

### Exponential Backoff

### Random Backoff

## A Ticket Spinlock

## A pthreads Spinlock

## Final Thoughts

Thanks for reading,

--Nick

### Link to the source code

- [Source Code: ](https://github.com/CoffeeBeforeArch/spinlocks)
- [My YouTube Channel: ](https://www.youtube.com/channel/UCsi5-meDM5Q5NE93n_Ya7GA?view_as=subscriber)
- [My GitHub Account: ](https://github.com/CoffeeBeforeArch)
- My Email: CoffeeBeforeArch@gmail.com

