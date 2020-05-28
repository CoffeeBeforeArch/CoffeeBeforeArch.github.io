---
layout: default
title: Thread Affinity
---

# Thread Affinity

When you write a multi-threaded application, there are a number of performance pitfalls to watch out for. If you partition your data poorly, you may suffer from false sharing. Likewise, if you don't fairly partition your work, you may end up with some threads overloaded with work, and others that are mostly idle. Another thing to consider is where threads are placed in relation to each other.

Placing threads that share data close to each other may allow you to exploit inter-thread locality. Likewise, placing threads that share data far apart may result in contention, especially if the threads are scheduled to different NUMA nodes. Unfortunately your operating system is mostly-oblivious to the needs of your application with respect to thread scheduling. As a result, it often makes sub-optimal decisions (if not given guidance). In this blog post, we'll be taking a look at exactly how we can give some scheduling guidance to our O.S. using Pthreads.

### Link to the source code
- [Source Code: ](https://github.com/CoffeeBeforeArch/misc_code/tree/master/thread_affinity)
- [My YouTube Channel: ](https://www.youtube.com/channel/UCsi5-meDM5Q5NE93n_Ya7GA?view_as=subscriber)
- [My GitHub Account: ](https://github.com/CoffeeBeforeArch)
- My Email: CoffeeBeforeArch@gmail.com

## Compiling and Running the Benchmarks

The benchmarks from this blog post were written using [Google Benchmark](https://github.com/google/benchmark). The following is the command I used to compile the benchmarks (on Ubuntu 18.04 using g++10).

```bash
g++ thread_affinity.cpp -lbenchmark -lpthread -O3 -march=native -mtune=native -o thread_affinity
```

## Relying on the O.S.

Let's craft a benchmark with pairs of threads fighting over shared pieces of data. The first thing we need to do is to have our threads compete over something. Let's use an atomic integer that we increment 100k times. Here's what that might look like.

```cpp
void work(std::atomic<int>& a) {
  for(int i = 0; i < 100000; ++i)
    a++;
}
```

Now we need some threads to do the fighting. Here's how the rest of my main benchmark loop was implemented.

```cpp
struct alignas(64) AlignedAtomic {
  AlignedAtomic(int v) { val = v; }
  std::atomic<int> val;
};

void os_scheduler() {
  AlignedAtomic a{0};
  AlignedAtomic b{0};

  // Create four threads and use lambda to launch work
  std::thread t0([&]() { work(a.val); });
  std::thread t1([&]() { work(a.val); });
  std::thread t2([&]() { work(b.val); });
  std::thread t3([&]() { work(b.val); });

  // Join the threads
  t0.join();
  t1.join();
  t2.join();
  t3.join();
}
```

Threads t0 and t1 compete for AlignedAtomic 'a', while threads t2 and t3 compete for AlignedAtomic 'b'. AlignedAtomic is just an atomic int wrapped in a struct which is aligned to 64B. This keeps our atomic ints from winding up on the same cache line (avoiding any false sharing).  

For this experiment, we're expecting heavy contention on two cache lines (the ones that have our AlignedAtomics). Let's take a look at both the execution time, and Shared Data Cache Line Table from `perf-c2c`. Here is the execution time (we'll focus on the wall clock time reported in the 'Time' column).

```
-----------------------------------------------------------------
Benchmark                       Time             CPU   Iterations
-----------------------------------------------------------------
osScheduling/real_time       5.09 ms        0.080 ms          100
```

Ok, some baseline numbers. Now let's take a look at some stats using `perf c2c record` and `perf c2c report`. Here's a truncated view of the Shared Data Cache Line Table.

```
Shared Data Cache Line Table     (2 entries, sorted on Total HITMs)
                             Total      Tot  ----- LLC Load Hitm -----
Index           Cacheline  records     Hitm    Total      Lcl      Rmt
    0      0x7fff1dede640     3242   53.67%     1630     1630        0     
    1      0x7fff1dede600     2703   46.23%     1404     1404        0 
```

Just as we expected, two cacheline under heavy contention. As a reminder, Hitm events are being used as a measure of contention. Hitm events occur when a thread loads data and finds that the cache line it's looking for is in the modified state in a different cache. Having a lot of these events indicate that 2 or more threads are fighting for exclusive access to a cache line so they can write to it. Let's zoom in on cache line `0x7fff1dede640` for some more information.

```
Cacheline 0x7fff1dede640
----- HITM -----                cpu
    Rmt      Lcl       Off      cnt
  0.00%   59.20%       0x0        4  (Thread t3)
  0.00%   40.80%       0x0        4  (Thread t4)
```

Two threads accessing the same part of the same cache line (AlignedAtomic 'b' in this case). What we should focus on is the `cpu cnt` column. This is the number of physical cores that samples came from (all 4 in my 4 core/8 thread CPU). This means that both of our threads hopped around to all 4 cores in our system during execution.

When the O.S. schedules a thread to a core, it doesn't just leave it there until it finishes. It may move the thread around if the scheduler thinks it is advantageous to do so (depending on the current load of the CPU). This means our performance may vary (and significantly). Let's compare the timing for a few runs.

```
-----------------------------------------------------------------
Benchmark                       Time             CPU   Iterations
-----------------------------------------------------------------
osScheduling/real_time       6.08 ms        0.149 ms          133
osScheduling/real_time       5.16 ms        0.140 ms          100
osScheduling/real_time       5.45 ms        0.078 ms          100
osScheduling/real_time       4.77 ms        0.074 ms          145
```

A wide range of results! Even in the better cases of thread scheduling (lower execution time), let's look at the the L1 data cache hit rate.

```
69,543,035    L1-dcache-loads          #   27.243 M/sec                  
35,919,314    L1-dcache-load-misses    #   51.65% of all L1-dcache hits  
```

We're missing almost half the time! Let's see if we can help our O.S. with some scheduling guidance.

## Setting Thread Affinity

Every major O.S. supports the ability to set thread affinity (i.e., inform the O.S. which physical cores a thread should be scheduled to). However, we can't do this solely with C++11's std::thread. We'll need to use the dedicated functions provided by the library used to implement std::thread. On my Linux machine, it's Pthreads. Luckily for us, we don't have to abandon C++11 threads entirely. We can use the std::thread method `native_handle()` which returns (in this case) a handle to our underlying pthread.

Let's re-write our benchmark's main loop using `pthread_setaffinity_np` to pin each pair of threads to a single core. Here's how my implementation looked.

```cpp
#include <pthread.h>

void thread_affinity() {
  AlignedAtomic a{0};
  AlignedAtomic b{0};

  // Create cpu sets for threads 0,1 and 2,3
  cpu_set_t cpu_set_1;
  cpu_set_t cpu_set_2;

  // Zero them out
  CPU_ZERO(&cpu_set_1);
  CPU_ZERO(&cpu_set_2);

  // And set the CPU cores we want to pin the threads too
  CPU_SET(0, &cpu_set_1);
  CPU_SET(1, &cpu_set_2);

  // Create thread 0 and 1, and pin them to core 0
  std::thread t0([&]() { work(a.val); });
  assert(pthread_setaffinity_np(t0.native_handle(), sizeof(cpu_set_t),
                                &cpu_set_1) == 0);
  std::thread t1([&]() { work(a.val); });
  assert(pthread_setaffinity_np(t1.native_handle(), sizeof(cpu_set_t),
                                &cpu_set_1) == 0);
 
  // Create thread 1 and 2, and pin them to core 1
  std::thread t2([&]() { work(b.val); });
  assert(pthread_setaffinity_np(t2.native_handle(), sizeof(cpu_set_t),
                                &cpu_set_2) == 0);
  std::thread t3([&]() { work(b.val); });
  assert(pthread_setaffinity_np(t3.native_handle(), sizeof(cpu_set_t),
                                &cpu_set_2) == 0);

  // Join the threads
  t0.join();
  t1.join();
  t2.join();
  t3.join();
}
```

Let's pick apart the new parts of our function. The first new addition `cpu_set_t`. This is used to specify the set of CPUs (physical cores, not threads) that a thread can be scheduled to. With `CPU_ZERO`, we can initialize our `cpu_set_t` to zero. We then can use `CPU_SET` to set which physical cores will be valid scheduling options for our threads. We can then call `pthread_set_affinity_np` with our native thread handle (a handle to a pthread), the size of the `cpu_set_t`, and the `cpu_set_t` itself to set the thread's affinity. [Here](https://www.man7.org/linux/man-pages/man3/pthread_setaffinity_np.3.html) is a link where you find more information on `pthread_set_affinity_np`.

For our purposes, we'll pin threads t0 and t1 to physical core 0, and threads t2 and t3 to physical core 1. This means that our pairs of threads can share the same copy of the cache line containing their respective AlignedAtomics instead of stealing it away from each other. Let's check our execution time.

```
-------------------------------------------------------------------
Benchmark                         Time             CPU   Iterations
-------------------------------------------------------------------
threadAffinity/real_time       1.32 ms        0.074 ms          536
```

Over 4x faster (on average) than when we just relied on the O.S. to do its best! Furthermore, this result far more stable because we're no longer relying on non-deterministic thread scheduling by the O.S. Let's take a look at the L1 hit rate.

```
281,131,290    L1-dcache-loads          #    177.685 M/sec
  7,969,745    L1-dcache-load-misses    #    2.83% of all L1-dcache hits

```

Very few L1 misses! This is because our pairs of threads are no longer invalidating each other. In my case, the cache lines containing the AlignedAtomics disappeared completely from the Shared Data Cache Line table from perf c2c! All that was left was ~30 cache lines with very few Hitm events (mainly from the kernel, so I've omitted that data from the post).

## Concluding remarks

You can't always escape sharing in multi-threaded applications. However, you do have some control in minimizing the performance impact when it occurs. Placing memory and threads is extremely important, especially when working with threads scattered across NUMA nodes. 

Thanks for reading,

--Nick

### Link to the source code

- [Source Code: ](https://github.com/CoffeeBeforeArch/misc_code/tree/master/thread_affinity)
- [My YouTube Channel: ](https://www.youtube.com/channel/UCsi5-meDM5Q5NE93n_Ya7GA?view_as=subscriber)
- [My GitHub Account: ](https://github.com/CoffeeBeforeArch)
- My Email: CoffeeBeforeArch@gmail.com


