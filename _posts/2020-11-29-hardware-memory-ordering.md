---
layout: default
title: Hardware Memory Re-Ordering
---

# Hardware Memory Re-Ordering

Even if we use software memory barriers to prevent compilers from re-ordering memory accesses in our generated assembly, they still might be re-ordered by the hardware during execution. How these accesses can be be re-ordered is a function of the processor's memory consistency model.

In this blog post, we'll be looking at an example of x86 Store-Load re-ordering in an implementation of Peterson's algorithm, and how we can prevent it using hardware barriers.

### Link to the source code

- [My YouTube Channel: ](https://www.youtube.com/channel/UCsi5-meDM5Q5NE93n_Ya7GA?view_as=subscriber)
- [My GitHub Account: ](https://github.com/CoffeeBeforeArch)
- My Email: CoffeeBeforeArch@gmail.com

## Background on Sequential Consistency

Hardware memory re-ordering occurs when memory accesses become globally visible in an order different that the accesses are appear in a program's assembly. How a processor is allowed to re-order memory accesses is a function of the processor's memory consistency model. The most intuitive of these models is sequential consistency.

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

You can find my C++ re-implementation of the benchmark [here](https://github.com/CoffeeBeforeArch/misc_code/blob/master/hw_barrier/hw_barrier.cpp). This can be compiled with:

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

### Running the Benchmark

## Final Thoughts

Thanks for reading,

--Nick

### Link to the source code

- [My YouTube Channel: ](https://www.youtube.com/channel/UCsi5-meDM5Q5NE93n_Ya7GA?view_as=subscriber)
- [My GitHub Account: ](https://github.com/CoffeeBeforeArch)
- My Email: CoffeeBeforeArch@gmail.com

