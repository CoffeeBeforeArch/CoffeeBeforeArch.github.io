---
layout: default
title: Atomic vs Mutex
---

# Atomic vs Mutex

Some parallel applications do not have any data sharing between threads. Others have sharing between threads that is read-only. But some applications are more complex, and require the coordiation of multiple threads that want to write to the same memory. In these situations, we may reach into our parallel programming toolbox for something like `std::mutex` to make sure only a single thread enters a critical section at a time. However, a mutex may be more than we actually require. In this blog post, we'll compare the performance of a mutex against directly using atomic operations in some basic situations.

### Link to the source code
- [Source Code: ](https://github.com/CoffeeBeforeArch/misc_code/tree/master/inc_bench)
- [My YouTube Channel: ](https://www.youtube.com/channel/UCsi5-meDM5Q5NE93n_Ya7GA?view_as=subscriber)
- [My GitHub Account: ](https://github.com/CoffeeBeforeArch)
- My Email: CoffeeBeforeArch@gmail.com

## Compiling and Running the Benchmarks

The benchmarks from this blog post were written using [Google Benchmark](https://github.com/google/benchmark). The following is the command I used to compile the benchmarks (on Ubuntu 18.04 using g++10).

```bash
g++ inc_bench.cpp -lbenchmark -lpthread -O3 -march=native -mtune=native -o thread_affinity
```

## Concluding remarks

Thanks for reading,

--Nick

### Link to the source code

- [Source Code: ](https://github.com/CoffeeBeforeArch/misc_code/tree/master/inc_bench)
- [My YouTube Channel: ](https://www.youtube.com/channel/UCsi5-meDM5Q5NE93n_Ya7GA?view_as=subscriber)
- [My GitHub Account: ](https://github.com/CoffeeBeforeArch)
- My Email: CoffeeBeforeArch@gmail.com


