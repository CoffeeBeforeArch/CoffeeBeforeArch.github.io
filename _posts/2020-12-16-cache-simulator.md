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

### Opening and Closing the Trace File

We will be using a `std::ifstream` object for reading our trace file. We can add this to our private data members and name it `infile`:

```cpp
// Input trace file
std::ifstream infile;
```

The first thing we need to set up for our `std::ifstream` object is opening and closing the file. We'll follow the programming technique RAII (resource acquisition is initialization) for this, which binds the life cycle of a resource (the file we'll be opening) to the lifetime of the object (our `CacheSim` object).

Specifically, we will be opening the file in the constructor of our `CacheSim` object, and closing it in the destructor. We'll start with our constructor which will take a single `std::string` (the path to the trace file to open). Inside the constructor we will call the `open` method for our `ifstream` object:

```cpp
// Constructor
CacheSim(std::string input) {
  // Initialize of input file stream object
  infile.open(input);
}
```

Next we will add the destructor which closes the file using the `close` method:

```cpp
// Destructor
~CacheSim() {
  infile.close();
}
```

### Reading and Parsing Trace File Lines

Now that we've set up the opening and closing of our trace file, we can focus on reading the instructions line-by-line and extracting the access type, address, and instruction count.

For this, we'll be implementing two methods:

1. `run` - A method that will keep reading until we run out of lines in the trace
2. `parse_line` - A method for extracting the access type, address, and instruction count from a line from the trace file

We can implement the `run` method as follows:

```cpp
// Run the simulation
void run() {
  // Keep reading data from a file
  std::string line;
  while (std::getline(infile, line)) {
    // We will parse the line from the file here
  }
}
```

Each iteration of the `while` loop, we read a new line from the trace file into our `std::string` named `line` using `std::getline`. When we hit the end of the file, we break out of the loop.

The next thing we will add is a helper method to extract the data stored in `line`. We'll implement that in our `parse_line` method:

```cpp
// Get memory access from the trace file
std::tuple<bool, std::uint64_t, int> parse_line(std::string access) {
  // What we want to parse
  int type;
  std::uint64_t address;
  int instructions;

  // Parse from the string we read from the file
  sscanf(access.c_str(), "# %d %lx %d", &type, &address, &instructions);

  return {type, address, instructions};
}
```

To our method we'll pass the `std::string` containing our memory access. We'll then create 3 temporary variables (`type`, `address`, and `instructions`) for the three values we want to extract from our string.

Next, we'll use `sscanf` to extract the three values into our temporary variables.

Finally, we'll return all three variables using a `std::tuple`.

All we need to do now is add the call to `parse_line` to our `run` method:

```cpp
// Run the simulation
void run() {
  // Keep reading data from a file
  std::string line;
  while (std::getline(infile, line)) {
    // Get the data for the access
    auto [type, address, instructions] = parse_line(line);
  }
}
```

We can even use structured bindings (C++17) to unpack our the three values in our `std::tuple` directly (as seen above) instead of using something like `std::tie` or `std::get`. 

### End of Section Thoughts

Great! We know have a nice setup for opening/closing our trace file, and reading/parsing the data from each line.

This is a good place to stop and verify that everything works. One good way to verify that everything is working correctly is to print out the values returned from `parse_line`, redirect the output to a file, and then `diff` that file with the original trace (they should perfectly match if everything is working correctly).

## Processing Cache Accesses

## Recording Statistics

## Final Thoughts

Thanks for reading,

--Nick

### Link to the source code

- [My YouTube Channel: ](https://www.youtube.com/channel/UCsi5-meDM5Q5NE93n_Ya7GA?view_as=subscriber)
- [My GitHub Account: ](https://github.com/CoffeeBeforeArch)
- My Email: CoffeeBeforeArch@gmail.com

