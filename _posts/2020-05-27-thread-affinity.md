---
layout: default
title: Thread Affinity
---

# Thread Affinity

When you write a multi-threaded application, there are a number of performance pitfalls to watch out for. If you partition your data poorly, you may suffer from false sharing. Likewise, if you don't fairly partition your work, you may end up with a mixture of threads overloaded with work, while other are mostly idle. Another thing to consider is where threads are placed in relation to each other.

Placing threads that share data close to each other may allow you to exploit inter-thread locality. Likewise, placing threads that share data far apart may result in contention, especially if threads are scheduled to different NUMA nodes. Unfortunately your operating system is mostly-oblivious to the needs of your application with respect to thread scheduling, and as a result, often makes sub-optimal decisions (if not given guidance). In this blog post, we'll be taking a look at exactly how we can give this scheduling guidance to our O.S. using Pthreads.

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

Let's craft a benchmark with pairs of threads fighting over shared pieces of data. The first thing we need to do is to have our threads compete over something. Let's use an atomic integer that we increment 100k times. Here's what that looks like.

```cpp
void work(std::atomic<int>& a) {
  for(int i = 0; i < 100000; ++i)
    a++;
}
```

Now we need some threads to do the fighting. Here's how the rest of the main benchmark loop was implemented.

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

Threads t0 and t1 compete for AlignedAtomic 'a', while threads t2 and t3 compete for AlignedAtomic 'b'. AlignedAtomic is just an atomic int wrapped in a struct which is aligned to 64B. This keeps our atomic ints from winding up on the same cache line.  

For this experiment, we're expecting heavy contention on two cache lines (the ones that have our AlignedAtomics. Let's take a look at both the execution time, and Shared Data Cache Line Table from `perf-c2c`. Here is the execution time (we'll focus on the wall clock time reported in the 'Time' column).

```
-----------------------------------------------------------------
Benchmark                       Time             CPU   Iterations
-----------------------------------------------------------------
osScheduling/real_time       5.09 ms        0.080 ms          100
```

Ok, some baseline timing, now let's take a look at some stats using `perf c2c record` and `perf c2c report`. Here's a truncated view of the Shared Data Cache Line Table.

```
Shared Data Cache Line Table     (2 entries, sorted on Total HITMs)
                             Total      Tot  ----- LLC Load Hitm -----
Index           Cacheline  records     Hitm    Total      Lcl      Rmt
    0      0x7fff1dede640     3242   53.67%     1630     1630        0     
    1      0x7fff1dede600     2703   46.23%     1404     1404        0 
```

Just as we expected, two cacheline under heavy contention. As a reminder, Hitm events are being used as a measure of contention. Hitm events occur when a thread loads data and finds the cache line it's looking for is in the modified state in a different cache. Many of these events mean 2 or more threads are fighting for exclusive access to a cache line so they can write to it. Let's zoom in on cache line `0x7fff1dede640` for some more information.

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

A wide range of results. Let's see if we can help our O.S. with some scheduling guidance.

## Using Perf C2C

Last time we showed that false sharing cause a ~3x slowdown compared to a serial implementation, with a ~40% miss-rate in the L1 cache. Let's see what information we can get with Perf C2C. Let's run the the application using the following:

```bash
perf c2c record ./false_sharing --benchmark_filter=falseSharing/real_time
```

![post run](/assets/perf_c2c/post_run.png)

Now we can load the data with perf. By default we have to options to look at; loads or stores.

```bash
perf report
```

![events](/assets/perf_c2c/events.png)

Let's look at the samples for loads. Unsurprisingly, we find that most of the samples are associated with the threads launched by our false sharing benchmark. The long names of the symbols come from the fact we are launching new threads using a lambda.

![loads lambdas](/assets/perf_c2c/loads_lambdas.png)

We find that the load and store samples are all associated with the atomic increment. Let's keep track of something from this perf window. Let's keep track of where this atomic increment is located in memory. We find the base address of the lamda is at `0x407cd0`, and the `incl` is at an offset by `0x10`. We'll therefore focus on `0x407ce0` (the sum of the base address and the offset).

![lambda 4 hotspot](/assets/perf_c2c/lambda_4_hotspot.png)

Now let's exit out, and load in the data with c2c. We get a "Shared Data Cache Line Table" with a single entry (the cache line that holds all 4 of the atmoic integers). Notice the high count of "Hitm". As a reminder, these are cases where a thread misses in its L1 cache, and finds the line in a modified state in a different L1 cache. This indicates that multiple threads are writing to the same cache line, and sending invalidations to each other.

```bash
perf c2c report
```

![perf c2c report](/assets/perf_c2c/perf_c2c_report.png)

We can get more information about a cache line by pressing `d` while it is selected in the table. When we do this, we find that cache line is being written at 4 places (our 4 threads performing atomic increments). One thing to observe here is the code address. Notice how one of the entries is `0x407ce0`. This was the address of the `incl` instruction we looked at earlier! The other code addresses correspond to the `incl` instruction being executed on the other 3 threads.

![cacheline view](/assets/perf_c2c/cacheline_view.png)

## Compilation in Debug Mode

If you compiled your application with the `-g` flag (instead of `-O3`), you may see a different hotspot in your application. Now, it looks like our hotspot is at `atomic_base.h`, a file that we did not write, or directly include.

![debug mode](/assets/perf_c2c/debug_mode.png)

However, if we look up what is actually at line 548 of `atomic_base.h`, we find that it is just the intrinsic for performing an atomic increment.

![fetch and add](/assets/perf_c2c/fetch_and_add.png)

In this, and many other cases, having a call graph can help you get the necessary context to debug your program. If we take the same code, but record with the `-g` flag, we can trace back how we got to `atomic_base.h`. We can load in our data to perf the same way we did previously.

```bash
perf c2c record -g ./false_sharing --benchmark_filter=falseSharing/real_time
```

```bash
perf c2c report
```

![fetch and add](/assets/perf_c2c/call_graph.png)

Now, we can see that our our atomic increment comes from `atomic_base.h`, which is called from the work function, which is running in our 4 threads launched with lambdas.

## Concluding remarks

This is by no means the only way to debug false sharing in an application. However, this should get you going in the right direction, and introduce you to some of the amazing tools out there for debugging performance problems in your code.

Thanks for reading,

--Nick

### Link to the source code

- [Source Code](https://github.com/CoffeeBeforeArch/spring_2020_tutorial/tree/master/false_sharing)

