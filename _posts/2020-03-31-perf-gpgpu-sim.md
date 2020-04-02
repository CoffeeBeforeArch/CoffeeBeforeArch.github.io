---
layout: default
title: Optimizing the Common Case with GPGPU-Sim
---

# Optimizing the Common Case with GPGPU-Sim

In a number of videos on my YouTube I have discussed performance tuning and benchmarking. However, the majority of those videos focus on microbenchmarks. In this post we'll be looking at how we can do some performance tuning on the GPGPU microarchitecture simulator GPGPU-Sim. Simulator performance is crucial, as it is the bottleneck for testing and verifying research ideas.

## Links

- [My YouTube Channel](https://www.youtube.com/channel/UCsi5-meDM5Q5NE93n_Ya7GA?view_as=subscriber)
- [My GitHub Account](https://github.com/CoffeeBeforeArch)
- [GPGPU-Sim Repository](https://github.com/purdue-aalp/gpgpu-sim_distribution/tree/dev)
- My Email: CoffeeBeforeArch@gmail.com

## Building GPGPU-Sim

For information on building GPGPU-Sim, check out the [first post](https://coffeebeforearch.github.io/2020/03/30/gpgpu-sim-1.html) from my series on working with GPGPU-Sim. This include guidance on debugging common problems, and how to run your first application on the simulator.

## Our Configuration

For this blog post we will be running a simple version of [CUDA matrix multiplication](https://github.com/CoffeeBeforeArch/cuda_programming/blob/master/matrixMul/baseline/mmul.cu) on GPGPU-Sim configured as an NVIDIA TITAN V GPU. This is the same setup as the blog post [here](https://coffeebeforearch.github.io/2020/03/30/gpgpu-sim-1.html).

## Finding Application Hot Spots

Optimizing for the common case is a core tenant of both high performance programming and computer architecture. But what exactly does that mean in the context of improving the performance of a simulator? For us, this means finding where the simulator is spending most of its time, and improving performance there first. This is where Linux perf tools can help.

Let's run our application with the simulator with a relatively small input (N = 128). Remember, using too large of matrix dimension will lead to a simulation that will not complete for hours. Here is a partial breakdown from where my run spent its time.

```bash
Samples: 289K of event 'cycles:ppp', Event count (approx.): 222166465721
Overhead  Command          Shared Object        Symbol
  16.13%  mmul             libcudart.so         [.] cache_stats::operator()
   7.78%  mmul             libcudart.so         [.] cache_stats::operator+=
   7.68%  mmul             libcudart.so         [.] tag_array::probe
   4.35%  mmul             libcudart.so         [.] sector_cache_block::is_reserved_line
   3.89%  mmul             libcudart.so         [.] ptx_thread_info::get_reg
   3.06%  mmul             libcudart.so         [.] Scoreboard::checkCollision
   2.36%  mmul             libcudart.so         [.] simt_stack::get_pdom_stack_top_info
   2.29%  mmul             libcudart.so         [.] pipelined_simd_unit::cycle
   1.96%  mmul             libcudart.so         [.] cache_stats::check_valid
   1.94%  mmul             libcudart.so         [.] scheduler_unit::sort_warps_by_oldest_dynamic_id
   1.93%  mmul             libcudart.so         [.] ptx_thread_info::ptx_exec_inst
   1.59%  mmul             libc-2.27.so         [.] cfree@GLIBC_2.2.5
   1.53%  mmul             libcudart.so         [.] sector_cache_block::is_invalid_line
   1.49%  mmul             libcudart.so         [.] cache_stats::operator()@plt
   1.46%  mmul             libcudart.so         [.] std::__detail::_Map_base<symbol const*, std::pair<symbol const* const, ptx_reg_t>, std::allocator<
   1.43%  mmul             libcudart.so         [.] shader_core_ctx::checkExecutionStatusAndUpdate
   1.43%  mmul             libcudart.so         [.] scheduler_unit::cycle
   1.36%  mmul             libcudart.so         [.] shader_core_ctx::execute
   1.22%  mmul             libc-2.27.so         [.] _int_malloc
   1.18%  mmul             libc-2.27.so         [.] malloc
   1.01%  mmul             libcudart.so         [.] cache_stats::check_valid@plt

```

Clearly there is something up in our implementation of `cache_stats::operator()`. Let's look back at the implementation in `gpu-cache.cc`.

```cpp
  if (fail_outcome) {
    if (!check_fail_valid(access_type, access_outcome))
      assert(0 && "Unknown cache access type or fail outcome");

    return m_fail_stats[access_type][access_outcome];
  } else {
    if (!check_valid(access_type, access_outcome))
      assert(0 && "Unknown cache access type or access outcome");

    return m_stats[access_type][access_outcome];
  }
```

Yikes - that's a lot of nested conditionals. Let's look at `check_valid(...)` and `check_fail_valid(...)` to see if things get better...

```cpp
bool cache_stats::check_valid(int type, int status) const {
  if ((type >= 0) && (type < NUM_MEM_ACCESS_TYPE) && (status >= 0) &&
      (status < NUM_CACHE_REQUEST_STATUS))
    return true;
  else
    return false;
}
```

```cpp
bool cache_stats::check_fail_valid(int type, int fail) const {
  if ((type >= 0) && (type < NUM_MEM_ACCESS_TYPE) && (fail >= 0) &&
      (fail < NUM_CACHE_RESERVATION_FAIL_STATUS))
    return true;
  else
    return false;
}
```

Things did not get better. But that's ok. The first step in writing code is often to get a functional, rather than performant solution. Now that we understand what is taking so much time, let's look at the disassembly to see if it's a specific part of the function that's slow.

```asm
cache_stats::operator()  /home/cba/forked_repos/gpgpu-sim_distribution/lib/gcc-6.5.0/cuda-9010/release/libcudart.so
Percent│                                                                                                                                             ◆
       │                                                                                                                                             ▒
       │                                                                                                                                             ▒
       │    Disassembly of section .text:                                                                                                            ▒
       │                                                                                                                                             ▒
       │    000000000010ff10 <cache_stats::operator()(int, int, bool) const>:                                                                        ▒
       │    _ZNK11cache_statsclEiib():                                                                                                               ▒
       │        return m_stats[access_type][access_outcome];                                                                                         ▒
       │      }                                                                                                                                      ▒
       │    }                                                                                                                                        ▒
       │                                                                                                                                             ▒
       │    unsigned long long cache_stats::operator()(int access_type, int access_outcome,                                                          ▒
       │                                               bool fail_outcome) const {                                                                    ▒
  0.01 │      push   %r12                                                                                                                            ▒
       │      ///                                                                                                                                    ▒
       │      /// Const accessor into m_stats.                                                                                                       ▒
       │      ///                                                                                                                                    ▒
       │      if (fail_outcome) {                                                                                                                    ▒
       │      test   %cl,%cl                                                                                                                         ▒
       │                                               bool fail_outcome) const {                                                                    ▒
  3.22 │      push   %rbp                                                                                                                            ▒
  6.03 │      movslq %edx,%rbp                                                                                                                       ▒
  0.02 │      push   %rbx                                                                                                                            ▒
  0.00 │      movslq %esi,%rbx                                                                                                                       ▒
  3.15 │      mov    %rdi,%r12                                                                                                                       ▒
       │        if (!check_fail_valid(access_type, access_outcome))                                                                                  ▒
  5.98 │      mov    %ebp,%edx                                                                                                                       ▒
  0.02 │      mov    %ebx,%esi                                                                                                                       ▒
       │      if (fail_outcome) {                                                                                                                    ▒
       │    ↓ je     10ff50 <cache_stats::operator()(int, int, 40                                                                                    ▒
       │        if (!check_fail_valid(access_type, access_outcome))                                                                                  ▒
  3.17 │    → callq  cache_stats::check_fail_valid(int, int) const@plt                                                                               ▒
       │      test   %al,%al                                                                                                                         ▒
       │    ↓ je     10ff90 <cache_stats::operator()(int, int, 80                                                                                    ▒
       │           *  out_of_range lookups are not defined. (For checked lookups                                                                     ▒
       │           *  see at().)                                                                                                                     ▒
       │           */                                                                                                                                ▒
       │          const_reference                                                                                                                    ▒
       │          operator[](size_type __n) const _GLIBCXX_NOEXCEPT                                                                                  ▒
       │          { return *(this->_M_impl._M_start + __n); }                                                                                        ▒
  0.15 │      mov    0x30(%r12),%rcx                                                                                                                 ▒
       │      lea    (%rbx,%rbx,2),%rax                                                                                                              ▒
       │        if (!check_valid(access_type, access_outcome))                                                                                       ▒
       │          assert(0 && "Unknown cache access type or access outcome");                                                                        ▒
       │                                                                                                                                             ▒
       │        return m_stats[access_type][access_outcome];                                                                                         ▒
       │      }                                                                                                                                      ▒
       │    }                                                                                                                                        ▒
  3.02 │      pop    %rbx                                                                                                                            ▒
  0.00 │      lea    (%rcx,%rax,8),%rax                                                                                                              ▒
       │        return m_fail_stats[access_type][access_outcome];                                                                                    ▒
 10.19 │      mov    (%rax),%rax                                                                                                                     ▒
  8.35 │      mov    (%rax,%rbp,8),%rax                                                                                                              ▒
       │    }                                                                                                                                        ▒
  1.53 │      pop    %rbp                                                                                                                            ▒
       │      pop    %r12                                                                                                                            ▒
       │    ← retq                                                                                                                                   ◆
       │      nop                                                                                                                                    ▒
       │        if (!check_valid(access_type, access_outcome))                                                                                       ▒
  5.94 │40: → callq  cache_stats::check_valid(int, int) const@plt                                                                                    ▒
  0.00 │      test   %al,%al                                                                                                                         ▒
       │    ↓ je     10ff71 <cache_stats::operator()(int, int, 61                                                                                    ▒
 10.30 │      mov    (%r12),%rdx                                                                                                                     ▒
  0.00 │      lea    (%rbx,%rbx,2),%rax                                                                                                              ▒
       │    }                                                                                                                                        ▒
  5.98 │      pop    %rbx                                                                                                                            ▒
  0.06 │      lea    (%rdx,%rax,8),%rax                                                                                                              ▒
       │        return m_stats[access_type][access_outcome];                                                                                         ▒
 15.42 │      mov    (%rax),%rax                                                                                                                     ▒
 12.94 │      mov    (%rax,%rbp,8),%rax                                                                                                              ▒
       │    }                                                                                                                                        ▒
  4.53 │      pop    %rbp                                                                                                                            ▒
       │      pop    %r12                                                                                                                            ▒
       │    ← retq                                                                                                                                   ▒
       │          assert(0 && "Unknown cache access type or access outcome");                                                                        ▒
       │61:   lea    cache_stats::operator()(int, int, bool) const::__PRETTY_FUNCTION__,%rcx                                                         ▒
       │      lea    frfcfs_scheduler::add_req(dram_req_t*)::__PRETTY_FUNCTION__+0x174,%rsi                                                          ▒
       │      lea    frfcfs_scheduler::add_req(dram_req_t*)::__PRETTY_FUNCTION__+0x978,%rdi                                                          ▒
       │      mov    $0x2cb,%edx                                                                                                                     ▒
       │    → callq  __assert_fail@plt                                                                                                               ▒
       │          assert(0 && "Unknown cache access type or fail outcome");                                                                          ▒
       │80:   lea    cache_stats::operator()(int, int, bool) const::__PRETTY_FUNCTION__,%rcx                                                         ▒
       │      lea    frfcfs_scheduler::add_req(dram_req_t*)::__PRETTY_FUNCTION__+0x174,%rsi                                                          ▒
       │      lea    frfcfs_scheduler::add_req(dram_req_t*)::__PRETTY_FUNCTION__+0x9e0,%rdi                                                          ▒
       │      mov    $0x2c6,%edx                                                                                                                     ▒
       │    → callq  __assert_fail@plt                                                                                                               ◆
```

What are the key things to take away from this disassembly? For one thing, most of the time is being spent on the return of a value from either `m_stats` or `m_fail_stats`. This indicates that we are probably missing in the cache on these accesses. Another key take away is that we seem to have a lot of small chunks of time being spent on calling `check_valid(...)` and `check_fail_valid(...)` functions and the related jumps.

If we wanted to tackle the cache misses, we'd have to dig deeper into things like reuse distance, locality, and where we might add some software prefetching. While this is a legitimate route to go, we might get an easy win by trying to simplify the control flow. I'm going to focus on simplifying the control flow because I see a glaring opportunity.

The glaring opportunity comes from the fact I see assert statements in the code. If the simulator crashes, that indicates a serious problem (either because the architecture is not being modeled properly, or a feature has not been implemented yet. However, the simulator does not crash for a large number of applications, especially those typically simulated in research works. This means that almost every time the simulator runs, it is wasting time performing these checks.

A natural question to ask is, how do other applications deal with this code? One common way is to just have different build modes. For example, an application may be set up to strip out all asserts when it is compiled in a release or performance mode, and leave them in when compiled in a debug mode. In this example, let's focus on what we can potentially gain by just removing the assertion checks. We can always come back later to implement a new build mode.

Let's modify the code for the `cache_stats::operator()` to the following.

```cpp
  if (fail_outcome) {
    return m_fail_stats[access_type][access_outcome];
  } else {
    return m_stats[access_type][access_outcome];
  }
```

All we have done is remove all the branches associated with the asserts (and the asserts themselves). Let's re-run our application and examine how the disassembly looks for our function that's the hotspot 

```asm
cache_stats::operator()  /home/cba/forked_repos/gpgpu-sim_distribution/lib/gcc-6.5.0/cuda-9010/release/libcudart.so
Percent│                                                                                                                                             ◆
       │                                                                                                                                             ▒
       │                                                                                                                                             ▒
       │    Disassembly of section .text:                                                                                                            ▒
       │                                                                                                                                             ▒
       │    000000000010f1f0 <cache_stats::operator()(int, int, bool) const>:                                                                        ▒
       │    _ZNK11cache_statsclEiib():                                                                                                               ▒
       │    unsigned long long cache_stats::operator()(int access_type, int access_outcome,                                                          ▒
       │                                               bool fail_outcome) const {                                                                    ▒
       │      ///                                                                                                                                    ▒
       │      /// Const accessor into m_stats.                                                                                                       ▒
       │      if (fail_outcome) {                                                                                                                    ▒
       │        return m_fail_stats[access_type][access_outcome];                                                                                    ▒
  0.00 │      movslq %esi,%rsi                                                                                                                       ▒
       │           *  out_of_range lookups are not defined. (For checked lookups                                                                     ▒
       │           *  see at().)                                                                                                                     ▒
       │           */                                                                                                                                ▒
       │          const_reference                                                                                                                    ▒
       │          operator[](size_type __n) const _GLIBCXX_NOEXCEPT                                                                                  ▒
       │          { return *(this->_M_impl._M_start + __n); }                                                                                        ▒
       │      lea    (%rsi,%rsi,2),%rax                                                                                                              ▒
  1.55 │      shl    $0x3,%rax                                                                                                                       ▒
       │      if (fail_outcome) {                                                                                                                    ▒
  7.67 │      test   %cl,%cl                                                                                                                         ▒
  4.52 │    ↓ jne    10f210 <cache_stats::operator()(int, int, 20                                                                                    ▒
 14.72 │      add    (%rdi),%rax                                                                                                                     ▒
       │      } else {                                                                                                                               ▒
       │        return m_stats[access_type][access_outcome];                                                                                         ▒
       │      movslq %edx,%rdx                                                                                                                       ▒
 22.60 │      mov    (%rax),%rax                                                                                                                     ▒
 23.23 │      mov    (%rax,%rdx,8),%rax                                                                                                              ▒
       │      }                                                                                                                                      ▒
       │    }                                                                                                                                        ▒
  0.01 │    ← retq                                                                                                                                   ▒
       │      nop                                                                                                                                    ▒
  0.25 │20:   add    0x30(%rdi),%rax                                                                                                                 ▒
       │        return m_fail_stats[access_type][access_outcome];                                                                                    ▒
       │      movslq %edx,%rdx                                                                                                                       ▒
 12.09 │      mov    (%rax),%rax                                                                                                                     ▒
 13.36 │      mov    (%rax,%rdx,8),%rax                                                                                                              ▒
  0.00 │    ← retq                                                                                                                                   ◆

```

The key things to take away from here is that we've removed a lot of the small overheads from extra branches and function calls, and primarily left with the overhead of caches misses. So, how well did we do? Let's just use the simulation time reported by the simulator for a rough estimate. On my machine, and for this relatively small input, my execution time went from 71 seconds to 61 seconds (a ~14% reduction in execution time). End-to-end execution time for the suite of applications used to qualify pull requests to the GPGPU-Sim repository decreased from 1h 34m to 1h 9m (with some minor variation from build to build).


## Concluding Remarks

Performance tuning is feedback based. You measure, analyze, tune, and repeat. In later posts we'll discuss different performance tuning opportunities for code bases like GPGPU-Sim, and show off to use some more exciting tools. Until then, we'll settle for our ~14% increase in perf. Not bad for removing 4 lines of code.

As always, feel free to contact me with questions.

Cheers,

--Nick

## Discussion Points

- Different applications may stress different parts of the simulator code. This would be an interesting thing to study in greater detail.

- The impact on these performance changes will likely change from system to system. Understanding why is can be an interesting but challenging exercise.

- Performance regressions are your friend! Tracking how code updates impact runtimes can help your code base from running at the speed of a turtle. 

## Links

- [My YouTube Channel](https://www.youtube.com/channel/UCsi5-meDM5Q5NE93n_Ya7GA?view_as=subscriber)
- [My GitHub Account](https://github.com/CoffeeBeforeArch)
- [GPGPU-Sim Repository](https://github.com/purdue-aalp/gpgpu-sim_distribution/tree/dev)
- My Email: CoffeeBeforeArch@gmail.com
