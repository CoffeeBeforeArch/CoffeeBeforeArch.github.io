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

In a sequentially consistent system, each memory access for a thread becomes globally visible in program order. However, memory accesses from different thread may still be interleaved. Consider the following simple pseudo-assembly example where `A` and `B` are both initially `0`:

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

### Takeaways

One thing to notice from our sequential consistency test cases is that the instructions from each thread execute in a sequential order (e.g, Thread 1 always executes `Write A, 1` before `Read B` regardless of how instructions from Thread 2 are interleaved in between). While reasoning about interleavings of instructions across threads can be difficult, we are safe from instructions being reordered within a single thread.

## Relaxed Memory Consistency Models (x86)

Computer architecture is all about tradeoffs. While sequential consistency is simple and intuitive, it places restrictions on execution that limit performance. By relaxing these restriction, we can increase performance at the cost of increasing hardware and programming model complexity.

To relax the memory model, we relax the rules around reordering reads and writes with respect to other reads and writes. In this post, we're going to focus on relaxation in the Intel x86 memory model. This is described in detail in section 8.2 of the [Intel Software Developer Manual](https://software.intel.com/content/www/us/en/develop/download/intel-64-and-ia-32-architectures-sdm-combined-volumes-1-2a-2b-2c-2d-3a-3b-3c-3d-and-4.html).

### Basics of the x86 Memory Model

## Final Thoughts

Thanks for reading,

--Nick

### Link to the source code

- [My YouTube Channel: ](https://www.youtube.com/channel/UCsi5-meDM5Q5NE93n_Ya7GA?view_as=subscriber)
- [My GitHub Account: ](https://github.com/CoffeeBeforeArch)
- My Email: CoffeeBeforeArch@gmail.com

