---
layout: default
title: Strength Reduction
---

# Strength Reduction

Compilers perform many optimizations to improve performance, and among these is strength reduction. Strength reduction is an optimizations by which a compiler replaces expensive operations with less expensive ones that produce a functionally equivilant result. In this blog post, we'll be taking a look at how strength reduction can be used to replace expensive operations like `idiv` with various other arithmetic operations, as well as looking at the difference in performance.

## Links

- [My YouTube Channel](https://www.youtube.com/channel/UCsi5-meDM5Q5NE93n_Ya7GA?view_as=subscriber)
- [My GitHub Account](https://github.com/CoffeeBeforeArch)
- My Email: CoffeeBeforeArch@gmail.com

## Strength Reduction of Integer Division

Division instructions are expensive. However, your compiler can play some tricks to get around them. Let's start by taking a look at a simple `is_odd` function on [Compiler Explorer](https://godbolt.org/).  

## Benchmarking

![assert test_enabled](/assets/strength_reduction/is_odd.png)
![assert test_enabled](/assets/strength_reduction/is_odd_asm.png)
![assert test_enabled](/assets/strength_reduction/remainder.png)
![assert test_enabled](/assets/strength_reduction/remainder_asm.png)
![assert test_enabled](/assets/strength_reduction/remainder_diff.png)
![assert test_enabled](/assets/strength_reduction/remainder_diff_asm.png)
![assert test_enabled](/assets/strength_reduction/remainder_div.png)
![assert test_enabled](/assets/strength_reduction/remainder_div_asm.png)

## Concluding Remarks

As always, feel free to contact me with questions.

Cheers,

--Nick

## Links

- [My YouTube Channel](https://www.youtube.com/channel/UCsi5-meDM5Q5NE93n_Ya7GA?view_as=subscriber)
- [My GitHub Account](https://github.com/CoffeeBeforeArch)
- My Email: CoffeeBeforeArch@gmail.com
