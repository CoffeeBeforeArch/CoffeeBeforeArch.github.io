---
layout: default
title: Duplicate Removal
---

# Duplicate Removal

One of the most imporant parts of planning for performance is choosing the right algorithms. Importantly, this extends to all parts of our code (not just just things like sorting!). If we choose the wrong algorithm, we may still be able to optimize it. However, the correct algorithm will generally allows us to scale performance much further than a "worse" algorithm that has been significantly optimized.

In this blog post, we will be taking a look at some different ways we can filter duplicate values from a `std::vector<int>`. We will study the performance of these methods and examine them under different circumstances (changing the size of the vectors and distributions of the input data).

### Link to the source code

- [My YouTube Channel](https://www.youtube.com/channel/UCsi5-meDM5Q5NE93n_Ya7GA?view_as=subscriber)
- [My GitHub Account](https://github.com/CoffeeBeforeArch)
- [Duplicate Removal Benchmark](https://github.com/CoffeeBeforeArch/misc_code/tree/master/duplicate_removal)
- My Email: CoffeeBeforeArch@gmail.com

## Baseline Implementation

The goal of our duplicate removal method will be to take in a vector of random number (e.g. `1, 2, 3, 1, 2, 3`), and return a vector without any duplicates (e.g., `1, 2, 3`). One of the most simple ways we can solve this is the brute-force approach.

For this method, we'll iterate for each element in the input vector and check if it is in the output vector. If it is not in the output vector, we will add it, otherwise we will proceed to the next element in the input vector.

We can implement this algorithm using as follows:

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

The amount of work done each iteration of the outer `for` loop monotonically increases over the loop iterations. This is because the number of elements we scan in `v_out` monotonically increases over the loop iterations as we find more non-duplicate elements to add to the output vector.

### Benchmark Setup

We can use Google benchmark to measure the performance of our filter implementations. Here is the basic structure of what I used:

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

We will be studying vectors with 2^10, 2^11, and 2^12 elements. Additionally, we will be looking at random numbers between 0 and 10, 100, 1000, and 10000. These are configured with a function named `custom_args`:

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

There are two noticeable trends from this data:

1. Execution time increases as we increase the size of our vectors
2. Execution time increases as the we widen the distribution of our input data

The first trend can be easily explained. Sifting through a larger number of elements takes longer than a sifting through smaller number of elements. For example, here are the results for the three vector sizes when the distribution of random numbers is fixed to between 0 and 100:

```txt
------------------------------------------------------------
Benchmark                  Time             CPU   Iterations
------------------------------------------------------------
baseline/10/100         14.2 us         14.2 us        49771
baseline/11/100         33.8 us         33.8 us        20354
baseline/12/100         74.1 us         74.1 us         9468
```

We see a ~2-3x increase in execution time (occasionally larger for different data distributions) each time `v_in` grows by 2x.

The second trend forces us to think about how our algorithm operates when the distribution of values inside `v_in` varies. As we widen the range of input data, the number of potential unique elements along with the expected number of unique values in `v_in` increases. This means that our output vector `v_out` is likely to grow larger and larger, and `std::find` will scan more and more elements in `v_out` on each iteration of the `for` loop.

Here are the results for when the size of `v_in` is fixed to 2^12 elements and we vary the data distribution:

```txt
------------------------------------------------------------
Benchmark                  Time             CPU   Iterations
------------------------------------------------------------
baseline/12/10          30.4 us         30.4 us        22736
baseline/12/100         74.1 us         74.1 us         9468
baseline/12/1000         340 us          340 us         2058
baseline/12/10000       1127 us         1127 us          617
```

At first, increasing the range of our random numbers by 10x only increases execution time by ~2x (from 30us to 74us). However, increasing the range by another 10x increases execution time by ~4.5x (from 74us to 340us), then by an additional ~3.3x (from 340us to 1127us) when the range is extended by another factor of 10.

Between our most narrow (0-10) and widest (0-10000) distributions of data, our execution time differs by ~37x to process the same number of elements (2^12)!

## Unordered Set

While there is nothing we can do about the increasing size of `v_in`, we may be able to decrease the time it takes to determine if a value is duplicate. Instead of using a `std::vector<int>` to store our output elements, we can use an `unordered_set<int>`.

The `unordered_set` is a set of unique objects of type `key` (where `key` is an `int` in this example), where each object is stored in a bucket according its hash value.

The benefit of this data structure is that we can change the `std::find` of our input value in `v_out` to an `insert` into our `unordered_set` (where the duplicate values simply overwrite each other). Unlike `std::find` which has a linear time complexity, `insert` has constant time complexity (the cost of hashing the object).

We can implement this as follows:

```cpp
// Insert each element into the unordered set
// Duplicate will be overridden
for (auto i : v_in) filter.insert(i);
```

`v_in` is still our `std::vector<int>`, and `filter` is our `std::unordered_set<int>`.

One important thing to note (depending on your use-case) is that we have broken the ordering of output elements w.r.t each other by moving from a `std::vector` to an `unordered_set`.

### Performance and Analysis

Using the same benchmark setup as our baseline, here are the performance results from my machine:

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

Both performance trends we saw in our baseline example still exist (larger vectors take longer to filter, and a wider data distrubtion takes longer to filter than a narrower ones). If we look at the 0-100 data distribution for our 3 vector sizes, we see a ~2x increase in execution time with each 2x increase in the size of `v_in`:

```txt
------------------------------------------------------------
Benchmark                  Time             CPU   Iterations
------------------------------------------------------------
hash_set/10/100         8.41 us         8.41 us        82617
hash_set/11/100         14.6 us         14.6 us        47795
hash_set/12/100         27.0 us         27.0 us        25299
```

This is similar to, if not slightly better than our baseline implementation (and approxiamtely true for all other data distributions in our results).

However, our `unordered_set` implementation scales *far* better than the baseline as we widen our data distributions.

Here are the results if we fix the size of the vector to 2^12:

```txt
------------------------------------------------------------
Benchmark                  Time             CPU   Iterations
------------------------------------------------------------
hash_set/12/10          24.8 us         24.8 us        28437
hash_set/12/100         27.0 us         27.0 us        25299
hash_set/12/1000        52.2 us         52.2 us        13070
hash_set/12/10000        118 us          118 us         5790
```

Unlike our baseline implementation, we see almost no difference in execution time as we increase the range of our data from 0-10, to 0-100 (24.8us vs 27.0us). Increasing the range of values by another factor of 10 increases execution time by less than 2x (from 27.0us to 52.2us). Increasing by a factor of 10 again leads to an execution time increase of just over 2x (from 52.2us to 118us).

Between our narrowest and widest data distribution, our performance only dropped by a factor of ~4.5x (much better than the ~37x from the baseline implementation).

Another interesting thing to note is that our baseline implemention often fares better than our `unordered_set`  when the data distribution is narrow (e.g., 0-10). This likely comes down to the cost of our linear search of the output vector `v_out` compared to the cost of insertion into the `unordered_set`.

When we have very few unique values in our input data, there will be very few values to scan in the output vector when we check for duplicates. That means that a linear search of the `v_out` vector will be relatively inexpensive, and can (potentially) be done faster than the hashing process of our `unordered_set`.

We can see an example of this with our vector with 2^10 input elements and data values between 0 and 10:

```txt
------------------------------------------------------------
Benchmark                  Time             CPU   Iterations
------------------------------------------------------------
baseline/10/10          2.80 us         2.80 us       258206
hash_set/10/10          6.34 us         6.34 us       110265
```

Our baseline is over 2x faster than the new implementation!

## Unordered Set + Copy

While our `unordered_set` implementation gave us a massive increase in performance, we altered the original problem. We're no longer returning a `std::vector<int>`, but a `std::unordered_set<int>`. To make this a truly fair comparison, we should also include the time it takes to copy from our `unordered_set` into an output vector.

We can implement this as follows:

```cpp
// Insert each element into the unordered set
// Duplicate will be overridden
for (auto i : v_in) filter.insert(i);
// Copy the unique elements to the output fector
for (auto i : filter) v_out.push_back(i);
```

In addition to the method of filtering from the previous example, we have an additional `for` loop where we copy the unique elements in our `std::unordered_map<int>` named `filter` to an output `std::vector<int>` named `v_out`.

### Performance and Analysis

Using the same benchmark setup as our baseline, here are the performance results from my machine:

```txt
-----------------------------------------------------------------
Benchmark                       Time             CPU   Iterations
-----------------------------------------------------------------
hash_set_copy/10/10          6.35 us         6.35 us       109623
hash_set_copy/10/100         8.62 us         8.62 us        80590
hash_set_copy/10/1000        22.9 us         22.9 us        29983
hash_set_copy/10/10000       31.5 us         31.5 us        21669
hash_set_copy/11/10          12.3 us         12.3 us        56491
hash_set_copy/11/100         14.8 us         14.8 us        47399
hash_set_copy/11/1000        37.5 us         37.5 us        18664
hash_set_copy/11/10000       63.6 us         63.6 us        10695
hash_set_copy/12/10          24.5 us         24.5 us        28655
hash_set_copy/12/100         27.2 us         27.2 us        25748
hash_set_copy/12/1000        53.3 us         53.3 us        12897
hash_set_copy/12/10000        128 us          128 us         5415
```

Our results are very similar to our original `unordered_set` implementation with the exception that most execution times are slightly longer. This can be attributed to the additional copy of data from our `unordered_set` to the output `std::vector`.

The copy overhead is most noticeable when the data distribution is very wide (e.g., 0-10000). This is because the expected number of unique values we must copy from `filter` to `v_out` is much larger than the expected number when the data distribution is very narrow (e.g., 0-10).

## Final Thoughts

Much of performance optimization is finding ways to do less work, be that algorithmically, through compiler optimizations, or through tuning our code to our hardware to avoid things like wasted data movement. In this blog post, we looked a fairly simple way to filter duplicates from a vector of integers using an `unordered_set` which allowed us to avoid a linear scan of a vector for each input element.

Through our experiments, we learned a few important lessons:

1. Performance can be influenced by both the size of the data and the values of the data
2. Different containers work better in different situations
3. Brute force solutions can be effective in certain circumstances

Thanks for reading,

--Nick

### Notes

- An `unordered_map` can be used instead of `unordered_set` if you need to additionally keep track of which elements were duplicates.
- Calculating the expected number of unique values in `v_in` is a worthwhile exercise to prove to yourself why widening the data distribution hurts performance.
- Studying different shapes of input data would be an interesting experiment
- Studying the performance of different hash set/table implementations for different data distributions would be an interesting experiment

### Link to the source code

- [My YouTube Channel](https://www.youtube.com/channel/UCsi5-meDM5Q5NE93n_Ya7GA?view_as=subscriber)
- [My GitHub Account](https://github.com/CoffeeBeforeArch)
- [Duplicate Removal Benchmark](https://github.com/CoffeeBeforeArch/misc_code/tree/master/duplicate_removal)
- My Email: CoffeeBeforeArch@gmail.com

