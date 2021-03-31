---
layout: default
title: Duplicate Removal
---

# Duplicate Removal

One of the most imporant parts of planning for performance is choosing the right algorithms. Importantly, this extend to all parts of your code (not just just things like sorting!). If we choose the wrong algorithm, we may still be able to optimize it. However, the correct algorithm generally allows us to scale performance much further.

In this blog post, we'll be taking a look at some different ways we can filter duplicate values from a `std::vector<int>`. We will study the performance of these different methods, and examine them under different circumstances (changing the size of the vectors, and distribution of random numbers in the vectors).

### Link to the source code

- [My YouTube Channel](https://www.youtube.com/channel/UCsi5-meDM5Q5NE93n_Ya7GA?view_as=subscriber)
- [My GitHub Account](https://github.com/CoffeeBeforeArch)
- [Duplicate Removal Benchmark](https://github.com/CoffeeBeforeArch/misc_code/tree/master/duplicate_removal)
- My Email: CoffeeBeforeArch@gmail.com

## Baseline Implementation

The goal of our duplicate removal method will be to take in a vector of random number (e.g. `1, 2, 3, 1, 2, 3`), and return a vector without any duplicates (e.g., `1, 2, 3`). One of the most simple ways we can solve this is the brute-force approach.

For this method, we'll iterate for each element in the input vector, and check if it is in the output vector. If it is not in the output vector, we will add it, otherwise, we'll continue to the next element in the input vector.

We can implement this using as follows:

```cpp
// For every value in the input vector
for (auto i : v_in) {
  // Check if it is not in the output vector
  if (std::find(begin(v_out), end(v_out), i) == v_out.end())
    // And put it in if it's not
    v_out.push_back(i);
}
```

For every element in `v_in`, we use `std::find` to check the element is in the output vector `v_out`. If it is not in the vector (`std::find` returned `v_out.end()`) we add the element to `v_out`.

The amount of work done each iteration of the outer `for` loop monotonically increases over the loop iterations. This is because the number of potential elements in `v_out` we have to scan monotonically increases over the loop iterations as we find non-duplicate elements.

### Benchmark Setup

We can use Google benchmark to measure the performance of our baseline implementation. Here is the basic structure:

```cpp
// Baseline benchmark used by sorting vectors
static void baseline(benchmark::State &s) {
  // Create input and output vectors
  int N = 1 << s.range(0);
  std::vector<int> v_in(N);
  std::vector<int> v_out;

  // Create our random number generators
  std::mt19937 rng;
  rng.seed(std::random_device()());
  std::uniform_int_distribution<int> dist(0, s.range(1));

  // Fill the input vector with random numbers
  std::generate(begin(v_in), end(v_in), [&] { return dist(rng); });

  // Benchmark loop
  for (auto _ : s) {
    // For every value in the input vector
    for (auto i : v_in) {
      // Check if it is not in the output vector
      if (std::find(begin(v_out), end(v_out), i) == v_out.end())
        // And put it in if it's not
        v_out.push_back(i);
    }

    // Clear each iteration
    v_out.clear();
  }
}
BENCHMARK(baseline)->Apply(custom_args)->Unit(benchmark::kMicrosecond);
```

We first create our vectors `v_in` and `v_out`, and fill `v_in` with random numbers. We then fall into our main benchmarking loop (`for (auto _ : s) {  }`) where we have our filter implementation.

We will be studying vectors with 2^10, 2^11, and 2^12 elements. Additionally, we will be looking at random numbers between 0 and 10, 100, 1000, and 10000. These are configured with our `custom_args` function:

```cpp
// Function for generating argument pairs
static void custom_args(benchmark::internal::Benchmark *b) {
  for (auto i : {10, 11, 12}) {
    for (auto j : {10, 100, 1000, 10000}) {
      b = b->ArgPair(i, j);
    }
  }
}
```

### Performance and Analysis

Here are the execution times measured on my machine:

```txt
------------------------------------------------------------
Benchmark                  Time             CPU   Iterations
------------------------------------------------------------
baseline/10/10          2.80 us         2.80 us       258206
baseline/10/100         14.2 us         14.2 us        49771
baseline/10/1000        60.3 us         60.3 us        11378
baseline/10/10000       94.9 us         94.9 us         7526
baseline/11/10          7.68 us         7.68 us        85437
baseline/11/100         33.8 us         33.8 us        20354
baseline/11/1000         153 us          153 us         4609
baseline/11/10000        331 us          331 us         2122
baseline/12/10          30.4 us         30.4 us        22736
baseline/12/100         74.1 us         74.1 us         9468
baseline/12/1000         340 us          340 us         2058
baseline/12/10000       1127 us         1127 us          617
```

There are two noticable trends from this data:

1. Execution time increases as we increase the size of our vectors
2. Execution time increases as the range distribution of random numbers increases

The first trend can be easily explained. If we have more elements to pass through our filter, it will take more time.

The second trend forces us to think about how our algorithm operates when the values in `v_in`. As the range of our random numbers increases, the number of potential unique elements increases, along with the expected number of unique values in `v_in`.

## Unordered Set

While there's nothing we can do about the increasing size of `v_in`, we may be able to decrease the search time of `v_out`. Instead of using a `std::vector<int>` to store our output elements, we can use an `unordered_set`.

The `unordered_set` contains a set of unique objects of type `key` (where `key` is an `int` for us), where each object is stored into a bucket according to the hash value of that object.

The benefit of this approach is that we can change the `std::find` of `v_out` to an `insert` into our `unordered_set` (where the duplicate values override each other). Unlike `std::find` which has a linear time complexity, `insert` has constant time complexity (the cost of hashing the object).

We can implement this as follows:

```cpp
// Insert each element into the unordered set
// Duplicate will be overridden
for (auto i : v_in) filter.insert(i);
```

Where `v_in` is still our `std::vector<int>` and `filter` is our `std::unordered_set<int>`.

### Performance and Analysis

Using the same benchmark setup as our baseline, here are the performance results:

```txt
------------------------------------------------------------
Benchmark                  Time             CPU   Iterations
------------------------------------------------------------
hash_set/10/10          6.34 us         6.34 us       110265
hash_set/10/100         8.41 us         8.41 us        82617
hash_set/10/1000        22.5 us         22.5 us        30708
hash_set/10/10000       30.3 us         30.3 us        23794
hash_set/11/10          12.5 us         12.5 us        55228
hash_set/11/100         14.6 us         14.6 us        47795
hash_set/11/1000        36.3 us         36.3 us        19184
hash_set/11/10000       61.3 us         61.3 us        11453
hash_set/12/10          24.8 us         24.8 us        28437
hash_set/12/100         27.0 us         27.0 us        25299
hash_set/12/1000        52.2 us         52.2 us        13070
hash_set/12/10000        118 us          118 us         5790
```

## Final Thoughts

Thanks for reading,

--Nick

### Link to the source code

- [My YouTube Channel](https://www.youtube.com/channel/UCsi5-meDM5Q5NE93n_Ya7GA?view_as=subscriber)
- [My GitHub Account](https://github.com/CoffeeBeforeArch)
- [Duplicate Removal Benchmark](https://github.com/CoffeeBeforeArch/misc_code/tree/master/duplicate_removal)
- My Email: CoffeeBeforeArch@gmail.com

