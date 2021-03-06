---
layout: default
title: A (Not So) fastMod
---

# A (Not So) fastMod

In performance, skepticism is your friend. When someone makes a claim, you ask for the numbers that support the claim. Likewise, when a result seems too good to be true, it may very well be. In this blog post we'll be re-examining the `fastMod` benchmarks from Chandler Carruth's CppCon 2015 talk titled ["Tuning C++: Benchmarks, and CPUs, and Compilers! Oh My!"](https://youtu.be/nXaxk27zwlk). What we'll focus on is whether the premise of the example is actually the one being tested by the benchmark. We'll do this by first re-collecting some of the performance numbers shown in the talk, then collecting a few more that show that the performance gains are largely coming from the synthetic input data, and that the optimization would be unwise for the compiler to make in many cases.

## Links

- [My YouTube Channel](https://www.youtube.com/channel/UCsi5-meDM5Q5NE93n_Ya7GA?view_as=subscriber)
- [My GitHub Account](https://github.com/CoffeeBeforeArch)
- [CppCon 2015 Talk](https://youtu.be/nXaxk27zwlk)
- [Benchmarks](https://github.com/CoffeeBeforeArch/misc_code/blob/master/code_scheduling)
- My Email: CoffeeBeforeArch@gmail.com

## Compiling the Code

All the benchmarks used in this blog post can be found [here](https://github.com/CoffeeBeforeArch/misc_code/blob/master/code_scheduling), and were writting using [Google Benchmark](https://github.com/google/benchmark). Below is the command I used to compile the benchmarks.

```bash
g++ bench_name.cpp -O3 -lbenchmark -lpthread -march=native -mtune=native -flto -fuse-linker-plugin
```

## Baseline Modulo Performance

- *"I want to briefly look at one other crazy example"* - [55:26](https://youtu.be/nXaxk27zwlk?t=3326)

Before we examine the modulo optimization in the video, we need to re-gather the baseline data. For this, we'll start with `baseMod`. Here is the C++ for the benchmark.

```cpp
// Baseline benchmark
static void baseMod(benchmark::State &s) {
  // Number of elements in the vectors
  int N = s.range(0);

  // Max for mod operator
  int ceil = s.range(1);

  // Vectors for input and output 
  std::vector<int> input;
  std::vector<int> output;
  input.resize(N);
  output.resize(N);

  // Generate random inputs (uniform random dist. between 0 & 255)
  std::mt19937 rng;
  rng.seed(std::random_device()());
  std::uniform_int_distribution<int> dist(0, 255);
  for (int &i : input) {
    i = dist(rng);
  }

  // Main benchmark loop
  while (s.KeepRunning()) {
    // Compute the modulo for each element
    for (int i = 0; i < N; i++) {
      output[i] = input[i] % ceil;
    }
  }
}
// Register the benchmark
BENCHMARK(baseMod)->Apply(custom_args);
```

We create a vector of input integers using a uniform random distribution of numbers between 0 and 255, an output vector of the same size, then we run our benchmark. In the benchmark loop where we are collecting the timing data (`for (auto _ : s )`), we simply perform perform a modulo on each input element, then store ther result in the output vector.

The benchmark uses argument pairs to test all possible combinations of vector sizes (16, 64, 256, 1024), and ceilings for our modulo (32, 128, 224). The ceiling values were selected based on our input data range (0-255 in a uniform random distribution). With a ceiling of 32, 7/8 of the numbers are greater >= 32. With a ceiling of 128, half of the possible input numbers are >= 128. At 224, only 1/8 of the input numbers are >= 224.

These inputs are generated by the following function.

```cpp
// Function for generating argument pairs
static void custom_args(benchmark::internal::Benchmark *b) {
  for (int i = 1 << 4; i <= 1 << 10; i <<= 2) {
    // Collect stats at 1/8, 1/2, and 7/8
    for (int j : {32, 128, 224}) {
      b = b->ArgPair(i, j);
    }
  }
}
```

Let's measure some baseline performance numbers.

```
-----------------------------------------------------------
Benchmark                 Time             CPU   Iterations
-----------------------------------------------------------
baseMod/16/32          29.2 ns         29.2 ns     23956748
baseMod/16/128         29.2 ns         29.2 ns     23933463
baseMod/16/224         29.2 ns         29.2 ns     24010859
baseMod/64/32           117 ns          117 ns      6014439
baseMod/64/128          117 ns          117 ns      5946117
baseMod/64/224          117 ns          117 ns      6027156
baseMod/256/32          470 ns          470 ns      1477802
baseMod/256/128         470 ns          470 ns      1484674
baseMod/256/224         472 ns          472 ns      1490478
baseMod/1024/32        1873 ns         1873 ns       374442
baseMod/1024/128       1870 ns         1870 ns       373702
baseMod/1024/224       1874 ns         1874 ns       375094
```

As you may have predicted, the vector size, not the ceiling value affects the performance of `baseMod`. That's because we're always going to compute the modulo for every input number, regardless of its value. Here's the inner-loop of the benchmark from the generated assembly.

```asm
  0.48 │230:┌─→mov        (%r10,%rcx,4),%eax                    
 11.98 │    │  cltd                                              
 62.20 │    │  idiv       %ebp                                   
 11.98 │    │  mov        %rcx,%rax                              
 12.76 │    │  mov        %edx,(%rdi,%rcx,4)                     
  0.01 │    │  inc        %rcx                                   
       │    ├──cmp        %rax,%rsi                              
  0.00 │    └──jne        230                                    
```

A faithful translation of what we had in our high-level C++. We load in a new value, use `idiv` to perform the modulo, store the result, increment the loop counter, and check to see if we're out of elements.

## "fastMod" Performance

- *"I have invented for you the world's fastest modulus operator"* - [55:53](https://youtu.be/nXaxk27zwlk?t=3346)

Now we'll take a look at the `fastMod` benchmark from the talk. We'll be exploiting the fact that the expensive modulus operator implemented with `idiv` can be skipped when the input value is less than the ceiling (e.g., 10 % 20 = 10). Here is my implementation based on the presented code.

```cpp
// Baseline for i
static void fastMod(benchmark::State &s) {
  // Number of elements
  int N = s.range(0);

  // Max for mod operator
  int ceil = s.range(1);

  // Vectors for input and output
  std::vector<int> input;
  std::vector<int> output;
  input.resize(N);
  output.resize(N);

  // Generate random inputs (uniform random dist. between 0 & 255)
  std::mt19937 rng;
  rng.seed(std::random_device()());
  std::uniform_int_distribution<int> dist(0, 255);
  for (int &i : input) {
    i = dist(rng);
  }

  // Main benchmark loop
  for (auto _ : s) {
    // DON'T compute the modulo for each element
    // Skip the expensive operation when we can
    for (int i = 0; i < N; i++) {
      output[i] = (input[i] >= ceil) ? input[i] % ceil : input[i];
    }
  }
}
// Register the benchmark
BENCHMARK(fastMod)->Apply(custom_args);
```

Now let's see how the performance compares to our baseline. Remember, we're testing different vector sizes, and different values for the ceiling. Our input values are still a random uniform distribution between 0 and 255. Here are the numbers.

```
-----------------------------------------------------------
Benchmark                 Time             CPU   Iterations
-----------------------------------------------------------
fastMod/16/32          26.1 ns         26.1 ns     25648581
fastMod/16/128         20.3 ns         20.3 ns     38350526
fastMod/16/224         13.4 ns         13.4 ns     51430885
fastMod/64/32           106 ns          106 ns      6749301
fastMod/64/128         81.0 ns         81.0 ns      8340513
fastMod/64/224         54.8 ns         54.8 ns     13379989
fastMod/256/32          424 ns          424 ns      1631738
fastMod/256/128         328 ns          328 ns      2237153
fastMod/256/224         215 ns          215 ns      3242629
fastMod/1024/32        1732 ns         1732 ns       404466
fastMod/1024/128       1281 ns         1281 ns       553820
fastMod/1024/224        952 ns          952 ns       752329
```

Looks great! We're better in every case than our `baseMod` benchmark. Let's first try to rationalize why. Our best case is when ceiling is 224. That's because most of our input data lies below the ceiling, so we're skipping the majority of the `idiv` instructions. Our worst case is when the ceiling is 32. It seems that while we aren't skipping many `idiv` instructions (compared with a ceiling of 224) the speedup we gain when we do makes us faster than `baseMod` (but only slightly). When the ceiling is 128, we're faster than when it is 32, but not quite a fast as when it is 224. Let's say that's because we're skipping more `idiv` instructions than when the ceiling is 32, but not quite as many as when the ceiling is 224.

Here is the assembly for our `fastMod`.

```
  0.00 │238:┌─→mov        %rax,%rcx                           
  8.87 │23b:│  mov        (%rdi,%rcx,4),%edx                   
  0.05 │    │  cmp        %ebp,%edx                            
  7.90 │    │↓ jl         247                                  
  2.61 │    │  mov        %edx,%eax                            
  0.25 │    │  cltd                                            
 53.35 │    │  idiv       %ebp                                 
 16.13 │247:│  mov        %edx,(%r8,%rcx,4)                    
  0.02 │    │  lea        0x1(%rcx),%rax                       
  0.06 │    ├──cmp        %rcx,%rsi                            
 10.07 │    └──jne        238                                                        
```

Not too much more complicated than our original inner-loop. Here, we're using a branch to conditionally skip over the `idiv` instruction (if the input value is less than the ceiling). We then store the output value, increment the loop counter, and check to see if we're out of input elements.

We're not done yet! We might be able to squeeze out some extra performance by passing a hint to our compiler. When we know the branch will always/almost always go a certain way, we can pass that information on to our compiler. This can lead to some better code scheduling, and better performance. Here's a look at the inner loop of `fastModHint`. Using `__builtin_expect`, we're telling the compiler to expect the input to be less than the ceiling.

```cpp
for (auto _ : s) {
  // DON'T compute the mod for each element
  // Skip the expensive operation using a simple compare
  for (int i = 0; i < N; i++) { 
    // Hint to the compiler that we usually skip the mod
    output[i] =
        __builtin_expect(input[i] >= ceil, 0) ? input[i] % ceil : input[i];
  }
}
```

Let's take a look at the performance numbers. Remember, this is an optimization for when we know if our input data is skewed in a certain direction. Be warned. If you provide a bad hint to your compiler, it can cause serious slowdowns.

```
---------------------------------------------------------------
Benchmark                     Time             CPU   Iterations
---------------------------------------------------------------
fastModHint/16/32          32.0 ns         32.0 ns     23148667
fastModHint/16/128         25.0 ns         25.0 ns     38520321
fastModHint/16/224         10.9 ns         10.9 ns     64085049
fastModHint/64/32           130 ns          130 ns      5484921
fastModHint/64/128         88.4 ns         88.4 ns      7852650
fastModHint/64/224         42.8 ns         42.8 ns     16516948
fastModHint/256/32          547 ns          547 ns      1289698
fastModHint/256/128         318 ns          318 ns      1909529
fastModHint/256/224         163 ns          163 ns      4623014
fastModHint/1024/32        2281 ns         2281 ns       290084
fastModHint/1024/128       1420 ns         1420 ns       506181
fastModHint/1024/224        706 ns          706 ns       906472
```

Our data matches our expectations! When the hint is bad, we're slower than our original `fastMod`, and even `baseMod`. However, when the hit is good (in this case, when ceiling is 224), we're significantly faster than `fastMod`! Let's see why by looking at the assembly.

```asm
  0.07 │238:┌─→mov        %rax,%rcx                             
 12.44 │23b:│  mov        (%rdi,%rcx,4),%edx                     
  0.11 │    │  cmp        %ebp,%edx                              
  2.12 │    │↓ jge        2c0                                    
 18.80 │242:│  mov        %edx,(%r8,%rcx,4)                      
  0.15 │    │  lea        0x1(%rcx),%rax                         
  0.04 │    ├──cmp        %rcx,%rsi                              
  6.97 │    └──jne        238                                    
.
.
.
  0.04 │2c0:│  mov        %edx,%eax                              
  0.11 │    │  cltd                                              
 58.17 │    │  idiv       %ebp                                   
  0.53 │    └──jmpq       242                                    
```

The fallthrough of the branch is now the case where we are *not* performing the division (the compiler will typically place the likely path as the fallthrough path). In fact, the division has been moved far away from our tight inner-loop! One final metric we can compare is the difference in the cylces where the divider is active. Let's compare the tests `baseMod/1024/224`, `fastMod/1024/224` `fastModHint/1024/224` for a constant number of iterations (1000000).

```
baseMod/1024/224      7,654,562,646      arith.divider_active
fastMod/1024/224        952,625,759      arith.divider_active
fastModHint/1024/224    933,148,930      arith.divider_active
```

Unsuprisingly, when 7/8 our data does not perform the modulo, we have 1/7 the number of cycles where the divider unit is active.

## "unstableMod" Performance

- *"So now if I rerun this, I should get slightly more interesting data"* - [59:14](https://youtu.be/nXaxk27zwlk?t=3554)

I'll now be introducing `unstableMod`. This benchmark has performance that is less predictable than `baseMod` and `fastMod`. Instead of starting out by explaining the C++, let's compare the performance of `unstableMod` to `baseMod`. I've also increased the vector sizes to 4096, 8192, and 16384. First, here are the numbers for `baseMod` (at the new vector sizes).

```
------------------------------------------------------------
Benchmark                  Time             CPU   Iterations
------------------------------------------------------------
baseMod/4096/32         7.15 us         7.15 us        96911
baseMod/4096/128        7.20 us         7.20 us        96776
baseMod/4096/224        7.21 us         7.21 us        96480
baseMod/8192/32         14.5 us         14.5 us        47878
baseMod/8192/128        14.5 us         14.5 us        48145
baseMod/8192/224        14.5 us         14.5 us        47846
baseMod/16384/32        28.7 us         28.7 us        24429
baseMod/16384/128       28.7 us         28.7 us        24316
baseMod/16384/224       28.7 us         28.7 us        24316
```

Nothing surprising. Larger vectors take larger to process than smaller vectors, and the timing isn't affected by different values of ceiling. Here are the results of `unstableMod`.

```
----------------------------------------------------------------
Benchmark                      Time             CPU   Iterations
----------------------------------------------------------------
unstableMod/4096/32         6.92 us         6.92 us        98341
unstableMod/4096/128        11.5 us         11.5 us        60539
unstableMod/4096/224        4.66 us         4.66 us       148384
unstableMod/8192/32         15.9 us         15.9 us        43786
unstableMod/8192/128        29.3 us         29.3 us        24037
unstableMod/8192/224        11.3 us         11.3 us        60260
unstableMod/16384/32        36.3 us         36.3 us        19286
unstableMod/16384/128       63.6 us         63.6 us        10975
unstableMod/16384/224       24.7 us         24.7 us        28259
```

Our `unstableMod` seems to be worse in most cases (sometimes by over 2x), and better in a small number of cases. And now or the code reveal!

```cpp
// Main benchmark loop
for (auto _ : s) {
  // DON'T compute the modulo for each element
  // Skip the expensive operation when we can
  for (int i = 0; i < N; i++) {
    output[i] = (input[i] >= ceil) ? input[i] % ceil : input[i];
  }
}
```

Ta da! Our benchmark is really just `fastMod`. The only thing we've changed (other than its name), is the size of the input vectors. This is odd. Why did `fastMod` seem like a good idea at small vector sizes, but inconsistent (and usually worse) at large vector sizes? To explain this, we need take another look at our results at small vector sizes. Specifically, let's look at our branch predictor unit statistics. Here are the branch miss percentages for `baseMod`.

```
baseMod/16/32       branch-misses    0.01% of all branches
baseMod/16/128      branch-misses    0.01% of all branches                        
baseMod/16/224      branch-misses    0.01% of all branches             
baseMod/64/32       branch-misses    0.01% of all branches                     
baseMod/64/128      branch-misses    0.01% of all branches                  
baseMod/64/224      branch-misses    0.01% of all branches             
baseMod/256/32      branch-misses    0.39% of all branches                   
baseMod/256/128     branch-misses    0.39% of all branches                   
baseMod/256/224     branch-misses    0.39% of all branches              
baseMod/1024/32     branch-misses    0.10% of all branches             
baseMod/1024/128    branch-misses    0.10% of all branches            
baseMod/1024/224    branch-misses    0.10% of all branches             
```

Nothing surprising here. The inner-loop of `baseMod` always does the same thing, so the branch predictor has almost (if not 0) miss-predictions. Here are the results for `fastMod`.

```
fastMod/16/32       branch-misses    0.00% of all branches
fastMod/16/128      branch-misses    0.00% of all branches                        
fastMod/16/224      branch-misses    0.00% of all branches             
fastMod/64/32       branch-misses    0.00% of all branches                     
fastMod/64/128      branch-misses    0.00% of all branches                  
fastMod/64/224      branch-misses    0.02% of all branches             
fastMod/256/32      branch-misses    0.07% of all branches                   
fastMod/256/128     branch-misses    0.00% of all branches                   
fastMod/256/224     branch-misses    0.00% of all branches              
fastMod/1024/32     branch-misses    0.03% of all branches             
fastMod/1024/128    branch-misses    0.25% of all branches            
fastMod/1024/224    branch-misses    0.44% of all branches             
```

Notice how `fastMod` has very few (if any) branch miss-predictions? That should be an immediate red flag. If our branch for conditionally performing modulo uses a random input, we'd expect that the branch predictor would have a difficult time guessing the outcomes. If the branch is skewed to one side, we'd expect the branch preductor unit (BPU) to perform better (i.e., for `ceil=32` and `ceil=224`). We'd expect the worst miss-prediction rate when the branch is not skewed (i.e., when `ceil=128`). So why are we seeing roughly the same miss-prediction rate for every input pair?

It's the input data! We created a single vector of random numbers before the main benchmark loop, and we re-use those random numbers every iteration of the benchmark. That means the BPU can learn the branch outcomes from the first few iterations of the benchmark, and predict them (with a high degree of accuracy) in the subsequent iterations. We're not measuring how `fastMod` performs with random input data, we're measuring how it performs with random input data *and* a warmed up BPU.

## Accounting for Branch Prediction

- *"And then I ran the benchmark. I measured."* - [1:13:31](https://youtu.be/nXaxk27zwlk?t=4411)

The BPU has a finite amount of state to track branch outcome history and branch targets. When we use larger vector sizes, the BPU stops being able to "memorize" all the branch outcomes that are repeated each iteration of the benchmark. It is only at this point that our input data starts to look a random input stream, and not just repeated values from a fixed-size vector. 

Don't believe me? Good! Stay skeptical. Make people prove their assertions. Let's place a cap on the number of iterations for `fastMod`. We'll test 1, 1,000, and 100,000 iterations. Note, with fewer iterations, the timing information will be less stable (i.e., you may see the inconsistent timing results). To roughly account for this, I ran the test multiple times, and am only reporting the best results for each iteration step.

Here were the results for `iterations=1`.

```
------------------------------------------------------------------------
Benchmark                              Time             CPU   Iterations
------------------------------------------------------------------------
fastMod/16/32/iterations:1          1331 ns          802 ns            1
fastMod/16/128/iterations:1          344 ns          314 ns            1
fastMod/16/224/iterations:1          262 ns          254 ns            1
fastMod/64/32/iterations:1           385 ns          388 ns            1
fastMod/64/128/iterations:1          461 ns          450 ns            1
fastMod/64/224/iterations:1          311 ns          311 ns            1
fastMod/256/32/iterations:1         1011 ns          992 ns            1
fastMod/256/128/iterations:1        1261 ns         1246 ns            1
fastMod/256/224/iterations:1         730 ns          722 ns            1
fastMod/1024/32/iterations:1        3078 ns         3041 ns            1
fastMod/1024/128/iterations:1      22683 ns        22681 ns            1
fastMod/1024/224/iterations:1       2115 ns         2117 ns            1
```

Terrible performance! While I wouldn't trust the exact timing numbers, we do see some of the same trends in performance as we did in `unstableMod`. Notice how a ceiling of 128 typically has the worst performance, and a ceiling of 224 has the best (same as `unstableMod`!)? Our inner-loop is nowhere near as slow as what's being recorded. What's likely going on is that the branches to the timing code are being miss-predicted since we are only running a single iteration. This will affect smaller vector sizes more than larger ones.

Let's continue our experiment with `iterations=1,000`.

```
---------------------------------------------------------------------------
Benchmark                                 Time             CPU   Iterations
---------------------------------------------------------------------------
fastMod/16/32/iterations:1000          41.2 ns         40.7 ns         1000
fastMod/16/128/iterations:1000         17.8 ns         17.8 ns         1000
fastMod/16/224/iterations:1000         30.6 ns         30.6 ns         1000
fastMod/64/32/iterations:1000           165 ns          165 ns         1000
fastMod/64/128/iterations:1000          117 ns          117 ns         1000
fastMod/64/224/iterations:1000         83.6 ns         83.6 ns         1000
fastMod/256/32/iterations:1000          453 ns          453 ns         1000
fastMod/256/128/iterations:1000         332 ns          332 ns         1000
fastMod/256/224/iterations:1000         218 ns          218 ns         1000
fastMod/1024/32/iterations:1000        1738 ns         1738 ns         1000
fastMod/1024/128/iterations:1000       1312 ns         1312 ns         1000
fastMod/1024/224/iterations:1000        924 ns          924 ns         1000
```

Our execution time seems to have reduced significantly (with 1000 iterations, we're likely not miss-predicting our branches to the timing code anymore). However, we also lost our pattern that a ceiling of 128 has the worst performance. In fact, we're beginning to align with our original measurements for `fastMod`.

Here are the results for `iterations=100,000`.

```
-----------------------------------------------------------------------------
Benchmark                                   Time             CPU   Iterations
-----------------------------------------------------------------------------
fastMod/16/32/iterations:100000          27.2 ns         27.2 ns       100000
fastMod/16/128/iterations:100000         21.5 ns         21.5 ns       100000
fastMod/16/224/iterations:100000         15.8 ns         15.8 ns       100000
fastMod/64/32/iterations:100000           105 ns          105 ns       100000
fastMod/64/128/iterations:100000         82.5 ns         82.5 ns       100000
fastMod/64/224/iterations:100000         56.4 ns         56.4 ns       100000
fastMod/256/32/iterations:100000          431 ns          431 ns       100000
fastMod/256/128/iterations:100000         309 ns          309 ns       100000
fastMod/256/224/iterations:100000         210 ns          210 ns       100000
fastMod/1024/32/iterations:100000        1643 ns         1643 ns       100000
fastMod/1024/128/iterations:100000       1236 ns         1236 ns       100000
fastMod/1024/224/iterations:100000        926 ns          926 ns       100000
```

We're just about back to where we started (our original `fastMod` results)! However, notice how our performance increased with the number of iterations. As we move from 1 to 1000 iterations, a lot of this improvement comes from no longer miss-predicting branches to the timing code. However, we still see an over 1.5x speedup (in some cases) as we increase the number of iterations from 1,000 and 100,000 iterations. Our BPU is continuing to learn! 

Now let's look at our branch miss-prediction rate for `fastMod` with a larger input vector sizes (the same test cases we used for `unstableMod`). Here are the results.

```
fastMod/4096/32     branch-misses    0.97% of all branches
fastMod/4096/128    branch-misses   14.80% of all branches
fastMod/4096/224    branch-misses    2.73% of all branches
fastMod/8192/32     branch-misses    2.42% of all branches
fastMod/8192/128    branch-misses   20.59% of all branches
fastMod/8192/224    branch-misses    4.23% of all branches
fastMod/16384/32    branch-misses    4.48% of all branches
fastMod/16384/128   branch-misses   22.98% of all branches
fastMod/16384/224   branch-misses    5.22% of all branches
```

Now our values match our expectations! When `ceil` is 32 or 224, we'd expect a slightly better hit rate (but not 0!). That's because our branches will either almost always be taken, or almost always not taken. We'd expect our BPU to perform the worst when `ceil=128`, because approximately half our values will take the branch, and the other half will not (and in a random order).

Let's take another look at the timing results of `fastMod` (previously named `unstableMod`) using larger vector sizes, and use our branch miss-prediction rate data to guide our analysis.

```
------------------------------------------------------------
Benchmark                  Time             CPU   Iterations
------------------------------------------------------------
fastMod/4096/32         6.92 us         6.92 us        98341
fastMod/4096/128        11.5 us         11.5 us        60539
fastMod/4096/224        4.66 us         4.66 us       148384
fastMod/8192/32         15.9 us         15.9 us        43786
fastMod/8192/128        29.3 us         29.3 us        24037
fastMod/8192/224        11.3 us         11.3 us        60260
fastMod/16384/32        36.3 us         36.3 us        19286
fastMod/16384/128       63.6 us         63.6 us        10975
fastMod/16384/224       24.7 us         24.7 us        28259
```

Our `ceil=128` cases perform the worst because of the extremely bad miss-prediction rate. Our `ceil=32` cases generally perform worse than our `baseMod`, but are not terrible. That's because the branch is biased towards one side, and the BPU does a decent job and predicting the outcomes. However, the loop is not skipping many modulo operations, and we still occasionally pay the price for miss-predictiion . The `ceil=224` cases are faster than `baseMod` still, but not by the same margins as when we were using small vectors. That's because the BPU is doing a good job predicting the biased branch outcomes and we're skipping the majority of the modulo operations. However, we are still occasionally paying the price of miss-prediction.

### Side Note - Avoiding Unnecessary Complexity

Why not just regenerate new random numbers each iteration? We can! However, now we're adding additional complexity to our benchmark. There are utilities for pausing the timing measurement withing the benchmarking loop (`PauseTiming` and `ResumeTiming`). Let's rewrite our inner-loop using these methods, and generate random numbers each iteration. We'll place this in a new benchmark called `fastModRandom`, which uses the original (small) vector sizes. Here are the modifications to the inner-loop.

```cpp
for (auto _ : s) {
  // Generate random numbers but don't profile it
  s.PauseTiming();
  for (int &i : input) {
    i = dist(rng);
  }
  s.ResumeTiming();

  // DON'T compute the mod for each element
  // Skip the expensive operation using a simple compare
  for (int i = 0; i < N; i++) {
    output[i] = (input[i] >= ceil) ? input[i] % ceil : input[i];
  }
}
```

And here are the timing results.

```
-----------------------------------------------------------------
Benchmark                       Time             CPU   Iterations
-----------------------------------------------------------------
fastModRandom/16/32           199 ns          200 ns      3495504
fastModRandom/16/128          221 ns          224 ns      3119579
fastModRandom/16/224          185 ns          187 ns      3754506
fastModRandom/64/32           311 ns          312 ns      2232891
fastModRandom/64/128          393 ns          393 ns      1778843
fastModRandom/64/224          266 ns          266 ns      2630969
fastModRandom/256/32          758 ns          756 ns       929540
fastModRandom/256/128        1066 ns         1067 ns       656544
fastModRandom/256/224         563 ns          564 ns      1246860
fastModRandom/1024/32        2660 ns         2659 ns       264438
fastModRandom/1024/128       3915 ns         3907 ns       181506
fastModRandom/1024/224       1890 ns         1891 ns       368629
```

Now our trends at the smaller vector sizes are the same as those at the larger vector sizes! However, notice the results can be 10x slower? That seems excessive. However, we did just add a huge overhead (random number generation) inside of our timing loop. It's unsurprising that it has affected the timing of our tight inner-loop, even if we do stop the timers. If we add the same random number generation to our baseline benchmark (we'll call it `baseModRandom`) we can see the same thing. Here are the timing results.

```
-----------------------------------------------------------------
Benchmark                       Time             CPU   Iterations
-----------------------------------------------------------------
baseModRandom/16/32           190 ns          191 ns      3703498
baseModRandom/16/128          189 ns          191 ns      3664350
baseModRandom/16/224          188 ns          189 ns      3694554
baseModRandom/64/32           272 ns          273 ns      2555042
baseModRandom/64/128          273 ns          275 ns      2557137
baseModRandom/64/224          274 ns          275 ns      2547418
baseModRandom/256/32          615 ns          617 ns      1124198
baseModRandom/256/128         613 ns          614 ns      1138165
baseModRandom/256/224         612 ns          614 ns      1141959
baseModRandom/1024/32        1951 ns         1953 ns       358473
baseModRandom/1024/128       1952 ns         1953 ns       358287
baseModRandom/1024/224       1952 ns         1954 ns       358137
```

Significantly slower than our original `baseMod`, but with the same performance trends!

The `PauseTiming` and `ResumeTiming` methods can be useful in approximately replicating performance trends. However, everything has a cost. Now our raw timing numbers are significantly different (by as much as 10x) than our original ones. It's usually a good idea to avoid extra complexity in a tight loop you are trying to profile. We were able to accomplish this by using slightly larger vectors. However, there are many ways to solve the same problem, and something as simple as increasing the size of a vector does not always work. Something to think about

## Concluding Remarks

- *"the most valuable thing I learned from this talk is to branch my modulos lol"* - Anonymous YouTube comment

One of the difficult parts of benchmarking is not falling the many subtle pitfalls. When we write a benchmark, that doesn't mean our work is done. We must first verify that we are measuring what we set out to measure. In our case, we intended to measure the performance of using a branch to avoid always performing the expensive modulo operation when we have random input data. However, what we actually measured the the performance of using a branch that is _almost always predicted correctly_ to avoid an expensive modulo operation.

While it seems that this benchmark from [Chandler's talk](https://youtu.be/nXaxk27zwlk) has some key problems, the talk intself contains a lot of valuable insights. One of the key points from the talk is to go out and measure things. In this case, it just so happens that we needed to measure a few more (the branch miss-prediction rates). Looks like that example wasn't so crazy after all.

As always, feel free to contact me with questions.

Cheers,

--Nick

## Links

- [My YouTube Channel](https://www.youtube.com/channel/UCsi5-meDM5Q5NE93n_Ya7GA?view_as=subscriber)
- [My GitHub Account](https://github.com/CoffeeBeforeArch)
- [Benchmarks](https://github.com/CoffeeBeforeArch/misc_code/blob/master/code_scheduling)
- My Email: CoffeeBeforeArch@gmail.com
