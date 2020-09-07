## CoffeeBeforeArch - The Github Page

Hey everyone I'm Nick, and welcome to the CoffeeBeforeArch GitHub page! I'm creating 100% open teaching content on a variety of topics, including introductory C++ programming, performance optimization and benchmarking, and GPU architecture.

The content I create reflects how I would have wanted to learn about programming, optimization, and architecture. Hopefully it helps others that are teaching themselves about computer science like I did.

## Recent Posts
- [Tutorial on False Sharing]({% post_url 2019-12-28-false-sharing-tutorial %})

- [Tutorial on Cache Associativity]({% post_url 2020-01-12-cache-associativity %})

- [False Sharing Detection with Perf C2C]({% post_url 2020-03-27-perf-c2c %})

- [Improving GPGPU-Sim Performance]({% post_url 2020-03-31-perf-gpgpu-sim %})

- [Disabling Asserts]({% post_url 2020-04-11-asserts %})

- [Aliasing]({% post_url 2020-04-12-aliasing %})

- [Vector Instructions]({% post_url 2020-04-23-vectorization %})

- [Avoiding Branches]({% post_url 2020-04-30-avoiding-branches %})

- [A (Not So) fastMod]({% post_url 2020-05-06-not-fast-mod %})

- [Thread Affinity]({% post_url 2020-05-27-thread-affinity %})

- [Strength Reduction]({% post_url 2020-06-06-strength-reduction %})

- [Optimizing Matrix Multiplication]({% post_url 2020-06-23-mmul %})

- [Atomic vs Mutex]({% post_url 2020-08-04-atomic-vs-mutex %})

- [Compiler Optimizations of a Clamp Function]({% post_url 2020-08-12-clamp-optimization %})

- [Compiler Optimizations of a Clamp Function]({% post_url 2020-09-06-compilation %})

## GPGPU-Sim Tutorials

- [Introduction]({% post_url 2020-03-30-gpgpu-sim-1 %})

- [Adding Configuration Options]({% post_url 2020-04-13-gpgpu-sim-2 %})

## Setup

A few people have been curious about my software/hardware setup. I've therefore provided a list of these things below, with a brief rational behind the choices.

### Software
#### Operating System - Ubuntu 18.04
If you're looking to try a GNU-Linux operating system, Ubuntu is a great place to start. It was the first distribution that I used, and supports all the software I need. There are more do-it-yourself distrubutions (like Arch), but I've never needed that level of customization.

#### Text Editor - VIM
I use VIM because it was the first text editor introduced to me. I have no ill will against people that use Emacs, Atom, etc., because it's a pointless debate. Use whatever you're comfortable with.

I use the [amix vimrc](https://github.com/amix/vimrc). It is incredibly easy to install, and contains almost every plugin you could want. The single modification I have made to this configuration is the addition of the [ClangFormat plugin](https://github.com/rhysd/vim-clang-format). [ClangFormat](http://clang.llvm.org/docs/ClangFormat.html) is a great tool for automatically formatting code. Personally, I use the Google C++ style. I have modified my .vimrc file to bind a function key to automatically run ClangFormat.

#### Compiler - GCC, Clang, NVCC, PGI
For many of my C++ programming videos, I use gcc/g++. This is because it's the default compiler on Linux, and many people are familier with. Clang is my preferred compiler becauase of how modular the infrastrucutre is. I have recently started doing more compiler work, and it is really the only choice. Performance seems to be better using Clang as well.

The majority of programming I do is for GPUs (specifically NVIDIA ones). As a result NVCC is the primary compiler I use when writing CUDA code. However, I do use Clang when doing GPU compiler research. I use the PGI compiler when looking into pragma-based GPU programming with OpenACC.

#### Debugging - Valgrind, GDB, CUDA-GDB, Address Sanitizer
Valgrind, GDB, and CUDA-GDB are the primary debuggers I use.  Address Sanitizer is another good option that can speed up the detection of memory corruption bugs compared to Valgrind.

#### Build Software - CMake
CMake is a common tool used by industry of building and testing software. Reduced down, it's a tool for automatically generating Makefiles. It's fairly easy to get started with CMake, and I've been gradually integrating it with all of my personal projects.

