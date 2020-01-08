---
layout: default
title: A Case Study on Hardware-Level Prediction Mechanisms
---

# A Case Study on Prediction Mechanisms

Modern CPUs contain hardware-level mechanisms that try to boost performance using prediction. Three example of this are speculative execution, branch prediction, and prefetching. Using benchmarks, we'll look at how our code-design, access patterns, and instruction scheduling influence our performance, on how this relates to these prediction mechanisms! 

The links below are to the source code used in this tutorial:
- [Branch Prediction](https://github.com/CoffeeBeforeArch/uarch_benchmarks/tree/master/branch_prediction)
- [Instruction Scheduling](https://github.com/CoffeeBeforeArch/cpp_20_samples/tree/master/condition_attributes)
- [Prefetching](https://github.com/CoffeeBeforeArch/uarch_benchmarks/tree/master/prefetching)

This tutorial was based on the following talks:
- [Want fast C++? Know your hardware!](https://youtu.be/a12jYibw0vs)
- [Tuning C++: Benchmarks, and CPUs, and Compilers! Oh My!](https://youtu.be/nXaxk27zwlk)

## Branch Prediction
### Background
Branch prediction allows you processor to continue speculative execution in the presence of conditional branches. However, only predicting if a branch is taken or not is insufficient. We rely on branch prediction units to also predict the target of a branch (what code needs to be executed). For example, consider the following code:

```cpp
void bp(bool condition) {
  if(condition)
    // BB1
  else
    // BB2
}
```

If we predict the condition to be true, we must also predict the address of the next instruction (at BB1). Likewise, if we predict the branch to resolve to false, we have to also predict the target (BB2) in the else block. However, there are less obvious scenarios where branch prediction is important. Consider the following code:

```cpp
void callMethod(Object o) {
  o.method();
}
```

This doesn't seem difficult to predict. If I call a method of an object, it's just a function call. This branch is not conditional, and the target should always be known. However, what if our method is actually a virtual function? Now things are slightly more complicated. Instead of directly calling a function, now we have to look up which function to call in a virtual function table.

Blindly calling virtual function can lead to performance problems. This is because the location of the next instruction to execute becomes difficult to predict. It's threfore important to treat a virtual functional call like a conditional branch (from a performance point of view). Paying a one-time or infrequent penalty for re-arranging our data may lead to significant performance improvement.

### Benchmarking

## Instruction Scheduling

## Prefetching

## Concluding remarks

Understanding how your underlying hardware works is incredibly important when you're trying to squeeze performance out of your code. False sharing is an example of how an obvious solution to a problem (like using multiple atomic integers) may not be a solution. We'll cover more optimization case studies like this in later tutorials.

Thanks for reading,

--Nick

### Links to source and YouTube Tutorial

- [Source Code](https://github.com/CoffeeBeforeArch/uarch_benchmarks/tree/master/false_sharing)
- [YouTube Video](https://youtu.be/FygXDrRsaU8)

### Additional discussion points

- Our benchmarks could be rewritten using atomic references that have been introduced in C++20. These provide an excellent way to have selective atomicity.
- Real workloads typically do more that have 4 threads only compete for a single cache-line. However, there may be regions of execution where this pathological case exists, and still causes a significant performance impact.
- Why do you need to use an atomic integer at all? [Here's](https://stackoverflow.com/a/39396999/12482128) a Stack Overflow response answering that question.
