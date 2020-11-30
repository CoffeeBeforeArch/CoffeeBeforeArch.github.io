---
layout: default
title: Hardware Memory Re-Ordering
---

# Hardware Memory Re-Ordering

Even if we use software memory barriers to prevent compilers from re-ordering memory accesses in our output assembly, they still might be re-ordered by the hardware during execution. Exactly how these accesses can be be re-ordered is dictated the processor's memory consistency model.

In this blog post, we'll be looking at an example of x86 Store-Load re-ordering, and how we can prevent it using hardware barriers.

### Link to the source code

- [My YouTube Channel: ](https://www.youtube.com/channel/UCsi5-meDM5Q5NE93n_Ya7GA?view_as=subscriber)
- [My GitHub Account: ](https://github.com/CoffeeBeforeArch)
- My Email: CoffeeBeforeArch@gmail.com

## Background

## Final Thoughts

Thanks for reading,

--Nick

### Link to the source code

- [My YouTube Channel: ](https://www.youtube.com/channel/UCsi5-meDM5Q5NE93n_Ya7GA?view_as=subscriber)
- [My GitHub Account: ](https://github.com/CoffeeBeforeArch)
- My Email: CoffeeBeforeArch@gmail.com

