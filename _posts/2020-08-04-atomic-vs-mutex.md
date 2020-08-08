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
  1.15 │278:┌─→mov    %rbx,%rdi
 19.31 │    │→ callq  pthread_mutex_lock@plt
  0.46 │    │  test   %eax,%eax
       │    │↓ jne    3b1
       │    │  mov    %rbx,%rdi
 36.78 │    │  incq   0x30(%rsp)
 20.69 │    │→ callq  pthread_mutex_unlock@plt
       │    ├──dec    %ebp
 21.61 │    └──jne    278
```

Pretty much what we described above! We call `pthread_mutex_lock`, use an `incq` to increment the value, then call `pthread_mutex_unlock`. Now let's compare the performance when we scale the number of threads.

```
-------------------------------------------------------------
Benchmark                   Time             CPU   Iterations
-------------------------------------------------------------
lock_guard_bench/1       2.00 ms         2.00 ms          350
lock_guard_bench/2       13.0 ms         11.9 ms           60
lock_guard_bench/3       20.3 ms         14.5 ms           40
lock_guard_bench/4       28.4 ms         16.7 ms           42
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
-------------------------------------------------------------
Benchmark                   Time             CPU   Iterations
-------------------------------------------------------------
atomic_bench/1          0.624 ms        0.624 ms         1117
atomic_bench/2           2.67 ms         2.34 ms          285
atomic_bench/3           3.99 ms         2.90 ms          224
atomic_bench/4           5.94 ms         4.15 ms          168
```

Way faster than using a mutex! But this shouldn't be terribly surprising. When we use a mutex, we're relying on software routines to lock and unlock the mutex. With our atomic operations, we're relying on the underlying hardware mechanisms to make the increment an indivisible operation.

## Concluding remarks

Thanks for reading,

--Nick

### Link to the source code

- [Source Code: ](https://github.com/CoffeeBeforeArch/misc_code/tree/master/inc_bench)
- [My YouTube Channel: ](https://www.youtube.com/channel/UCsi5-meDM5Q5NE93n_Ya7GA?view_as=subscriber)
- [My GitHub Account: ](https://github.com/CoffeeBeforeArch)
- My Email: CoffeeBeforeArch@gmail.com


