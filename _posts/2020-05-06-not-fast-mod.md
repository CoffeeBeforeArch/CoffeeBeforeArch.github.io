---
layout: default
title: A Not So Fast Modulo
---

# A Not So Fast Modulo

In performance, skepticism is your friend. When someone makes a claim, you ask for the numbers that support said claim. Likewise, when an result seems too good to be true, it may very well be. In this blog post we'll be re-examining the `fastMod` benchmarks from Chandler Carruth's CppCon 2015 talk titled ["Tuning C++: Benchmarks, and CPUs, and Compilers! Oh My!"](https://youtu.be/nXaxk27zwlk). What we'll focus on is whether the premise of the example is actually the one being tested by the benchmark. We'll do this by first re-collecting the performance numbers shown in the talk, then collecting a few more that show that the performance gains are largely coming from the synthetic input data, and would be an unwise optimization in many cases.

## Links

- [My YouTube Channel](https://www.youtube.com/channel/UCsi5-meDM5Q5NE93n_Ya7GA?view_as=subscriber)
- [My GitHub Account](https://github.com/CoffeeBeforeArch)
- [CppCon 2015 Talk](https://youtu.be/nXaxk27zwlk)
- [Benchmarks](https://github.com/CoffeeBeforeArch/misc_code/blob/master/code_scheduling)
- My Email: CoffeeBeforeArch@gmail.com

## Compiling the Code

All the benchmarks used in this blog post can be found [here](https://github.com/CoffeeBeforeArch/misc_code/blob/master/code_scheduling), and were writting using [Google Benchmark](https://github.com/google/benchmark). Below is the command I used to compile the benchmarks.

```bash
g++ bench_name.cpp -O3 -lbenchmark -lpthread -march=native -mtune=native
```

## Baseline Modulo Performance

- *"I want to briefly look at one other crazy example"*

Before we examine the modulo optimization in the video, we first need to re-gather the baseline data. For this, we'll start with base made. Here is the C++ code.

```cpp
// Baseline for intuitive modulo operation
static void baseMod(benchmark::State &s) {
  // Number of elements
  int N = s.range(0);

  // Max for mod operator
  int ceil = s.range(1);

  // Vector for input and output of modulo
  std::vector<int> input;
  std::vector<int> output;
  input.resize(N);
  output.resize(N);

  // Generate random inputs
  std::mt19937 rng;
  rng.seed(std::random_device()());
  std::uniform_int_distribution<int> dist(0, 255);
  for (int &i : input) {
    i = dist(rng);
  }

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

We create a vector of input integers using a uniform random distribution (of numbers between 0 and 255), another vector for our output results, then run our benchmark. In the benchmark loop where we're collecting timing data (`for (auto _ : s )`), we simply perform the modulo on each input element, then store ther result in the output.

The benchmark uses argument pairs to test all possible combinations of vector sizes (16, 64, 256, 1024), and ceilings for our modulo (32, 128, 224). The modulo value were selected based on the input random number rang (0-255). With %32, most of the numbers are greater than 32. With %128, half the possible input numbers are greater than or equal to 128, and half are below. With 224, most of the input numbers are less than 224.

Here is the input function that generates these custom arguments.

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

Here are the performance results.

```
---------------------------------------------------------------------
Benchmark                           Time             CPU   Iterations
---------------------------------------------------------------------
baseMod/16/32                    49.3 ns         49.2 ns     14135978
baseMod/16/128                   48.9 ns         48.8 ns     13556154
baseMod/16/224                   49.1 ns         49.0 ns     13949133
baseMod/64/32                     197 ns          196 ns      3507126
baseMod/64/128                    194 ns          194 ns      3584375
baseMod/64/224                    197 ns          197 ns      3543025
baseMod/256/32                    787 ns          786 ns       896642
baseMod/256/128                   794 ns          793 ns       874863
baseMod/256/224                   845 ns          782 ns       869559
baseMod/1024/32                  3323 ns         3244 ns       213629
baseMod/1024/128                 3385 ns         3299 ns       210451
baseMod/1024/224                 3308 ns         3221 ns       212943
```

As expected, the number we're performing modulo by does not have an impact on performance. That's because no-matter what, we're always doing the modulo on every input number. Here's the inner-loop of the benchmark from the generated assembly.

```asm
  1.48 │1f8:┌─→mov    (%r8,%rcx,4),%eax                             
 10.00 │    │  cltd                                             
 62.94 │    │  idiv   %ebx                                      
 10.20 │    │  mov    %rcx,%rax                              
 14.57 │    │  mov    %edx,(%rdi,%rcx,4)                 
  0.47 │    │  inc    %rcx                               
  0.00 │    ├──cmp    %rax,%rsi                              
  0.07 │    └──jne    1f8                                     
```

A faithful translation of what we had in our high-level C++. We load in a new value, use `idiv` to perform the modulo, store the result, increment the loop counter, and check to see if we're out of elements.

## "fastMod" Performance

Now we'll take a look at the fastMod benchmark performance. Here the version that I created based on the code presented in the presentation.

```cpp
// Baseline for intuitive modulo operation
static void fastMod(benchmark::State &s) {
  // Number of elements
  int N = s.range(0);

  // Max for mod operator
  int ceil = s.range(1);

  // Vector for input and output of modulo
  std::vector<int> input;
  std::vector<int> output;
  input.resize(N);
  output.resize(N);

  // Generate random inputs
  std::mt19937 rng;
  rng.seed(std::random_device()());
  std::uniform_int_distribution<int> dist(0, 255);
  for (int &i : input) {
    i = dist(rng);
  }

  for (auto _ : s) {
    // DON'T compute the mod for each element
    // Skip the expensive operation using a simple compare
    for (int i = 0; i < N; i++) {
      output[i] = (input[i] >= ceil) ? input[i] % ceil : input[i];
    }
  }
}
// Register the benchmark
BENCHMARK(fastMod)->Apply(custom_args);
```

The main idea here is that division is expensive, so if we don't need to perform it, we shouldn't. Therefore, if an input number is less than the ceiling (32, 128, or 224), we simply store the input number to the output. If the input number is greater than or equal to `ceil`, we have to perform the modulo. Here are the performance results I measured.

```
---------------------------------------------------------------------
Benchmark                           Time             CPU   Iterations
---------------------------------------------------------------------
fastMod/16/32                    51.2 ns         51.1 ns     15028050
fastMod/16/128                   43.7 ns         43.7 ns     15989641
fastMod/16/224                   27.3 ns         27.1 ns     25246534
fastMod/64/32                     199 ns          192 ns      3793905
fastMod/64/128                    158 ns          153 ns      4687761
fastMod/64/224                    111 ns          107 ns      5816871
fastMod/256/32                    767 ns          744 ns       893204
fastMod/256/128                   627 ns          605 ns      1085240
fastMod/256/224                   464 ns          449 ns      1342627
fastMod/1024/32                  2986 ns         2912 ns       249818
fastMod/1024/128                 2373 ns         2302 ns       293129
fastMod/1024/224                 1867 ns         1859 ns       373717
```

Looks great! We see at worst the same, if not better performance than `baseMod` when `ceil` is 32, and better with every other input pair. When `ceil` is 224, we have the best performance. That's because we skip the modulo for almost every input number. Let's take a look at how this translated to the assembly.

```
  0.06 │1f8:┌─→mov    %rax,%rcx                   
 12.44 │1fb:│  mov    (%rdi,%rcx,4),%edx  
  0.37 │    │  cmp    %ebx,%edx            
  5.99 │    │↓ jl     207                                
  2.26 │    │  mov    %edx,%eax                           
  0.23 │    │  cltd                              
 51.60 │    │  idiv   %ebx                       
 19.33 │207:│  mov    %edx,(%r8,%rcx,4)         
  0.54 │    │  lea    0x1(%rcx),%rax               
  0.08 │    ├──cmp    %rcx,%rsi                 
  6.62 │    └──jne    1f8                             
```

Unsurprisingly our `idiv` has a branch just before it that skips over the modulo if the input is less than `ceil`. If we know most of our input values are below the `ceil`, we can provide a compiler hint like `__builtin_expect` (as shown in the video), to help the compiler schedule the code more optimiallly. Here is the change to the benchmarking loop.

```cpp
for (auto _ : s) {
  // DON'T compute the mod for each element
  // Skip the expensive operation using a simple compare
  for (int i = 0; i < N; i++) {
    output[i] =
        __builtin_expect(input[i] >= ceil, 0) ? input[i] % ceil : input[i];
  }
}
```

Here, we're expecting the result of the comparison to be 0 (i.e., the input is less than `ceil`. Let's take a look at the performance numbers for the largest vector size, and a ceil of 224. Remember, this is an optimization for when we know if our input data is skewed a certain direction (which we can select with the benchmark).

```
---------------------------------------------------------------
Benchmark                     Time             CPU   Iterations
---------------------------------------------------------------
fastModHint/1024/224       1284 ns         1283 ns      2210614
```

Significantly faster! Let's see why by looking at the assembly.

```asm
  0.50 │1f8:┌─→mov    %rax,%rcx                                     
 19.36 │1fb:│  mov    (%rdi,%rcx,4),%edx                   
  2.21 │    │  cmp    %ebx,%edx                         
  1.72 │    │↓ jge    268                         
 18.64 │202:│  mov    %edx,(%r8,%rcx,4)               
  0.70 │    │  lea    0x1(%rcx),%rax                  
  0.07 │    ├──cmp    %rcx,%rsi                             
 10.58 │    └──jne    1f8                          
.
.
.
  0.17 │268:│  mov    %edx,%eax                                  
  0.20 │    │  cltd                                       
 44.01 │    │  idiv   %ebx                             
  1.79 │    └──jmp    202                                  
```

The fallthrough of the branch is now the case where we are *not* performing the division. In fact, the division has be moved far away from our tight inner-loop! Another metric we can compare between baseMod and fastMod is the difference in the cylces where the divider is active. Let's compare the tests `baseMod/1024/224` and `fastModHint/1024/224` for a constant number of iteratios (1000000).

```
baseMod/1024/224      7,462,530,966      arith.divider_active
fastModHint/1024/224  1,008,987,160      arith.divider_active
```

Unsuprisingly, when 7/8's our data does not perform the modulo, we have 1/7 the number of cycles of the divider unit being active.

## "unstableMod Performance"

I'll now be introducing `slowMod`. This benchmark performs modulo even slower than than `baseMod` for all cases. Instead of starting out by explaining the C++, let's look at the performance results side-by-side with `baseMod`. I've also increased the vector sizes from (4096, 8192, and 16384). Here are the numbers for baseMod.

```
------------------------------------------------------------
Benchmark                  Time             CPU   Iterations
------------------------------------------------------------
baseMod/4096/32         10.7 us         10.7 us       260846
baseMod/4096/128        10.7 us         10.7 us       261573
baseMod/4096/224        10.7 us         10.7 us       261521
baseMod/8192/32         21.4 us         21.4 us       129617
baseMod/8192/128        21.4 us         21.4 us       130356
baseMod/8192/224        21.4 us         21.4 us       130386
baseMod/16384/32        43.7 us         43.6 us        64128
baseMod/16384/128       44.5 us         44.4 us        63685
baseMod/16384/224       45.0 us         45.0 us        64040
```

Here are the results of `unstableMod`.

```
----------------------------------------------------------------
Benchmark                      Time             CPU   Iterations
----------------------------------------------------------------
unstableMod/4096/32         13.3 us         13.3 us       211063
unstableMod/4096/128        16.7 us         16.7 us       160856
unstableMod/4096/224        9.61 us         9.60 us       288420
unstableMod/8192/32         27.7 us         27.7 us        99420
unstableMod/8192/128        43.3 us         43.3 us        66535
unstableMod/8192/224        20.0 us         20.0 us       100000
unstableMod/16384/32        59.0 us         59.0 us        47468
unstableMod/16384/128       94.4 us         94.4 us        29949
unstableMod/16384/224       41.3 us         41.3 us        68449
```

Our `unstableMod` seems to be worse in most cases (sometimes by more than 2x), and better in only a small number. And now or the code reveal!

```
for (auto _ : s) {
  // DON'T compute the mod for each element
  // Skip the expensive operation using a simple compare
  for (int i = 0; i < N; i++) {
    output[i] = (input[i] >= ceil) ? input[i] % ceil : input[i];
  }
}
```

Ta da! It's just `fastMod`. This seems... odd. Why did `fastMod` seem like a good idea at small vector sizes, but a terrible at large sizes? To explain this, we first need to look at the branch prediction rate. Here are branch misses for each input combination of `baseMod` for the original input sizes.

```
baseMod/16/32       42,529          branch-misses             #    0.01% of all branches
baseMod/16/128      51,761          branch-misses             #    0.01% of all branches                        
baseMod/16/224      67,636          branch-misses             #    0.01% of all branches             
baseMod/64/32       42,040          branch-misses             #    0.01% of all branches                     
baseMod/64/128      47,719          branch-misses             #    0.01% of all branches                  
baseMod/64/224      58,502          branch-misses             #    0.02% of all branches             
baseMod/256/32      1,182,454       branch-misses             #    0.40% of all branches                   
baseMod/256/128     1,179,800       branch-misses             #    0.40% of all branches                   
baseMod/256/224     1,185,037       branch-misses             #    0.40% of all branches              
baseMod/1024/32     414,946         branch-misses             #    0.11% of all branches             
baseMod/1024/128    414,029         branch-misses             #    0.11% of all branches            
baseMod/1024/224    423,683         branch-misses             #    0.11% of all branches             
```

And here are the results for `fastMod`.

```
fastMod/16/32       44,052          branch-misses             #    0.00% of all branches
fastMod/16/128      45,134          branch-misses             #    0.00% of all branches                        
fastMod/16/224      165,187         branch-misses             #    0.01% of all branches             
fastMod/64/32       69,907          branch-misses             #    0.01% of all branches                     
fastMod/64/128      75,747          branch-misses             #    0.01% of all branches                  
fastMod/64/224      972,921         branch-misses             #    0.07% of all branches             
fastMod/256/32      1,401,461       branch-misses             #    0.22% of all branches                   
fastMod/256/128     151,744         branch-misses             #    0.01% of all branches                   
fastMod/256/224     11,978,222      branch-misses             #    0.71% of all branches              
fastMod/1024/32     4,972,570       branch-misses             #    0.66% of all branches             
fastMod/1024/128    6,387,262       branch-misses             #    0.65% of all branches            
fastMod/1024/224    29,981,203      branch-misses             #    2.79% of all branches             
```

Notice how there are almost 0% branch misses for both benchmarks? That should be an immediate red flag. If our branch for conditionally performing modulo is based on a random input, we'd expect the branch predictor have a difficult time predicting the outcome. If the branch is skewed one direction, we'd expect the branch preductor unit (BPU) to perform better (i.e., for the `ceil=32` and `ceil=224`, cases where we almost always perform modulo, or almost never do). So why are we seeing roughly the same miss-prediction rate for all three cases?

It's the input data! We create one vector of random numbers, and re-use those random numbers for every iteration of the benchmarking loop. That means that the branch predictor can learn the branch outcomes from the first few iterations of the benchmark, and predict them with a high degree of accuracy for the remainder of the iterations. When we increase the size of the vectors, we put more pressure on the BPU. We're not measuring how `fastMod` performs with random input data, we're measuring how it performs with random input data *and* almost perfect branch prediction.

The BPU has a finite amount of state to record information about branch outcome history. When we use larger vector sizes, the BPU stops being able to "memorize" all the branch outcomes. It's only at this point where our input stream of data starts to look like a random input stream of data, and not just a repeated pattern.

Don't believe me? Good! Stay skeptical. Make people prove their assertions. Let's place a cap on the number of iterations of our `fastMod` benchmark. We'll test 1, 1000, and 100000 iterations. Note, with fewer iterations, the timing information will be less stable (i.e., you may see the inconsistent timing results). To roughly account for this, I ran the test multiple times am reporting the best results for each iteration step. We don't need to be extremely precise here, as there will already be a _huge_ difference in performance.

Here were the results for iterations=1.

```
------------------------------------------------------------------------
Benchmark                              Time             CPU   Iterations
------------------------------------------------------------------------
fastMod/16/32/iterations:1          2388 ns         1078 ns            1
fastMod/16/128/iterations:1         1008 ns          837 ns            1
fastMod/16/224/iterations:1          820 ns          758 ns            1
fastMod/64/32/iterations:1          1025 ns          989 ns            1
fastMod/64/128/iterations:1         1204 ns         1090 ns            1
fastMod/64/224/iterations:1          938 ns          898 ns            1
fastMod/256/32/iterations:1         1716 ns         1674 ns            1
fastMod/256/128/iterations:1        2320 ns         2207 ns            1
fastMod/256/224/iterations:1        1524 ns         1404 ns            1
fastMod/1024/32/iterations:1        5049 ns         4886 ns            1
fastMod/1024/128/iterations:1       6792 ns         6627 ns            1
fastMod/1024/224/iterations:1       3578 ns         3409 ns            1
```

Here are the results for iterations=1,000.

```
---------------------------------------------------------------------------
Benchmark                                 Time             CPU   Iterations
---------------------------------------------------------------------------
fastMod/16/32/iterations:1000          59.9 ns         58.6 ns         1000
fastMod/16/128/iterations:1000         41.0 ns         40.8 ns         1000
fastMod/16/224/iterations:1000         36.7 ns         36.7 ns         1000
fastMod/64/32/iterations:1000           242 ns          242 ns         1000
fastMod/64/128/iterations:1000          129 ns          129 ns         1000
fastMod/64/224/iterations:1000          143 ns          143 ns         1000
fastMod/256/32/iterations:1000          966 ns          965 ns         1000
fastMod/256/128/iterations:1000         660 ns          660 ns         1000
fastMod/256/224/iterations:1000         393 ns          393 ns         1000
fastMod/1024/32/iterations:1000        3125 ns         3126 ns         1000
fastMod/1024/128/iterations:1000       2071 ns         2072 ns         1000
fastMod/1024/224/iterations:1000       1315 ns         1301 ns         1000
```

And finally, the results for iterations=100,000.

```
-----------------------------------------------------------------------------
Benchmark                                   Time             CPU   Iterations
-----------------------------------------------------------------------------
fastMod/16/32/iterations:100000          60.4 ns         60.4 ns       100000
fastMod/16/128/iterations:100000         43.4 ns         43.4 ns       100000
fastMod/16/224/iterations:100000         25.7 ns         25.7 ns       100000
fastMod/64/32/iterations:100000           216 ns          216 ns       100000
fastMod/64/128/iterations:100000          123 ns          123 ns       100000
fastMod/64/224/iterations:100000          103 ns          103 ns       100000
fastMod/256/32/iterations:100000          671 ns          671 ns       100000
fastMod/256/128/iterations:100000         483 ns          483 ns       100000
fastMod/256/224/iterations:100000         317 ns          317 ns       100000
fastMod/1024/32/iterations:100000        2712 ns         2712 ns       100000
fastMod/1024/128/iterations:100000       1837 ns         1837 ns       100000
fastMod/1024/224/iterations:100000       1277 ns         1277 ns       100000

```

Notice how the performance seems to improve the more iterations we run (and by almost 20x is some cases)? Our branch predictor is learning!

Now let's compare the prediction rates of `fastMod` when we use a larger input size (N > 2^12).

```
fastMod/4096/32     18,577,245      branch-misses             #    3.48% of all branches
fastMod/4096/128    55,620,265      branch-misses             #   12.88% of all branches
fastMod/4096/224    33,225,762      branch-misses             #    4.95% of all branches
fastMod/8192/32     27,709,832      branch-misses             #    4.68% of all branches
fastMod/8192/128    44,043,830      branch-misses             #   19.35% of all branches
fastMod/8192/224    45,040,339      branch-misses             #    5.88% of all branches
fastMod/16384/32    20,877,808      branch-misses             #    5.79% of all branches
fastMod/16384/128   60,975,087      branch-misses             #   22.51% of all branches
fastMod/16384/224   60,882,352      branch-misses             #    6.67% of all branches
```

Now we have some values that make sense. When we do a modulo of 32 or 224, we'd expect a slightly better hit rate. That's because our branches will either almost always be taken or not taken. We'd expect our BPU to perform the wors with modulo 128, because approximately half our values will take the branch, and the other half will not, and in no particular order.

Let's take a look at our original performance measurements for the larger vector sizes, and use the branch miss-prediction rates to guide our analysis.

```
------------------------------------------------------------
Benchmark                  Time             CPU   Iterations
------------------------------------------------------------
fastMod/4096/32         13.3 us         13.3 us       211063
fastMod/4096/128        16.7 us         16.7 us       160856
fastMod/4096/224        9.61 us         9.60 us       288420
fastMod/8192/32         27.7 us         27.7 us        99420
fastMod/8192/128        43.3 us         43.3 us        66535
fastMod/8192/224        20.0 us         20.0 us       100000
fastMod/16384/32        59.0 us         59.0 us        47468
fastMod/16384/128       94.4 us         94.4 us        29949
fastMod/16384/224       41.3 us         41.3 us        68449
```

Our %128 cases perform the worst because of the bad miss-prediction rate. Our %32 cases are slightly worse than our `baseMod`, but not terrible. That's because the branch is biased towards one side, and the BPU does a decent job and predicting the outcomes. However, the loop is not skipping many modulo operations, and we still occasionally pay the price for miss-predicting. The %224 cases are similar performance, if not slightly better than `baseMod`. That's because the BPU is predicting the branch outcomes faily well because the branch is biased, and we're skipping many of the modulo operations.

## Concluding Remarks

- *"the most valuable thing I learned from this talk is to branch my modulos lol"* - Anonymous YouTube comment

One difficult part of benchmarking is not falling the many subtle pitfalls. When we write a benchmark, that doesn't mean our work is done. We must first verify that we are measuring what we set out to measure. In our case, we intended to measure the performance of using a branch to avoid always performing the expensive modulo operation when we have random input data. However, what we actually measured the the performance of using a branch that is _almost always predicted correctly_ to avoid an expensive modulo operation.

While it seems that this benchmark from [Chandler's talk](https://youtu.be/nXaxk27zwlk) has some key problems, it still contains a lot of valuable insights. One of the key points from the talk is to go out and measure things. In this case, it just so happens that we needed to measure a few more things (namely the branch miss-prediction rate). Looks like the example wasn't so crazy after all.

As always, feel free to contact me with questions.

Cheers,

--Nick

## Links

- [My YouTube Channel](https://www.youtube.com/channel/UCsi5-meDM5Q5NE93n_Ya7GA?view_as=subscriber)
- [My GitHub Account](https://github.com/CoffeeBeforeArch)
- [Benchmarks](https://github.com/CoffeeBeforeArch/misc_code/blob/master/conditions)
- My Email: CoffeeBeforeArch@gmail.com
