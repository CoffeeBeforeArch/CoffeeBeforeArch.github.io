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

Fundamentally, a spinlock requires three elements for functionality:

1. State to maintain whether or not the spinlock is currently locked
2. A method for locking the spinlock
3. A method for unlocking the spinlock

Here is a naive way you could implement a spinlock as a class in C++:

```cpp
 class Spinlock {
  private:
   // Spinlock state
   std::atomic<bool> locked{false};
 
  public:
   // Lock the spinlock
   void lock() {
     while (locked.exchange(true))
       ;
   }
   
   // Unlock the spinlock
   void unlock() { locked.store(false); }
 };
```

### Spinlock State

For our spinlock state (`locked`) we're using a `std::atomic<bool>`. This has been made atomic because multiple threads will be competing to modify the spinlock state (lock the spinlock) at the same time.

### The Lock Method

Our `lock` method is simple `while` loop where we repeatedly do an atomic exchange of `true` with the current state of the lock (`locked`). If the lock is free (`locked == false`), the atomic exchange will set `locked` to `true`, and return `false`. This will make the thread break out of the `while` loop, and return from the `lock` method.

If the spinlock was already locked by another thread (`locked == true`), the atomic exchange will repeatedly swap out `true` (the state of the `locked`) with `true` (the value passed the `.exchange()`) method. This means the thread waiting will get stuck in the `while` loop until the lock is eventually freed (by another thread), and the waiting thread does an exchange that returns `false`.

### The Unlock Method

The `unlock` method for our thread is incredibly simple. When the thread holding the lock wants to release the lock, it simply sets `locked` to `false`. The next atomic exchange from a waiting thread in the `lock` method will then grab the lock.

### Low Level Assembly

Let's take a look at the low level assembly for our benchmark to see what is being executed by the processor:

```assembly
  0.33 │20:┌─→mov     %edi,%ecx     
 77.99 │   │  xchg    %cl,(%rdx)    
  0.03 │   │  test    %cl,%cl       
  0.18 │   │↑ jne     20            
  8.43 │   │  incq    (%rsi)        
 13.05 │   │  xchg    %cl,(%rdx)    
       │   ├──dec     %eax          
  0.00 │   └──jne     20            
```

The first thing we see is our loop with an atomic exchange (`xchg`). This is our `lock` method where we spin in a `while` loop until our atomic exchange returns false.

The next thing we have is an increment instruction (`incq`). This is the increment of our shared variable in the `inc` fucntion our threads are running.

Finally, we have another atomic exchange `xchg` to release the lock, before decrementing the loop counter for our `for` loop, and repeating the process if the 100k iterations are not done.

### Performance

Here are the end-to-end performance numbers for 1, 2, 4, and 8 threads:

```txt
------------------------------------------------------------
Benchmark                  Time             CPU   Iterations
------------------------------------------------------------
naive/1/real_time       1.05 ms        0.054 ms          686
naive/2/real_time       14.2 ms        0.090 ms           47
naive/4/real_time       66.8 ms        0.129 ms           10
naive/8/real_time        247 ms        0.255 ms            3
```

We might have initially assumed that doubling the number of threads would double the execution time. This is because the amount of work (number of total increments of our shared value) doubles as we double the number of running threads. However, this is not what we see from our timing results. Instead of our 2-thread benchmark taking 2x as long as the single-threaded, it takes 14x longer. You can observe a similar trend between the other benchmark cases as well.

We must have some inter-thread contention that is causing our execution time to expload. Because our benchmark is simply doing a few loads and stores to memory, a good place to start would be looking at our caches. Here is our L1 cache miss-rate for the 8 thread benchmark:

```txt
104,434,391      L1-dcache-loads           #   10.438 M/sec                  
 48,181,819      L1-dcache-load-misses     #   46.14% of all L1-dcache hits  
```

It seems like our current spinlock implementation is causing a huge number of L1 cache misses. This is something we'll tackle with our first optimization called locally spinning.

## A Spinlock with Locally Spinning

To tackle our L1 cache misses observed in our naive implementation, we first have to understand where our cache misses are coming from. One way we can approach this problem is by categorizing our misses by why the miss occured. For this, we can use the 4-Cs of cache misses.

### The 4-Cs of Cache Misses

Cache misses are clasically categorized into 4 types:

1. Cold-Start/Compulsory Misses
2. Capacity Misses
3. Conflict Misses
4. Coherence Misses

Let's reason about which category our misses fall into so that it can guide our plan for optimization.

#### Cold-Start/Compulsory Misses

Cold-start misses (sometimes called compulsory misses) occur when we access data that for the first time. With the exception of prefetching, data accessed for the first time will not be in our caches, and we will see cache misses.

Is the large percentage of L1 cache loads being misses a result of cold-start misses? 

Unlikely. Let's thing about the loop each thread in our benchmark is running:

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

We're repeatedly accessing the same two pieces of memory (the state of the spinlock in our `lock` and `unlock` methods, and the shared value `val`). While we may initially miss in the first iteration of our loop, we should not in the thousands of remaining iterations.

#### Capacity Misses

Capacity misses occur when the amount of data we're accessing is simply too large to fit into our cache. For example, if we're iterating over 1GB of data, that won't completely fit into a 32kB L1 cache.

Is the large percentage of L1 cache loads being misses a result of cold-start misses? 

Unlikely. Let's think about the size of the data we're accessing. We're still only accessing two pieces of data we're accessing (our lock state, and shared value `val`). Our lock state is a `std::atomic<bool>` with conservatively only takes up 1 bytes. Our value, `val`, is a `std::uint64_t`, which is 8 bytes. That means together we're only using 9 bytes of data.

9 bytes of data is not large enough to cause capacity misses in a modern L1 data cache.

#### Conflict Misses

#### Coherence Misses

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

