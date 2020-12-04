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

### Case 1: A = 1, and B = 1

Consider the follwing interleaving of memory instructions:

| Instruction | Issuing Thread |
|:------------|:---------------|
|`Write A, 1` |    Thread 1    |
|`Write B, 1` |    Thread 2    |
|`Read B`     |    Thread 1    |
|`Read A`     |    Thread 2    |

In this case, the writes to `A` and `B` both occur before the reads of `A` and `B`.

## Final Thoughts

Thanks for reading,

--Nick

### Link to the source code

- [My YouTube Channel: ](https://www.youtube.com/channel/UCsi5-meDM5Q5NE93n_Ya7GA?view_as=subscriber)
- [My GitHub Account: ](https://github.com/CoffeeBeforeArch)
- My Email: CoffeeBeforeArch@gmail.com

