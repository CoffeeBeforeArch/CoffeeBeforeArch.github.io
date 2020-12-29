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

In this next section, we'll focus on how to model a cache and process the accesses we read and process from our trace file.

### Cache State

Before we can implement the funcionality of our cache model, we first need to add the appropriate state (data members) to our `CacheSim` class.

The first thing we'll add are the configuration parameters. These describe the shape of our cache (block size, associativity, and capactiy), and the overheads (miss penalty and dirty writeback penalty) for each access.

```cpp
// Cache settings
unsigned block_size;
unsigned associativity;
unsigned capacity;
unsigned miss_penalty;
unsigned dirty_wb_penalty;
```

The next thing we'll add to our cache are some offsets and masks that we'll use when extracting the set number and tag from a memory address:

```cpp
// Access settings
unsigned set_offset;
unsigned tag_offset;
unsigned set_mask;
```

The final thing we'll add for functionality is actual state of our cache:

```cpp
// The actual cache state
std::vector<std::uint64_t> tags;
std::vector<char> dirty;
std::vector<char> valid;
std::vector<int> priority;
```

For each cache block, we'll store the tag at the location, if the cache block is dirty, if the cache block is valid, and the priority of the cache block w.r.t. the LRU replacement policy.

### Setting Up the Cache

Now that we have the data members in our `CacheSim` class, we need to initialize them. This is something we can do in our class' constructor.

We'll start by expanding our constructor to also take the configuration options (block size, associativity, capacity, miss-penalty, and dirty writeback penalty) as parameters.

```cpp
// Constructor
CacheSim(std::string input, unsigned bs, unsigned a, unsigned c, unsigned mp,
         unsigned wbp) {
  // Initialize of input file stream object
  infile.open(input);

  // Set all of our cache settings
  block_size = bs;
  associativity = a;
  capacity = c;
  miss_penalty = mp;
  dirty_wb_penalty = wbp;
```

The next thing we need to do is resize our vectors based on the number of cache blocks for which we want to maintain state. This will simply be the capacity of our cache divided by the cache block size:

```cpp
// Set all of our cache settings
block_size = bs;
associativity = a;
capacity = c;
miss_penalty = mp;
dirty_wb_penalty = wbp;

// Calculate the number of blocks
// Assume this divides evenly
auto num_blocks = capacity / block_size;

// Create our cache based on the number of blocks
tags.resize(num_blocks);
dirty.resize(num_blocks);
valid.resize(num_blocks);
priority.resize(num_blocks);
}
```

Note, we don't actually need to worry about the shape of our cache with sizing the vectors. This is because the shape of our cache will only affect the traversal of the vectors, not the number of total elements (which will always be equal to the number of cache blocks).

The final thing we need to initialize are the offsets and mask to extract the set number and tag from a memory address. We'll be using the following memory address format as a basis for these offsets and masks.

```txt
       X-Bits         Y-Bits        Z-Bits
|****** TAG ******|**** SET ****|** OFFSET **|
```

The first thing we'll calculate is the `set_offset`, which is the number of bits we need to shift over over to extract the set number (Z-Bits from our diagram). Assuming byte-addressable memory and a cache block size of 16-bytes, the number of bits (Z-Bits) would be 4. This is because 16 bytes (for a cache line) can be indexed using only 4 bits (using values `0b0000` to `0b1111`).

The fundamental calculation we need is log base 2 of the cache block size (e.g. log base 2 of 16 is 4). A simple way to calculate this is by subtracting 1 from the block size then using `std::popcount` to count the number of set bits. Consider the following example where the `block_size` is 16 bytes:

```txt
block_size           = 16 = 0b00010000
block_size - 1       = 15 = 0b00001111
popcount(0b00001111) = 4
```

In C++, we can do this as follows:

```cpp
auto set_offset = std::popcount(block_size - 1);
```

This calculation is reliant on the `block_size` being a power-of-2. For all practical intents and purposes, this will always be the case.

We next thing we need to do calculate is our `set_mask`. This will be a mask of 1s equal to the number of set bits (Y-Bits from our diagram) in our memory address. We can calculate the number of sets with the following:

```cpp
auto sets = capacity / (block_size * associativity);
```

The size of each set is the equal to the associativity multiplied by the block size. We can therefore figure out the total number of sets buy dividing the total capacity of the cache by the size of each set.

To create a mask of ones equal to the number of set bits (Y-bits from our diagram), we can simply subtract 1 from the the number of sets. Consider the following example where the number sets is 16:

```txt
sets           = 16 = 0b00010000
sets - 1       = 15 = 0b00001111
```

Because the number of sets will be a power-of-2 for all practical intents and purposes, subtracting one from than number will give us a mask of ones equal to the number of set bits (for the same reasoning as with the `set_offset` calculation).

```cpp
set_mask = sets - 1;
```

The final thing we need to calculate is the `tag_offset` (the number of bits to shift the address over to extract the tag). This will simply be the `set_offset` plus the number number of set bits. We can calculate this as follows:

```cpp
auto set_bits = std::popcount(set_mask);
tag_offset = set_offset + set_bits;
```

We do not need a mask to extract the tag bits because all the upper-bits of the address shifted by `tag_offset` will be zeros.

Here is the full implementation of our constructor:

```cpp
// Constructor
CacheSim(std::string input, unsigned bs, unsigned a, unsigned c, unsigned mp,
         unsigned wbp) {
  // Initialize of input file stream object
  infile.open(input);

  // Set all of our cache settings
  block_size = bs;
  associativity = a;
  capacity = c;
  miss_penalty = mp;
  dirty_wb_penalty = wbp;

  // Calculate the number of blocks
  // Assume this divides evenly
  auto num_blocks = capacity / block_size;

  // Create our cache based on the number of blocks
  tags.resize(num_blocks);
  dirty.resize(num_blocks);
  valid.resize(num_blocks);
  priority.resize(num_blocks);

  // Calculate values for traversal
  // Cache lines come in the following format:
  // |****** TAG ******|**** SET ****|** OFFSET **|
  // Calculate the number of offset bits
  auto block_bits = std::popcount(block_size - 1);

  // Calculate the number of set bits, and create a mask of 1s
  set_offset = block_bits;
  auto sets = capacity / (block_size * associativity);
  set_mask = sets - 1;
  auto set_bits = std::popcount(set_mask);

  // Calculate the bit-offset for the tag and create a mask of 1s
  // Always use 64-bit addresses
  tag_offset = block_bits + set_bits;
}
```

### Accessing the Cache

Now that we have initialized our cache state, we can move into the logic for processing cache accesses. For this, we will implement the following methods:

1. `probe` - A method that performs the cache access
2. `get_set` - A helper method to get the set number from an address
3. `get_tag` - A helper method to get the tag from an address

Additionally, we will need to update our `run` method with our call to `probe`.

#### Helper Method Implementations

Our helper method `get_set` uses our `set_offset` and `set_mask` initialized in the constructor:

```cpp
// Extract the set number
// Shift the set to the bottom then extract the set bits
int get_set(std::uint64_t address) {
  auto shifted_address = address >> set_offset;
  return shifted_address & set_mask;
}
```

To get the set number, we simply shift the input memory address down to get rid of the offset bits, then extract just the set bits from the address (ignoring the tag).

Our `get_tag` method is similar except that we do not need to use a mask to extract the tag bits (because the upper bits will all be 0 in our shifted address).

```cpp
// Extract the tag
// Shift the tag to the bottom
// No need to use mask (tag is all upper remaining bits)
std::uint64_t get_tag(std::uint64_t address) {
  return address >> tag_offset;
}
```

#### Probe Implementation

Our `probe` method will simulate the actual cache access. To our method we will pass the type of access (read/write), and the address of the access. We will return if the access was a hit or a miss, and if the access caused the writeback of a dirty cache line.

```cpp
// Probe the cache
std::tuple<bool, bool> probe(bool type, std::uint64_t address) { }
```

The first thing we will add is our calls to the helper methods to extract the set number and tag:

```cpp
// Calculate the set from the address
auto set = get_set(address);
auto tag = get_tag(address);
```

Using the cache set number, we'll create a `std::span` (C++20) for each of the vectors containing our cache state:

```cpp
// create a spans for our set
auto base = set * associativity;
std::span local_tags{tags.data() + base, associativity};
std::span local_dirty{dirty.data() + base, associativity};
std::span local_valid{valid.data() + base, associativity};
std::span local_priority{priority.data() + base, associativity};
```

`std::span` provides a nice way to only view a subset of a container (our vectors) at once. In our case, we can view just the data for one set from each of our vectors.

Next, we start implementing the actual cache line functionality. To start we implement the logic for checking if the access is a hit or a miss:

```cpp
  // Check each cache line in the set
  auto hit = false;
  int invalid_index = -1;
  int index;
  for (auto i = 0u; i < local_valid.size(); i++) {
    // Check if the block is invalid
    if (!local_valid[i]) {
      // Keep track of invalid entries in case we need them
      invalid_index = i;
      continue;
    }

    // Check if the tag matches
    if (tag != local_tags[i]) continue;

    // We found the line, so mark it as a hit
    hit = true;
    index = i;

    // Update dirty flag
    local_dirty[index] |= type;

    // Break out of the loop
    break;
  }
```

Our `for` loop goes over each line in the set of our cache, first checking if the current cache line is invalid. If the line is invalid, we keep track of the index (for use in replacement), and `continue` to check the next cache block (or exit if there are no more blocks to check).

If the cache block is valid, we check if the tag of the access matches the block in the cache. If it does not match, we `continue` to check the next cache block (or exit if there are no more blocks to check).

If the cache block is not invalid, and the tag of the access and block match, we have a cache hit! We therefore mark the access as a hit, save the index (for use in the replacement policy), mark the dirty bit for the block if it is a write, and break out of the loop.

If the access was not a hit, we must find a cache line to replace:

```cpp
// Find an element to replace if it wasn't a hit
auto dirty_wb = false;
if (!hit) {
  // First try and use an invalid line (if available)
  if (invalid_index >= 0) {
    index = invalid_index;
    local_valid[index] = 1;
  }
  // Otherwise, evict the lowest-priority cache block (largest value)
  else {
    auto max_element = std::ranges::max_element(local_priority);
    index = std::distance(begin(local_priority), max_element);
    dirty_wb = local_dirty[index];
  }

  // Update the tag and dirty state
  local_tags[index] = tag;
  local_dirty[index] = type;
}
```

If we found an invalid cache line when initially searching the set, we will that for replacement. Otherwise, we will replace the element with the lowest priority (where the highest priority value is 0, and the lowest is `associativity - 1`). Additionally, we will set the tag and dirty bit if the access was a write.

We next need to update the priority for our cache replacement policy. For this, we will use `std::transform`:

```cpp
// Update the priority
// Go through each element
// Increase the priority of all the blocks with a lower priority than the
// one we are accessing
// High priority -> Low priority = 0 -> associativity - 1
std::transform(begin(local_priority), end(local_priority),
               begin(local_priority), [&](int p) {
                 if (p <= local_priority[index] && p < associativity)
                   return p + 1;
                 else
                   return p;
               });

// Currently accessed block has the highest priority (0)
local_priority[index] = 0;
```

Here, we decrease the priority of all elements in the span that have a higher original priority than the cache block being accessed. Then, we set the accessed block to have the highest priority (`0`).

Finally, we return if the access was a hit/miss, and if the access caused a dirty writeback:

```cpp
return {hit, dirty_wb};
```

#### Modification to the Run Method

The final step to functionally model our cache is to add a call to `probe` to our `run` method:

```cpp
  // Run the simulation
  void run() {
    // Keep reading data from a file
    std::string line;
    while (std::getline(infile, line)) {
      // Get the data for the access
      auto [type, address, instructions] = parse_line(line);

      // Probe the cache
      auto [hit, dirty_wb] = probe(type, address);
    }
  }
```

Just like we did with `parse_line`, we use structured bindings (C++17) to unpack the two values returned from the `probe` method.

### End of Section Thoughts

We implemented many things in this section! However, it's a good idea to test each of these methods as you go.

For example, I would first verify that your `get_set` and `get_tag` methods are properly extracting the fields from the memory addresses.

Next, I would create a sample trace with just a few accesses (these could be copied from one of the larger trace files) that can be easily manually verified. For example, you could initially check that two accesses to the same cache line result in one hit and one miss.

## Recording Statistics

The final step thing we need for our simulator is statistic collection. We'll implement this using a simple `update_stats` method, and small modification to our `run` method. We will also add a `dump_stats` method for printing the cache statistics (and place a call to this method in the destructor).

### State for our Statistics

To keep track our our statistics, we'll add a few more data members to our class:

```cpp
// Cache statistics
std::int64_t writes_ = 0;
std::int64_t mem_accesses_ = 0;
std::int64_t misses_ = 0;
std::int64_t dirty_wb_ = 0;
std::int64_t instructions_ = 0;
```

Initially, all our statistics will be `0`.

A good thing to keep in mind is that we don't need to have a unique counter for every statistic. For example, the number of reads can be calculated by subtracting `writes_` from the total number memory accesses (`mem_accesses_`).

### The Update Stats Method

Our `update_stats` method is incredibly simple:

```cpp
// Update the stats
void update_stats(int instructions, bool type, bool hit, bool dirty_wb) {
  mem_accesses_++;
  writes_ += type;
  misses_ += !hit;
  instructions_ += instructions;
  dirty_wb_ += dirty_wb;
}
```

As parameters, our method takes the number of instructions executed, the type of memory access (read/write), if the access was a hit/miss, and if the access resulted in a dirty writeback.

We then update the internal stats accordingly.

### The Dump Stats Method

The final method we'll implement is `dump_stats`. This method will simply calculate any statistics that need to be derived, and print them to the screen:

```cpp
// Dump the statistics from simulation
void dump_stats() {
  // Print the cache settings
  std::cout << "CACHE SETTINGS\n";
  std::cout << "       Cache Size (Bytes): " << capacity << '\n';
  std::cout << "           Associativity : " << associativity << '\n';
  std::cout << "       Block Size (Bytes): " << block_size << '\n';
  std::cout << "    Miss Penalty (Cycles): " << miss_penalty << '\n';
  std::cout << "Dirty WB Penalty (Cycles): " << dirty_wb_penalty << '\n';
  std::cout << '\n';

  // Print the access breakdown
  std::cout << "CACHE ACCESS STATS\n";
  std::cout << "TOTAL ACCESSES: " << mem_accesses_ << '\n';
  std::cout << "         READS: " << mem_accesses_ - writes_ << '\n';
  std::cout << "        WRITES: " << writes_ << '\n';
  std::cout << '\n';

  // Print the miss-rate breakdown
  std::cout << "CACHE MISS-RATE STATS\n";
  double miss_rate = (double)misses_ / (double)mem_accesses_ * 100.0;
  auto hits = mem_accesses_ - misses_;
  std::cout << "     MISS-RATE: " << miss_rate << '\n';
  std::cout << "        MISSES: " << misses_ << '\n';
  std::cout << "          HITS: " << hits << '\n';
  std::cout << '\n';

  // Print the instruction breakdown
  std::cout << "CACHE IPC STATS\n";
  auto cycles = miss_penalty * misses_;
  cycles += dirty_wb_penalty * dirty_wb_;
  cycles += instructions_;
  double ipc = (double)instructions_ / (double)cycles;
  std::cout << "           IPC: " << ipc << '\n';
  std::cout << "  INSTRUCTIONS: " << instructions_ << '\n';
  std::cout << "        CYCLES: " << cycles << '\n';
  std::cout << "      DIRTY WB: " << dirty_wb_ << '\n';
}
```

We will also add a call to this method in the destructor:

```cpp
// Destructor
~CacheSim() {
  infile.close();
  dump_stats();
}
```

### Modification to the Run Method

We can complete our `run` method by adding a call to `update_stats`:

```cpp
// Run the simulation
void run() {
  // Keep reading data from a file
  std::string line;
  while (std::getline(infile, line)) {
    // Get the data for the access
    auto [type, address, instructions] = parse_line(line);

    // Probe the cache
    auto [hit, dirty_wb] = probe(type, address);

    // Update the cache statistics
    update_stats(instructions, type, hit, dirty_wb);
  }
}
```

### End of Section Thoughts

We've finished implementing our cache simulator! However, it's a good idea to implement these statistics as you go (and not wait until the end).

For example, I'd first add statistics for reads/writes and instruction count before implementing the logic for hits/misses (since they are independent of each other).

## Running the Simulator

If all has gone according to plan, you should be able to run your simulator! Here is the `main` function I used to create and start my simulator:

```cpp
int main(int argc, char *argv[]) {
  // Kill the program if we didn't get an input file
  assert(argc == 2);

  // File location
  std::string location(argv[1]);

  // Hard coded cache settings
  unsigned block_size = 1 << 4;
  unsigned associativity = 1 << 0;
  unsigned capacity = 1 << 14;
  unsigned miss_penalty = 30;
  unsigned dirty_wb_penalty = 2;

  // Create our simulator
  CacheSim simulator(location, block_size, associativity, capacity,
                     miss_penalty, dirty_wb_penalty);
  simulator.run();

  return 0;
}
```

While I simply hard-coded the cache configuration parameters in the `main` function, you could optionally add the as command line arguments.

## Final Thoughts

We have made it to the end of our journey! You can test your simulator using the traces found at [the original course website](https://occs.oberlin.edu/~ctaylor/classes/210SP13/cache.html), or write your own!

As with all the code in these blog posts, there are many things we could look at improving. For example, the cache simulator we created is actually pretty slow! There are numerous optimizations we could perform, especially concerning the how we read accesses from the trace file. However, we'll leave that discussion to another blog post.

Thanks for reading,

--Nick

### Link to the source code

- [My YouTube Channel: ](https://www.youtube.com/channel/UCsi5-meDM5Q5NE93n_Ya7GA?view_as=subscriber)
- [My GitHub Account: ](https://github.com/CoffeeBeforeArch)
- My Email: CoffeeBeforeArch@gmail.com

