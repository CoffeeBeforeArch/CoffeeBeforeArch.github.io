---
layout: default
title: Compiler Memory Re-Ordering
---

# Compiler Memory Re-Ordering

Compilers perform instruction scheduling to improve instruction-level parallelism (ILP). While better scheduling often improves performance, re-ordering of memory accesses can lead to subtle bugs in multithreaded applications.

In this blog post, we'll look at how GCC can re-order memory accesses, discuss how this can lead to bugs, and look at how we can guide the compiler to generate the code we want (and often expect).

### Link to the source code

- [My YouTube Channel: ](https://www.youtube.com/channel/UCsi5-meDM5Q5NE93n_Ya7GA?view_as=subscriber)
- [My GitHub Account: ](https://github.com/CoffeeBeforeArch)
- My Email: CoffeeBeforeArch@gmail.com

## Background

One area where compiler memory re-ordering is especially relevant is in the development of synchronization devices (e.g., locks). Without proper guidance, a compiler may re-order the release of a lock before accesses to shared data in a critical section.

To show this, consider the following `memory_reorder` function that will be executed by multiple threads:

```cpp
// Shared values for our threads
extern volatile int now_serving;
extern int shared_value;

// Some expensive computation
int compute(int a);

void memory_reorder(int ticket, int a) {
    // Wait for the lock
    while(now_serving != ticket);

    // Our critical section
    shared_value = compute(a);

    // Release the lock
    now_serving = ticket + 1;
}
```

Inside our `memory_reorder` function, each thread waits in an infinite `while` loop for its `ticket` number to equal the `now_serving` number. After the numbers our equal, the thread enters the critical section where it performs some computation, and updates `shared_value`. Finally, the thread with the lock passes it to the next in line by incrementing the `now_serving` number by `1`.

We're essentially looking at is the core part of a ticket spinlock protecting a critical section. The `while` loop is where the thread waits for the lock, the call to `compute` and store to `shared_value` is the critical section, and the store to `now_serving` is the release of the lock.

You can find more information in [this](https://coffeebeforearch.github.io/2020/11/07/spinlocks-6.html) blog post about ticket spinlocks.

## Understanding the Low-Level Assembly

To understand how a compiler may re-order access to memory locations, we have to look at some assembly. Here is the resulting assembly for our `memory_reorder` function compiled with GCC-10.2 and with the flags `-O1 -march=native -mtune=native`:

```assembly
memory_reorder(int, int):
        push    rbx
        mov     ebx, edi
        mov     edi, esi
.L2:
        mov     eax, DWORD PTR now_serving[rip]
        cmp     eax, ebx
        jne     .L2
        call    compute(int)
        mov     DWORD PTR shared_value[rip], eax
        inc     ebx
        mov     DWORD PTR now_serving[rip], ebx
        pop     rbx
        ret
```

The resulting assembly is an almost 1-to-1 translation of our C++. Starting at label `.L2`, our threads wait in our `while` loop where they read the `now_serving` number from memory (`mov`), compare it against the `ticket` number (`cmp`), and either jump back to the top of the loop or break out (`jne`).

When we break out of the loop, we call our `compute` function (`call`), then store the result to memory (`mov`). We finally store our incremented ticket number to `now_serving` (`inc` + `mov`) and return (`ret`) from the function.

## Instruction Scheduling at Higher Optimization Levels

Let's see what happens if we bump up our compiler optimization level from `-O1` to `-O2`. Here was the resulting assembly for the compiler flags `-O2 -march=native -mtune=native`:

```assembly
memory_reorder(int, int):
        push    rbx
        mov     ebx, edi
        mov     edi, esi
.L2:
        mov     eax, DWORD PTR now_serving[rip]
        cmp     eax, ebx
        jne     .L2
        call    compute(int)
        inc     ebx
        mov     DWORD PTR now_serving[rip], ebx
        mov     DWORD PTR shared_value[rip], eax
        pop     rbx
        ret
```

While our instructions have remained the same, their order has not. Specifically, we can see that our increment (`inc`) of `ticket` and store of that result to `now_serving` was  hoisted above our store of the result of `compute()` to `shared_val`. This is bad for us! Essentially what the compiler has done is release the lock before we have updated the shared value in our critical section!

But why did the compiler do this if it's so bad? The short answer is, the compiler does not see it as bad. To the compiler, the store to `shared_value` is independent of the store to `now_serving`. Because these two operations are independent, the compiler can re-order them as it sees fit. The store to `shared_value` happening before the store to `now_serving` in the high-level program does *NOT* imply a happens-before relationship between the two.

But now to the million-dollar question. How do we stop the compiler from doing this? Software memory barriers!

## Software Memory Barrier 

Just like how instruction sets will have barrier instructions to prevent the re-ordering of memory accesses that can occur in hardware, there are software barriers to prevent memory re-ordering done by the compiler. Let's take a look at how we can use a software barrier to prevent the re-ordering of our store to `shared_val`:

```cpp
// Shared values for our threads
extern volatile int now_serving;
extern int shared_value;

// Some expensive computation
int compute(int a);

void memory_reorder(int ticket, int a) {
    // Wait for the lock
    while(now_serving != ticket);

    // Our critical section
    shared_value = compute(a);

    // Prevent compiler reordering w/ software barrier
    asm volatile("" : : : "memory");

    // Release the lock
    now_serving = ticket + 1;
}
```

All we've added is the fake inline assembly instruction `asm volatile("" : : : "memory")` that performs the function of a software barrier. This represents a non-instruction that could touch all memory, thereby preventing accesses from being re-ordered because of the dependency (e.g., the update of `shared_value` can not be moved after the barrier, and the update of `now_serving` can not be moved before it by the compiler).

Let's see how our final output assembly looks (compiled with `-O2 -march=native -mtune=native` that led to the bad memory re-ordering):

```assembly
memory_reorder(int, int):
        push    rbx
        mov     ebx, edi
        mov     edi, esi
.L2:
        mov     eax, DWORD PTR now_serving[rip]
        cmp     eax, ebx
        jne     .L2
        call    compute(int)
        mov     DWORD PTR shared_value[rip], eax
        inc     ebx
        mov     DWORD PTR now_serving[rip], ebx
        pop     rbx
        ret
```

Our re-ordering is gone! Our software barrier prevented the update of `shared_value` from being moved after the increment of `now_serving`! You can also see that the inline assembly does not actually lead to a hardware barrier in the output (it really is just a hint to the compiler).

## Final Thoughts

Writing parallel programs is difficult, and it's critical to understand not just memory consistency model the architecture you are working with, but the rules that guide code generation and instruction scheduling for you language and compiler.

Thanks for reading,

--Nick

### Link to the source code

- [My YouTube Channel: ](https://www.youtube.com/channel/UCsi5-meDM5Q5NE93n_Ya7GA?view_as=subscriber)
- [My GitHub Account: ](https://github.com/CoffeeBeforeArch)
- My Email: CoffeeBeforeArch@gmail.com

