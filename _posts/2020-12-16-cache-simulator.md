---
layout: default
title: Writing a Trace-Based Cache Simulator
---

# Writing a Trace - Based Cache Simulator

Computer architects use many tools to evaluate proposed architectures. They may use coarse-grained analytical models to quickly rule out sub-optimal designs, or complex RTL simulation to get an accurate view of how the real hardware will behave. Another common tool that provides a compromise on speed in accuarcy between these is micro-architecture simulators.

Microarchitecture simulators can provide researchers with a decently accurate view of how hardware will behave in a reasonable amount of time, and are used extensively in both research and industry. This makes them a great thing to get familiar with if you are interested in hardware and architecture.

In this blog post we'll be looking at how to write a trace-based cache simulator. This should provide a (relatively) simple entry points for those interested in simulator design. We will then close with a discussion on simulator performance.

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

In the following sections, we will how to implement this functionality in greater detail as we implement methods and add data members to our `CacheSim` class:

```cpp
// Our cache simulator
class CacheSim {
 public:
 private:
};
```

## Reading and Parsing Memory Accesses

The first step in building a cache simulator is building the infrastructure to read/parse the parse the instructions. We'll start by looking at the format of the instruction trace we'll be processing, then look at the code we'll use to parse the instructions.

### Format of the Instruction Trace

The instruction traces my simulator is based on are from [this course assignment](https://occs.oberlin.edu/~ctaylor/classes/210SP13/cache.html). Each line of the trace files looks like the following:

```txt
# 0 7fffed80 1
```

Following the `#` we have the type of memory access (`0` for read, and `1` for write), the address of the piece of memory being accessed, and then the number of instructions executed between the previous memory access and this one.

### Reading and Parsing the Trace Files

Before we can extract the access type, address, and instruction count, we need to first read the line from the file. We'll implement the opening and closing of our trace file in the constructor and destructor of our `CacheSim` class. This follows the RAII (resource acquisition is initialization) programming technique which binds the life cycle of a resource (our trace file) to the lifetime of an object (our `CacheSim` object).

The actual resource our constructor and destructor will open and close is a `std::ifstream` object. We will add this as a data member of our class and name it `infile`:

```cpp
// Input trace file
std::ifstream infile;
```

This is what we will use to open/close our trace file, and read the trace file line-by-line.

We will additionally add a constructor to our `CacheSim` class that takes a `std::string` that contains the location of the trace file to open. We will additionally open the file in the constructor:

```cpp
// Constructor
CacheSim(std::string input) {
  // Initialize of input file stream object
  infile.open(input);
}
```

We will also add a destructor to our classes that closes the file:

```cpp
// Destructor
~CacheSim() {
  infile.close();
}
```

## Processing Cache Accesses

## Recording Statistics

## Final Thoughts

Thanks for reading,

--Nick

### Link to the source code

- [My YouTube Channel: ](https://www.youtube.com/channel/UCsi5-meDM5Q5NE93n_Ya7GA?view_as=subscriber)
- [My GitHub Account: ](https://github.com/CoffeeBeforeArch)
- My Email: CoffeeBeforeArch@gmail.com

