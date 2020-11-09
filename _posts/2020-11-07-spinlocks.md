---
layout: default
title: Spinlocks
---

# Spinlocks

One of the my important parts of parallel programming is synchronization and coordinating access to shared data. An important tool in that domain is the spinlock. Spinlocks provide a way for threads to busy-wait for a lock instead of yielding their remaining time to the O.S.'s thread scheduler. This can be increadibly useful when you know your threads will not be waiting very long for access to the lock.

In this blog post, we'll look at how spinlocks and their optimizations can be implemented in C++.

### Link to the source code

- [Source Code: ](https://github.com/CoffeeBeforeArch/spinlocks)
- [My YouTube Channel: ](https://www.youtube.com/channel/UCsi5-meDM5Q5NE93n_Ya7GA?view_as=subscriber)
- [My GitHub Account: ](https://github.com/CoffeeBeforeArch)
- My Email: CoffeeBeforeArch@gmail.com

## A Naive Spinlock

## A Spinlock with Locally Spinning

## A Spinlock with Active Backoff

## A Spinlock with Passive Backoff

## A Spinlock with Non-Constant Backoff

### Exponential Backoff

### Random Backoff

## A Ticket Spinlock

## Final Thoughts

Thanks for reading,

--Nick

### Link to the source code

- [Source Code: ](https://github.com/CoffeeBeforeArch/spinlocks)
- [My YouTube Channel: ](https://www.youtube.com/channel/UCsi5-meDM5Q5NE93n_Ya7GA?view_as=subscriber)
- [My GitHub Account: ](https://github.com/CoffeeBeforeArch)
- My Email: CoffeeBeforeArch@gmail.com

