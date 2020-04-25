---
layout: default
title: Vectorization
---

# Vectorization

The majority of arithmetic hardware in modern processors is for vector instructions. This means that if we want to make effective use of our silicon, we need to manually vectorize our programs or get some help from the compiler. In this blog post, we'll be taking a look at the vectorization of a dot product, and be focusing on programmability, readability, and performance.

## Links

- [My YouTube Channel](https://www.youtube.com/channel/UCsi5-meDM5Q5NE93n_Ya7GA?view_as=subscriber)
- [My GitHub Account](https://github.com/CoffeeBeforeArch)
- [Dot Product Benchmarks](https://github.com/CoffeeBeforeArch/misc_code/blob/master/dot_product/dp.cpp)
- My Email: CoffeeBeforeArch@gmail.com

## Compiling the Code

The following results were collected using [Google Benchmark](https://github.com/google/benchmark) and the [perf Linux profiler](https://perf.wiki.kernel.org/index.php/Main_Page). The benchmark code was compiled using the following was compiled using:

```bash
g++ dp.cpp -std=c++2a -O3 -lbenchmark -lpthread -ltbb -march=native -mtune=native
```

The Google benchmark code all take roughly the following form.

```cpp
// Benchmark the baseline C-style dot product
static void DP(benchmark::State &s) {
  // Get the size of the vector
  size_t N = 1 << s.range(0);

  // Initialize the vectors
  std::vector<float> v1;
  std::fill_n(std::back_inserter(v1), N, rand() % 100);
  std::vector<float> v2;
  std::fill_n(std::back_inserter(v2), N, rand() % 100);

  // Keep the result from being optimized away
  volatile float result = 0;

  // Our benchmark loop
  while (s.KeepRunning()) {
    result = dot_product#(v1, v2);
  }
}
BENCHMARK(DP)->DenseRange(8, 10);
```

All we're doing is creating 2 vectors of 2^N elements and passing them into our `dot_product1(...)` function. I've marked the variable that stores the value returned by the function as volatile to prevent it from being optimized away by the compiler. Based on our `DenseRange(8, 10)` at the end of the benchmark, we'll be testing vector sizes of 2^8, 2^9, and 2^10 elements.

## Baseline Dot Product

Dot product can be simply explained as the sum of products of two sequences of numbers. As such, it's fairly easy to write a function that computes it. For example, we can write the following as a first pass.

```cpp
int dot_product1(std::vector<float> &__restrict v1,
                 std::vector<float> &__restrict v2) {
  auto tmp = 0;
  for (size_t i = 0; i < v1.size(); i++) {
    tmp += v1[i] * v2[i];
  }
  return tmp;
}
```

Nothing crazy in this first implementation. All we're doing is using a for loop to iterate through both vectors, multiplying the elements together, and accumulating the partial results into a temporary variable. One thing to note is that this is a very C-style way to write this function (with the exception of using a vector). It's not incredibly difficult to discern how things are being calculated, but it also isn't immedaitely clear.

Let's run our benchmark and see what we get as results.

```
-----------------------------------------------------
Benchmark           Time             CPU   Iterations
-----------------------------------------------------
baseDP/8          276 ns          276 ns     10200185
baseDP/9          612 ns          612 ns      4578259
baseDP/10        1284 ns         1284 ns      2181952
```

We don't really have any context if this is good or bad at this point, but we can look at the assembly to see what we're doing, and where the hotspots of our code is. Here is some profiling done with `perf record`.

```asm
       │1c8:┌─→vmovups (%rcx,%r8,1),%ymm4                                                      ▒
  3.10 │    │  vmulps (%rdx,%r8,1),%ymm4,%ymm1                                                 ▒
  0.45 │    │  add    $0x20,%r8                                                                ▒
  9.64 │    │  vaddss %xmm1,%xmm0,%xmm0                                                        ▒
  0.02 │    │  vshufps $0x55,%xmm1,%xmm1,%xmm3                                                 ▒
       │    │  vshufps $0xff,%xmm1,%xmm1,%xmm2                                                 ▒
 12.52 │    │  vaddss %xmm0,%xmm3,%xmm3                                                        ▒
  0.03 │    │  vunpckhps %xmm1,%xmm1,%xmm0                                                     ▒
 12.04 │    │  vaddss %xmm3,%xmm0,%xmm0                                                        ▒
 12.22 │    │  vaddss %xmm0,%xmm2,%xmm2                                                        ▒
  0.40 │    │  vextractf128 $0x1,%ymm1,%xmm0                                                   ◆
       │    │  vshufps $0x55,%xmm0,%xmm0,%xmm1                                                 ▒
 11.73 │    │  vaddss %xmm2,%xmm0,%xmm2                                                        ▒
 12.80 │    │  vaddss %xmm2,%xmm1,%xmm2                                                        ▒
  0.43 │    │  vunpckhps %xmm0,%xmm0,%xmm1                                                     ▒
       │    │  vshufps $0xff,%xmm0,%xmm0,%xmm0                                                 ▒
 11.68 │    │  vaddss %xmm2,%xmm1,%xmm1                                                        ▒
 12.18 │    │  vaddss %xmm0,%xmm1,%xmm0                                                        ▒
  0.00 │    ├──cmp    %r8,%r9                                                                  ▒
  0.43 │    └──jne    1c8                                                                      ▒

```

It looks like the compiler was able to do some auto-vectorization for us! Without trying to decipher exactly what is going on, we can see that the majority of our time is being spent on `vaddss`, a vector add instruction using the 128 bit `xmm` registers, and `vmulps`, a vector multiplication instruction using the 256 bit `ymm` registers.

## A Modern C++ Dot Product

Modern C++ has a number of useful features can enable faster and more expressive code. The algorithms in the STL (standard template library) are examples of this. Let's update the following code to use both the STL and execution policies that are part of the Parallelism TS.

```cpp
// Modern C++ dot product
float dot_product2(std::vector<float> &__restrict v1,
                 std::vector<float> &__restrict v2) {
  return std::transform_reduce(std::execution::unseq, begin(v1), end(v1),
                               begin(v2), 0.0f);
}
```

Our entire function has been reduced to a single call to `std::transform_reduce(...)`. By default, transform reduce will perform a pair-wise multiplication of two ranges of elements, then reduce them using addition into an initial value (`0.0f` for us). We will even pass in the `std::execution::unseq` execution policy to give the hint that this operation should be vectorized.

Let's go ahead and run this benchmark and collect the performance numbers for the same input matrix sizes.

```
------------------------------------------------------
Benchmark            Time             CPU   Iterations
------------------------------------------------------
modernDP/8         287 ns          283 ns      9827647
modernDP/9         626 ns          625 ns      4457001
modernDP/10       1325 ns         1324 ns      2146599
```

Pretty close to our original function! Now let's see what the assembly looks like.

```asm
  0.05 │1c8:┌─→vmovups (%rdx,%r8,1),%ymm4                                                                                                  ▒
  3.07 │    │  vmulps (%rax,%r8,1),%ymm4,%ymm1                                                                                             ▒
  0.47 │    │  add    $0x20,%r8                                                                                                            ▒
  9.72 │    │  vaddss %xmm1,%xmm0,%xmm0                                                                                                    ▒
  0.02 │    │  vshufps $0x55,%xmm1,%xmm1,%xmm3                                                                                             ▒
       │    │  vshufps $0xff,%xmm1,%xmm1,%xmm2                                                                                             ▒
 12.12 │    │  vaddss %xmm0,%xmm3,%xmm3                                                                                                    ▒
  0.04 │    │  vunpckhps %xmm1,%xmm1,%xmm0                                                                                                 ▒
 11.95 │    │  vaddss %xmm3,%xmm0,%xmm0                                                                                                    ▒
 12.65 │    │  vaddss %xmm0,%xmm2,%xmm2                                                                                                    ▒
  0.52 │    │  vextractf128 $0x1,%ymm1,%xmm0                                                                                               ▒
  0.00 │    │  vshufps $0x55,%xmm0,%xmm0,%xmm1                                                                                             ▒
 11.60 │    │  vaddss %xmm2,%xmm0,%xmm2                                                                                                    ▒
 12.68 │    │  vaddss %xmm2,%xmm1,%xmm2                                                                                                    ▒
  0.42 │    │  vunpckhps %xmm0,%xmm0,%xmm1                                                                                                 ▒
  0.00 │    │  vshufps $0xff,%xmm0,%xmm0,%xmm0                                                                                             ▒
 11.61 │    │  vaddss %xmm2,%xmm1,%xmm1                                                                                                    ▒
 12.22 │    │  vaddss %xmm0,%xmm1,%xmm0                                                                                                    ◆
       │    ├──cmp    %r8,%r9                                                                                                              ▒
  0.44 │    └──jne    1c8                                                                                                                  ▒

```

The same inner-loop! We got just as optimized of a dot product implementation using the STL as we did with a hand-rolled loop.

## Modern Dot Product w/ Double Initial Value

One interesting thing I observed with `transform_reduce(...)` is the difference in performance when the type of the initial value is changed. For our `modernDP` run, we accumulated the dot product of two vectors of floats into a floating point number (`0.0f`). However, let's see what happens when we change the precision to double (e,g, `0.0`). These are the performance numbers I measured.

```asm
-------------------------------------------------------------
Benchmark                   Time             CPU   Iterations
-------------------------------------------------------------
modernDP_double/8         196 ns          195 ns     14348574
modernDP_double/9         388 ns          388 ns      7210663
modernDP_double/10        776 ns          776 ns      3573320
```

Our performance... increased (and significantly)! Let's take a look at the assembly to see what happened.

```asm
  1.13 │ a0:   vmovss (%rax),%xmm8                                                                                                         ▒
  0.08 │       vmovss 0x4(%rax),%xmm9                                                                                                      ◆
  3.67 │       vmovss 0x8(%rax),%xmm10                                                                                                     ▒
  0.18 │       vmovss 0xc(%rax),%xmm11                                                                                                     ▒
  1.23 │       vmovss 0x10(%rax),%xmm12                                                                                                    ▒
  0.11 │       vmovss 0x14(%rax),%xmm13                                                                                                    ▒
  3.77 │       vmovss 0x18(%rax),%xmm14                                                                                                    ▒
  0.21 │       vmovss 0x1c(%rax),%xmm15                                                                                                    ▒
  1.34 │       vcvtsd2ss %xmm5,%xmm5,%xmm5                                                                                                 ▒
  3.94 │       vcvtsd2ss %xmm6,%xmm6,%xmm6                                                                                                 ▒
  3.39 │       vfmadd231ss (%rdx),%xmm8,%xmm5                                                                                              ▒
  3.70 │       vfmadd231ss 0x4(%rdx),%xmm9,%xmm6                                                                                           ▒
  1.40 │       vcvtsd2ss %xmm7,%xmm7,%xmm7                                                                                                 ▒
  3.72 │       vcvtsd2ss %xmm1,%xmm1,%xmm1                                                                                                 ▒
  3.15 │       vfmadd231ss 0x8(%rdx),%xmm10,%xmm7                                                                                          ▒
  3.26 │       vfmadd231ss 0xc(%rdx),%xmm11,%xmm1                                                                                          ▒
  1.37 │       vcvtsd2ss %xmm4,%xmm4,%xmm4                                                                                                 ▒
  3.70 │       vcvtsd2ss %xmm3,%xmm3,%xmm3                                                                                                 ▒
  2.03 │       vfmadd231ss 0x10(%rdx),%xmm12,%xmm4                                                                                         ▒
  4.12 │       vfmadd231ss 0x14(%rdx),%xmm13,%xmm3                                                                                         ▒
  0.99 │       vcvtsd2ss %xmm2,%xmm2,%xmm2                                                                                                 ▒
  4.21 │       vcvtsd2ss %xmm0,%xmm0,%xmm0                                                                                                 ▒
  2.01 │       vfmadd231ss 0x18(%rdx),%xmm14,%xmm2                                                                                         ▒
  3.78 │       vfmadd231ss 0x1c(%rdx),%xmm15,%xmm0                                                                                         ▒
  0.29 │       add    $0x20,%rax                                                                                                           ▒
  0.77 │       add    $0x20,%rdx                                                                                                           ▒
  4.59 │       vcvtss2sd %xmm5,%xmm5,%xmm5                                                                                                 ▒
  3.55 │       vcvtss2sd %xmm6,%xmm6,%xmm6                                                                                                 ▒
  4.80 │       vcvtss2sd %xmm7,%xmm7,%xmm7                                                                                                 ▒
  4.18 │       vcvtss2sd %xmm1,%xmm1,%xmm1                                                                                                 ▒
  5.17 │       vcvtss2sd %xmm4,%xmm4,%xmm4                                                                                                 ▒
  4.79 │       vcvtss2sd %xmm3,%xmm3,%xmm3                                                                                                 ▒
  5.18 │       vcvtss2sd %xmm2,%xmm2,%xmm2                                                                                                 ▒
  3.96 │       vcvtss2sd %xmm0,%xmm0,%xmm0                                                                                                 ▒
  0.00 │       cmp    %r8,%rax                                                                                                             ▒
  0.18 │     ↑ jne    407f80 <dot_product2(std::vector<float, std::allocator<float> >&, std::vector<float, std::allocator<float> >&)+0xa0> ▒
```

Interesting! An inner-loop with more instructions (a number of which are just of conversion) seems to be faster than one with fewer instructions! We can start understanding the performance using the built in metric groups in `perf`. We'll start with the `Pipeline` metric group.

Here are the results for accumulating into a float.

```
-------------------------------------------------------------------------
Benchmark                               Time             CPU   Iterations
-------------------------------------------------------------------------
modernDP/10/iterations:2000000       1287 ns         1287 ns      2000000

 Performance counter stats for './a.out --benchmark_filter=modernDP/10':

     5,176,229,852      uops_retired.retire_slots #      1.0 UPI                    
    10,362,438,744      inst_retired.any                                            
     7,889,094,313      cycles                                                      
     5,175,967,371      uops_executed.thread      #      2.6 ILP                    
     4,028,989,500      uops_executed.core_cycles_ge_1                                   
```

And here are the results of accumulating into a double.

```
--------------------------------------------------------------------------------
Benchmark                                      Time             CPU   Iterations
--------------------------------------------------------------------------------
modernDP_double/10/iterations:2000000        778 ns          778 ns      2000000

 Performance counter stats for './a.out --benchmark_filter=modernDP_double/10':

    13,043,778,115      uops_retired.retire_slots #      1.4 UPI                    
    18,467,776,758      inst_retired.any                                            
     4,759,288,779      cycles                                                      
    15,103,369,042      uops_executed.thread      #      6.4 ILP                    
     4,736,740,748      uops_executed.core_cycles_ge_1      
```

Some interesting results! It looks like our instruction-level parallelism (ILP) is over 2x greater in the case where we accumulate into a double. Let's dig a little deeper, and collect the number of pipeline resource stall for each benchmark.

Here are the results from accumulating into a float.

```
-------------------------------------------------------------------------
Benchmark                               Time             CPU   Iterations
-------------------------------------------------------------------------
modernDP/10/iterations:2000000       1273 ns         1273 ns      2000000

 Performance counter stats for './a.out --benchmark_filter=modernDP/10':

     5,577,130,248      resource_stalls.any                                         

       2.567031313 seconds time elapsed

```

And here are the results from accumulating into a double.

```
--------------------------------------------------------------------------------
Benchmark                                      Time             CPU   Iterations
--------------------------------------------------------------------------------
modernDP_double/10/iterations:2000000        768 ns          768 ns      2000000

 Performance counter stats for './a.out --benchmark_filter=modernDP_double/10':

     1,276,749,470      resource_stalls.any                                         

       1.558682718 seconds time elapsed

```

It looks like we encounter ~3.5x more pipeline resource stalls when accumulating into a float than into a double!

So what happened? My initial impression is that when we accumulate into a double, there is a greater number of independant instructions than our loop that accumulates results into a float. By having more instructions with fewer dependancies between them, we can better utilize our available hardware resouces instead of stalling and waiting on dependencies. This seems to compensate larger number of instructions executed.

## Hand-Tuned Dot Product

The compiler vectorized our program (to the best of it's ability), but that doesn't mean it selected the correct instructions (from a performance standpoint). One key challenge of translating high-level code into low-level instructions is understanding what the programmer intended. While it's obvious to us that we're performing a dot product, it's not necessarily obvious to the compiler.

Let's write a final version of our dot product function that uses the dot product instrinsic. Here's how mine looks.

```cpp
// Hand-vectorized dot product
float dot_product4(const float *__restrict v1, const float *v2,
                   const size_t N) {
  auto tmp = 0.0f;
  for (size_t i = 0; i < N; i += 8) {
    // Temporary variables to help with intrinsic
    float r[8];
    __m256 rv;

    // Our dot product intrinsic
    rv = _mm256_dp_ps(_mm256_load_ps(v1 + i), _mm256_load_ps(v2 + i), 0xf1);

    // Avoid type punning using memcpy
    std::memcpy(r, &rv, sizeof(float) * 8);

    tmp += r[0] + r[4];
  }
  return tmp;
}
```

Here, we're directly using the vector dot product intrinsic `mm_256_dp_ps(...)`. This does a dot product on 8 floating point numbers at a time. One thing to note is that this intrinsic does not perform the full reduction (it does a partial reduction of the upper and lower products only). This explains the `tmp += r[0] + r[4]` at the end of the loop.

One thing to note here is that intrinsics usually require memory that has been aligned to some boundary. For the case of `_mm256_dp_ps(...)`, I've aligned the arrays of floats to 32 bytes. Your program will likely crash if you just call `new` or `malloc` instead of something like `aligned_alloc` or `posix_memalign`.

Here is how the Google benchmark code looks:

```cpp
// Benchmark our hand-tuned dot product
static void handTunedDP(benchmark::State &s) {
  // Get the size of the vector
  size_t N = 1 << s.range(0);

  // Initialize the vectors
  // Align memory to 32 bytes for the vector instruction
  float *v1 = (float *)aligned_alloc(32, N * sizeof(float));
  float *v2 = (float *)aligned_alloc(32, N * sizeof(float));
  for (size_t i = 0; i < N; i++) {
    v1[i] = rand() % 100;
    v2[i] = rand() % 100;
  }

  // Keep our result from being optimized away
  volatile float result = 0;

  // Our benchmark loop
  while (s.KeepRunning()) {
    result = dot_product4(v1, v2, N);
  }
}
BENCHMARK(handTunedDP)->DenseRange(8, 10)->Iterations(2000000);
```

Fairly close to the other benchmarks. Let's take a look at the performance results.

```
---------------------------------------------------------
Benchmark               Time             CPU   Iterations
---------------------------------------------------------
handTunedDP/8        80.4 ns         80.4 ns     34932124
handTunedDP/9         147 ns          147 ns     18977751
handTunedDP/10        298 ns          298 ns      9396390
```

Significantly faster than all three previous benchmarks! Now, let's take a look at the assembly that was generated.

```asm
  0.30 │108:┌─→vmovaps 0x0(%r13,%rax,4),%ymm3                                                 ▒
 29.56 │    │  vdpps  $0xf1,(%r14,%rax,4),%ymm3,%ymm0                                         ▒
  0.20 │    │  add    $0x8,%rax                                                               ▒
  0.40 │    │  vmovaps %xmm0,%xmm1                                                            ▒
  0.03 │    │  vextractf128 $0x1,%ymm0,%xmm0                                                  ▒
 25.46 │    │  vaddss %xmm0,%xmm1,%xmm0                                                       ▒
 43.43 │    │  vaddss %xmm0,%xmm2,%xmm2                                                       ▒
       │    ├──cmp    %rax,%rbx                                                               ▒
  0.08 │    └──ja     108                                                                     ▒

```

We can see the instruction we manually selected near the top of the loop (`vdpps`). Intrinsics are really just wrappers around assembly instructions. 

## Pitfalls in Performance Analysis

Let's take a look at some of those pipeline statistics from earlier and discuss how they can be misleading if used incorrectly.

```
----------------------------------------------------------------------------
Benchmark                                  Time             CPU   Iterations
----------------------------------------------------------------------------
handTunedDP/10/iterations:2000000        386 ns          386 ns      2000000

 Performance counter stats for './a.out --benchmark_filter=handTunedDP/10':

     3,353,158,730      uops_retired.retire_slots #      1.4 UPI                    
     4,663,122,016      inst_retired.any                                            
     2,386,337,167      cycles                                                      
     3,103,026,491      uops_executed.thread      #      3.4 ILP                    
     1,839,325,335      uops_executed.core_cycles_ge_1                                                              

```

It looks like we have lower instruction-level parallelism (ILP) than the `modernDP_double` benchmark from earlier. However, we are still over 2x faster! ILP isn't everything! We're working with a very complicated multi-variate problem (CPU performance). This means the explanation of one performance difference may give no insights into another. Let's take a look at the FLOPs metric group to gain some better insights.

Here are the results from the `modernDP_double`.

```
--------------------------------------------------------------------------------
Benchmark                                      Time             CPU   Iterations
--------------------------------------------------------------------------------
modernDP_double/10/iterations:2000000        776 ns          776 ns      2000000

 Performance counter stats for './a.out --benchmark_filter=modernDP_double/10':

     4,053,377,963      fp_arith_inst_retired.scalar_single #      2.6 GFLOPs                   (66.57%)
                38      fp_arith_inst_retired.scalar_double                                     (66.56%)
                 0      fp_arith_inst_retired.128b_packed_double                                     (66.56%)
                 0      fp_arith_inst_retired.128b_packed_single                                     (66.68%)
                 0      fp_arith_inst_retired.256b_packed_double                                     (66.87%)
         6,007,313      fp_arith_inst_retired.256b_packed_single                                     (66.76%)

```

And here are the results for `handTunedDP`.

```
----------------------------------------------------------------------------
Benchmark                                  Time             CPU   Iterations
----------------------------------------------------------------------------
handTunedDP/10/iterations:2000000        396 ns          396 ns      2000000

 Performance counter stats for './a.out --benchmark_filter=handTunedDP/10':

       512,503,977      fp_arith_inst_retired.scalar_single #      5.7 GFLOPs                   (66.80%)
                37      fp_arith_inst_retired.scalar_double                                     (66.81%)
                 0      fp_arith_inst_retired.128b_packed_double                                     (66.80%)
                 0      fp_arith_inst_retired.128b_packed_single                                     (66.80%)
                 0      fp_arith_inst_retired.256b_packed_double                                     (66.40%)
       514,838,204      fp_arith_inst_retired.256b_packed_single                                     (66.40%)

```

For our hand-tuned version, we're primarily doing 256-bit packed single-precision operations, while in the `modernDP_double` version, we're primarily doing scalar single-precision opreations with only a few 256-bit packed single-precision operations. This likely explains why we have almost ~2x the GFLOPs!

## Concluding Remarks

Vector instructions give us the opportunity to exploit data-level parallelism in our programs. While our compiler does a good job of accelerating our programs using these instructions, it can still have trouble interpreting what exactly we are trying to do.

Furthermore, performance is non-intuitive! We saw how a loop with more instructions wound up being faster than one with fewer, and how a lower level of instruction level parallelism still could lead to a faster program because of which instructions were being executed.

As always, feel free to contact me with questions.

Cheers,

--Nick

## Links

- [My YouTube Channel](https://www.youtube.com/channel/UCsi5-meDM5Q5NE93n_Ya7GA?view_as=subscriber)
- [My GitHub Account](https://github.com/CoffeeBeforeArch)
- My Email: CoffeeBeforeArch@gmail.com
