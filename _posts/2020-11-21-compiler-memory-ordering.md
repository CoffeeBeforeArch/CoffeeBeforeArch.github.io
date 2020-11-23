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

To show this example, consider the following `memory_reorder` function that will be executed by multiple threads:

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

Inside our `memory_reorder` function, each thread waits in an infinite `while` loop for its `ticket` number to equal the `now_serving` number. After the numbers our equal, the thread enters the critical section where it performs some computation, and updates `shared_value`. Finally, the thread with the lock passes it to the next in line by incrementing the `now_serving` number by 1.

What we are essentially looking at is the core part of a ticket spinlock protecting a critical section (the call to the `compute` function and store to `shared_value`).

## Understanding the Low-Level Assembly

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

## Software Memory Barrier

```cpp
// Shared values for our threads
extern volatile int now_serving;
extern int shared_value;

// Some expensive computation
int compute(int a);

void memory_reorder(int ticket, int a) {
    // Wait for values to match
    while(now_serving != ticket);

    // Update the value
    shared_value = compute(a);

    // Prevent compiler reordering w/ software barrier
    asm volatile("" : : : "memory");

    // Tell the next thread to go
    now_serving = ticket + 1;
}
```

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

## Final Thoughts

Thanks for reading,

--Nick

### Link to the source code

- [My YouTube Channel: ](https://www.youtube.com/channel/UCsi5-meDM5Q5NE93n_Ya7GA?view_as=subscriber)
- [My GitHub Account: ](https://github.com/CoffeeBeforeArch)
- My Email: CoffeeBeforeArch@gmail.com

