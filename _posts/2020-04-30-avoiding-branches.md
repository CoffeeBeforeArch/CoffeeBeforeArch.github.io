---
layout: default
title: Avoiding Branches
---

# Avoiding Branches

Modern processors speculate whether or not a branch is taken, and target address of the next instruction. If a processor speculates incorrectly, it must rewind all incorrectly executed instructions, and proceed down the correct control-flow path. This can lead to significant performance problems when a branch is difficult to predict, and the processor misspeculates frequently. While the compiler can remove some branches using techniques like scalarization, many can only be removed by direct programmer-intervention.

In this blog post, we'll be looking at how we can remove branches by integrating a condition with a mathematical operation. We will also be discussing value-dependant and type-dependant performance.

## Links

- [My YouTube Channel](https://www.youtube.com/channel/UCsi5-meDM5Q5NE93n_Ya7GA?view_as=subscriber)
- [My GitHub Account](https://github.com/CoffeeBeforeArch)
- [Benchmarks](https://github.com/CoffeeBeforeArch/misc_code/blob/master/conditions)
- My Email: CoffeeBeforeArch@gmail.com

## Compiling the Code

All the benchmarks used in this blog post can be found [here](https://github.com/CoffeeBeforeArch/misc_code/blob/master/conditions), and were writting using [Google Benchmark](https://github.com/google/benchmark). Below is the command I used to compile the benchmarks.

```bash
g++ bench_name.cpp -O3 -lbenchmark -lpthread -march=native -mtune=native
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
  std::vector<some_type> v_in(N);
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
BENCHMARK(branchBench)->DenseRange(some_range);
```

For each benchmark, we create a vector of boolean values, and initialize them using a pseudo-random number generator. Inside of our main benchmarking loop, we're going to be testing the performance of conditionally adding a constant value to our `sink` variable. `sink` was dynamically allocated to keep it from being removed by the compiler, but retain the optimizations that were disabled when I marked it as `volatile` or with `benchmark::DoNotOptimize(...)`. We'll test various input sizes (all powers of two).

## Getting to a Baseline Benchmark

For our baseline, we will simply use an if-statement to conditionally add a value to our `sink` variable.

```cpp
// Benchmark main loop
while (s.KeepRunning()) {
  for (auto b : v_in)
    if (b) *sink += 41;
}
```

Let's run our benchmark and see what we get as results.

```
---------------------------------------------------------------
Benchmark                     Time             CPU   Iterations
---------------------------------------------------------------
branchBenchRandom/10        814 ns          814 ns      3236121
branchBenchRandom/11       1825 ns         1823 ns      1604909
branchBenchRandom/12      13262 ns        13261 ns       210503
```

Some interesting results right from the start. Increasing the number of operations from 2^10 to 2^11 just over doubled the execution time. However, doubling the number of booleans again from 2^11 to 2^12 increased the execution time by over 7x! Let's first take a look at some hot-spot information at the assembly level using the [perf Linux profiler](https://perf.wiki.kernel.org/index.php/Main_Page).

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

What we're looking at is the core inner-loop (where the conditional add occurs). We're shifting a register value over to extract a single bit (our boolean), conditionally adding 0x29 (41 in decimal) based on the value, then checking if we need to load another 64 booleans stored as individual bits in a 64-bit register by comparing the interation count against 0x3f (63 decimal).

Let's take a look at some branch predicition statistics to see how things differ with our three different input sizes. I will also fix the number of iterations to 1,000,000. That can done by adding `->Iterations(1000000)` to the line where the benchmark is registered.

```
2,630,213       branch-misses             #    0.09% of all branches
12,441,999      branch-misses             #    0.20% of all branches
1,367,006,312   branch-misses             #   11.06% of all branches
```

_Very_ unexpected results. We'd expect that our branch predictor would always do a poor job with our benchmark when the input data is random. However, it seems to do a good job when our input vector size is small (N < 2^12). Do we blame the random number generator? Probably not. We're already using `std::mt19937` instead of `rand()`. What we should be thinking about is the way Google Benchmark runs our code.

The benchmark will repeat whatever is in the `for (auto _ : s )` loop until either some internal measurements stabilize, or a fixed time or number of iterations has completed. However, our random number generation occurs outside this loop. While we may have high-quality random numbers and are averaging timing across millions of iterations, each iteration is using *the exact same random numbers*. That means each iteration of the `for (auto _ : s)` loop will have the exact same branch outcomes, and the branch predictor unit eventually learns what they are.

We can test this hypothesis using two more benchmarks. We'll now look at `branchBenchFalse` and `branchBenchTrue`. In these benchmarks, our loop either never adds the constant value to `sink`, or always does. This should be a trivial for our branch predictor unit to learn. Here are the timing results for the smallest vector size (2^10).

```
----------------------------------------------------------------
Benchmark                      Time             CPU   Iterations
----------------------------------------------------------------
branchBenchFalse/10         1198 ns         1198 ns      2339510
branchBenchTrue/10          1761 ns         1761 ns      1589511
branchBenchRandom/10         842 ns          842 ns      3283319
```

And here are the number of branch misses.

```
False:     24,883,530      branch-misses          #    0.81% of all branches
True:       1,145,795      branch-misses          #    0.04% of all branches
Random:     2,379,407      branch-misses          #    0.08% of all branches
```

`branchBenchFalse` and `branchBenchTrue` should have almost no misses because the because the branch always goes a single direction. `branchBenchRandom` has nearly 0 misses as well. This is because branch predictor unit learns the branch outcomes from first few iterations of our benchmark (that all use the same input data).

Branch predictor units (BPUs) are effective, but have their limits (i.e., the have a fixed amount of storage for branch history and targets). For our benchmark, it can "remember" the results of 2^10 and 2^11 iterations, but does a poor job with 2^12. Let's compare the results of the three benchmarks for the largest vector size. 

```
---------------------------------------------------------------
Benchmark                     Time             CPU   Iterations
---------------------------------------------------------------
branchBenchFalse/12        4814 ns         4813 ns       583745
branchBenchTrue/12         5526 ns         5524 ns       516838
branchBenchRandom/12      12981 ns        12980 ns       216861
```

And here are the number of branch misses.

```
 False:    95,850,193      branch-misses          #    0.78% of all branches
  True:     1,373,892      branch-misses          #    0.01% of all branches
Random: 1,352,117,345      branch-misses          #   11.00% of all branches
```

Still almost 0 misses for our false and true benchmarks, but a significant number for our random one! Since our benchmark uses the same vector of booleans each iteration, we need to ensure that vector is of a sufficient size such that the the BPU can't easily learn the outcomes from previous iterations. Otherwise our input is no longer "random", just a pattern that can be "memorized".

But wait! We're skipped over another interesting result. Our `branchBenchRandom` was faster than `branchBenchFalse` for small vector sizes! `branchBenchRandom` sometimes skips adding our constant value to `sink`, so while we may expect it to be faster than `branchBenchTrue` (which always performs the addition), we would expect it to be slower than `branchBenchFalse` (that never performs the addition)!

Modern Intel pipelines can get uops from a Decoded Stream Buffer (DSB), or a slower Micro-instruction Translation Engine (MITE). Let's see which one is providing the uops to our processor's backend. Here are the results for `branchBenchFalse`.

```
     2,265,959,520      idq.dsb_uops                      
     3,368,665,296      idq.mite_uops                    
```

And here are the results for `branchBenchRandom`.

```
     6,294,916,475      idq.dsb_uops                                 
        15,795,857      idq.mite_uops                               
```

Our random benchmark avoids many of the penalites in the legacy MITE pipeline, leading to slightly better performance, despite sometimes performing the addition. Our false benchmark seems to get most of its instructions from the MITE. We'll leave any further discussion of this issue to another blog post.

Our key takeaway from this section should be our baseline benchmark results. Vector sizes of 2^10 and 2^11 don't work because the BPU can learn the branch outcomes. We'll therefore be using the sizes 2^12, 2^13, and 2^14. Here are the timing results (now in microseconds).

```
---------------------------------------------------------------
Benchmark                     Time             CPU   Iterations
---------------------------------------------------------------
branchBenchRandom/12       13.1 us         13.1 us       214146
branchBenchRandom/13       31.8 us         31.8 us        88274
branchBenchRandom/14       69.2 us         69.2 us        40548
```

## Multiplying by a Boolean

One way we can remove a branch is by integrating a boolean's value into an expression. Multiplication and addition are cheap (compared to miss-speculation) so we may see a performance improvement by always performing these operations instead of conditionally skipping them.

Let's remove the branch by multiplying the constant, 41, by the boolean (1 or 0). Here's the inner benchmark loop.

```cpp
// Benchmark main loop
while (s.KeepRunning()) {
 for (auto b : v_in) *sink += 41 * b;
}
```

We've completely removed the branch with the price of sometimes multiplying by 0. Here are the timing results.

```
---------------------------------------------------------------
Benchmark                     Time             CPU   Iterations
---------------------------------------------------------------
boolBenchNonPower/12       4.11 us         4.11 us       688606
boolBenchNonPower/13       8.09 us         8.09 us       345301
boolBenchNonPower/14       16.1 us         16.1 us       173933
```

We're about 3 (or more) times faster! Let's look at the assembly to what the low-level differences are.

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

Some new code! We're extracting the boolean through `shlx` + `and` + `setne`, and using a combination of `movzbl` + `lea` + `add` to perform `*sink += 41 * b;`.

## Value-Dependent Performance

Not all constants are treated equally! For both of the previous benchmarks, we've been conditionally adding 41 to our `sink` variable. Let's change our benchmark to add 32 (a power of 2) to the sink variable. Here are the only changes to the inner-loop.

```cpp
// Benchmark main loop
while (s.KeepRunning()) {
  for (auto b : v_in) *sink += 32 * b;
}
```

Let's take a look at the execution times.

```
------------------------------------------------------------
Benchmark                  Time             CPU   Iterations
------------------------------------------------------------
boolBenchPower/12       3.42 us         3.42 us       819065
boolBenchPower/13       6.91 us         6.91 us       410612
boolBenchPower/14       14.2 us         14.2 us       196806
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

For our inner-loop, we've replaced the `lea` instructions that compute the value 41 with a left shift of the boolean (either 0 or 1) by 5 using `shl`, which creates a value of 32 if the boolean was true (1). We still removed the branch, and now we're executing fewer instructions.

Let's take a look at instruction counts of the two benchmarks Here are the previous results from adding 41 (for a set number of iterations).

```
 9,863,822,693      instructions
19,721,451,501      instructions
39,439,292,114      instructions
```

And here are the results for when we add a power of two.

```
 9,045,005,043      instructions
18,083,435,007      instructions
36,161,678,782      instructions
```

Unsuprising results. We decreased the number of instructions executed in our inner-loop, which decreased out total number of instructions executed, and decreased execution time. To make this statement, we have to use the assumption that the time it takes to execute instructions is roughly equivilant. This assumption works decently well when comparing these two loops because they are quite similar, and we don't have any extroadinarily expensive operations (like `idiv`) that would obviously break this assumption.

Another scenario we can explore is when the value we are conditionally adding is not known at compile-time. Let's take a look at what happens we change this to be a run-time value. For simplicity, I'll just use a value from `DenseRange` (10, 11, and 12). The important thing here is not the actual value of the number, just that the compiler does not know the value at compile-time.

Here is what the inner-loop looks like. The compiler should be smart enough to hoist loop-invariants, like the value loaded by `s.range(0)`. If you're unsure, you could always just use something like `rand()` to get a random number to add.

```cpp
// Benchmark main loop
while (s.KeepRunning()) {
  for (auto b : v_in) *sink += s.range(0) * b;
}
```

And here were the performance numbers.

```
------------------------------------------------------------
Benchmark                  Time             CPU   Iterations
------------------------------------------------------------
boolBenchInput/12       3.90 us         3.90 us       720875
boolBenchInput/13       7.82 us         7.82 us       360674
boolBenchInput/14       15.6 us         15.6 us       178741
```

Slower than adding a power of two. Let's see why by looking at the assembly.

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

Our assembly closely resembles our high-level C++. We extract a boolean that was stored as a bit, check the condition, then multiply the result with our run-time input, and add it to `sink`. We seem to have the same number of instructions as the power of two input case (that used a shift), but let's measure the numbers just to be sure (using a constant number of iterations again).

Here are the number of instructions counts again for the power-of-two value known at compile-time.

```
 9,045,005,043      instructions
18,083,435,007      instructions
36,161,678,782      instructions
```

And here are the results when the value is only known at run-time.

```
 9,046,302,760      instructions
18,084,190,742      instructions
36,160,339,674      instructions
```

Roughly the same instruction count! Now we have to give up our assumption that arithmetic instructions take approximately the same time. However, we can farily easily root-cause the issue by looking `imul` and `shl` instructions in Agner Fog's [instruction tables](https://www.agner.org/optimize/instruction_tables.pdf). For my processor, `imul` takes 3-4c (depending on the inputs), and that `shl` takes only 0.5c (for the case of a register and immediate input). A likely explanation for why our performance decreased.

But wait! Let's look at the timing results of the non-power of two input and run-time input side-by side.

```
--------------------------------------------------------------------
Benchmark                     Time             CPU   Iterations
--------------------------------------------------------------------
boolBenchNonPower/12       4.11 us         4.11 us       688606
boolBenchNonPower/13       8.09 us         8.09 us       345301
boolBenchNonPower/14       16.1 us         16.1 us       173933
boolBenchInput/12          3.90 us         3.90 us       720875
boolBenchInput/13          7.82 us         7.82 us       360674
boolBenchInput/14          15.6 us         15.6 us       178741
```

Our performance using an `imul` to conditionally add a value to `sink` is faster than the case where we're always adding the same constant, 41, that the compiler cleverly added using multiple `lea` instructions. Looks like we found at least one scenario where clever choices by the compiler are slower than the straight-forward solution.

## Data-Size-Dependant Performance

We just saw how the compiler's knowledge of a value can change both the generated assembly and performance of an application. Unsurprisingly, the data types can have a similar effect. One thing to consider is the space they take up. Let's look at the size of a 2^12 vector storing booleans, chars, and integers. Here are some results I collected on my machine.

```
std::vector<bool> w/ 4096 elements = 512B!
std::vector<char> w/ 4096 elements = 4096B!
std::vector<int>  w/ 4096 elements = 16384B!
```

std::vector<bool> ends up being a specialized implementation that stores booleans as individual bits (a bit-vector)! We can therefore pack 8 booleans into a single byte (like a bit mask), and store all 4096 in only 512B. std::vector<char> and <int> end up using exactly as much memory as we'd expect. 4096 chars take 4096B, and 4096 ints take 16384B (4 B/int * 4096 ints).

Having a smaller memory footprint can be good for performance, as it means we have a smaller footprint in our caches. However, memory footprint isn't everything. In fact, many optimizations are centered around using extra memory to improve performance. Let's start at the extreme. We'll create a new benchmark where we replace our vector of booleans with an vector of integers that are only values 0 and 1. To represent each boolean, we're now wasting 31 bits.

Here are the performance numbers.

```
--------------------------------------------------------------
Benchmark                    Time             CPU   Iterations
--------------------------------------------------------------
intBenchNonPower/12      0.349 us        0.349 us      8053063
intBenchNonPower/13      0.709 us        0.709 us      3960530
intBenchNonPower/14       1.83 us         1.82 us      1593989
```

Whoah! That's roughly 10x faster than our previous best results! Let's go ahead and take a look at the assembly to see why.

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

Our compiler vectorized the code for us! In our previous benchmarks, the compiler could not vectorize the code because all the booleans were packed in a bit-vector. Each bit therefore has to be extracted using shift and `and` operations. When we're using integers, we don't have this problem, and the compiler can extract multiple integers that we're using to store booleans using a vector memory operation like `vmovdqu`.

However, we didn't quite make and apples-to-apples comparison. It's not a surprising conclusion that exploiting data parallelism can speed up an application. However, the compiler may not want to be able to vectorize this operation for some reason (like aliasing) in a real application. Let's take a step back and and re-run the test with vectorization disabled. All we need to do is add the `-fno-tree-vectorize` to our compiler flags. 

Here were the results.

```
--------------------------------------------------------------
Benchmark                    Time             CPU   Iterations
--------------------------------------------------------------
intBenchNonPower/12       2.53 us         2.53 us      1103132
intBenchNonPower/13       5.16 us         5.16 us       548831
intBenchNonPower/14       10.1 us         10.1 us       274542
```

Significantly faster than using a boolean, but definitely not the 10x speedup from vectorization. Let's dig into the assembly and see what we find.

```asm
  0.72 │340:┌─→mov    (%rax),%edx                                     ▒
 32.76 │    │  add    $0x4,%rax                                       ▒
  0.51 │    │  lea    (%rdx,%rdx,4),%esi                              ▒
 65.03 │    │  lea    (%rdx,%rsi,8),%edx                              ▒
  0.36 │    │  add    %edx,%ecx                                       ▒
       │    ├──cmp    %rbx,%rax                                       ▒
  0.08 │    └──jne    340                                             ▒
```

Very similar to `boolBenchNonPower`, except we can use the input boolean stored in an integer directly instead of having to extract the condition value from a bit-vector! This likely accounts for the small improvement in performance over the equivilant code for a bool. We also see the same compiler optimization of using `lea` instructions instead of `imul` to conditionally add a constant value of 41 to `sink`.

Let's explore the same train of thought we did for out boolean benchmarks. We'll look at two more benchmarks. One that adds a power-of-two constant, and another that adds a value known only at run-time. For both of these cases, we'll be using a vector of integers. Here are the results.

```
-----------------------------------------------------------
Benchmark                 Time             CPU   Iterations
-----------------------------------------------------------
intBenchPower/12       2.10 us         2.10 us      1000000
intBenchPower/13       4.22 us         4.22 us      1000000
intBenchPower/14       8.45 us         8.45 us      1000000
intBenchInput/12       1.98 us         1.98 us      1000000
intBenchInput/13       4.02 us         4.02 us      1000000
intBenchInput/14       7.88 us         7.87 us      1000000
```

In the assembly, we find that both benchmarks are extremely tight loops. Here is the loop for the power-of-two input.

```asm
  0.44 │340:┌─→mov    $0x5,%edx                                    ▒
 47.96 │    │  shlx   %edx,(%rax),%edx                             ▒
  0.06 │    │  add    $0x4,%rax                                    ▒
 50.77 │    │  add    %edx,%ecx                                    ▒
       │    ├──cmp    %rbx,%rax                                    ▒
  0.09 │    └──jne    340                                          ▒
```

We can see that each iteration we shifts the boolean input left by 5 (to create either 0 or 32), and adds the result to `sink`.

Here is the assembly when a constant with a value only known at run-time.

```
  0.56 │348:┌─→mov    (%rax),%ecx                                  ▒
  0.17 │    │  add    $0x4,%rax                                    ▒
 50.82 │    │  imul   %esi,%ecx                                    ▒
 47.85 │    │  add    %ecx,%edx                                    ▒
       │    ├──cmp    %rax,%rbx                                    ▒
  0.11 │    └──jne    348                                          ▒
```

All we're doing is multiplying the input variable by the run-time constant, and adding the result to `sink`.

An interesting result here is that again, our generalized implementation (using `imul`) seems outperform clever specializations by the compiler (like using `lea` and even `shlx` in this case). Both implementations are also faster than adding a non-power-of-two.

To change gears, we are going to re-focus on type-based performance differences. We saw that with an integer we get performance improvement from vectorization or not having to unpack boolean bit from a bit-vector. However, we're only using 1/32 bits in an integer. If we're only using the values 1 and 0, we could use a smaller data type. Let's use a char! It's 1/4 the size of an integer, and also will not require bit-shifting and mask operations to extract the value (as needed with the bit-vector).

Let's re-enable vectorization and try a vector of chars that only hold the values 0 or 1. Here were my performance numbers.

```
---------------------------------------------------------------
Benchmark                     Time             CPU   Iterations
---------------------------------------------------------------
charBenchNonPower/12      0.382 us        0.382 us      7376064
charBenchNonPower/13      0.762 us        0.762 us      3665293
charBenchNonPower/14       1.52 us         1.52 us      1842032
```

Very similar results to using an integer, and even slightly better at the largest input size. Here is the assmebly generated

```
  0.52 │398:┌─→vmovdqu (%rdx),%ymm2                                        ▒
  1.06 │    │  add    $0x20,%rdx                                           ▒
  8.83 │    │  vpmovsxbw %xmm2,%ymm6                                       ▒
  0.43 │    │  vpmullw %ymm3,%ymm6,%ymm6                                   ▒
  0.50 │    │  vextracti128 $0x1,%ymm2,%xmm2                               ▒
  1.10 │    │  vpmovsxbw %xmm2,%ymm2                                       ▒
 11.43 │    │  vpmullw %ymm3,%ymm2,%ymm2                                   ▒
  1.11 │    │  vpmovsxwd %xmm6,%ymm1                                       ▒
  1.06 │    │  vextracti128 $0x1,%ymm6,%xmm6                               ▒
  4.49 │    │  vpaddd %ymm0,%ymm1,%ymm0                                    ▒
  9.84 │    │  vpmovsxwd %xmm6,%ymm6                                       ▒
 16.24 │    │  vpaddd %ymm0,%ymm6,%ymm6                                    ▒
  1.54 │    │  vpmovsxwd %xmm2,%ymm0                                       ▒
  0.90 │    │  vextracti128 $0x1,%ymm2,%xmm2                               ▒
 13.91 │    │  vpaddd %ymm6,%ymm0,%ymm0                                    ▒
  2.05 │    │  vpmovsxwd %xmm2,%ymm2                                       ▒
 23.97 │    │  vpaddd %ymm0,%ymm2,%ymm0                                    ▒
       │    ├──cmp    %rdx,%rcx                                            ◆
  0.08 │    └──jne    398                                                  ▒

```

Vectorization again, but significantly more instructions than with an integer. This is likely because each 128-bit register can now hold 16 booleans stored as chars instead of only 4 booleans stored as integers. While we have more instruction, we're processing many more elements each iteration of the loop.

Let's take a step back and disable vectorization to make an apples-to-apples comparison. Here are my three results (for a non-power-of-two, a power-of-two, and a run-time value input).

```
---------------------------------------------------------------
Benchmark                     Time             CPU   Iterations
---------------------------------------------------------------
charBenchNonPower/12       2.55 us         2.55 us      1112562
charBenchNonPower/13       5.13 us         5.13 us       546261
charBenchNonPower/14       10.2 us         10.2 us       273814
charBenchPower/12          1.86 us         1.86 us      1500524
charBenchPower/13          3.74 us         3.74 us       750482
charBenchPower/14          7.53 us         7.53 us       373262
charBenchInput/12          1.95 us         1.95 us      1440667
charBenchInput/13          3.88 us         3.88 us       724444
charBenchInput/14          7.76 us         7.76 us       356929

```

Similar performance to using integers, but with 1/4 the storage requirement (and slightly faster in some cases)! Let's take a quick look at the instructions to see if we can understand why. 

Here is for our non power of two constant.

```asm
  0.57 │320:┌─→movsbl (%rax),%edx                                ▒
 33.15 │    │  inc    %rax                                       ▒
  0.34 │    │  lea    (%rdx,%rdx,4),%esi                         ▒
 65.66 │    │  lea    (%rdx,%rsi,8),%edx                         ▒
  0.17 │    │  add    %edx,%ecx                                  ▒
       │    ├──cmp    %rax,%rbx                                  ▒
  0.05 │    └──jne    320                                        ▒
```

We see the same types of optimizations as the other two cases (with `lea`).

Let's examing the other two cases to see if the follow the same trends as bool and int. Here is the listing for the power of two constant.

```asm
 20.40 │320:┌─→movsbl (%rax),%edx                                ▒
 18.00 │    │  inc    %rax                                       ▒
 23.89 │    │  shl    $0x5,%edx                                  ▒
 22.84 │    │  add    %edx,%ecx                                  ▒
       │    ├──cmp    %rax,%rbx                                  ▒
 14.60 │    └──jne    320                                        ▒
```

And finally, for the run-time value.

```asm
 17.04 │330:┌─→movsbq (%rax),%rdx                                ▒
 17.97 │    │  inc    %rax                                       ▒
 26.09 │    │  imul   %rsi,%rdx                                  ▒
 24.56 │    │  add    %edx,%ecx                                  ▒
       │    ├──cmp    %rax,%rbx                                  ▒
 14.09 │    └──jne    330                                        ▒
```

Very similar code to what we saw for integers! Any performance differece is likely coming from our smaller usage of memory. Furthermore, the compiler's choice of using `lea` instead of `imul` again seems to be a poor choice. While we _can_ see performance improvements with specializations (as seen with the power-of-two inputs), using `imul` generally seems to be better than constructing the value using `lea` instructions.

As a reminder, here were the results from our original branch benchmarks.

```
---------------------------------------------------------------
Benchmark                     Time             CPU   Iterations
---------------------------------------------------------------
branchBenchRandom/12       13.1 us         13.1 us       214146
branchBenchRandom/13       31.8 us         31.8 us        88274
branchBenchRandom/14       69.2 us         69.2 us        40548
```

It looks like we're seeing an almost 10x performance improvement (depending on the input size) by integrating the comparison in with the computation, and understanding the impact of values and data-types on performance. Pretty cool stuff.

## Concluding Remarks

Many interesting results to think about! Our choice of values and data types can impact performance significantly. It also is a reminder for us to be vigilant about performance. Just because the compiler specializes a function because it knows a value at compile-time doesn't mean it will be better performance. This was the case for using `lea` instructions instead `imul` in all three examples.

Another important thing to consider is how we benchmark code. I've seen large C++ conference talks fall for our baseline pitfall of using the same random numbers each iteration. Furthermore, I found huge performance variation (>10%) depending on which benchmarks were located in the same file, and which settings were enabled (e.g., the number of iterations). So be careful. Even if you're measuring using great tools, that doesn't mean there are not other issues hiding in your benchmarks.

As always, feel free to contact me with questions.

Cheers,

--Nick

## Links

- [My YouTube Channel](https://www.youtube.com/channel/UCsi5-meDM5Q5NE93n_Ya7GA?view_as=subscriber)
- [My GitHub Account](https://github.com/CoffeeBeforeArch)
- [Benchmarks](https://github.com/CoffeeBeforeArch/misc_code/blob/master/conditions)
- My Email: CoffeeBeforeArch@gmail.com
