---
layout: default
title: Hardware Memory Reordering
---

# Hardware Memory Reordering

Even if we use software memory barriers to prevent compilers from reordering memory accesses in our generated assembly, they still might be reordered by the hardware during execution. How these accesses can be be reordered is a function of the processor's memory consistency model.

In this blog post, we'll be looking at an example of x86 Store-Load reordering in an implementation of Peterson's algorithm, and how we can prevent it using hardware barriers.

### Link to the source code

- [My YouTube Channel: ](https://www.youtube.com/channel/UCsi5-meDM5Q5NE93n_Ya7GA?view_as=subscriber)
- [My GitHub Account: ](https://github.com/CoffeeBeforeArch)
- My Email: CoffeeBeforeArch@gmail.com

## Background on Sequential Consistency

Hardware memory reordering occurs when memory accesses become globally visible in an order different that the accesses are appear in a program's assembly. How a processor is allowed to reorder memory accesses is a function of the processor's memory consistency model. The most intuitive of these models is sequential consistency.

In a sequentially consistent system, each memory access for a thread becomes globally visible in program order. However, memory accesses from different threads may still be interleaved. Consider the following simple pseudo-assembly example where `A` and `B` are both initially `0`:

| Thread 1 | Thread 2 |
|:------------|:------------|
|`Write A, 1` |`Write B, 1` |
|`Read B`     |`Read A`     |

In a sequentially consistent system, what combinations of values are possible for `A` and `B`? There are three possible scenarios. Let's take a look at the globally visable order of instructions for each.

### Case 1: A = 1, B = 1

Consider the follwing interleaving of memory instructions (initially `A == 0` and `B == 0`):

| Instruction | Issuing Thread | A, B |
|:------------|:---------------|:-----|
|`Write A, 1` |    Thread 1    | 1, 0 |
|`Write B, 1` |    Thread 2    | 1, 1 |
|`Read B`     |    Thread 1    | 1, 1 |
|`Read A`     |    Thread 2    | 1, 1 |

In this case, the writes to `A` and `B` occur before the reads of `A` and `B`. This leads to the result of `A == 1` and `B == 1` as the final result.

This same result could be achieved if the order of the writes is reversed, or the order of reads is reversed.

### Case 2: A = 0, B = 1

Consider the follwing interleaving of memory instructions (initially `A == 0` and `B == 0`):

| Instruction | Issuing Thread | A, B |
|:------------|:---------------|:-----|
|`Write B, 1` |    Thread 2    | 0, 1 |
|`Read A`     |    Thread 2    | 0, 1 |
|`Write A, 1` |    Thread 1    | 1, 1 |
|`Read B`     |    Thread 1    | 1, 1 |


In this case, Thread 2 executes its write and read before Thread 1 does either. This leads to the result of `A == 0`, and `B == 1`

### Case 3: A = 1, B = 0

Consider the follwing interleaving of memory instructions (initially `A == 0` and `B == 0`):

| Instruction | Issuing Thread | A, B |
|:------------|:---------------|:-----|
|`Write A, 1` |    Thread 1    | 1, 0 |
|`Read B`     |    Thread 1    | 1, 0 |
|`Write B, 1` |    Thread 2    | 1, 1 |
|`Read A`     |    Thread 2    | 1, 1 |

In this case, Thread 1 executes its write and read before Thread 2 does either. This leads to the result of `A == 1`, and `B == 0` (the opposite of Case 2).

### Takeaways - Sequential Consistency

One thing to notice from our sequential consistency test cases is that the instructions from each thread execute in a sequential order (e.g, Thread 1 always executes `Write A, 1` before `Read B` regardless of how instructions from Thread 2 are interleaved in between). While reasoning about interleavings of instructions across threads can be difficult, we are safe from instructions being reordered within a single thread.

## Relaxed Memory Consistency Models (x86)

Computer architecture is all about tradeoffs. While sequential consistency is simple and intuitive, it places restrictions on execution that limit performance. These restrictions include the following rules about memory reordering:

|   Reordering   |                Description                 |
|:---------------|:-------------------------------------------|
| Write -> Write | Writes are not reordered with older writes |
| Write -> Read  | Reads are not reordered with older writes  |
| Read -> Read   | Reads are not reordered with older Reads   |
| Read -> Write  | Writes are not reordered with older Reads  |

By relaxing these restriction, we can increase performance at the cost of increasing hardware and programming model complexity. In this post, we're going to focus on relaxation in the Intel x86 memory model. This is described in detail in section 8.2 of the [Intel Software Developer Manual](https://software.intel.com/content/www/us/en/develop/download/intel-64-and-ia-32-architectures-sdm-combined-volumes-1-2a-2b-2c-2d-3a-3b-3c-3d-and-4.html).

### Basics of the x86 Memory Model

The x86 architecture follows a consistency model called Processor Consistency (referred to in the developer manual as Processor Ordering). This model is very similar to sequential consistency, but with the following exception:

- _"Reads and writes always appear in programmed order at
the system bus—except for the following situation where processor ordering is exhibited. Read misses are permitted to go ahead of buffered writes on the system bus when all the buffered writes are cache hits and, therefore, are not directed to the same address being accessed by the read miss."_

Processor consistency relaxes our Write -> Read ordering constraint, allowing writes to be reordered with subsequent reads. This introduces a fourth possible scenario for interleaved instructions for our two threads.

### Case 4: A = 0, B = 0

Consider the follwing interleaving of memory instructions (initially `A == 0` and `B == 0`):

| Instruction | Issuing Thread | A, B |
|:------------|:---------------|:-----|
|`Read A`     |    Thread 2    | 0, 0 |
|`Read B`     |    Thread 1    | 0, 0 |
|`Write A, 1` |    Thread 1    | 1, 0 |
|`Write B, 1` |    Thread 2    | 1, 1 |

In this case, Thread 1 and Thread 2 both perform their reads before performing their writes. This leads to the result of `A == 1`, and `B == 0` (the opposite of Case 2). The same order could be achieved by swapping the order of reads, or swapping the order of writes.

### Takeaways - Processor Consistency

Processor consistency allows for instructions to be reordered within a single thread (e.g., Thread 1's read of `B` can complete before its write to `A`). It's easy to see how this complicates the programming model, making things less intuitive to programmers.

Even if a programmer orders operations correctly in their high-level language (e.g., C++), and ensures the operations are ordered correctly in the compiler-generated assembly, the hardware may still reorder a read with an older write.

Jeff Preshing's blog post [Memory Reodering Caught in the Act](https://preshing.com/20120515/memory-reordering-caught-in-the-act/) is a great read that includes a short benchmark to shows this exact example of hardware Write -> Read reordering.

You can find my C++ implementation of the benchmark from that post [here](https://github.com/CoffeeBeforeArch/misc_code/blob/master/hw_barrier/hw_barrier.cpp). This can be compiled with:

```bash
g++ hw_barrier.cpp -o hw_barrier -O2 -lpthread
```

## Peterson's Algorithm for Mutual Exclusion

To understand how hardware memory reordering can cause real-world bugs, let's take a look at a C++ implementation of Peterson's algorithm for two threads. [Peterson's algorithm](https://en.wikipedia.org/wiki/Peterson%27s_algorithm) is an algorithm for mutual exclusion where threads communicate through shared memory.

We'll start by looking at:

- What state is required for our implementation
- How a thread gets access to a critical section
- How a thread notifies it's done with a critical section

After this we'll run a short benchmark to show how memory reordering fundamentally breaks this implementation, and how we can fix this with a dedicated memory barrier instruction.

You can find the full benchmark [here](https://github.com/CoffeeBeforeArch/misc_code/blob/master/hw_barrier/peterson.cpp).

### Required State

Peterson's algorithm fundamentally requires two pieces of state:

- `interested` - An array of elements (equal to the number of threads) indicating each thread's interest in entering the critical section
- `turn` - An element indicating which thread currently has priority to enter the critical section

We can implement these as data members of a class called `Peterson`.

```cpp
// Simple class implementing Peterson's algorithm for mutual exclusion
class Peterson {
 private:
  // Is this thread interested in the critical section
  volatile int interested[2] = {0, 0};

  // Who's turn is it?
  volatile int turn = 0;
};
```

Note, we have both our variables marked as `volatile` to keep them from being cached in registers (we need access of these variables to be in memory so that they are visable to both threads).

### Gaining Access to the Critical Section

The next things we need is a method where a thread will wait for access to a critical section. We'll call this method `lock`. Here is how I implemented mine:

```cpp
  // Method for locking w/ Peterson's algorithm
  void lock(int tid) {
    // Mark that this thread wants to enter the critical section
    interested[tid] = 1;

    // Assume the other thread has priority
    int other = 1 - tid;
    turn = other;

    // Wait until the other thread finishes or is not interested
    while (turn == other && interested[other])
      ;
  }
```

The first thing the calling thread does is index into our `interested` array using its thread ID (assumed to be either `0` or `1`), and write `1` to say that it is interested in entering the critical section.

Next, the thread yields priority to the other thread by setting `turn` equal to the other thread's ID.

Finally the thread enters a `while` loop waiting for either the other thread to yield priority, or to no longer be interested in entering the critical section.

### Exiting the Critical Section

The final thing we need is a method to notify that a thread is done with the critical section. We can call this method `unlock`. Here is how I implemented mine:

```cpp
  // Method for unlocking w/ Peterson's algorithm
  void unlock(int tid) {
    // Mark that this thread is no longer interested
    interested[tid] = 0;
  }
```

Notifying that a thread is done with the critical section is simple. The exiting thread just needs to say it is no longer interested in the critical section by writing `0` to its state in `interested`

### Our Benchmark

To test our implementation of Peterson's algorithm, we'll be using it to prevent simultaneous increments to a non-atomic integer. Here is the function called `work` that we'll have two thread run:

```cpp
// Work function
void work(Peterson &p, int &val, int tid) {
  for (int i = 0; i < 1e8; i++) {
    // Lock using Peterson's algorithm
    p.lock(tid);
    // Critical section
    val++;
    // Unlock using Peterson's algorithm
    p.unlock(tid);
  }
}
```

Here, we're using Peterson's algorithm to protect the increment of our shared value `val`.

The `main` function that drives our benchmark is incredibly simple:

```cpp
int main() {
  // Shared value
  int val = 0;
  Peterson p;

  // Create threads
  std::thread t0([&] { work(p, val, 0); });
  std::thread t1([&] { work(p, val, 1); });

  // Wait for the threads to finish
  t0.join();
  t1.join();

  // Print the result
  std::cout << "FINAL VALUE IS: " << val << '\n';

  return 0;
}
```

Here, we create our shared value `val` and `Peterson` object, spawn two threads (with thread IDs `0` and `1` respectively), wait for them to finish running the `work` function, then print out the final result value of `val`.

### Compiling and Running the Benchmark

We can compile our benchmark `peterson.cpp` using the following command:

```bash
g++ peterson.cpp -o peterson -O2 -lpthread
```

What we'd expect to be printed at the end of execution (if our implementation of Peterson's algorithm gives us mutual exclusion) is 2 million (from 1 million increments performed by each thread). If our implementation does not give us mutual exclusion, the final value of `val` may be less than 2 million because of overlapping increments from our two threads.

Here are the results from a few runs of the benchmark:

```txt
FINAL VALUE IS: 199999998
FINAL VALUE IS: 199999991
FINAL VALUE IS: 199999993
FINAL VALUE IS: 199999992
```

Definitely a bug! `1e8 + 1e8` does not equal `199999998` (or any of our other values)! So what happened? 

We can first look at our assembly to see if the order of our instructions is the problems. Here's are the output from perf:

```assembly
  0.00 │18:┌─→movl    $0x1,0x4(%rax)     
  0.20 │   │  movl    $0x0,0x8(%rax)     
       │   │↓ jmp     36                 
       │   │  nop                        
 25.73 │30:│  mov     (%rax),%edx        
 34.10 │   │  test    %edx,%edx          
  0.00 │   │↓ je      3d                 
 29.10 │36:│  mov     0x8(%rax),%edx     
  0.22 │   │  test    %edx,%edx          
  3.79 │   │↑ je      30                 
  6.86 │3d:│  incl    (%rsi)             
       │   │  movl    $0x0,0x4(%rax)     
       │   ├──dec     %ecx               
       │   └──jne     18                 
```

Nothing stands out as immediately incorrect. First, our threads say they are interested in entering the critical section and yield priority to each other using the `movl` instructions starting at label `18:`.

Next, our threads wait inside of the `while` loop starting at label `30:`, where they evaluate the two conditions (at labels `30:` and `36:`). 

If one of these conditions becomes false, a thread does the increment of the shared value (`incl`), stores `0` to say it is no longer intersted in the critical section, and continues executing the `for` loop in our `work` function.

Since the ordering of these instructions make sense in our binary, something must be getting reordered in hardware. Let's think about where that could be happening in our high-level C++. As a reminder, here's our `lock` method:

```cpp
  // Method for locking w/ Peterson's algorithm
  void lock(int tid) {
    // Mark that this thread wants to enter the critical section
    interested[tid] = 1;

    // Assume the other thread has priority
    int other = 1 - tid;
    turn = other;

    // Wait until the other thread finishes or is not interested
    while (turn == other && interested[other])
      ;
  }
```

We know that Processor Consistency says writes may be reordered with subsequent reads, so we will focus on those relationships in our program. Specifically, we will focus on the conditions being checked in our `while` loop because that is where our threads wait for permission to enter the critical section.

First, let's consider `turn`. Each thread yields priority by setting `turn = other`, then waiting on `turn == other` in the `while` loop. If the read of `turn` for the comparison is reordered before the write, the thread could break out of the `while` loop and exist the critical section. Is this possible?

No! This write and read can not be reordered because they are the same location! The developer manual specifically says this is not allowed:

- _"Reads may be reordered with older writes to different locations but not with older writes to the same location."_

Let's now consider the write-read pair of `interested[tid]` and `interested[other]`. Our thread first says it's interested in entering the critical section by writing to `1` to `interested[tid]`. In the `while` loop, it then waits for the other thread to no longer be interested in the critical section by evaluating `interested[other] == 1`.

If the two reads (one by each thread) of `interested[other]` are reordered before the two writes (one by each thread) to `interested[tid]`, each thread would think that the other is not interested in the critical section, break out of the `while` loop, and enter the critical section. Is this possible?

Yes! `interested[tid]` and `interested[other]` are two different memory locations, meaning that this write-read pair can be reordered!

Now that we understand where our bug is coming from, we can ask the million dollar question. How do we prevent this reordering?

### Hardware Memory Barriers

To prevent the reordering of memory operations, x86 provides memory barriers in the form of fence instructions that we directly insert using intrinsics. These fences include:

- `_mm_sfence`
  - _"Perform a serializing operation on all store-to-memory instructions that were issued prior to this instruction. Guarantees that every store instruction that precedes, in program order, is globally visible before any store instruction which follows the fence in program order."_
- `_mm_lfence`
  - _"Perform a serializing operation on all load-from-memory instructions that were issued prior to this instruction. Guarantees that every load instruction that precedes, in program order, is globally visible before any load instruction which follows the fence in program order."_
- `_mm_mfence`
  - _"Perform a serializing operation on all load-from-memory and store-to-memory instructions that were issued prior to this instruction. Guarantees that every memory access that precedes, in program order, the memory fence instruction is globally visible before any memory instruction which follows the fence in program order."_

Now the question is, which one do we need in our program?

`_mm_sfence` only deals with the the ordering of store instructions, and `_mm_lfence` only deals with the ordering of load instructions, so neither of these (at least on their own) are what we're looking for. However, `_mm_mfence` looks like a perfect fit!

`_mm_mfence` gives us the guarantee that prior stores (like the write to `interested[tid]`) will have become globally visable before and subsequent memory operation (like the read of `interested[other]`). Let's add it between our write of `interested[tid]` and read of `interested[other]`:

```cpp
  // Method for locking w/ Peterson's algorithm
  void lock(int tid) {
    // Mark that this thread wants to enter the critical section
    interested[tid] = 1;

    // Assume the other thread has priority
    int other = 1 - tid;
    turn = other;

    // HW memory barrier
    _mm_mfence();

    // Wait until the other thread finishes or is not interested
    while (turn == other && interested[other])
      ;
  }
```

If we recompile and rerun our application a few times we get the following result:

```txt
FINAL VALUE IS: 200000000
FINAL VALUE IS: 200000000
FINAL VALUE IS: 200000000
```

Finally! The correct answer! Let's take a look at the assembly:

```assembly
       │18:┌─→movl    $0x1,(%rax)         
  0.16 │   │  movl    $0x1,0x8(%rax)      
 20.13 │   │  mfence                      
  1.15 │   │↓ jmp     37                  
       │   │  nop                         
 19.13 │30:│  mov     0x4(%rax),%ecx      
  3.12 │   │  test    %ecx,%ecx           
       │   │↓ je      3f                  
  1.55 │37:│  mov     0x8(%rax),%ecx      
  0.13 │   │  cmp     $0x1,%ecx           
 33.72 │   │↑ je      30                  
 20.90 │3f:│  incl    (%rsi)              
  0.01 │   │  movl    $0x0,(%rax)         
       │   ├──dec     %edx                
       │   └──jne     18                  
```

The only major difference is that we have an `mfence` instrcution after our stores to `interested[tid]` and `turn`.

## Does Hardware Reodering Require OoO Execution?

It's natural to think that hardware memory reordering requires out-of-order (OoO) execution. However, this is not the case. Hardware memory reordering can happen even in an in-order processor! To understand this, let's take a look at the [architecture block diagram](https://en.wikichip.org/wiki/intel/microarchitectures/kaby_lake) for Intel's Kaby Lake Processors (which does OoO execution) from wikichip.

![kaby lake arch](https://en.wikichip.org/w/images/7/7e/skylake_block_diagram.svg)

In the Memory Subsystem, you can see a block labeled "Store Buffer & Forwarding". A store buffer is there to do exactly what it sounds like: buffer stores. Unlike loads, stores are not typically on the critical path (instructions often stall waiting on data to be loaded, not on data being written). Instead of making stores immediately visible, they can first be buffered in a store buffer, and written back gradually as bandwidth becomes available.

In an in-order processor, a thread could first perform a store that goes into the store buffer. At this point, the store is the memory subsystem's problem (to the core, that operation is complete). The thread could then issue a read which completes before the previous store has drained out of the store buffer! No OoO execution required!

## How Do Fence Instruction Work?

It's natural to be inquisitve about how the the x86 instructions work at the hardware level. Below are some quick thoughts on how they could work:

- `_mm_sfence`
  - To ensure all previous stores have become globally visible, the hardware could stall all later stores (in program order), until all earlier stores have retired and the store buffer has been drained, before the `sfence` instruction itself retires.
- `_mm_lfence`
  - Similar to the `_mm_sfence`, the architecture could stall later loads (in program order), until all earlier loads have completed, before the `lfence` instruction itself retires.
- `_mm_mfence`
  - Naturally, the `mfence` instruction is a combination of the previous two. The architecture could stall all later loads and stores (in program order) until all ealier loads and stores have retired and the store buffer has been drained, before the `mfence` instruction itself retuires.

There are of course exceptions and nuances not covered by these descriptions, but I'll leave the [patent reading](https://www.freepatentsonline.com/8959314.html) as an exercise for the reader.

## Final Thoughts

Memory consistency can be difficult to understand. Even with the relatively strict consistency model of x86 (compared to weaker models used by ARM and POWER), bugs related to hardware memory reordering often go against our intuition, and often require some very careful thought to get around.

Thanks for reading,

--Nick

### Link to the source code

- [My YouTube Channel: ](https://www.youtube.com/channel/UCsi5-meDM5Q5NE93n_Ya7GA?view_as=subscriber)
- [My GitHub Account: ](https://github.com/CoffeeBeforeArch)
- My Email: CoffeeBeforeArch@gmail.com

