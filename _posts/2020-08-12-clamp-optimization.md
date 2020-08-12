---
layout: default
title: Compiler Optimization of a Clamp Function
---

# Compiler Optimization of a Clamp Function

Modern processors are incredibly complex, and writing functionally correct code for even a moderately complex application can be a painful and teadious endeavor. Luckily for us, we have compilers that allow us to write code in high level languages like C++ that produce not just correct code, but highly optimized code as well. In this blog post, we'll be looking at a simple clamp benchmark, and compiling it at different optimization levels, and with different optimization flags. We'll then look at both the performance and generated assembly for each experiment to understand how things change based on the compiler optimizations.

### Link to the source code

- [Source Code: ](https://github.com/CoffeeBeforeArch/misc_code/tree/master/clamp)
- [My YouTube Channel: ](https://www.youtube.com/channel/UCsi5-meDM5Q5NE93n_Ya7GA?view_as=subscriber)
- [My GitHub Account: ](https://github.com/CoffeeBeforeArch)
- My Email: CoffeeBeforeArch@gmail.com

### Misc. Links

For more detail about the GCC optimization flags, check out [this](https://gcc.gnu.org/onlinedocs/gcc/Optimize-Options.html) link.

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

All of these benchmarks were compiled using the following command:

```bash
g++ clamp_bench.cpp -lbenchmark -lpthread -O0 -o clamp
```

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

And here is the output assembly:

```assembly
  3.34 │307:┌─→lea     -0x2780(%rbp),%rax                                         
       │    │  mov     %rax,%rdi                                                  
  1.53 │    │→ callq   std::vector<int, std::allocator<int> >::size               
  0.31 │    │  cmp     %rax,-0x27e0(%rbp)                                         
  1.64 │    │  setb    %al                                                        
  3.53 │    │  test    %al,%al                                                    
       │    │↓ je      390                                                        
       │    │  mov     -0x27e0(%rbp),%rdx                                         
  0.14 │    │  lea     -0x2780(%rbp),%rax                                         
  1.18 │    │  mov     %rdx,%rsi                                                  
  3.36 │    │  mov     %rax,%rdi                                                  
  0.14 │    │→ callq   std::vector<int, std::allocator<int> >::operator[]         
  8.44 │    │  mov     (%rax),%eax                                                
  1.03 │    │  cmp     $0x200,%eax                                                
  1.24 │    │↓ jg      363                                                        
 17.13 │    │  mov     -0x27e0(%rbp),%rdx                                         
  0.32 │    │  lea     -0x2780(%rbp),%rax                                         
  0.78 │    │  mov     %rdx,%rsi                                                  
  0.62 │    │  mov     %rax,%rdi                                                  
  2.73 │    │→ callq   std::vector<int, std::allocator<int> >::operator[]         
  7.38 │    │  mov     (%rax),%ebx                                                
  0.55 │    │↓ jmp     368                                                        
 18.42 │363:│  mov     $0x200,%ebx                                                
  5.05 │368:│  mov     -0x27e0(%rbp),%rdx                                         
  0.36 │    │  lea     -0x2760(%rbp),%rax                                         
  3.13 │    │  mov     %rdx,%rsi                                                  
  0.60 │    │  mov     %rax,%rdi                                                  
  2.47 │    │→ callq   std::vector<int, std::allocator<int> >::operator[]         
 12.72 │    │  mov     %ebx,(%rax)                                                
  0.09 │    │  addq    $0x1,-0x27e0(%rbp)                                         
  1.56 │    └──jmpq    307                                                        
```

Seems like a lot of assembly for such simple operation, but remember, this is largely unoptimized. Let's break it down into some key key components.

First, we see a call to `std::vector<int, std::allocator<int> >::size` each iteration of our loop. This is our for-loop's condition range check (notice, our compiler didn't hoist this loop invariant).

Next, we see three calls to `::operator[]`. Remember, a vector is really just a class that implements the `[]` operator, which is just another method. The first call to the `[]` operator is our read of `v_in`. The remaining two `[]` operators are for writing to `v_out` (one for if we have to clamp the input, and the other if we don't).

Now let's look at the performance:

```
------------------------------------------------------------------------
Benchmark                              Time             CPU   Iterations
------------------------------------------------------------------------
clamp_bench/8                       2066 ns         1947 ns       360128
clamp_bench/9                       4566 ns         4296 ns       161397
clamp_bench/10                      9871 ns         9481 ns        74752
```

This looks fairly slow, but we won't know for certain until we look at some more data.

### Clamp with Raw Pointers

We seem to have a lot of extra overhead from using C++ STL containers (our calls to `operator[]` and `size` for our vectors). Let's run another experiment where we swap out our `std::vector` containers with raw pointers.

Here is what that looks like:

```cpp
// Benchmark for a clamp function
// Uses raw pointers to avoid overhead in unoptimized code
static void clamp_bench_raw_ptr(benchmark::State &s) {
  // Number of elements in the vector
  auto N = 1 << s.range(0);

  // Create our random number generators
  std::mt19937 rng;
  rng.seed(std::random_device()());
  std::uniform_int_distribution<int> dist(0, 1024);

  // Create a vector of random integers
  int *v_in = new int[N]();
  int *v_out = new int[N]();
  std::generate(v_in, v_in + N, [&]() { return dist(rng); });

  // Main benchmark loop
  for (auto _ : s) {
    for (int i = 0; i < N; i++) {
      v_out[i] = (v_in[i] > 512) ? 512 : v_in[i];
    }
  }

  delete[] v_in;
  delete[] v_out;
}
BENCHMARK(clamp_bench_raw_ptr)->DenseRange(8, 10);
```

And here is the output assembly:

```assembly
 14.20 │32e:┌─→mov     -0x27b0(%rbp),%eax                                                 
  0.57 │    │  cmp     -0x27ac(%rbp),%eax                                                 
       │    │↓ jge     38b                                                                
  0.22 │    │  mov     -0x27b0(%rbp),%eax                                                 
  0.34 │    │  cltq                                                                       
 12.82 │    │  lea     0x0(,%rax,4),%rdx                                                  
  0.54 │    │  mov     -0x27a8(%rbp),%rax                                                 
  0.42 │    │  add     %rdx,%rax                                                          
 21.44 │    │  mov     (%rax),%eax                                                        
  6.38 │    │  mov     -0x27b0(%rbp),%edx                                                 
       │    │  movslq  %edx,%rdx                                                          
       │    │  lea     0x0(,%rdx,4),%rcx                                                  
  6.65 │    │  mov     -0x27a0(%rbp),%rdx                                                 
  7.04 │    │  add     %rcx,%rdx                                                          
       │    │  mov     $0x200,%ecx                                                        
       │    │  cmp     $0x200,%eax                                                        
  6.23 │    │  cmovg   %ecx,%eax                                                          
 21.78 │    │  mov     %eax,(%rdx)                                                        
  0.67 │    │  addl    $0x1,-0x27b0(%rbp)                                                 
       │    └──jmp     32e                                                                
```

Still un-optimized, but a lot cleaner than our std::vector implementation. Furthermore, you can see we don't have a branch anymore for our clamp. It has been replaced by a `cmovg`, which conditionally moves a value if it is greater than another based on the result of the previous compare (`cmp`). 

In previous blog posts, we've looked at how branchless alternatives to code are often beneficial because of the large price of branch misprediction. Let's measure run our benchmark to see how much performance improved by getting rid of the branch and `std::vector` operator overhead.

Here are the performance results:

```
------------------------------------------------------------------------
Benchmark                              Time             CPU   Iterations
------------------------------------------------------------------------
clamp_bench_raw_ptr/8                551 ns          528 ns      1000000
clamp_bench_raw_ptr/9                860 ns          830 ns       838950
clamp_bench_raw_ptr/10              1731 ns         1650 ns       426407
```

Fairly large (4-5x)! Let's keep track on how this changes as we increase the optimization levels.

### Clamp with Vectors and std::transform

Another way we can implement our clamp function is using STL algorithm `std::transform` with the a function object for our clamp. 

Here's how that looks like using `std::vector`:

```cpp
// Benchmark for a clamp function
static void clamp_bench_lambda(benchmark::State &s) {
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

  // Our clamp function
  auto clamp = [](int in) { return (in > 512) ? 512 : in; };

  // Main benchmark loop
  for (auto _ : s) {
    std::transform(begin(v_in), end(v_in), begin(v_out), clamp);
  }
}
BENCHMARK(clamp_bench_lambda)->DenseRange(8, 10);
```

This is functionally the same as our previous two implementations, but now we're relying more heavily on the STL.

Let's take a look at the assembly:

```assembly
       │    000000000000b622 <__gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > > std::transform<__gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >, __gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >, clamp_bench_lambda(benchmark::State&)::{lambda(int)#2}>(__gnu_cxx::__normal_it
       │    _ZSt9transformIN9__gnu_cxx17__normal_iteratorIPiSt6vectorIiSaIiEEEES6_ZL18clamp_bench_lambdaRN9benchmark5StateEEUliE0_ET0_T_SC_SB_T1_():
       │      endbr64
       │      push    %rbp
       │      mov     %rsp,%rbp
       │      push    %r12
       │      push    %rbx
       │      sub     $0x20,%rsp
       │      mov     %rdi,-0x18(%rbp)
       │      mov     %rsi,-0x20(%rbp)
       │      mov     %rdx,-0x28(%rbp)
       │      lea     -0x20(%rbp),%rdx
       │      lea     -0x18(%rbp),%rax
       │      mov     %rdx,%rsi
  6.68 │      mov     %rax,%rdi
  0.03 │    → callq   __gnu_cxx::operator!=<int*, std::vector<int, std::allocator<int> > >
  0.03 │      test    %al,%al
       │    → je      b69d <__gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > > std::transform<__gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >, __gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> >
       │      lea     -0x18(%rbp),%rax
  1.91 │      mov     %rax,%rdi
  4.18 │    → callq   __gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >::operator*
 33.23 │      mov     (%rax),%r12d
  0.13 │      lea     -0x28(%rbp),%rax
       │      mov     %rax,%rdi
  7.57 │    → callq   __gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >::operator*
       │      mov     %rax,%rbx
  1.64 │      lea     -0x29(%rbp),%rax
  5.18 │      mov     %r12d,%esi
  0.03 │      mov     %rax,%rdi
  2.12 │    → callq   clamp_bench_lambda(benchmark::State&)::{lambda(int)#2}::operator()
 15.85 │      mov     %eax,(%rbx)
  0.03 │      lea     -0x18(%rbp),%rax
  0.03 │      mov     %rax,%rdi
  7.51 │    → callq   __gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >::operator++
  3.07 │      lea     -0x28(%rbp),%rax
  0.13 │      mov     %rax,%rdi
  4.21 │    → callq   __gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >::operator++
  6.03 │    → jmp     b63d <__gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > > std::transform<__gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > >, __gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> >
  0.36 │      mov     -0x28(%rbp),%rax
       │      add     $0x20,%rsp
  0.03 │      pop     %rbx
       │      pop     %r12
       │      pop     %rbp
       │    ← retq
```

Let's parse what's going on here. Each iteration of the loop, we access our input and output elements through our vector iterators with `::operator*`. Our input is then clamped with a call to our lambda. After we store the result, we move to the next element in our iterators using `::operator++`. Let's look at the code generated for our clamp lambda in greater detail:

```assembly
       │    000000000000ac00 <clamp_bench_lambda(benchmark::State&)::{lambda(int)#2}::operator()(int) const>:
       │    _ZZL18clamp_bench_lambdaRN9benchmark5StateEENKUliE0_clEi():
 29.45 │      push   %rbp
  0.71 │      mov    %rsp,%rbp
 10.84 │      mov    %rdi,-0x8(%rbp)
 24.26 │      mov    %esi,-0xc(%rbp)
  0.18 │      mov    $0x200,%eax
  0.72 │      cmpl   $0x200,-0xc(%rbp)
 25.04 │      cmovle -0xc(%rbp),%eax
  8.43 │      pop    %rbp
  0.37 │    ← retq
```

Very similar to our raw pointer implementation. Instead of a branch, our compiler opted to use a conditional move (`cmov`) instruction again (this time using a less-than comparison).

Now let's check out the performance:

```assembly
------------------------------------------------------------------------
Benchmark                              Time             CPU   Iterations
------------------------------------------------------------------------
clamp_bench_lambda/8                2883 ns         2853 ns       244106
clamp_bench_lambda/9                5749 ns         5692 ns       122683
clamp_bench_lambda/10              11894 ns        11372 ns        62469
```

Our worst performance yet! Despite using a `cmov` instruction, we still have too much overhead from our unoptimized C++ operators.

### Clamp with Raw Pointers and std::transform

A final thing we can try is using raw pointers and std::transform. Here's my implementation:

```cpp
// Benchmark for a clamp function
// Uses raw pointers to avoid overhead in unoptimized code
static void clamp_bench_raw_ptr_lambda(benchmark::State &s) {
  // Number of elements in the vector
  auto N = 1 << s.range(0);

  // Create our random number generators
  std::mt19937 rng;
  rng.seed(std::random_device()());
  std::uniform_int_distribution<int> dist(0, 1024);

  // Create a vector of random integers
  int *v_in = new int[N]();
  int *v_out = new int[N]();
  std::generate(v_in, v_in + N, [&]() { return dist(rng); });

  // Our clamp function
  auto clamp = [](int in) { return (in > 512) ? 512 : in; };

  // Main benchmark loop
  for (auto _ : s) {
    std::transform(v_in, v_in + N, v_out, clamp);
  }

  delete[] v_in;
  delete[] v_out;
}
BENCHMARK(clamp_bench_raw_ptr_lambda)->DenseRange(8, 10);
```

Let's see how our assembly changed using raw pointers:

```assembly
       │    000000000000b6ec <int* std::transform<int*, int*, clamp_bench_raw_ptr_lambda(benchmark::State&)::{lambda(int)#2}>(int*, int*, clamp_bench_raw_ptr_lambda(benchmark::State&)::{lambda(int)#2}, clamp_bench_raw_ptr_lambda(benchmark::State&)::{lambda(int)#2})>:
       │    _ZSt9transformIPiS0_ZL26clamp_bench_raw_ptr_lambdaRN9benchmark5StateEEUliE0_ET0_T_S6_S5_T1_():                                                                                                                         
  0.05 │      endbr64                                                                                                                                                                            
       │      push    %rbp                                                                                                                                                                                                                                                         
       │      mov     %rsp,%rbp                                                                                                                                                                                                                                                    
  0.03 │      sub     $0x20,%rsp                                                                                                                                                                                                                                                   
       │      mov     %rdi,-0x8(%rbp)                                                                                                                                                                                                                                              
       │      mov     %rsi,-0x10(%rbp)                                                                                                                                                                                                                                             
       │      mov     %rdx,-0x18(%rbp)                                                                                                                                                                                                                                             
  3.28 │      mov     -0x8(%rbp),%rax                                                                                                                                                                                                                                              
  0.67 │      cmp     -0x10(%rbp),%rax                                                                                                                                                                                                                                             
  0.03 │    → je      b734 <int* std::transform<int*, int*, clamp_bench_raw_ptr_lambda(benchmark::State&)::{lambda(int)#2}>(int*, int*, clamp_bench_raw_ptr_lambda(benchmark::State&)::{lambda(int)#2}, clamp_bench_raw_ptr_lambda(benchmark::State&)::{lambda(int)#2})+0x48>      
 14.56 │      mov     -0x8(%rbp),%rax                                                                                                                                                                                                                                              
 15.91 │      mov     (%rax),%edx                                                                                                                                                                                                                                                  
  1.81 │      lea     -0x19(%rbp),%rax                                                                                                                                                                                                                                             
  0.08 │      mov     %edx,%esi                                                                                                                                                                                                                                                    
  8.53 │      mov     %rax,%rdi                                                                                                                                                                                                                                                    
  8.82 │    → callq   clamp_bench_raw_ptr_lambda(benchmark::State&)::{lambda(int)#2}::operator()                                                                                                                                                                                   
  0.21 │      mov     -0x18(%rbp),%rdx                                                                                                                                                                                                                                             
 28.54 │      mov     %eax,(%rdx)                                                                                                                                                                                                                                                  
  2.83 │      addq    $0x4,-0x8(%rbp)                                                                                                                                                                                                                                              
 14.13 │      addq    $0x4,-0x18(%rbp)                                                                                                                                                                                                                                             
       │    → jmp     b704 <int* std::transform<int*, int*, clamp_bench_raw_ptr_lambda(benchmark::State&)::{lambda(int)#2}>(int*, int*, clamp_bench_raw_ptr_lambda(benchmark::State&)::{lambda(int)#2}, clamp_bench_raw_ptr_lambda(benchmark::State&)::{lambda(int)#2})+0x18>      
  0.42 │      mov     -0x18(%rbp),%rax                                                                                                                                                                                                                                             
  0.03 │      leaveq                                                                                                                                                                                                                                                               
  0.08 │    ← retq                                                                                                                                                                                                                                                                 
```

Much more clean than our implementation using vectors. Now, our `std::transform` accesses memory directly rather than calling the iterator dereference operator. However, we still see a call to our clamp lambda. Here is how our clamp was implemented:

```assembly
       │    000000000000b09a <clamp_bench_raw_ptr_lambda(benchmark::State&)::{lambda(int)#2}::operator()(int) const>:
       │    _ZZL26clamp_bench_raw_ptr_lambdaRN9benchmark5StateEENKUliE0_clEi():
  0.21 │      push   %rbp
 25.43 │      mov    %rsp,%rbp
  3.31 │      mov    %rdi,-0x8(%rbp)
  0.16 │      mov    %esi,-0xc(%rbp)
 12.88 │      mov    $0x200,%eax
 29.61 │      cmpl   $0x200,-0xc(%rbp)
 21.14 │      cmovle -0xc(%rbp),%eax
  0.11 │      pop    %rbp
  7.15 │    ← retq
```

Roughly the same as our implementation with `std::vector`. We are still using a `cmovle` instead of a branch for our clamp.

Let's see how our performance compares to our previous three implementations:

```
------------------------------------------------------------------------
Benchmark                              Time             CPU   Iterations
------------------------------------------------------------------------
clamp_bench_raw_ptr_lambda/8         553 ns          552 ns      1159259
clamp_bench_raw_ptr_lambda/9        1127 ns         1088 ns       643189
clamp_bench_raw_ptr_lambda/10       2199 ns         2168 ns       322517
```

Not bad! Only slightly slower than our implementation that used raw pointers and a hand-rolled for-loop! Based on our generated assembly, this is likely because we still have some lingering C++ operator overhead that doesn't exists in the other implementation.

### -O0 Optimization Summary

So what did we learn in this section? From our initial measurements, writing code in a more C-style seems to be faster (and significantly!). However, let's keep in mind this is *WITHOUT OPTIMIZATIONS*. This not a typical situation (except when debugging). Let's continue our experiments with `-O1` optimizations in the next section.

## Clamp at -O1 Optimization

Now that we have a solid understanding about our performance with un-optimized code, let's enabled -O1 optimziations. For GCC, this enables the following optimization flags:

```bash
-fauto-inc-dec
-fbranch-count-reg
-fcombine-stack-adjustments
-fcompare-elim
-fcprop-registers
-fdce
-fdefer-pop
-fdelayed-branch
-fdse
-fforward-propagate
-fguess-branch-probability
-fif-conversion
-fif-conversion2
-finline-functions-called-once
-fipa-profile
-fipa-pure-const
-fipa-reference
-fipa-reference-addressable
-fmerge-constants
-fmove-loop-invariants
-fomit-frame-pointer
-freorder-blocks
-fshrink-wrap
-fshrink-wrap-separate
-fsplit-wide-types
-fssa-backprop
-fssa-phiopt
-ftree-bit-ccp
-ftree-ccp
-ftree-ch
-ftree-coalesce-vars
-ftree-copy-prop
-ftree-dce
-ftree-dominator-opts
-ftree-dse
-ftree-forwprop
-ftree-fre
-ftree-phiprop
-ftree-pta
-ftree-scev-cprop
-ftree-sink
-ftree-slsr
-ftree-sra
-ftree-ter
-funit-at-a-time
```

All of these benchmarks were compiled using the following command:

```bash
g++ clamp_bench.cpp -lbenchmark -lpthread -O1 -o clamp
```

### Clamp w/ Vectors

Let's re-evaluate our clamp benchmark that uses `std::vector` containers and for-loop. Here is the generated assembly with `-O1` optimizations enabled:

```assembly
 23.13 │246:┌─→cmpl    $0x200,(%rcx,%rax,4) 
  2.01 │    │  mov     %esi,%edi            
 15.36 │    │  cmovle  (%rcx,%rax,4),%edi   
 10.47 │    │  mov     0x30(%rsp),%rdx    
 12.63 │    │  mov     %edi,(%rdx,%rax,4) 
  2.44 │    │  add     $0x1,%rax          
  4.48 │    │  mov     0x10(%rsp),%rcx    
 10.25 │    │  mov     0x18(%rsp),%rdx    
 10.59 │    │  sub     %rcx,%rdx          
  2.42 │    │  sar     $0x2,%rdx          
  0.01 │    ├──cmp     %rdx,%rax          
  4.50 │    └──jb      246                
```

Much better! Our compiler was able to get rid of our `::operator[]` overhead and turn our branch into a `cmovle` instruction. Let's see how much our performance improved:

```
------------------------------------------------------------------------
Benchmark                              Time             CPU   Iterations
------------------------------------------------------------------------
clamp_bench/8                        201 ns          201 ns      3447808
clamp_bench/9                        399 ns          399 ns      1764295
clamp_bench/10                       798 ns          798 ns       867648
```

Much better! Trimming that overhead and changing to a `cmovle` instruction made each input size about 10x faster than it was previously!

### Clamp with Raw Pointers

Let's now re-evaluate our clamp with raw pointers and for-loop. Here is the generated assembly with `-O1` optimizations enabled:

```assembly
 49.04 │1ff:┌─→cmpl    $0x200,(%rbx,%rax,4)                                    
  0.03 │    │  mov     %ecx,%edx                                               
  0.55 │    │  cmovle  (%rbx,%rax,4),%edx                                      
 49.79 │    │  mov     %edx,0x0(%rbp,%rax,4)                                   
  0.02 │    │  add     $0x1,%rax                                               
       │    ├──cmp     %rsi,%rax                                                   
       │    └──jne     1ff                                                         
```

An even-tighter inner-loop! We still have a `cmovle` instruction, and our optimizations trimmed away quite a few instructions!

Now let's see if this improved our performance:

```
------------------------------------------------------------------------
Benchmark                              Time             CPU   Iterations
------------------------------------------------------------------------
clamp_bench_raw_ptr/8                116 ns          116 ns      6026730
clamp_bench_raw_ptr/9                226 ns          226 ns      3071095
clamp_bench_raw_ptr/10               456 ns          456 ns      1539512
```

Somewhere between 4x and 5x faster than when we compiled with `-O0` optimizations, and about 2x as fast as our implementations with `std::vector` and a for-loop.

### Clamp with Vectors and std::transform

Now let's see how our worst-performing implementation faired with `-O1` optimizations enabled. Here's our new assembly:

```assembly
 48.00 │1d0:┌─→cmpl    $0x200,(%rdx,%rax,1)                     
  0.02 │    │  mov     %r8d,%ecx                                
  1.15 │    │  cmovle  (%rdx,%rax,1),%ecx                       
 49.59 │    │  mov     %ecx,(%rdi,%rax,1)                       
  0.03 │    │  add     $0x4,%rax                                
       │    ├──cmp     %rsi,%rax                                
       │    └──jne     1d0                                      
```

Very similar to our clamp benchmark with raw pointers and a for-loop when `-O1` optimizations are enabled! Let's measure the performance:

```
------------------------------------------------------------------------
Benchmark                              Time             CPU   Iterations
------------------------------------------------------------------------
clamp_bench_lambda/8                 117 ns          117 ns      6026575
clamp_bench_lambda/9                 227 ns          227 ns      3056295
clamp_bench_lambda/10                510 ns          510 ns      1357904
```

As you probably expected, the performance is similar to our last example, beacuse the assebly is almost the same.

### Clamp with Raw Pointers and std::transform

Let's see if we still get better performance by using raw pointers with `std::transform`. Here is the generated assembly:

```assembly
 48.89 │1e8:┌─→cmpl    $0x200,(%rbx,%rax,1)                 
  0.03 │    │  mov     %ecx,%edx                            
  1.12 │    │  cmovle  (%rbx,%rax,1),%edx                   
 48.94 │    │  mov     %edx,0x0(%rbp,%rax,1)                
  0.01 │    │  add     $0x4,%rax                            
       │    ├──cmp     %rax,%r12                            
       │    └──jne     1e8                                  
```

More of the same assembly we saw previously. Let's check the performance just to make sure nothing else is going on:

```assembly
------------------------------------------------------------------------
Benchmark                              Time             CPU   Iterations
------------------------------------------------------------------------
clamp_bench_raw_ptr_lambda/8         115 ns          115 ns      6017905
clamp_bench_raw_ptr_lambda/9         226 ns          226 ns      3078079
clamp_bench_raw_ptr_lambda/10        509 ns          509 ns      1355150
```

Looks good! Only our implementation that uses `std::vector` and a hand-rolled for-loop is still under-performing.

### -O1 Optimization Summary

What we've seen so far is that using things like STL containers and algorithms can have substantial overheads, but those largely dissapear when you enable just basic optimizations. In the next section, we'll continue our discussion by enabling `-O2` optimizations for our benchmarks and analyzing the results.

## Clamp at -O2 Optimization

When we enable `-O2` optimizations, we get all the previous optimizations enabled, along with the following flags in GCC:

```bash
-falign-functions  -falign-jumps 
-falign-labels  -falign-loops 
-fcaller-saves 
-fcode-hoisting 
-fcrossjumping 
-fcse-follow-jumps  -fcse-skip-blocks 
-fdelete-null-pointer-checks 
-fdevirtualize  -fdevirtualize-speculatively 
-fexpensive-optimizations 
-ffinite-loops 
-fgcse  -fgcse-lm  
-fhoist-adjacent-loads 
-finline-functions 
-finline-small-functions 
-findirect-inlining 
-fipa-bit-cp  -fipa-cp  -fipa-icf 
-fipa-ra  -fipa-sra  -fipa-vrp 
-fisolate-erroneous-paths-dereference 
-flra-remat 
-foptimize-sibling-calls 
-foptimize-strlen 
-fpartial-inlining 
-fpeephole2 
-freorder-blocks-algorithm=stc 
-freorder-blocks-and-partition  -freorder-functions 
-frerun-cse-after-loop  
-fschedule-insns  -fschedule-insns2 
-fsched-interblock  -fsched-spec 
-fstore-merging 
-fstrict-aliasing 
-fthread-jumps 
-ftree-builtin-call-dce 
-ftree-pre 
-ftree-switch-conversion  -ftree-tail-merge 
-ftree-vrp
```

`-O2` enables nearly all optimizations, as long as they don't involve a space-speed tradeoff.


The following benchmark was compiled using the following command:

```bash
g++ clamp_bench.cpp -lbenchmark -lpthread -O2 -o clamp
```

### Benchmark Results

Unlike the previous optimization levels, we will no-longer be analyzing 4 benchmarks. Why? Because they generate roughly the same assembly, and have the same performance!

Let's use the most-C++ of our implementations (using `std::vector` containers and `std::transform`) for the reaminder of our experiments. Here is the generated assembly for that benchmark:

```assembly
 49.05 │2a0:┌─→cmpl    $0x200,(%rdx,%rax,1)                        
  0.01 │    │  mov     %esi,%ecx                                   
  1.21 │    │  cmovle  (%rdx,%rax,1),%ecx                          
 48.38 │    │  mov     %ecx,(%r8,%rax,1)                           
  0.01 │    │  add     $0x4,%rax                                   
       │    ├──cmp     %rdi,%rax                                   
       │    └──jne     2a0                                         
```

The exact same assembly as we had with `-O1` optimizations. Here are the performance numbers:

```
----------------------------------------------------------------
Benchmark                      Time             CPU   Iterations
----------------------------------------------------------------
clamp_bench_lambda/8         119 ns          119 ns      5874782
clamp_bench_lambda/9         232 ns          232 ns      2998641
clamp_bench_lambda/10        521 ns          521 ns      1336188
```

Unsurprisingly, We get the about same performance as previouslly reported!

### -O2 Optimization Summary

Do `-O2` optimizations help at all? Yes, absolutely! However, we are optimizing an incredibly simple operation (a clamp function). With such a simple operation, there is a finite set of optimizations that are applicable, and none of those in the `-O2` category, in this case were helpful for our tight inner-loop.

We'll continue our exploration with -O3 optimizations in the next section.

## Clamp at -O3 Optimization

When we enabled `-O3` optimizations, we get all the previous optimization, plus the following flags in GCC:

```bash
-fgcse-after-reload
-fipa-cp-clone
-floop-interchange
-floop-unroll-and-jam
-fpeel-loops
-fpredictive-commoning
-fsplit-loops
-fsplit-paths
-ftree-loop-distribution
-ftree-loop-vectorize
-ftree-partial-pre
-ftree-slp-vectorize
-funswitch-loops
-fvect-cost-model
-fvect-cost-model=dynamic
-fversion-loops-for-strides
```

We have some very important optimizations at this level (e.g., vectorization).

Our benchmark was compiled using the following command:

```bash
g++ clamp_bench.cpp -lbenchmark -lpthread -O3 -o clamp
```

As a reminder, we wll only be looking at a single benchmark since they all now generate the same assembly.

### Benchmark Results

Let's see if our assembly changed when we enabled `-O3` optimizations:

```assembly
 18.70 │290:┌─→movdqu  0x0(%rbp,%rax,1),%xmm0                               
  1.39 │    │  movdqu  0x0(%rbp,%rax,1),%xmm3                               
 16.98 │    │  movdqa  %xmm1,%xmm2                                          
  2.73 │    │  pcmpgtd %xmm1,%xmm0                                          
 17.90 │    │  pand    %xmm0,%xmm2                                          
  1.56 │    │  pandn   %xmm3,%xmm0                                          
 18.13 │    │  por     %xmm2,%xmm0                                          
  1.50 │    │  movups  %xmm0,(%r12,%rax,1)                                  
 17.72 │    │  add     $0x10,%rax                                           
  0.03 │    ├──cmp     %rdx,%rax                                            
  1.40 │    └──jne    
```

Some new code! What happened? Our compiler performed vectorization! Vectorization allows our processor to process multiple elements in a single instruction. In this case, we're packing 4 elements into each 128-bit `xmm` register that is used. Let's see how this changed our performance:

```
------------------------------------------------------------------------
Benchmark                              Time             CPU   Iterations
------------------------------------------------------------------------
clamp_bench_lambda/8                36.5 ns         36.5 ns     19224165
clamp_bench_lambda/9                78.2 ns         78.2 ns      8927309
clamp_bench_lambda/10                154 ns          154 ns      4508519
```

Quite a significant improvement! In fact, almost 4x faster! This is somewhat to be expected. If we're using instructions that process 4 elements at a time, we would optimistically expect a 4x speedup. We don't quite hit that mark, likely because we have a few extra instructions to handle the vectorization, and maybe some minor throughput differences in the instructions.

### -O3 Optimization Summary

Vectorization can have a huge impact on performance (as seen in this example). However, many things can get into the way of the auto-vectorizer (e.g., aliasing) that can prevent vectorization. In the next section, we'll look at a few additional flags we can pass to our compiler beyond just changing the optimization level.

## Clamp at -O3 Optimization with Native Tuning

One final thing we will try are the `-march=native` and `-mtune=native` flags. These flags tell the compiler to produce code for the native system's processor architecture, and tune for the native architecture respectively. 

Our benchmark was compiled using the following command:

```bash
g++ clamp_bench.cpp -lbenchmark -lpthread -O3 -march=native -mtune=native -o clamp
```

### Benchmark Results

Let's see if compiling and tuning for the native architecture made any difference to our performance. Here is the generated assembly:

```assembly
 46.07 │2c8:┌─→vpminsd    (%r12,%rax,1),%ymm1,%ymm0                     
 14.49 │    │  vmovdqu    %ymm0,0x0(%r13,%rax,1)                        
 18.79 │    │  add        $0x20,%rax                                    
  0.26 │    ├──cmp        %rax,%rcx                                     
 18.78 │    └──jne        2c8                                           
```

Two important changes! For one, we're using 256-bit vector instructions (`ymm` registers are 256 bits) which process 8 integers at a time instead of the 4 we were previously. Secondly, we have fewer instructions, because we're using `vpminsd`, a dedicated instruction for extracting the minimum of packed numbers. Let's see how this changes performance:

```
------------------------------------------------------------------------
Benchmark                              Time             CPU   Iterations
------------------------------------------------------------------------
clamp_bench_lambda/8                10.6 ns         10.6 ns     66181966
clamp_bench_lambda/9                19.4 ns         19.4 ns     36134751
clamp_bench_lambda/10               57.3 ns         57.3 ns     12117304
```

Another ~3-4x performance improvement (somewhat expected based on the changes in the assembly)!

### -O3 Optimization with Native Tuning Summary

Without informing the compiler about the architecture it is compiling for, it has to be conservative, and it will not use any features (e.g. wider SIMD instruction) that it doens't know are supported. Telling our compiler to perform native tuning for our clamp benchmark meant that it knew it could generated 8-wide SIMD instructions, and use a dedicated instruction for extracting minimums from packed numbers.

## Final Thoughts

Code can change drastically at different optimization levels, and with different optimization flags. Understanding how these optimizations work can help us improve the performance of our applications, and guide how we write code in our high-level languages. One important parting thought is this. Before you try and perfrom micro-optimizations yourself, make sure the compiler doesn't have a switch that will do it for you (and maybe much more!).

Thanks for reading,

--Nick

### Link to the source code

- [Source Code: ](https://github.com/CoffeeBeforeArch/misc_code/tree/master/clamp)
- [My YouTube Channel: ](https://www.youtube.com/channel/UCsi5-meDM5Q5NE93n_Ya7GA?view_as=subscriber)
- [My GitHub Account: ](https://github.com/CoffeeBeforeArch)
- My Email: CoffeeBeforeArch@gmail.com

