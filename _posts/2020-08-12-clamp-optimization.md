---
layout: default
title: Compiler Optimization of a Clamp Function
---

# Compiler Optimization of a Clamp Function

Modern processors are incredibly complex, and writing functionally correct code for even a moderately complex application can be a painful and teadious endeavor. Luckily for us, we have compilers that allow us to write code in high level languages like C++ that produce not just correct code, but highly optimized code as well. In this blog post, we'll be looking at a simple clamp benchmark, and compiling it at different optimization levels, and with different optimization flags, and comparing the performance and generated assembly for each experiment.

### Link to the source code

- [Source Code: ](https://github.com/CoffeeBeforeArch/misc_code/tree/master/clamp)
- [My YouTube Channel: ](https://www.youtube.com/channel/UCsi5-meDM5Q5NE93n_Ya7GA?view_as=subscriber)
- [My GitHub Account: ](https://github.com/CoffeeBeforeArch)
- My Email: CoffeeBeforeArch@gmail.com

## The Clamp Function

Before we start looking at optimizations, we need to understand the function we're optimizing. We'll be studying a clamp function in thei blog post. A clamp function simply clamps a given unput number to a range of values. In this case, we'll be looking at a function that clamps an integer to a maximum value (as opposed to a clamp function that clamps both an upper and lower bound).

Here's an example implementation in C++:

```cpp
for (int i = 0; i < N; i++)
  v_out[i] = (v_in[i] > ceiling) ? ceiling : v_in[i];
```

For each element in an array/vector, we compare the input to the ceiling. If the input is greater than the ceiling, we store ceiling in the output array/vector. Otherwise, we store the input value to the output array/vector.

## Clamp at -O0 Optimization

Let's start our journey by looking at some un-optimized code. When we compile using gcc w/o and optimization flags, it defaults to the -O0 optimization level which generates unoptimized code. Let's compare the results of a few different C++ implementations of our clamp function.

### Clamp w/ Vectors

This implementation of our clamp function uses a for loop to clamp the vectors from an input vector, and store the results into an output vector.

```cpp
 // Benchmark for a clamp function
 static void clamp_bench(benchmark::State &s) {
   // Number of elements in the vector
   auto N = 1 << s.range(0);
 
   // Create our random number generators
   std::mt19937 rng;
   rng.seed(std::random_device()());
   std::uniform_int_distribution<int> dist(0, 1024);
 
   // Create a vector of random integers
   std::vector<int> v_in(N);
   std::vector<int> v_out(N);
   std::generate(begin(v_in), end(v_in), [&]() { return dist(rng); });
 
   // Main benchmark loop
   for (auto _ : s) {
     for (std::size_t i = 0; i < v_in.size(); i++) {
       v_out[i] = (v_in[i] > 512) ? 512 : v_in[i];
     }
   }
 }
 BENCHMARK(clamp_bench)->DenseRange(8, 10);
```

We can compile this using the following command:

```bash
g++ clamp_bench.cpp -lbenchmark -lpthread -O0 -o clamp
```

And here is the output assembly:

```
  3.50 │307:┌─→lea     -0x2780(%rbp),%rax                                            
       │    │  mov     %rax,%rdi                                                     
  2.24 │    │→ callq   std::vector<int, std::allocator<int> >::size                  
  0.34 │    │  cmp     %rax,-0x27e0(%rbp)                                            
  1.57 │    │  setb    %al                                                           
  3.37 │    │  test    %al,%al                                                       
       │    │↓ je      390                                                           
  0.09 │    │  mov     -0x27e0(%rbp),%rdx                                            
  0.27 │    │  lea     -0x2780(%rbp),%rax                                            
  1.43 │    │  mov     %rdx,%rsi                                                     
  3.58 │    │  mov     %rax,%rdi                                                     
  0.28 │    │→ callq   std::vector<int, std::allocator<int> >::operator[]            
  9.40 │    │  mov     (%rax),%eax                                                   
  1.15 │    │  cmp     $0x200,%eax                                                   
  1.48 │    │↓ jg      363                                                           
 17.41 │    │  mov     -0x27e0(%rbp),%rdx                                            
  0.48 │    │  lea     -0x2780(%rbp),%rax                                            
  1.25 │    │  mov     %rdx,%rsi                                                     
  0.46 │    │  mov     %rax,%rdi                                                     
  2.96 │    │→ callq   std::vector<int, std::allocator<int> >::operator[]            
  7.95 │    │  mov     (%rax),%ebx                                                   
  0.77 │    │↓ jmp     368                                                           
 14.62 │363:│  mov     $0x200,%ebx                                                   
  3.80 │368:│  mov     -0x27e0(%rbp),%rdx                                            
  0.78 │    │  lea     -0x2760(%rbp),%rax                                            
  2.97 │    │  mov     %rdx,%rsi                                                     
  0.78 │    │  mov     %rax,%rdi                                                     
  2.30 │    │→ callq   std::vector<int, std::allocator<int> >::operator[]            
 12.23 │    │  mov     %ebx,(%rax)                                                   
  0.28 │    │  addq    $0x1,-0x27e0(%rbp)                                            
  1.78 │    └──jmpq    307                                                           
```

Seems like a lot of assembly for such simple operation, but remember, this is largely unoptimized. Let's break it down into some key key components.

First, we see a call to `std::vector<int, std::allocator<int> >::size` each iteration of our loop. This is our for-loop's condition range check (notice, our compiler didn't hoist this loop invariant).

Next, we see three calls to `std::vector<int, std::allocator<int> >::operator[]`. Remember, a vector is really just a class that implements the `[]` operator, which is just another method. The first call to the `[]` operator is our read to `v_in`. The other two `[]` operators are for writing to `v_out` (one for if we have to clamp the input, and the other if we don't).

Now let's look at the performance:

```
------------------------------------------------------------------------
Benchmark                              Time             CPU   Iterations
------------------------------------------------------------------------
clamp_bench/8                       1711 ns         1711 ns       391179
clamp_bench/9                       4077 ns         4077 ns       173661
clamp_bench/10                      8977 ns         8976 ns        77978
```

This looks fairly slow, but we won't know for certain until we look at some more data.

### Clamp with Raw Pointers

## Final Thoughts

Thanks for reading,

--Nick

### Link to the source code

- [Source Code: ](https://github.com/CoffeeBeforeArch/misc_code/tree/master/clamp)
- [My YouTube Channel: ](https://www.youtube.com/channel/UCsi5-meDM5Q5NE93n_Ya7GA?view_as=subscriber)
- [My GitHub Account: ](https://github.com/CoffeeBeforeArch)
- My Email: CoffeeBeforeArch@gmail.com


