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

Division instructions are expensive. However, your compiler can play some tricks to get around them. Let's start by taking a look at a simple `is_odd` function on [Compiler Explorer](https://godbolt.org/). One way we can implement this function is with the modulo operator. We can do modulo 2 in order to get the remainder when dividing our input value by two. Let's see how this looks in C++.

![assert test_enabled](/assets/strength_reduction/is_odd.png)

Now lets look at the assembly generated for this function.

![assert test_enabled](/assets/strength_reduction/is_odd_asm.png)

Our compiler was pretty clever! Instead of using an expensive `idiv`, our compiler used a bitwise AND instruction by 1. This checks if the least significant bit of an input is set (allowing us to check if the number is even or odd). Pretty neat! However, checking if a number is odd is rather trivial. Let's give our compiler a more difficult problem.

Let's transform our `is_odd` function into one called `remainder` that returns the remainder when the input is divided by some constant (1245 in this case). Here's what that may look like in C++.

![assert test_enabled](/assets/strength_reduction/remainder.png)

Now let's take a look at the assembly.

![assert test_enabled](/assets/strength_reduction/remainder_asm.png)

Looks like our compiler found a way around using an `idiv` instruction again! This time, it used a combination of shifts, multiplies, subtracts, etc. to functionally replace the expensive division instruction. Let's try another value (892) to see what the assembly looks like. Here's the high-level C++.

![assert test_enabled](/assets/strength_reduction/remainder_diff.png)

And here is the assembly.

![assert test_enabled](/assets/strength_reduction/remainder_diff_asm.png)

Looks like we got a similar output! The compiler found another combination of arithmetic (outside of division) to replace the `idiv` instruction.

Here's a good question. When can't the compiler replace the `idiv` instruction in some clever way? One obvious reason why a compiler may not be able to get rid of a division instruction is if we aren't doing modulo by some constant known at compile-time. If the compiler doesn't know what the value we're dividing by is, it has to be conservative, and generate code that works in all possible cases. Let's replace the constant we're doing modulo by a second input to our function. Here's the C++ with those changes.

![assert test_enabled](/assets/strength_reduction/remainder_div.png)

And here is the corresponding assembly.

![assert test_enabled](/assets/strength_reduction/remainder_div_asm.png)

No strength reduction!

## Benchmarking Strength Reduction

Are all the extra instructions really worth it? Let's benchmark it and find out! Here's a benchmark for the strength reduction implemenation.

```cpp
static void srMod(benchmark::State &s) {
  std::vector<int> v_in(4096);
  std::vector<int> v_out(4096);

  for (auto _ : s) {
    for (size_t i = 0; i < v_in.size(); i++)
      v_out[i] = v_in[i] % 1245;
  }
}
BENCHMARK(srMod);
```

And here is a benchmark using that uses the division instruction.

```cpp
static void baseMod(benchmark::State &s) {
  std::vector<int> v_in(4096);
  std::vector<int> v_out(4096);

  for (auto _ : s) {
    for (size_t i = 0; i < v_in.size(); i++)
      v_out[i] = v_in[i] % s.range(0);
  }
}
BENCHMARK(baseMod)->Arg(1245);
```

Our benchmarks just create a few vectors of 0s, performs modulo 1245 on all the elements, and stores them to memory. We're able to avoid the strength reduction optimization in our `baseMod` by indirectly passing 1245 thought `Arg`. Both benchmarks were compiled using the following command line flag.

## Concluding Remarks

As always, feel free to contact me with questions.

Cheers,

--Nick

## Links

- [My YouTube Channel](https://www.youtube.com/channel/UCsi5-meDM5Q5NE93n_Ya7GA?view_as=subscriber)
- [My GitHub Account](https://github.com/CoffeeBeforeArch)
- My Email: CoffeeBeforeArch@gmail.com
