---
layout: default
title: Biased Branches
---

# Biased Branches

When a branch is biased (e.g., mostly taken or mostly not-taken), we can inform the compiler about this things like GCC's `__builtin_expect`, or the more portable C++20 attribute `[[likely]]`/`[[unlikely]]`. While providing good hints and improve code scheduling and performance, providing bad hints can worsen performance.

In this blog post, we will look at how performance can vary depending on how a branch is biased, then explore what benefits we can get from compiler hints. We will then compare the results to branchless versions of our benchmarks, and analyze the tradoffs between the implementations.

### Link to the source code

- [Source Code: ](https://github.com/CoffeeBeforeArch/misc_code/tree/master/biased_branches)
- [My YouTube Channel: ](https://www.youtube.com/channel/UCsi5-meDM5Q5NE93n_Ya7GA?view_as=subscriber)
- [My GitHub Account: ](https://github.com/CoffeeBeforeArch)
- My Email: CoffeeBeforeArch@gmail.com


# Conditional Add

## Baseline

The benchmark we'll look at in this post is a simple for-loop that conditionally adds a constant value to a total based on outcomes stored in a `std::vector<bool>`:

```cpp
for (auto b : v_in)
  if (b) *sink += s.range(0);
```

The outcomes in our `std::vector<bool>` were generated using the following random number generator:

```cpp
// Get the distribution
double skew = s.range(1) / 100.0;

// Create random number generator
std::random_device rd;
std::mt19937 gen(rd());
std::bernoulli_distribution d(skew);

// Create a vector of random booleans
std::vector<bool> v_in(N);
std::generate(begin(v_in), end(v_in), [&]() { return d(gen); });
```

Our outcomes are generated using `std::bernoulli_distribution` We can pass a decimal between `0.0` and `1.0` to our `std::bernoulli_distribution` to skew the likelihood of generating `true`/`false` outcomes. For example, an input of `0.5` means that `true` and `false` are equally likely (50%), while an input of `0.1` means that ~90% of the time we'll generate a `false` outcome.

Let's take a look at the performance where we vary the skew from `0.1` to `0.9`:

```text
------------------------------------------------------------------
Benchmark                        Time             CPU   Iterations
------------------------------------------------------------------
branchBenchRandom/14/10       30.2 us         30.2 us        46742
branchBenchRandom/14/20       44.2 us         44.2 us        31818
branchBenchRandom/14/30       61.9 us         61.9 us        22675
branchBenchRandom/14/40       78.2 us         78.2 us        17904
branchBenchRandom/14/50       85.8 us         85.8 us        16262
branchBenchRandom/14/60       82.1 us         82.0 us        17242
branchBenchRandom/14/70       65.9 us         65.9 us        21906
branchBenchRandom/14/80       45.5 us         45.5 us        30706
branchBenchRandom/14/90       32.0 us         32.0 us        44435
```

And now let's take a look at where we're spending our time in the assembly:

```assembly
  5.50 │1d0:┌─→shlx   %rax,%rdi,%rdx          
  4.75 │    │  and    (%rcx),%rdx                                 
  7.91 │    │↓ je     1ef                                       
 28.34 │    │  mov    0x20(%rbp),%rdx                             
  3.55 │    │  cmp    %rdx,0x28(%rbp)                             
  0.00 │    │↓ je     373                                          
  6.89 │    │  mov    (%rdx),%rdx                                 
 15.79 │    │  add    %edx,(%r12)                                  
 21.03 │1ef:│  cmp    $0x3f,%eax                                      
  0.38 │    │↓ je     260      
  0.34 │    │  inc    %eax     
       │    ├──cmp    %rcx,%rsi
  4.36 │    └──jne    1d0
```

The first thing our code does is extract our condition from the `std::vector<bool>` which is implemented as a bit-vector. This is done with a shift (`shlx`) and logical AND (`and`) instruction. The following `je` instruction based on the logical AND is used to skip the `add` instruction. The remaining code is used loop bounds checking, and loading in another 64 condition results from the bit vector (e.g., `cmp    $0x3f,%eax`).

From the performance results, we see that our time is best near the tails (`0.1` and `0.9`). This is where almost all the branches are not taken, or they are almost all taken. Modern branch predictors to a good job with heavily skewed branches like this, so we should see a very small miss-prediction rate in our performance counters.

Here is the miss-prediction rate for `branchBenchRandom/14/10`:

```text
     2,917,050,188      branches             # 1670.033 M/sec
        94,664,206      branch-misses        #    3.25% of all branches
```

And here is the miss-rediction rate for `branchBenchRandom/14/90`:

```text
     3,537,483,346      branches             # 1955.373 M/sec
        71,681,192      branch-misses        #    2.03% of all branches
```

Both are relatively low, in the ~2-4% range.

The second thing to see is that performance is the worst at `branchBenchRandom/14/50`. This is where the taken and not-taken outcome is equally likely, making things very difficult for the branch predcitor. Here is the miss-prediction rate:

```text
     1,578,311,040      branches             #  669.491 M/sec
       213,765,913      branch-misses        #   13.54% of all branches
```

A 4-5x higher miss-prediction rate than at the tails.

Another trend to notice is that performance is slightly worse moving towards the mostly-taken side compared to the mostly not-taken side. 

For example, we're ~4us faster at `branchBenchRandom/14/40` compared to `branchBenchRandom/14/60`. 

```text
------------------------------------------------------------------
Benchmark                        Time             CPU   Iterations
------------------------------------------------------------------
branchBenchRandom/14/40       78.2 us         78.2 us        17904
branchBenchRandom/14/60       82.1 us         82.0 us        17242
```

While both sides should have similar branch miss-prediction rates, the mostly not-taken side should be favorable for performance because it is doing less work (it skips the instructions for performing the add).

## Adding Compiler Hints

Now that we understand the performance of our benchmark without any hints, let's inform the compiler that our branch is skewed. We'll start by informing the compiler that our branch is mostly not-taken using the C++20 `[[unlikely]]` attribute:

```cpp
for (auto b : v_in)
  if (b) [[unlikely]] *sink += s.range(0);
```

And here are the performance numbers:

```text
---------------------------------------------------------------------------
Benchmark                                 Time             CPU   Iterations
---------------------------------------------------------------------------
branchBenchRandom_unlikely/14/10       26.5 us         26.5 us        53583
branchBenchRandom_unlikely/14/20       43.6 us         43.6 us        33011
branchBenchRandom_unlikely/14/30       62.2 us         62.2 us        23361
branchBenchRandom_unlikely/14/40       79.4 us         79.4 us        17961
branchBenchRandom_unlikely/14/50       85.6 us         85.6 us        16276
branchBenchRandom_unlikely/14/60       80.0 us         80.0 us        17548
branchBenchRandom_unlikely/14/70       65.8 us         65.8 us        21262
branchBenchRandom_unlikely/14/80       52.2 us         52.2 us        27187
branchBenchRandom_unlikely/14/90       40.9 us         40.9 us        33721
```

Let's start by comparing the biased not-taken side with and without the hint:

```text
---------------------------------------------------------------------------
Benchmark                                 Time             CPU   Iterations
---------------------------------------------------------------------------
branchBenchRandom/14/10                30.2 us         30.2 us        46742
branchBenchRandom/14/20                44.2 us         44.2 us        31818
branchBenchRandom/14/30                61.9 us         61.9 us        22675
branchBenchRandom/14/40                78.2 us         78.2 us        17904
branchBenchRandom_unlikely/14/10       26.5 us         26.5 us        53583
branchBenchRandom_unlikely/14/20       43.6 us         43.6 us        33011
branchBenchRandom_unlikely/14/30       62.2 us         62.2 us        23361
branchBenchRandom_unlikely/14/40       79.4 us         79.4 us        17961
```

For our heavily skewed branch (`0.1`), we see that a relatively large improvement in performance (30.2us to 26.5us). When there is very little skew (`0.4`), or performance is slightly worse with the hint (78.2us to 79.4 us).

Let's take a look at the assembly to see how our code as changed:

```assembly
  3.41 │1d0:┌─→shlx   %rax,%rdi,%rdx
  7.79 │    │  and    (%rcx),%rdx
  8.84 │    │↓ jne    26e
 25.66 │1de:│  cmp    $0x3f,%eax
  0.46 │    │↓ je     250
  0.73 │    │  inc    %eax
       │    ├──cmp    %rcx,%rsi
  4.66 │    └──jne    1d0       
```

In this first section of the assembly, we have our main loop which tests the condition, and skips the addition. However, the case we've marked as unlikely (performing the addition), as been moved far lower in the executable, starting at address `26e` (I've omitted the unrelated code in-between):

```assembly
 23.71 │26e:   mov    0x20(%rbp),%rdx
  4.00 │       cmp    %rdx,0x28(%rbp)
       │     ↓ je     383
  3.24 │       mov    (%rdx),%rdx
 15.98 │       add    %edx,(%r12)
  0.01 │     ↑ jmpq   1de
```

Code that is unlikely to be executed is often moved away from the hot path to avoid pollution of the instruction cache and decoded stream buffer which caches decoded micro-ops. You will also typically see the prioritized side of the branch moved to the not-taken side of the branch. We can find the following excerpt from the Intel Software Optimization Manual:

- "Even though this BPU mechanism generally eliminates the penalty for taken branches, software
should still regard taken branches as consuming more resources than do not-taken branches."

If we look at the [Instruction Table book](https://www.agner.org/optimize/instruction_tables.pdf) from Agner Fog, we can see that predicted taken branches can be scheduled to two ports (0 and 6), and predicted taken branches can only be scheduled to one port (6).

## Final Thoughts

Compiler hints are tricky, and can introduce new performance problems when used incorrectly. However, when applied carefully, they can be impactful for performance, especially when applied to branches in a tight loop. You'll find that the Linux kernel applies these hints using the macros `likely` and `unlikely`.

Thanks for reading,

--Nick

### Link to the source code

- [Source Code: ](https://github.com/CoffeeBeforeArch/misc_code/tree/master/biased_branches)
- [My YouTube Channel: ](https://www.youtube.com/channel/UCsi5-meDM5Q5NE93n_Ya7GA?view_as=subscriber)
- [My GitHub Account: ](https://github.com/CoffeeBeforeArch)
- My Email: CoffeeBeforeArch@gmail.com

