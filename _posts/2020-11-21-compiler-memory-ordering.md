---
layout: default
title: Compiler Memory Re-Ordering
---

# Compiler Memory Re-Ordering

Compilers perform instruction scheduling to improve instruction-level parallelism (ILP). While better scheduling often improves performance, re-ordering of memory accesses can lead to subtle bugs in multithreaded applications.

In this blog post, we'll use Compiler Explorer to see how GCC can re-order memory accesses, discuss how this can lead to bugs, and finally how we can guide the compiler to generate the code we want (and often expect) it to generate.

### Link to the source code

- [My YouTube Channel: ](https://www.youtube.com/channel/UCsi5-meDM5Q5NE93n_Ya7GA?view_as=subscriber)
- [My GitHub Account: ](https://github.com/CoffeeBeforeArch)
- My Email: CoffeeBeforeArch@gmail.com

## Final Thoughts

Thanks for reading,

--Nick

### Link to the source code

- [My YouTube Channel: ](https://www.youtube.com/channel/UCsi5-meDM5Q5NE93n_Ya7GA?view_as=subscriber)
- [My GitHub Account: ](https://github.com/CoffeeBeforeArch)
- My Email: CoffeeBeforeArch@gmail.com

