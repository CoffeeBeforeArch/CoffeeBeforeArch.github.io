---
layout: default
title: Avoiding Branches
---

# Avoiding Branches

Modern processors speculate if branches are taken and their targets. If a processor speculates incorrectly, it must rewind all incorrectly executed instructions, and continue down the correct control flow path. This can lead to significant performance problems when a branch is difficult to predict, and the processor misspeculates frequently. While the compiler can remove some branches using techniques like scalarization, some problematic ones require programmer intervention.

In this blog post, we'll be looking at how we can remove branches by integrating a condition with a mathematical operation. We'll some different ways we can approach this problem, as well as dicussing how both our choice of types and immediate values can impact performance.

## Links

- [My YouTube Channel](https://www.youtube.com/channel/UCsi5-meDM5Q5NE93n_Ya7GA?view_as=subscriber)
- [My GitHub Account](https://github.com/CoffeeBeforeArch)
- [Benchmarks](https://github.com/CoffeeBeforeArch/misc_code/blob/master/conditions)
- My Email: CoffeeBeforeArch@gmail.com

## Compiling the Code

All the benchmarks used in this blog post can be found [here](https://github.com/CoffeeBeforeArch/misc_code/blob/master/conditions), and were writting using [Google Benchmark](https://github.com/google/benchmark). Below is the command I used to compile the benchmarks.

```bash
g++ conditions.cpp -O3 -lbenchmark -lpthread -march=native -mtune=native
```

Additionally, there is a short program that prints out the sizes of some dynamic allocations by overloading the `new` operator. That can be compiled as follows.

```bash
g++ sizes.cpp
```

The Google benchmark code follows the following form.

```cpp
static void Bench(benchmark::State &s) {
  // Get the input vector size
  auto N = 1 << s.range(0);

  // Create random number generator
  std::random_device rd;
  std::mt19937 gen(rd());
  std::bernoulli_distribution d(0.5);

  // Create a vector of random booleans
  std::vector<bool> v_in(N);
  std::generate(begin(v_in), end(v_in), [&]() { return d(gen); });

  // Output element
  // Dynamically allocated int isn't optimized away
  int *sink = new int;
  *sink = 0;

  // Benchmark main loop
  while (s.KeepRunning()) {
    // Conditionally add to "sink" here
  }

  // Free our memory
  delete sink;
}
BENCHMARK(branchBench)->DenseRange(10, 12);
```
To set up the experiment, we create a vector of boolean values, and initialize them using a pseudo-random number generator. Inside of our main benchmarking loop, we're going to be testing the performance of conditionally adding a constant value to our `sink` variable. `sink` was dynamically allocated to keep it from being removed by the compiler, but retain the optimizations that were disabled when I marked it as `volatile` or with `benchmark::DoNotOptimize(...)`. We'll be testing input vector sizes of 2^10, 11, and 12.

## Baseline Benchmark

Our baseline benchmark will be one that uses a branch to conditionally add numbers to our sink. Here is the inner benchmark loop.

```cpp
// Benchmark main loop
while (s.KeepRunning()) {
  for (auto b : v_in)
    if (b) *sink += 41;
}
```

Nothing crazy in this first implementation. All we're doing is going through all of our boolean conditions, and conditionally adding the value of 41 to `sink` if the condition is true. 

Let's run our benchmark and see what we get as results.

```
---------------------------------------------------------
Benchmark               Time             CPU   Iterations
---------------------------------------------------------
branchBench/10        814 ns          814 ns      3236121
branchBench/11       1825 ns         1823 ns      1604909
branchBench/12      13262 ns        13261 ns       210503
```

Some interesting results right from the start. Increasing the number of operations from 2^10 to 2^11 seems to have doubled the execution time. However, doubling the number of booleans again from 2^11 to 2^12 increased the execution time by over 7x! Let's first take a look at some hot-spot information at the assembly level using the [perf Linux profiler](https://perf.wiki.kernel.org/index.php/Main_Page).

```asm
 12.76 │1a5:┌─→shlx   %rax,%rbx,%rdx                                     ▒
  3.25 │    │  and    (%rcx),%rdx                                        ▒
 10.36 │    │↓ je     1b4                                                ▒
 48.66 │    │  addl   $0x29,(%r12)                                       ▒
 10.02 │1b4:│  cmp    $0x3f,%eax                                         ▒
  0.40 │    │↓ je     220                                                ▒
  0.33 │    │  inc    %eax                                               ▒
  0.01 │    ├──cmp    %rcx,%rsi                                          ▒
 13.46 │    └──jne    1a5                                                ▒

```

This isn't the entire loop, but it is the core part. All we're doing is shifting a register value over to extract a single bit (our boolean), conditionally adding 0x29 (41 in decimal), then checking if we need to load another 64 booleans stored as individual bits in our 64-bit register by comparing the interation count against 0x3f (63 decimal).

Let's take a look at some other perf statistics to see how things differ between our three different input sizes. First we'll compare the branch missprediction rates (we'll also fix the number of iterations to 1,000,000).

```
2,630,213       branch-misses             #    0.09% of all branches
12,441,999      branch-misses             #    0.20% of all branches
1,367,006,312   branch-misses             #   11.06% of all branches
```

Our number of misses increase by first 6x, then by ~100x! That likely explains our huge drop in performance! It seems our branch predictor gets worse the more we abuse it with random numbers!

## Multiplying by a Boolean

One way we can remove a branch is by integrating a boolean's value into an expression. Let's remove the branch by multiplying the constant, 41, by the boolean (1 or 0). That code looks like this.

```cpp
// Benchmark main loop
while (s.KeepRunning()) {
 for (auto b : v_in) *sink += 41 * b;
}
```
We've completely removed the branch with the price of sometimes multiplying by 0. Here are the timing results.

```
--------------------------------------------------------------------
Benchmark                          Time             CPU   Iterations
--------------------------------------------------------------------
logicBenchBoolNonPower/10       1379 ns         1379 ns      2029115
logicBenchBoolNonPower/11       2807 ns         2803 ns       955068
logicBenchBoolNonPower/12       5482 ns         5481 ns       506292
```

More interesting results! We seem to be slightly worse than the benchmark using a branch for the smaller input sizes, but over 4x better at the large input size! Let's look at the assembly to understand how this loop differs.

```asm
 20.35 │1c8:┌─→shlx   %rax,%rbx,%rdx                                               ▒
  4.65 │    │  and    (%rcx),%rdx                                                  ▒
  8.39 │    │  setne  %dl                                                          ▒
  8.42 │    │  movzbl %dl,%edx                                                     ▒
 17.74 │    │  lea    (%rdx,%rdx,4),%edi                                           ▒
 10.59 │    │  lea    (%rdx,%rdi,8),%edx                                           ▒
 17.89 │    │  add    %edx,%esi                                                    ▒
  1.97 │    │  cmp    $0x3f,%eax                                                   ▒
  0.05 │    │↓ je     248                                                          ▒
  6.65 │    │  inc    %eax                                                         ▒
  0.00 │1e5:├──cmp    %rcx,%r8                                                     ▒
  1.70 │    └──jne    1c8                                                          ▒

```

Some new code! This time, we're extracting the boolean through shift + and, the using a combination of `movzbl` and `lea` instructions to perform `*sink += 41 * b;`. But why is the code only gets faster for the largest input size? Let's start by understanding the amount of work each benchmark does. We'll be using instruction count as a proxy for work done. While instructions don't all have the same overhead, we're looking at fairly similar inner-loops, and the instructions are mostly cheap arithmetic operations. We'll therefore start under the simplifying assumption that the benchmark executing fewer instructions should be faster.

Here are the results for the branch benchmarks.

```
8,754,489,905       instructions              #    3.26  insn per cycle
17,512,225,151      instructions              #    2.87  insn per cycle
34,983,320,195      instructions              #    0.85  insn per cycle
```

And here are the results for our benchmark that integrates the boolean with the addition of a constant.

```
12,341,422,456      instructions              #    2.84  insn per cycle
24,663,237,309      instructions              #    2.85  insn per cycle
49,313,876,329      instructions              #    2.85  insn per cycle
```

Let's try to understand the trends here. It's likely that the branch benchmarks are faster at smaller input sizes simply because we're doing less work (fewer instructions). However, when we misspeculate significantly (>11% misses, as we observed for the large input size), we're silently doing more work by having to rewind executed instructions and restart execution.

For our benchmark that multiplies the constant with a bool, our work scales linearly. This is because we're always performing the same operations, giving us ~100% correctly predicted branches. The largest size only beats the branch benchmark because the branch benchmark beats itself (due to misspeculation)!

## Value-Dependent Performance

Not all constants are treated equally! For both of the previous benchmarks, we've been conditionally adding 41 to out `sink` variable. Let's change our benchmark to add 32 (a power of 2) to the sink variable. Here are the only changes to the inner-loop.

```cpp
// Benchmark main loop
while (s.KeepRunning()) {
  for (auto b : v_in) *sink += 32 * b;
}
```

Let's take a look at the execution times.

```
-----------------------------------------------------------------
Benchmark                       Time             CPU   Iterations
-----------------------------------------------------------------
logicBenchBoolPower/10        919 ns          919 ns      3141008
logicBenchBoolPower/11       1788 ns         1788 ns      1546853
logicBenchBoolPower/12       3554 ns         3554 ns       784270
```

Even more interesting results! Our performance improved by adding a power of 2 value instead of 41. Let's go ahead and take a look at the assembly.

```asm
 18.76 │1c8:   shlx   %rax,%rbx,%rdx
  4.06 │       and    (%rcx),%rdx
  7.34 │       setne  %dl
 10.93 │       movzbl %dl,%edx
 19.84 │       shl    $0x5,%edx
 18.05 │       add    %edx,%esi
  2.89 │       cmp    $0x3f,%eax
  0.05 │     ↓ je     248
  6.59 │       inc    %eax
  0.01 │1e2:   cmp    %rcx,%rdi
  9.91 │     ↑ jne    1c8

```

For our inner-loop, we're setting the lower value of `%edx` based on the boolean, and shifting that value (either 0 or 1) by 5 using `shl`, which creates a value of 32 if the boolean was true (1). We don't have any new branches in our code, and now we're executing fewer instructions than the case where we added 41.

Let's take a look at instruction count information again. Here are the previous results from adding 41.

```
12,341,422,456      instructions              #    2.84  insn per cycle
24,663,237,309      instructions              #    2.85  insn per cycle
49,313,876,329      instructions              #    2.85  insn per cycle
```

And here are the results for when we add a power of two.

```
11,316,660,610      instructions              #    4.27  insn per cycle
22,614,193,920      instructions              #    4.27  insn per cycle
45,208,283,098      instructions              #    4.30  insn per cycle
```

Unsuprising results. We decreased the number of instructions in our inner-loop, which decreased out total number of instructions executed, and decreased execution time. Note, we're still relying on our assumption that our inner-loops are very similar, and that the time all the arithmetic instructions we're looking at take roughly the same amount of time.

Another use-case we can explore is when the value we are conditionally adding is not known at compile-time. Let's take a look at what happens we change this to be a run-time value. For simplicity, I'll just use a value from `DenseRange` (10, 11, and 12).

Here is what the inner-loop looks like. The compiler should be smart enough to hoist loop-invariants, like the value loaded by s.range(0). If you're unsure, you could always just use something like `rand()` to get a random number.

```cpp
// Benchmark main loop
while (s.KeepRunning()) {
  for (auto b : v_in) *sink += s.range(0) * b;
}
```

And here were the performance numbers.

```
-----------------------------------------------------------------
Benchmark                       Time             CPU   Iterations
-----------------------------------------------------------------
logicBenchBoolInput/10       1128 ns         1128 ns      2512900
logicBenchBoolInput/11       2224 ns         2224 ns      1239874
logicBenchBoolInput/12       4485 ns         4485 ns       629310
```

Let's take a look at assembly.

```asm
 13.83 │1be:   shlx   %rax,%rbx,%rdx
  5.49 │       and    (%rcx),%rdx
  5.49 │       setne  %dl
  9.34 │       movzbl %dl,%edx
 27.13 │       imul   %r8,%rdx
 15.98 │       add    %edx,%esi
  2.37 │       cmp    $0x3f,%eax
  0.08 │     ↓ je     240
  7.42 │       inc    %eax
  0.00 │       cmp    %rcx,%rdi
  9.02 │     ↑ jne    1be
```

Now our assembly closely resembles what our high-level C++ does. We extract a boolean that was stored as a bit, check the condition, then multiply the result with our run-time value (stored in a register), and add it to `sink`. We seem to have the same number of instructions as the power of two input (that used a shift), but let's measure the numbers just to be sure.

Here are the number of instructions executed when we were adding a power of two value that is known at compile-time.

```
11,316,660,610      instructions              #    4.27  insn per cycle
22,614,193,920      instructions              #    4.27  insn per cycle
45,208,283,098      instructions              #    4.30  insn per cycle
```

And here are the results when the value is only known at run-time.

```
11,307,458,507      instructions              #    3.26  insn per cycle
22,588,609,872      instructions              #    3.25  insn per cycle
45,153,939,781      instructions              #    3.27  insn per cycle
```

Roughly the same instruction count! Now we have to give up our assumption that arithmetic instructions take approximately the same time. However, we can farily easily root-cause the issue by looking `imul` and `shl` instructions in Agner Fog's [instruction tables](https://www.agner.org/optimize/instruction_tables.pdf). For my processor, `imul` takes 3-4c (depending on the inputs), and that `shl` takes only 0.5c (for the case of a register and immediate input). A likely explanation for why our performance decreased.

But wait! Let's look at the timing results of the non-power of two input and run-time input side-by side.

```
--------------------------------------------------------------------
Benchmark                          Time             CPU   Iterations
--------------------------------------------------------------------
logicBenchBoolNonPower/10       1379 ns         1379 ns      2029115
logicBenchBoolNonPower/11       2807 ns         2803 ns       955068
logicBenchBoolNonPower/12       5482 ns         5481 ns       506292
logicBenchBoolInput/10          1102 ns         1102 ns      2549754
logicBenchBoolInput/11          2194 ns         2194 ns      1277546
logicBenchBoolInput/12          4358 ns         4358 ns       642253
```

Our performance using an `imul` to conditionally add a value to `sink` is faster than the case where we're always adding the same constant (42) using `lea` instructions. Looks like we found at least one scenario where clever choices by the compiler are slower than the straight-forward solution.

## Data-Size-Dependant Performance

We just saw how the compiler's knowledge of the value of constant can change both the generated assembly and performance of an application. Unsurprisingly, the data types we use can have a similar effect. One thing to consider is the space they take up. Let's look at the size of a 2^12 vector (our largest vector size for our benchmarks) storing booleans, chars, and integers. Here are some results I collected on my machine.

```
std::vector<bool> w/ 4096 elements = 512B!
std::vector<char> w/ 4096 elements = 4096B!
std::vector<int>  w/ 4096 elements = 16384B!
```

std::vector<bool> ends up being a specialized implementation that stores booleans as individual bits! We can therefore back 8 booleans into a single byte (like a bit mask), and store all 4096 in 512B. std::vector<char> and <int> end using exactly as much memory as we'd expect. 4096 chars take 4096B, and 4096 ints take 16384B (4 B/int * 4096 ints).

Having a smaller memory footprint can be good for performance, as it means we have a smaller footprint in our caches. However, memory footprint isn't everything. In fact, many optimizations are centered around trading off memory for performance. Let's try a new benchmark where we replace our vector of booleans with an vector of integers holding only the values 0 and 1. For each boolean value, we're now wasting 31 bits.

Here are the performance numbers.

```
-----------------------------------------------------------
Benchmark                 Time             CPU   Iterations
-----------------------------------------------------------
logicBenchInt/10        104 ns          104 ns     27398795
logicBenchInt/11        207 ns          207 ns     13450254
logicBenchInt/12        380 ns          380 ns      7365575
```

Whoah! That's almost 10x faster than our previous best results! Let's go ahead and take a look at the assembly to see why.

```asm
  1.48 │398:┌─→vmovdqu (%rax),%ymm1                                        ▒
 13.26 │    │  add    $0x20,%rax                                           ▒
 31.82 │    │  vpslld $0x2,%ymm1,%ymm0                                     ▒
  0.97 │    │  vpaddd %ymm1,%ymm0,%ymm0                                    ▒
  3.50 │    │  vpslld $0x3,%ymm0,%ymm0                                     ▒
 14.64 │    │  vpaddd %ymm1,%ymm0,%ymm0                                    ▒
 30.54 │    │  vpaddd %ymm0,%ymm2,%ymm2                                    ▒
  0.01 │    ├──cmp    %rax,%rbx                                            ◆
  0.19 │    └──jne    398                                                  ▒
```

Our compiler vectorized the code for us! In our previous benchmarks, the compiler could not vectorize the code because all the booleans were packed in a bit-vector. Each bit therefore has to be extracted using shift and `and` operations. When we're using integers, we don't have this problem, and the compiler can extrace multiple integers that we're using to store booleans using a vector memory operation like `vmovdqu`.

However, we didn't quite make and apples-to-apples comparison. It's not a surprising conclusion exploiting data parallelism can speed up an application. However, the compiler may not want to be able to vectorize this operation for some reason (like aliasing) in a real application. Let's take a step back and and re-run the test with vectorization disabled. All we need to do is add the `-fno-tree-vectorize` to our compiler flags. 

Here were the results.

```
logicBenchIntNonPower/10       1004 ns         1004 ns      2788922
logicBenchIntNonPower/11       2000 ns         2000 ns      1395663
logicBenchIntNonPower/12       3996 ns         3996 ns       700189
```

Better performance than `logicBenchBoolNonPower`, but definitely not the 10x speedup from vectorization. Let's dig into the assembly and see what we find.

```asm
  0.72 │340:┌─→mov    (%rax),%edx                                     ▒
 32.76 │    │  add    $0x4,%rax                                       ▒
  0.51 │    │  lea    (%rdx,%rdx,4),%esi                              ▒
 65.03 │    │  lea    (%rdx,%rsi,8),%edx                              ▒
  0.36 │    │  add    %edx,%ecx                                       ▒
       │    ├──cmp    %rbx,%rax                                       ▒
  0.08 │    └──jne    340                                             ▒
```

Very similar to `logicBenchBoolNonPower`, except we can use the input boolean stored in an integer directly instead of having to extract the condition value from a bit-vector. This likely accounts for the small improvement in performance over the equivilant code for a bool. We also see the same compiler optimization of using `lea` instructions instead of `imul` to conditionally add a constant value of 41 to `sink`.

Let's explore the same train of thought we did for out boolean benchmarks. We'll look at the impact that adding a power of two constant, and a run-time constant to the `sink` variable, but again, using the integer type to hold the condition variable. Here are the timing results.

```
logicBenchIntPower/10           672 ns          672 ns      4158108
logicBenchIntPower/11          1343 ns         1343 ns      2093035
logicBenchIntPower/12          2676 ns         2676 ns      1045922
logicBenchIntInput/10           673 ns          673 ns      4068004
logicBenchIntInput/11          1337 ns         1337 ns      2094360
logicBenchIntInput/12          2668 ns         2668 ns      1048372
```

What we find in the assembly for both are extremely tight loops. Here is the loop for the power of two input.

```asm
  0.44 │340:┌─→mov    $0x5,%edx                                    ▒
 47.96 │    │  shlx   %edx,(%rax),%edx                             ▒
  0.06 │    │  add    $0x4,%rax                                    ▒
 50.77 │    │  add    %edx,%ecx                                    ▒
       │    ├──cmp    %rbx,%rax                                    ▒
  0.09 │    └──jne    340                                          ▒
```

We can see that each iteration we simply shifts the boolean input left by 5 (to create either 0 or 32), and add the result to `sink`.

Here is the assembly when a constant with a value determined at run-time.

```
  0.56 │348:┌─→mov    (%rax),%ecx                                  ▒
  0.17 │    │  add    $0x4,%rax                                    ▒
 50.82 │    │  imul   %esi,%ecx                                    ▒
 47.85 │    │  add    %ecx,%edx                                    ▒
       │    ├──cmp    %rax,%rbx                                    ▒
  0.11 │    └──jne    348                                          ▒
```

All we're doing is multiplying the input variable by the run-time constant, and adding the result to `sink`.

To change gears, we again going to re-focus on type-based performance differences instead of value-based ones. We saw with an integer we get some performance improvement from vectorization or not having to unpack boolean bit from a bit-vector. However, we're only using 1/32 bits in an integer. If we're only using the values 1 and 0, we could use a smaller data type. Let's use a char! 1/4 the size of an integer, but won't require bit-shifting and mask operations to extract the value.

Let's re-enable vectorization and try a vector of chars with the value of either 0 or 1. Here were my performance numbers.

```
logicBenchCharNonPower/10       94.5 ns         94.5 ns     29632602
logicBenchCharNonPower/11        189 ns          189 ns     14861750
logicBenchCharNonPower/12        378 ns          378 ns      7397448
```

Very similar results to using an integer! Here's what the assmebly looked like.

```
  0.53 │388:┌─→vmovdqu (%rax),%ymm2                                                    ▒
  0.35 │    │  add    $0x20,%rax                                                       ▒
  9.88 │    │  vpmovsxbw %xmm2,%ymm3                                                   ▒
  0.31 │    │  vpmullw 0x41173(%rip),%ymm3,%ymm3        # 449c40 <_IO_stdin_used+0x200>▒
  0.46 │    │  vextracti128 $0x1,%ymm2,%xmm2                                           ▒
  1.73 │    │  vpmovsxwd %xmm3,%ymm1                                                   ▒
 10.54 │    │  vextracti128 $0x1,%ymm3,%xmm3                                           ▒
  3.52 │    │  vpaddd %ymm0,%ymm1,%ymm0                                                ▒
  0.33 │    │  vpmovsxbw %xmm2,%ymm2                                                   ▒
  0.35 │    │  vpmovsxwd %xmm3,%ymm3                                                   ▒
 16.38 │    │  vpaddd %ymm0,%ymm3,%ymm3                                                ▒
  0.29 │    │  vpmullw 0x41148(%rip),%ymm2,%ymm2        # 449c40 <_IO_stdin_used+0x200>▒
  6.23 │    │  vpmovsxwd %xmm2,%ymm0                                                   ▒
  2.15 │    │  vextracti128 $0x1,%ymm2,%xmm2                                           ▒
 16.29 │    │  vpaddd %ymm3,%ymm0,%ymm0                                                ▒
  0.18 │    │  vpmovsxwd %xmm2,%ymm2                                                   ◆
 27.36 │    │  vpaddd %ymm0,%ymm2,%ymm0                                                ▒
  0.01 │    ├──cmp    %r12,%rax                                                        ▒
  0.27 │    └──jne    388                                                              ▒
```

Vectorization again, but significantly more instructions than with an integer. This is likely because each 128-bit register can now hold 16 booleans stored as chars instead of only 4 booleans stored as integers. While we have more instruction, we can make a greater amount of forward progress.

Once again, let's take a step back and disable vectorization to make an apples-to-apples comparison. Here are my three results.

```
--------------------------------------------------------------------
Benchmark                          Time             CPU   Iterations
--------------------------------------------------------------------
logicBenchCharNonPower/10        637 ns          637 ns      4394048
logicBenchCharNonPower/11       1265 ns         1265 ns      2214036
logicBenchCharNonPower/12       2521 ns         2520 ns      1110496
logicBenchCharPower/10           474 ns          474 ns      5928468
logicBenchCharPower/11           939 ns          939 ns      2988854
logicBenchCharPower/12          1867 ns         1866 ns      1499018
logicBenchCharInput/10           494 ns          494 ns      5704480
logicBenchCharInput/11           975 ns          974 ns      2875398
logicBenchCharInput/12          1945 ns         1944 ns      1443183
```

Interesting! A similar trend to what we saw with integers (but slightly faster)! Let's take a quick look at the instructions to see if we can understand why. 

Here is for our non power of two constant.

```asm
 19.48 │328:┌─→movsbl (%rax),%edx                                                      ▒
  0.29 │    │  inc    %rax                                                             ▒
 36.17 │    │  lea    (%rdx,%rdx,4),%esi                                               ▒
  6.71 │    │  lea    (%rdx,%rsi,8),%edx                                               ▒
 36.49 │    │  add    %edx,%ecx                                                        ▒
       │    ├──cmp    %rax,%rbx                                                        ▒
  0.15 │    └──jne    328                                                              ▒
```

We see the same types of optimizations as the other two cases (with `lea`).

Let's examing the other two cases to see if the follow the same trends as bool and int. Here is the listing for the power of two constant.

```asm
 19.81 │328:┌─→movsbl (%rax),%edx                                                      ▒
 17.87 │    │  inc    %rax                                                             ▒
 24.09 │    │  shl    $0x5,%edx                                                        ▒
 22.84 │    │  add    %edx,%ecx                                                        ▒
  0.02 │    ├──cmp    %rax,%rbx                                                        ▒
 14.35 │    └──jne    328                                                              ▒

```

And finally, for the run-time input.

```asm
 16.95 │338:┌─→movsbq (%rax),%rdx                                                      ▒
 17.02 │    │  inc    %rax                                                             ▒
 27.05 │    │  imul   %rsi,%rdx                                                        ▒
 25.09 │    │  add    %edx,%ecx                                                        ▒
  0.01 │    ├──cmp    %rax,%rbx                                                        ▒
 13.12 │    └──jne    338                                                              ▒
```

Very similar code! Our performance gainst are likely coming from the fact that we need 1/4 the number of iterations of our loop because our data is 4x as dense! For a third time, the compiler's choice of using `lea` instead of `imul` seems to be a poor choice.

## Concluding Remarks

Every part of program can impact performance. From the immediate values of constants to the data types we use to store values. Understanding how tease-out better performance from the compiler is often an art that is very feedback-based. In this blog we showed that how we conditionally add a constant value can vary significantly in performance. We showed that htis performance can be based on both the types of the inputs, and the actual data values.

We also showed that the compiler does not always make the right decision for performance. While this is certainly not an attack on compiler writers, it is a call for those that care about performance to remain vigilant as to what your compilers are doing, and that it's never a bad idea to sanity check.

As always, feel free to contact me with questions. 

Cheers,

--Nick

## Links

- [My YouTube Channel](https://www.youtube.com/channel/UCsi5-meDM5Q5NE93n_Ya7GA?view_as=subscriber)
- [My GitHub Account](https://github.com/CoffeeBeforeArch)
- My Email: CoffeeBeforeArch@gmail.com
- [Dot Product Benchmarks](https://github.com/CoffeeBeforeArch/misc_code/blob/master/dot_product/dp.cpp)
