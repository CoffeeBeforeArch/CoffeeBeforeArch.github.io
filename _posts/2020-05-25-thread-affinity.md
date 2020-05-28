---
layout: default
title: Thread Affinity
---

# Thread Affinity

Multi-threading can enable substantial speedups to our applications, but only when it used effectively. Blindly adding threads to a serial application may lead to little or no improvement in runtime. In the worst case, our parallel implementation may even run slower! When using threads, some obvious things to consider are how we partition our data, and how we divide work betweent threads. Another critical thing to consider is _where_ our threads get scheduled. If we place our threads intelligently, we can take advantage of inter-thread locality, and reduce the chance of expensive cache misses that must fetch data from another core, or another NUMA (non-uniform memory architecture) node. In this blog post, we'll be looking at how can pin threads to specific cores, and how it can improve performance.

### Link to the source code

## Concluding remarks

Thanks for reading,

--Nick

### Link to the source code
