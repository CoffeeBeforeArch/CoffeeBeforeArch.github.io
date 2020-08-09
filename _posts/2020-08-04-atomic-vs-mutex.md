---
layout: default
title: Mutex vs Atomic
---

# Mutex vs Atomic

Some parallel applications do not have any data sharing between threads. Others have sharing between threads that is read-only. But some applications are more complex, and require the coordiation of multiple threads that want to write to the same memory. In these situations, we can reach into our parallel programming toolbox for something like `std::mutex` to make sure only a single thread enters a critical section at a time. However, a mutex can be significantly more overhead than we actually need. In this blog post, we'll be comparing the performance between using a mutex and directly using atomic operations in a few benchmarks.

### Link to the source code
- [Source Code: ](https://github.com/CoffeeBeforeArch/misc_code/tree/master/inc_bench)
- [My YouTube Channel: ](https://www.youtube.com/channel/UCsi5-meDM5Q5NE93n_Ya7GA?view_as=subscriber)
- [My GitHub Account: ](https://github.com/CoffeeBeforeArch)
- My Email: CoffeeBeforeArch@gmail.com

## Compiling and Running the Benchmarks

The benchmarks from this blog post were written using [Google Benchmark](https://github.com/google/benchmark). Below is the command I used to compile the simple increment benchmarks (on Ubuntu 18.04 using g++10).

```bash
g++ inc_bench.cpp -lbenchmark -lpthread -O3 -march=native -mtune=native -flto -fuse-linker-plugin -o inc_bench
```

The matrix multiplication benchmarks can be built using the existing makefile in the repository.

## Parallel Increment - The Bad Way

Consider the following simple application:

```cpp
 int main() {
   // Shared value for our threads
   int shared_val = 0;
 
   // Number of iterations (65536)
   int N = 1 << 16;
 
   // Lambda that performs an increment
   auto inc_func = [&]() {
     for (auto i = 0; i < N; i++) shared_val++;
   };
 
   // Create two threads
   std::thread t1(inc_func);
   std::thread t2(inc_func);
 
   // Join the threads
   t1.join();
   t2.join();
 
   // Print the result
   std::cout << "FINAL VALUE IS: " << shared_val << '\n';
 
   return 0;
 }
```

What is the value of `shared_val` when it is printed to the screen? Our expected result is 2^16 + 2^16 (2^17) because that's how many increments we perform. However, the real answer is far less straight-forward. The real answer is that we won't know for certain until runtime because we have a race condition where both of our threads are trying to write to the same location at the same time.

Each of our increments to `shared_val` really consists of three operations: a read of `shared_val`, an increment of `shared_val`, and a write to `shared_val`. When our two threads try and perform these operations at the same time, they can interleave in nasty ways that give us unexpected results. Consider a simple example where our threads take turns updating `shared_val`.

First, thread 1 reads `shared_val`, then increments it, then writes the value back to memory. Sometime later, thread 2 performs the same 3 operations (uninterrupted), and this goes back and forth until each thread increments `shared_val` 2^16 times. Sounds great! We would expect the final value to be 2^16 + 2^16 (2^17). Now let's consider another scenrio where the threads do not politely take turns.

First, thread 1 reads `shared_val`. Then, thread 2 reads `shared val`. Next, thread 1 increments the value it read, followed by thread 2 incrementing the value it read. Finally, both threads write the value they incremented to memory, and this set of steps repeat until each thread performs 2^16 increments. What is the value now? 2^16, not 2^17! Let's substitute in some values to make this make this more clear.

Let's start with `shared_val = 0`. First, both threads read `shared_val`, and find a value of `0`. Next both threads increment this value to `1`. Finally, both threads write the value `1` to memory. Each thread performed an increment, but the final value of `shared_val` doesn't reflect this because of the interleaving!

Here are a few print outs from when I ran the program:

```
FINAL VALUE IS: 65536
FINAL VALUE IS: 131072
FINAL VALUE IS: 85026
```

In the first run, it looks like `shared_val` was incremented half as many times as it should have been. In the second case, we got the expected result (2^17). In the final run, we got something in-between. We clearly need some way to restrict which orderings are allowed.

### Additional Notes

#### Detecting Race Conditions

One way we can detect some race conditions is using Google's thread sanitizer (by adding `-fsanitize=thread` to our compile flags). Here is a sample output from running bad increment program with thread sanitizer enabled:

```
==================
WARNING: ThreadSanitizer: data race (pid=31049)
  Read of size 4 at 0x7fffb18f4d28 by thread T2:
    #0 std::thread::_State_impl<std::thread::_Invoker<std::tuple<main::{lambda()#1}> > >::_M_run() <null> (a.out+0x401379)
    #1 execute_native_thread_routine ../../../../../libstdc++-v3/src/c++11/thread.cc:80 (libstdc++.so.6+0xd62af)

  Previous write of size 4 at 0x7fffb18f4d28 by thread T1:
    #0 std::thread::_State_impl<std::thread::_Invoker<std::tuple<main::{lambda()#1}> > >::_M_run() <null> (a.out+0x40138f)
    #1 execute_native_thread_routine ../../../../../libstdc++-v3/src/c++11/thread.cc:80 (libstdc++.so.6+0xd62af)

  Location is stack of main thread.

  Location is global '<null>' at 0x000000000000 ([stack]+0x00000001ed28)

  Thread T2 (tid=31052, running) created by main thread at:
    #0 pthread_create ../../../../libsanitizer/tsan/tsan_interceptors_posix.cpp:962 (libtsan.so.0+0x5be22)
    #1 __gthread_create /home/cba/forked_repos/gcc/build/x86_64-pc-linux-gnu/libstdc++-v3/include/x86_64-pc-linux-gnu/bits/gthr-default.h:663 (libstdc++.so.6+0xd6524)
    #2 std::thread::_M_start_thread(std::unique_ptr<std::thread::_State, std::default_delete<std::thread::_State> >, void (*)()) ../../../../../libstdc++-v3/src/c++11/thread.cc:135 (libstdc++.so.6+0xd6524)
    #3 main <null> (a.out+0x401136)

  Thread T1 (tid=31051, finished) created by main thread at:
    #0 pthread_create ../../../../libsanitizer/tsan/tsan_interceptors_posix.cpp:962 (libtsan.so.0+0x5be22)
    #1 __gthread_create /home/cba/forked_repos/gcc/build/x86_64-pc-linux-gnu/libstdc++-v3/include/x86_64-pc-linux-gnu/bits/gthr-default.h:663 (libstdc++.so.6+0xd6524)
    #2 std::thread::_M_start_thread(std::unique_ptr<std::thread::_State, std::default_delete<std::thread::_State> >, void (*)()) ../../../../../libstdc++-v3/src/c++11/thread.cc:135 (libstdc++.so.6+0xd6524)
    #3 main <null> (a.out+0x401127)

SUMMARY: ThreadSanitizer: data race (/home/cba/forked_repos/misc_code/inc_bench/a.out+0x401379) in std::thread::_State_impl<std::thread::_Invoker<std::tuple<main::{lambda()#1}> > >::_M_run()
==================
FINAL VALUE IS: 131072
ThreadSanitizer: reported 1 warnings
```

#### Race Conditions In a Single Instruction

You may have wondered how there can be any interleaving when our increment in C++ gets translated to a single x86 instruction (like `add` or `inc`). This is beause x86 processors don't actually execute x86 instructions (at least not directly). These instructions get translated into micro-operations (uops), and the uops can be interleaved between threads.

Check out [uops.info](https://www.uops.info/) for more information.

## Parallel Increment - Mutex and Atomic

In this section, we'll discuss how to avoid the race condition in our parallel increment example using mutexes and atomics.

### Parallel Increment - Mutex

The first way we can avoid our race condition is using a mutex (`std::mutex` in this example). Below is the definition of `std::mutex` from [en.cppreference.com](https://en.cppreference.com/w/cpp/thread/mutex):

- The mutex class is a synchronization primitive that can be used to protect shared data from being simultaneously accessed by multiple threads.

Sounds like exactly what we are looking for (a way to serialize access to shared data)! Let's look at an updated function that will be called from multiple threads where we use a `std::mutex` to serialize access to `shared_val`.

```cpp
 // Function to increment using a lock
 void inc_mutex(std::mutex &m, int &shared_val) {
   for (int i = 0; i < (1 << 16); i++) {
     std::lock_guard<std::mutex> g(m);
     shared_val++;
   }
 }
```

Let's step through what's going on here. Each iteration of the loop, we use a `std::lock_guard` to lock our `std::mutex`. If a thread finds the `std::mutex` is already locked, it waits for the lock to be released. If a thread finds the `std::mutex` unlocked, it locks the `std::mutex`, increments `shared_val`, then unlocks the mutex (the `std::lock_guard` object locks the mutex when it is created, and releases it when it is destroyed).

Let's take a looks at the low-level assembly:

```
       │20:┌─→mov   %r12,%rdi
 22.62 │   │→ callq pthread_mutex_lock@plt
  2.51 │   │  test  %eax,%eax
       │   │↓ jne   60
 49.50 │   │  incq  0x0(%rbp)
  0.75 │   │  mov   %r12,%rdi
 11.33 │   │→ callq pthread_mutex_unlock@plt
       │   ├──dec   %ebx
 13.29 │   └──jne   20
```

Pretty much what we described above! We call `pthread_mutex_lock`, use an `incq` to increment the value, then call `pthread_mutex_unlock`. Now let's compare the performance when we scale the number of threads.

```
-----------------------------------------------------------------------
Benchmark                             Time             CPU   Iterations
-----------------------------------------------------------------------
lock_guard_bench/1/real_time       1.30 ms         1.23 ms          543
lock_guard_bench/2/real_time       8.02 ms         7.56 ms           82
lock_guard_bench/3/real_time       8.55 ms         7.41 ms           80
lock_guard_bench/4/real_time       19.4 ms         16.0 ms           36
lock_guard_bench/5/real_time       24.5 ms         20.3 ms           29
lock_guard_bench/6/real_time       32.8 ms         28.3 ms           22
lock_guard_bench/7/real_time       39.4 ms         33.2 ms           18
lock_guard_bench/8/real_time       44.8 ms         38.5 ms           16
```

Unsurprisingly, our performance gets worse as the number of threads increases. That's because with more threads, we have more contention for our lock.

### Parallel Increment - Atomic

As usual in programming, there are many different ways to solve a problem. Using a mutex to solve our parallel increment race condition is actually overkill. What we should be using are atomic operations. Here is overview of the atomic operations library from [en.cppreference.com](https://en.cppreference.com/w/cpp/atomic)

- The atomic library provides components for fine-grained atomic operations allowing for lockless concurrent programming. Each atomic operation is indivisible with regards to any other atomic operation that involves the same object. Atomic objects are free of data races.

Basically, our increment that requires a read, modify, and write of `shared_val` becomes a single indivisible operation. That means there can be no interleaving of operations across threads.

Let's take a look at how we can do this in C++:

```cpp
// Function to incrememnt atomic int
void inc_atomic(std::atomic<int> &shared_val) {
  for (int i = 0; i < (1 << 16); i++) shared_val++;
}
```

Each increment of `shared_val` is now an atomic increment! Let's see how this translates to the low-level assembly.

```
 98.89 │10:   lock   incq (%rdx)
  0.03 │      dec    %eax
  1.08 │    ↑ jne    10
```

A much more streamlined routine. We perform `incq` instructions with the `lock` prefix 2^16 times on each thread. This `lock` prefix ensures that the core performing the instruction has exclusive ownership of the cache line we are writing to for the entire operation. This is how the increment is made into an indivisible unit.

Now let's look at the performance with a varying number of threads, and compare the performance to using a mutex:

```
-----------------------------------------------------------------------
Benchmark                             Time             CPU   Iterations
-----------------------------------------------------------------------
atomic_bench/1/real_time          0.400 ms        0.400 ms         1737
atomic_bench/2/real_time           3.24 ms         3.13 ms          224
atomic_bench/3/real_time           5.34 ms         5.11 ms          100
atomic_bench/4/real_time           7.18 ms         6.95 ms           96
atomic_bench/5/real_time           8.43 ms         6.03 ms           78
atomic_bench/6/real_time           9.78 ms         7.78 ms           72
atomic_bench/7/real_time           11.5 ms         10.4 ms           59
atomic_bench/8/real_time           13.4 ms         11.3 ms           52
```

Way faster than using a mutex (by over 3x in some cases)! But this shouldn't be terribly surprising. When we use a mutex, we're relying on software routines to lock and unlock the mutex. With our atomic operations, we're relying on the underlying hardware mechanisms to make the increment an indivisible operation.

However, it's important to note that what we're profiling here is repeated locks and unlocks for our mutex, and repeated atomic increments for our atomic benchmark. This is largely an artifical scenario, so we should be cautious in thinking cases where we can replace a mutex with an atomic operation will give us a massive speedup.

## Matrix Multiplication - Mutex and Atomics

In this section, we'll look at a few different implementations of matrix multiplication, and compare the difference in performance when elements are statically mapped to threads, or dynamically mapped using mutexes or atomics.

### Statically Mapped Elements

We can parallelize matrix multiplication without without synchronization mechanisms if we have threads work on different parts of the output matrix. Here is an example implementation that performs some cache tiling for even better performance:

```cpp
 // Blocked column parallel implementation w/o atomic
 void blocked_column_multi_output_parallel_gemm(const double *A, const double *B,
                                                double *C, std::size_t N,
                                                std::size_t tid,
                                                std::size_t stride) {
   // For each chunk of columns
   for (std::size_t col_chunk = tid * 16; col_chunk < N; col_chunk += stride)
     // For each chunk of rows
     for (std::size_t row_chunk = 0; row_chunk < N; row_chunk += 16)
       // For each block of elements in this row of this column chunk
       // Solve for 16 elements at a time
       for (std::size_t tile = 0; tile < N; tile += 16)
         // For apply that tile to each row of the row chunk
         for (std::size_t row = 0; row < 16; row++)
           // For each row in the tile
           for (std::size_t tile_row = 0; tile_row < 16; tile_row++)
             // Solve for each element in this tile row
             for (std::size_t idx = 0; idx < 16; idx++)
               C[(row + row_chunk) * N + col_chunk + idx] +=
                   A[(row + row_chunk) * N + tile + tile_row] *
                   B[tile * N + tile_row * N + col_chunk + idx];
 }
```

Each thread process 16 columns of output elements each iteration of the outer-most loop until every element has been processed (we're assuming square matrices that have a dimension `N` that is divisible by 16).

Let's measure the performance when we statically partition the matrix like this for matrices of dimension `2^8 + 16`, `2^9 + 16`, and `2^10 + 16`. We are not directly using a power-of-two dimension to avoid cache assosciativity problems (conflict misses).

Here are the results:

```
-------------------------------------------------------------------------
Benchmark                               Time             CPU   Iterations
-------------------------------------------------------------------------
static_mmul_bench/8/real_time       0.913 ms        0.462 ms         1771
static_mmul_bench/9/real_time        4.01 ms         2.51 ms          340
static_mmul_bench/10/real_time       29.1 ms         23.6 ms           57
```

### Dynamically Mapped Using Mutexes and Atomics

One way we can try to make the performance better is by dynamically mapping elements to threads. The rationale behind this is that threads complete work at different rates due to things like runtime scheduling differences. If work is statically and evenly divided between threads, some threads may finish their work faster than others (due to advantageous thread schedules), and be sitting around waiting for the others to finish. With dynamically mapped work, Threads that finish their work early simply ask for more work instead of sitting idle.

Let's first take a look at an implementation with mutexes:

```cpp
 // Fetch and add routine using a mutex
 uint64_t fetch_and_add(uint64_t &pos, std::mutex &m) {
   std::lock_guard<std::mutex> g(m);
   auto ret_val = pos;
   pos += 16;
   return ret_val;
 }
 
 // Blocked column parallel implementation w/ atomics
 void blocked_column_multi_output_parallel_mutex_gemm(const double *A,
                                                      const double *B, double *C,
                                                      std::size_t N,
                                                      uint64_t &pos,
                                                      std::mutex &m) {
   // For each chunk of columns
   for (std::size_t col_chunk = fetch_and_add(pos, m); col_chunk < N;
        col_chunk = fetch_and_add(pos, m))
     // For each chunk of rows
     for (std::size_t row_chunk = 0; row_chunk < N; row_chunk += 16)
       // For each block of elements in this row of this column chunk
       // Solve for 16 elements at a time
       for (std::size_t tile = 0; tile < N; tile += 16)
         // For apply that tile to each row of the row chunk
         for (std::size_t row = 0; row < 16; row++)
           // For each row in the tile
           for (std::size_t tile_row = 0; tile_row < 16; tile_row++)
             // Solve for each element in this tile row
             for (std::size_t idx = 0; idx < 16; idx++)
               C[(row + row_chunk) * N + col_chunk + idx] +=
                   A[(row + row_chunk) * N + tile + tile_row] *
                   B[tile * N + tile_row * N + col_chunk + idx];
 }
```

In the outer-most loop, each thread gets a chunk of 16 columns to solve from our `fetch_and_add` routine. This routine takes the position (`pos`), saves the current value, increments `pos` by 16, and returns the original value. Access to `pos` is restricted using a `std::mutex` and `std::lock_guard`.

Let's measure the performance:

```
------------------------------------------------------------------------
Benchmark                              Time             CPU   Iterations
------------------------------------------------------------------------
mutex_mmul_bench/8/real_time       0.519 ms        0.474 ms         1345
mutex_mmul_bench/9/real_time        2.80 ms         2.33 ms          242
mutex_mmul_bench/10/real_time       21.3 ms         19.1 ms           33
```

A decent improvement compared to our statically mapped version (~30% improvement). Now let's look at an implementation where we directly atomic `fetch_add` for a `std::atomic`:

```cpp
// Blocked column parallel implementation w/ atomics
void blocked_column_multi_output_parallel_atomic_gemm(
    const double *A, const double *B, double *C, std::size_t N,
    std::atomic<uint64_t> &pos) {
  // For each chunk of columns
  for (std::size_t col_chunk = pos.fetch_add(16); col_chunk < N;
       col_chunk = pos.fetch_add(16))
    // For each chunk of rows
    for (std::size_t row_chunk = 0; row_chunk < N; row_chunk += 16)
      // For each block of elements in this row of this column chunk
      // Solve for 16 elements at a time
      for (std::size_t tile = 0; tile < N; tile += 16)
        // For apply that tile to each row of the row chunk
        for (std::size_t row = 0; row < 16; row++)
          // For each row in the tile
          for (std::size_t tile_row = 0; tile_row < 16; tile_row++)
            // Solve for each element in this tile row
            for (std::size_t idx = 0; idx < 16; idx++)
              C[(row + row_chunk) * N + col_chunk + idx] +=
                  A[(row + row_chunk) * N + tile + tile_row] *
                  B[tile * N + tile_row * N + col_chunk + idx];
}

```

Pretty much the same as our mutex, except we don't have to write our own `fetch_and_add` routine. Let's take a look at the performance results.

```
-------------------------------------------------------------------------
Benchmark                               Time             CPU   Iterations
-------------------------------------------------------------------------
atomic_mmul_bench/8/real_time       0.516 ms        0.458 ms         1349
atomic_mmul_bench/9/real_time        2.78 ms         2.31 ms          246
atomic_mmul_bench/10/real_time       21.0 ms         18.8 ms           33
```

Interestingly enough, they're about the same as when we used a `std::mutex` (and still ~30% faster than the static mapping)! In these examples, most of our time is being spent doing fused multiply-add operations in the inner loops, and not performing the fetch and add in the outer-most loop. Because the fetch and add is not the bottleneck, we see little-to-no performance difference between our mutex and atomic implementation.

## Final Thoughts

Are mutexes and atomics the same thing? No, certainly not. However, both can be used to solve parallel programming problems where threads want to access the same piece of data. Mutexes allow us to serialize access to a region of code through locking and unlocking. Atomic operations allow us to directly make use of hardware locking mechanisms for fundamental operations like increment, add, exchange, etc.. What's important is that we understand the tools in our parallel programming toolbox, and the costs/benefits of each.

Thanks for reading,

--Nick

### Link to the source code

- [Source Code: ](https://github.com/CoffeeBeforeArch/misc_code/tree/master/inc_bench)
- [My YouTube Channel: ](https://www.youtube.com/channel/UCsi5-meDM5Q5NE93n_Ya7GA?view_as=subscriber)
- [My GitHub Account: ](https://github.com/CoffeeBeforeArch)
- My Email: CoffeeBeforeArch@gmail.com


