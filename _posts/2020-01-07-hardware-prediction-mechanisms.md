---
layout: default
title: A Case Study on Hardware Prediction Mechanisms
---

# A Case Study on Prediction Mechanisms

Modern CPUs contain hardware mechanisms that try to boost performance using prediction. Three example of this are speculative execution, branch prediction, and prefetching. Using benchmarks, we'll look at how our code-design, access patterns, and instruction scheduling influence our performance, on how this relates to these prediction mechanisms! 

The links below are to the source code used and video version of this tutorial:
- [Branch Prediction](https://github.com/CoffeeBeforeArch/uarch_benchmarks/tree/master/branch_prediction)
- [Pre-fetching](https://github.com/CoffeeBeforeArch/uarch_benchmarks/tree/master/branch_prediction)
- [Pre-fetching](https://github.com/CoffeeBeforeArch/uarch_benchmarks/tree/master/branch_prediction)
- [YouTube Video](https://youtu.be/FygXDrRsaU8)

This tutorial was based on the following talks:
- [Want fast C++? Know your hardware!](https://youtu.be/a12jYibw0vs)
- [Tuning C++: Benchmarks, and CPUs, and Compilers! Oh My!](https://youtu.be/nXaxk27zwlk)

## Background


### Cache Coherence


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
