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

Is the large percentage of L1 cache loads being misses a result of capacity misses? 

Unlikely. Let's think about the size of the data we're accessing. We're still only accessing two pieces of data we're accessing (our lock state, and shared value `val`). Our lock state is a `std::atomic<bool>` with conservatively only takes up 1 bytes. Our value, `val`, is a `std::uint64_t`, which is 8 bytes. That means together we're only using 9 bytes of data.

9 bytes of data is not large enough to cause capacity misses in a modern L1 data cache.

#### Conflict Misses

Conflict misses occur when we access cache lines that all map to the same set (or subset of total sets) in our set-associative caches. For example, modern L1 caches usually have 8-way associativity. This means that each set has 8 slots for cache lines mapped to that set. If we access 8 cache lines that map to the same set, they can all be stored in our cache. However, if we access a 9th cache line, this will cause one of our cache lines in that set to be evicted, which can
lead to later cache misses.

Is the large percentage of L1 cache loads being misses a result of conflict misses? 

Unlikely, and for the same reason that our misses are not from capacity misses. We are simply not accessing enough data for conflict misses to be an issue. Realistically, the 9 bytes we are repeatedly accessing would at most be split across two different cache lines (one line for our spinlock state, and one line for our shared value `val`).

#### Coherence Misses

Coherence misses occur because of writes in multi-threaded applications. Before a thread can write to data on a cache line, it first must get that cache line in the modified (`M`) state. This state means that the thread has exclusive access to the cache line, and no other copies of the cache line exist in any other cache.

In order to guarantee that there are no copies of the cache line in any other cache, invalidations are sent out to all other cores/caches. Coherence misses occur when a cache line in one core is invalidated by a thread doing a write on a different core.

Is the large percentage of L1 cache loads being misses a result of coherence misses? 

Yes! Our benchmark is running multiple threads, all of which are writing to the same two pieces of memory (the spinlock state and the shared value `val`). All of our threads are fighting over the same cache line(s) and sending invalidations to each other.

One way we can prove this out is by eliminating all doubt that our misses are from the first 3-Cs of cache misses (cold-start, capacity, and conflict misses). These should exist even in the single-threaded benchmark. Here are the cache miss numbers when we only run a single thread:

```
254,289,668      L1-dcache-loads           #  321.707 M/sec                  
  2,906,087      L1-dcache-load-misses     #    1.14% of all L1-dcache hits  
```

Only ~1%! Our cache misses only spike up to ~50% when we run our multi-threaded benchmarks (2, 4, or 8 threads)

### Addressing Coherence Misses

Now that we know that our cache misses are due to coherence (specifically writes), what do we do about it? Intuitively, we want to limit the number of writes we perform, thereby reducing our cache line invalidation. Let's look at the three main places where we are performing writes:

1. The increment of our shared value `val`
2. The `unlock` method of our spinlock
3. The `lock` method of our spinlock

We can narrow down _where_ we will try and optimize by reasoning about the potential opportunity at each location.

#### The Increment

Can we limit the number of writes to our shared value `val`?

No, because that would not be addressing any of the issues with our spinlock. The increment of `val` is just the arbitrary piece of work we have given our threads to perform, and is completely independant from the implementation of our spinlock. We should instead be focusing on our spinlock implemntation, because this is what we have designed.

#### The Unlock Method

Can we limit the number of writes done by our `unlock` method?

No, because the number of writes our `unlock` method does is at minimum. Let's take a look at how we implemented this method in our `naive` implementation:

```cpp
   // Unlock the spinlock
   void unlock() { locked.store(false); }
```

Each call to `unlock` only performs a single store which is necessary to mark the lock as free. We'll have to look for opportunities elsewhere.

#### The Lock Method

Can we limit the number of writes done by our `lock` method?

Yes! Let's re-examine our `lock` implementation from our `naive` spinlock:

```cpp
   // Lock the spinlock
   void lock() {
     while (locked.exchange(true))
       ;
   }
```

As long as a thread is waiting in the `while` loop for the spinlock to be free, it is writing to the spinlock state (with the atomic exchange). That means that our cache line with the lock state is being torn between the different cores on our chip as fast as different threads can issue atomic writes. But how do we address this? 

We can start by rethinking _when_ we try and grab the lock. Our current `lock` method constantly tries and grabs the lock using atomic exchanges. This is usually a bad idea, because in many cases we will fail to get the lock (especially when as we increase the number of threads). What we can instead do is only issue exchange instructions (writes) when the read the lock state has become free. That leads us to our spinning locally optimization.

### The Spinning Locally Optimization

Let's take a look at our `lock` method with the spinning locally optimization:

```cpp
   // Lock the spinlock
   void lock() {
     // Keep trying until we get the lock
     while (1) {
       // Try and grab the lock
       if (!locked.exchange(true)) return;
 
       // Read the spinlock state
       while (locked.load())
         ;
     }
   }
```

When a thread tries to lock our spinlock it falls into an infinite `while` loop. Inside the that loop, the thread first tried and grab the lock using an atomic exchange. If it gets the lock, the thread returns from the `lock` method. Otherwise, it falls into the second `while` loop where the thread repeatedly reads the state of the spinlock until it becomes free. When the thread reads the lock has once again become free, it breaks out of the second `while` loop, and attempts to get the
lock again.

What we've done is replace our writes while the lock is take with reads. This is good because read-only copies of a cache line exist in the caches of multiple cores at the same time. This means that each of our waiting threads can read a local copy of the spinlock state from their own L1 caches until the thread with the lock writes that the lock is free. This will invalidate the all copies of the cache line, and the waiting threads will then read the update that the spinlock has become free.

### Low-Level Assembly

Let's take a look at how our low-level assembly for our benchmark looks with this new optimization:

```assembly
  0.25 │20:┌─→mov     %edi,%ecx       
 36.12 │   │  xchg    %cl,(%rax)      
  0.03 │   │  test    %cl,%cl         
  0.06 │   │↓ je      40              
  0.28 │   │  nop                     
 35.84 │30:│  movzbl  (%rax),%ecx     
  0.10 │   │  test    %cl,%cl         
  5.87 │   │↑ jne     30              
  1.10 │   │↑ jmp     20              
       │   │  nop                     
  4.78 │40:│  incq    (%rsi)         
 15.57 │   │  xchg    %cl,(%rax)    
  0.00 │   ├──dec     %edx           
       │   └──jne     20             
```

The first thing our code does is try and grab the lock using an atomic exchange (`xchg`). If we get the lock, we jump down the increment of our shared value `val` (`incq`), decrement our loop counter, and jump back to the top of the loop if our loop is not done.

Where things differ from our original assembly is when we fail to get the lock. Instead of immediately retrying the atomic exchange, we fall into a tight loop where we read the state of the spinlock (`movzbl`), test if the spinlock is now free, then either jump up to the atomic exchange if the lock is free, or re-read the spinlock state if the lock is still take.

### Performance

Here are the end-to-end performance numbers for 1, 2, 4, and 8 threads:

```txt
-------------------------------------------------------------------
Benchmark                         Time             CPU   Iterations
-------------------------------------------------------------------
spin_locally/1/real_time       1.01 ms        0.046 ms          698
spin_locally/2/real_time       10.4 ms        0.070 ms           68
spin_locally/4/real_time       28.9 ms        0.123 ms           19
spin_locally/8/real_time       91.4 ms        0.148 ms            8
```

A very large improvement over our `naive` spinlock! We've improved from 14.2ms to 10.4ms for 2 threads, 66.8ms to 28.9ms for 4 threads, and 247ms to 91.4ms for 8 threads. Let's take a look at our L1 cache miss-rate for the 8 thread case to see if we helped reduce the number of coherence cache misses:

```txt
1,496,253,736      L1-dcache-loads           #  253.267 M/sec                  
   70,272,048      L1-dcache-load-misses     #    4.70% of all L1-dcache hits  
```

Down from ~50% to ~5%! A huge improvement. While this is a great improvement, we can still do better. While we removed the constant contention for the cache line, we have replaced it with bursty contention (when a thread releases the lock). We can look at relieving this with an optimization called backoff.

## A Spinlock with Active Backoff

Before we jump into our active backoff optimization implementation, let's try and understand where we have bursty contention in our spinlock implementation.

### Bursty Contention

Our bursty contention stems from when the thread currently holding the lock releases the lock. Let's take another look at our lock method to understand this:

```cpp
   // Lock the spinlock
   void lock() {
     // Keep trying until we get the lock
     while (1) {
       // Try and grab the lock
       if (!locked.exchange(true)) return;
 
       // Read the spinlock state
       while (locked.load())
         ;
     }
   }
```
 
When the lock is released, all threads spinning in the second `while` loop (our locally spinning optimization) read that the lock is released, and race to the atomic exchange to try and grab the lock. However, only one thread out of the potentially many that try for the lock will get the lock. All the other threads will then wastefully still perform their atomic exchange, fail to get the lock, and go back to reading the lock state, waiting for it to become free again.

To solve this, we need to limit the number of threads that can break out of our waiting loop when the lock is freed. We can do this by adding backoff to this loop.

### The Active Backoff Optimization

To keep our threads from immediately breaking out of the second `while` loop, we will be adding a small delay between each read of the spinlock state. Here is our modified `lock` method:

```cpp
   // Locking mechanism
   void lock() {
     while (1) {
       // Try and grab the lock
       if (!locked.exchange(true)) return;
 
       // Wait for the lock to be free
       do {
         // Pause for some number of iterations
         for (volatile int i = 0; i < 150; i += 1)
           ;
         // Read the lock state
       } while (locked.load());
     }
   }
```

Our `lock` method is almost identical to our spinning locally implemention, with the exception of our loop where we read the state of the lock until it is free. Instead of reading the lock state as fast as possible, we'll insert a configurable delay using a dummy `for` loop.

But why insert a delay here? If threads read the state of the lock as fast as possible, it's likely that almost all the threads all break out at and compete for the lock at the same time. However, if we add a delay in this loop, we increase the probability that some threads will get caught up executing the delay instructions, while others (but not _all_ the threads) break out at try and get the lock. 

### Low-Level Assembly

Let's take a look at how our low-level assembly for our benchmark looks with this new optimization:

```assembly
  0.42 │20:┌─→mov     %edi,%ecx              
 17.29 │   │  xchg    %cl,(%rax)             
  0.01 │   │  test    %cl,%cl                
  0.01 │   │↓ je      67                     
  0.09 │   │  nop                            
  0.35 │30:│  movl    $0x0,-0x4(%rsp)        
  0.07 │   │  mov     -0x4(%rsp),%ecx        
  0.02 │   │  cmp     $0x95,%ecx             
       │   │↓ jg      5e                     
  0.06 │   │  nop                            
  1.77 │48:│  mov     -0x4(%rsp),%ecx        
  1.44 │   │  inc     %ecx                   
 15.57 │   │  mov     %ecx,-0x4(%rsp)        
 21.50 │   │  mov     -0x4(%rsp),%ecx        
  0.07 │   │  cmp     $0x95,%ecx             
  8.49 │   │↑ jle     48                     
 13.96 │5e:│  movzbl  (%rax),%ecx            
  0.04 │   │  test    %cl,%cl                
  0.02 │   │↑ jne     30                     
  0.35 │   │↑ jmp     20                     
  2.54 │67:│  incq    (%rsi)                 
 15.93 │   │  xchg    %cl,(%rax)             
       │   ├──dec     %edx                   
  0.00 │   └──jne     20                     
```

Our assembly is very similar to our spinning locally implementation, with the exception of the extra instructions for our delay `for` loop. This loop is set up at label `30:`, and begins at label `48:`.

### Performance

Here are the end-to-end performance numbers for 1, 2, 4, and 8 threads:

```
---------------------------------------------------------------------
Benchmark                           Time             CPU   Iterations
---------------------------------------------------------------------
active_backoff/1/real_time       1.05 ms        0.054 ms          679
active_backoff/2/real_time       2.60 ms        0.081 ms          281
active_backoff/4/real_time       9.49 ms        0.121 ms           75
active_backoff/8/real_time       50.5 ms        0.287 ms           14
```

Another huge improvement! Compared to our spinlock with only the locally spinning optimization, our end-to-end execution time went down from 10.4ms to 2.60ms for 2 threads, 28.9ms to 9.49ms for 4 threads, and 91.4ms to 50.5ms for 8 threads. While this is a great improvement in performance, we're wasting power executing all the instructions for the `for` loop that generates our delay. We can look at addressing this using a passive form of backoff.

## A Spinlock with Passive Backoff

While everyone cares about performance (to some degree), an increasing number of people also care about power-consumption (especially in massive data-centers). One way we can intuitively save on power is by improving performance so that our application finishes faster (the processor is active for a shorter amount of time). However, there are ways we can improve power-consumption for applications that run for (approximately) the same length of time.

For spin-wait loops like our spinlock, we can use the `_mm_pause()` intrinsic for this exact purpose.

### The Pause Intrinsic

[Here](https://software.intel.com/sites/landingpage/IntrinsicsGuide/#text=_mm_pause&expand=4141) is the `_mm_pause` intrinsic from the intel Intrinsics Guide. If we look at the description, it says:

```txt
Provide a hint to the processor that the code sequence is a spin-wait loop. This can help improve the performance and power consumption of spin-wait loops.
```

With our original active backoff implementation, we got a performance benefit from creating a finite pause within our loop that polls (reads) the state of our spinlock. However, we got this at the cost of thousands, millions, or billions of extra instructions (depending on how long our application runs) executed for our dummy `for` loop.

We can use the `_mm_pause()` intrinsic to get this to the same effect (generating a delay between reads of our spinlock state), but without executing a huge number of extra instructions.

### The Passive Backoff Optimization

Let's take a look at our spinlock's `lock` method but with passive instead of active backoff. For this, we will be swapping out our `for` loop with calls to the `_mm_pause()` intrinsic. Here's how that may look:

```cpp
   // Locking mechanism
   void lock() {
     while (1) {
       // Try and grab the lock
       if (!locked.exchange(true)) return;
 
       do {
         // Pause for some number of iterations
         for (int i = 0; i < 4; i++) _mm_pause();
 
         // Check to see if the lock is free
       } while (locked.load());
     }
   }
```

The core functionality is exactly the same as our active backoff implementation, with the sole exception of how we generate our delays. Just like we could configure the number of iterations of our for loop to control our delay in the previous implementation, we can tune how many times we use the pause intrinsic as well.

### Low-Level Assembly

Let's take a look at how our low-level assembly for our benchmark looks with this new optimization:

```assembly
  0.34 │20:┌─→mov     %edi,%ecx                  
 17.32 │   │  xchg    %cl,(%rax)                 
  0.01 │   │  test    %cl,%cl                    
  0.01 │   │↓ je      41                         
  0.09 │   │  nop                                
 12.61 │30:│  pause                              
 12.39 │   │  pause                              
 12.32 │   │  pause                              
 12.06 │   │  pause                              
 15.37 │   │  movzbl  (%rax),%ecx                
  0.03 │   │  test    %cl,%cl                    
  0.05 │   │↑ jne     30                         
  0.42 │   │↑ jmp     20                         
  2.75 │41:│  incq    (%rsi)                     
 14.23 │   │  xchg    %cl,(%rax)                 
       │   ├──dec     %edx                       
  0.00 │   └──jne     20                        
```

No major structural changes compare to our previous implementation. We start by trying to get the lock with an atomic exchange (`xchg`). If we get the lock, we increment the shared value (`incq`), free the lock with another atomic exchange (`xchg`), decrement the loop counter (`dec`), and return to the top of the loop if there are more iterations.

If we didn't get the look, we go to our spinning locally `do while` loop with passive backoff starting at label `30:`. There, we see 4 `pause` instructions (coming from our calls to the `_mm_pause()` intrinsic), followed by our read of the spinlock state that decides whether we pause again, or try and grab the lock.

### Performance

Here are the end-to-end performance numbers for 1, 2, 4, and 8 threads:

```txt
----------------------------------------------------------------------
Benchmark                            Time             CPU   Iterations
----------------------------------------------------------------------
passive_backoff/1/real_time       1.03 ms        0.054 ms          676
passive_backoff/2/real_time       2.48 ms        0.082 ms          276
passive_backoff/4/real_time       9.59 ms        0.110 ms           70
passive_backoff/8/real_time       46.7 ms        0.236 ms           15
```

I've roughly matched the performance of our active backoff and passive backoff benchmarks. However, you may see slight differences. This is to be expected because thread scheduling is non-deterministic, and is heavily influenced by the current load of the CPUs. Therefore, we'll deem the values close enough.

### Power

To reason about the power-consumption of our applications, we'll be looking at the total number of instructions executed to compare the amount of work being done, then directly query power performance counters.

#### Instructions Executed

Let's start our power-consumption analysis by comparing the total number of instructions executed. Let's compare the 8-thread benchmark case for our active and passive passive backlock implementations with the number of benchmark iterations set to `50`.

Here are the number instructions executed for our active backoff implementation:

```txt
55,227,656,500      instructions              #    0.76  insn per cycle
```

Around 55 billion. Because thread scheduling is non-deterministic, this number varied from run to run, but generally fell between the 45-60 billion instruction range.

Now let's see how many instructions were executed for our passive backoff implementation:

```txt
1,012,803,760      instructions              #    0.01  insn per cycle         
```

A 55x reduction in instructions! But this should make sense. In our active backoff implementation, we're executing billions of extra instructions just to induce the delay inside of our locally spinning loop. In our passive backoff implementation, we're using a dedicated instruction that adds a finite delay to our pipeline, allowing us to simply stop doing work for short periods of time.

Intuitively, we can reason that our passive backoff implementation that executes 55x fewer instructions and with roughly equivilant end-to-end execution time as our active backoff should consume less power while running.

##### Additional Statistics

One thing you may notice if you look at our other performance counter stats for our passive backoff benchmark is that we have a very high branch miss-prediction and L1 data cache miss rate:

```txt
216,996,864      branches                  #   12.154 M/sec                  
 35,495,595      branch-misses             #   16.36% of all branches        
240,163,142      L1-dcache-loads           #   13.452 M/sec                  
147,908,665      L1-dcache-load-misses     #   61.59% of all L1-dcache hits  
```

What's going on? Let's think about what we know about our benchmarks and spinlock implementations. In both benchmarks, we have a heavy contention scenario where threads fight over the cache line(s) containing our spinlock state and shared value `val`. Furthermore, when a thread actually sees the lock as free or successfully gets the lock is non-deterministic, and could be difficult for our branch predictor to guess correctly. 

These things explain our observations in passive backoff, but why do these not look like a problem in our active backoff implementation?

```txt
 7,871,605,800      branches                  #  576.093 M/sec                  
    76,998,863      branch-misses             #    0.98% of all branches        
15,543,237,429      L1-dcache-loads           # 1137.551 M/sec                  
   106,735,606      L1-dcache-load-misses     #    0.69% of all L1-dcache hits  
```

Just look at the raw counter values! We're only doing ~217 million branches in our passive backoff benchmark, but ~7.9 billion in our active backoff benchmark. Likewise, our L1 data cache load increased from ~240 million to ~15.5 billion! Most of these loads and branches are from the dummy `for` loop we added to insert a delay. These are hiding the underlying randomness and lack of cache-locality that is fundamental to our application.

#### Power Performance Counters


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

