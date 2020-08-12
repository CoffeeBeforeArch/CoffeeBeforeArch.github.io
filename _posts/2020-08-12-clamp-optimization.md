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

Next, we see threeYou will find that most people that do the research and design work at places like Intel, NVIDIA, AMD, etc will have at least a masters or PhD. It’s possible you could eventually work your way up with only a bachelors, but it would likely take far longer, and it is not the common case. Ultimately a research position requires experience with research. Something most undergraduates have little-to-no formal experience with at the level being asked.

 calls to `std::vector<int, std::allocator<int> >::operator[]`. Remember, a vector is really just a class that implements the `[]` operator, which is just another method. The first call to the `[]` operator is our read to `v_in`. The other two `[]` operators are for writing to `v_out` (one for if we have to clamp the input, and the other if we don't).

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

We can compile the code with the following commmand:

```bash
g++ clamp_bench.cpp -lbenchmark -lpthread -O0 -o clamp
```

And here is the output assembly:

```
 13.27 │2a3:┌─→mov    -0x24(%rbp),%eax                                                        
  1.39 │    │  cmp    -0x28(%rbp),%eax                                                        
       │    │↓ jge    2eb                                                                    
  0.73 │    │  mov    -0x24(%rbp),%eax                                                        
  0.61 │    │  cltq                                                                         
 11.88 │    │  lea    0x0(,%rax,4),%rdx                                                     
  0.92 │    │  mov    -0x30(%rbp),%rax                                                      
  0.98 │    │  add    %rdx,%rax                                                               
 22.11 │    │  mov    (%rax),%eax                                                             
  6.20 │    │  mov    -0x24(%rbp),%edx                                                        
  0.12 │    │  movslq %edx,%rdx                                                               
  0.03 │    │  lea    0x0(,%rdx,4),%rcx                                                       
  6.01 │    │  mov    -0x38(%rbp),%rdx                                                        
  6.11 │    │  add    %rcx,%rdx                                                               
  0.15 │    │  mov    $0x200,%ecx                                                             
  0.06 │    │  cmp    $0x200,%eax                                                             
  6.87 │    │  cmovg  %ecx,%eax                                                               
 20.54 │    │  mov    %eax,(%rdx)                                                             
  1.24 │    │  addl   $0x1,-0x24(%rbp)                                                        
  0.08 │    └──jmp    2a3                                                                     
```

Still un-optimized, but a lot cleaner than our std::vector implementation. Furthermore, you can see we don't have a branch anymore for our clamp. It's been replaced by a `cmovg`, which conditionally moves a value based on the result of the previous compare (`cmp`). 

In previous blog posts, we've looked at how branchless alternatives to code are often beneficial because of the large price of branch misprediction. Let's measure run our benchmark to see how much performance improved by getting rid of the branch on `std::vector` overhead.

Here are the performance results:

```
------------------------------------------------------------------------
Benchmark                              Time             CPU   Iterations
------------------------------------------------------------------------
clamp_bench_raw_ptr/8                686 ns          682 ns      1033335
clamp_bench_raw_ptr/9               1347 ns         1339 ns       524740
clamp_bench_raw_ptr/10              2831 ns         2810 ns       267156
```

Fairly large! Let's keep track on how this changes as we increase the optimization levels.

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

Functionally the same as our previous two implementations, but we're relying more heavily on the STL.

Let's take a look at the assembly:

```
  0.15 │17:┌─→lea    -0x20(%rbp),%rdx                                                                                                                 
       │   │  lea    -0x18(%rbp),%rax                                                                                                                 
  0.04 │   │  mov    %rdx,%rsi                                                                                                                        
  5.72 │   │  mov    %rax,%rdi                                                                                                                        
  0.19 │   │→ callq  bool __gnu_cxx::operator!=<int*, std::vector<int, std::allocator<int                                                             
  1.65 │   │  test   %al,%al                                                                                                                          
  0.04 │   │↓ je     408d41 <__gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > > std::transform<__gnu_cxx::__normal_iterator<i
  0.12 │   │  lea    -0x18(%rbp),%rax                                                                                                                 
  4.53 │   │  mov    %rax,%rdi                                                                                                                        
  1.96 │   │→ callq  __gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int                                                           
 31.49 │   │  mov    (%rax),%r12d                                                                                                                     
  0.88 │   │  lea    -0x28(%rbp),%rax                                                                                                                 
       │   │  mov    %rax,%rdi                                                                                                                        
  6.75 │   │→ callq  __gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int                                                           
  0.08 │   │  mov    %rax,%rbx                                                                                                                        
  2.34 │   │  lea    -0x29(%rbp),%rax                                                                                                                 
  0.08 │   │  mov    %r12d,%esi                                                                                                                       
  3.53 │   │  mov    %rax,%rdi                                                                                                                        
  2.61 │   │→ callq  clamp_bench_lambda(benchmark::State&)::{lambda(int)#2}::operator()(int) const                                                    
 15.96 │   │  mov    %eax,(%rbx)                                                                                                                      
  0.50 │   │  lea    -0x18(%rbp),%rax                                                                                                                 
  0.15 │   │  mov    %rax,%rdi                                                                                                                        
  6.83 │   │→ callq  __gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int                                                           
  4.49 │   │  lea    -0x28(%rbp),%rax                                                                                                                 
  0.12 │   │  mov    %rax,%rdi                                                                                                                        
  1.99 │   │→ callq  __gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int                                                           
  7.25 │   └──jmp    408ce1 <__gnu_cxx::__normal_iterator<int*, std::vector<int, std::allocator<int> > > std::transform<__gnu_cxx::__normal_iterator<i
```

Let's parse what's going on here. Each iteration of the loop, we access our input element through our vector iterators. Our input is then passed to our clamp lambda which returns either the ceiling or the input value. Finally, we store that result through a vector iterator. Let's look at the code generated for our lambda in greater detail:

```
       │    0000000000408432 <clamp_bench_lambda(benchmark::State&)::{lambda(int)#2}::operato
       │    _ZZL18clamp_bench_lambdaRN9benchmark5StateEENKUliE0_clEi():
  0.75 │      push   %rbp
 18.47 │      mov    %rsp,%rbp
 12.87 │      mov    %rdi,-0x8(%rbp)
  1.49 │      mov    %esi,-0xc(%rbp)
 16.98 │      mov    $0x200,%eax
 10.07 │      cmpl   $0x200,-0xc(%rbp)
 19.78 │      cmovle -0xc(%rbp),%eax
  2.61 │      pop    %rbp
 16.98 │    ← retq
```

Very similar to our raw pointer implementation. Instead of a branch, our compiler again opted to use a conditional move (`cmov`) instruction (this time using a less-than comparison).

Now let's check out the performance:

```
------------------------------------------------------------------------
Benchmark                              Time             CPU   Iterations
------------------------------------------------------------------------
clamp_bench_lambda/8                5054 ns         4972 ns       134519
clamp_bench_lambda/9                9882 ns         9714 ns        71631
clamp_bench_lambda/10              19120 ns        18930 ns        37458
```

## Final Thoughts

Thanks for reading,

--Nick

### Link to the source code

- [Source Code: ](https://github.com/CoffeeBeforeArch/misc_code/tree/master/clamp)
- [My YouTube Channel: ](https://www.youtube.com/channel/UCsi5-meDM5Q5NE93n_Ya7GA?view_as=subscriber)
- [My GitHub Account: ](https://github.com/CoffeeBeforeArch)
- My Email: CoffeeBeforeArch@gmail.com


