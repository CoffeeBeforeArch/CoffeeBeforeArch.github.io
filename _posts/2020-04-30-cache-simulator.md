---
layout: default
title: Cache Simulator
---

# Cache Simulator

A common project for undergraduate and graduate computer engineering students is writing a trace-based cache simulator. To help out those with similar assignments or those just interested in architecture, I decided to write one. This blog posts covers the overall architecture of the simulator that I wrote, why I made certain design decisions, and some discussion on interesting avenues for future development/improvement of the project. 

## Links

- [My YouTube Channel](https://www.youtube.com/channel/UCsi5-meDM5Q5NE93n_Ya7GA?view_as=subscriber)
- [My GitHub Account](https://github.com/CoffeeBeforeArch)
- [Cache Simulator Repository](https://github.com/CoffeeBeforeArch/cache_simulator)
- My Email: CoffeeBeforeArch@gmail.com

## Compiling and Running the Simulator

Compiling the simulator a compiler that supports some C++20 (like `std::popcount`). Currently, I am developing the project with `g++ 10.0.1`.

While I haven't written a makefile for the project yet, the below commands are how you can build the simulator. 

First you will need to compile the Google protobuf.

```bash
protoc --cpp_out=./ cacheConfig.proto
```

Then you will have to compile the rest of the simulator.

```bash
g++ driver.cpp simulation.cpp access.cpp cacheConfig.pb.cc cache_set.cpp data_cache.cpp statistics.cpp -O3 -flto -march=native -mtune=native --std=c++2a -lprotobuf -o driver
```
To comment on some of the compilation flags, `-03` is to enable almost all optimizations, `-flto` enables link-time optimizaiton, `-march/-mtune=native` is for tuning the code for the current architecture, and `--std=c++2a` is to enable C++20 features.

## Simulator Settings

### Next Steps

## Driving the Simulator

### Next Steps

## Reading Instruction Traces

## Next Steps

## The Data Cache

### Cache Sets

### Replacement Policy

### Statistics

### Next Steps

## Concluding Remarks

As always, feel free to contact me with questions.

Cheers,

--Nick

## Links

- [My YouTube Channel](https://www.youtube.com/channel/UCsi5-meDM5Q5NE93n_Ya7GA?view_as=subscriber)
- [My GitHub Account](https://github.com/CoffeeBeforeArch)
- [Cache Simulator Repository](https://github.com/CoffeeBeforeArch/cache_simulator)
- My Email: CoffeeBeforeArch@gmail.com
