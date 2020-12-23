---
layout: default
title: Writing a Trace-Based Cache Simulator
---

# Writing a Trace-Based Cache Simulator

Computer architects use many tools to evaluate proposed architectures. They may use coarse-grained analytical models to quickly rule out sub-optimal designs, or complex RTL simulation to get an accurate view of how the real hardware will behave. Another common tool that provides a compromise on speed in accuarcy between these is micro-architecture simulators.

Microarchitecture simulators can provide researchers with a decently accurate view of how hardware will behave in a reasonable amount of time, and are used extensively in both research and industry. This makes them a great thing to get familiar with if you are interested in hardware and architecture.

In this blog post we'll be looking at how to write a trace-based cache simulator. This should provide a (relatively) simple entry points for those interested in simulator design.

### Link to the source code

- [My YouTube Channel: ](https://www.youtube.com/channel/UCsi5-meDM5Q5NE93n_Ya7GA?view_as=subscriber)
- [My GitHub Account: ](https://github.com/CoffeeBeforeArch)
- My Email: CoffeeBeforeArch@gmail.com

## Trace-Based Simulator Basics

Before we dig into the code, we should discuss what we are actually trying to design, starting with what "trace-based" simulation means.

Simulators often come in two flavors: trace-based and execution-driven. Execution-driven simulators drive simulation by directly executing a program's instructions. Conversely, trace-driven simulators drive simulation by reading/parsing a pre-recorded trace of instructions. For our simple cache simulator, we will be using a trace-based design.

Our simulator must fundamentally do three things:

1. Read+Parse instructions from a trace of memory accesses
2. Process the cache accesses
3. Record statistics about what happened

## Final Thoughts

Thanks for reading,

--Nick

### Link to the source code

- [My YouTube Channel: ](https://www.youtube.com/channel/UCsi5-meDM5Q5NE93n_Ya7GA?view_as=subscriber)
- [My GitHub Account: ](https://github.com/CoffeeBeforeArch)
- My Email: CoffeeBeforeArch@gmail.com

