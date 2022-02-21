---
layout: post
title: "Lost in std::is_permutation complexity"
description: ""
excerpt: "This post is dedicated to an STL algorithm which caused me some serious performance issues at my first use of it: std::is_permutation."
categories: [modern-cpp]
tags: [Functional-Programming, C++]
---

This post is dedicated to an STL algorithm I discovered only recently, and which caused me some serious performance issue at my first use of it: `std::is_permutation`.

The algorithm appeared in the STL with C++11 and in principle is quite a useful one. It is however deeply penalized by its inefficiency, which make this algorithm almost impractical.

In this post, we will first describe the algorithm itself and describe some of the use cases that makes it interesting in principle.

Then we will talk a bit about my misadventure with it. This misadventure could have been avoided by reading the manual and not use `std::is_permutation` in the first place. But I suspect other might fall into the trap as well, for reasons that we will describe as well.

We will conclude by discussing the design decision that makes this algorithm implementation inefficient by construction, and discuss some alternatives.

## Motivation for std::is_permutation

From the the [cppreference documentation](http://en.cppreference.com/w/cpp/algorithm/is_permutation), `std::is_permutation` is an algorithm that takes two ranges as input. It indicates whether there exists a rearrangement of the elements that would make both ranges be equal. The equality criteria can be parameterized.

### Anagrams

The simplest use case for `std::is_permutation` would be to implement a function that would tell if two strings are anagrams of each others.

An anagram is “the result of rearranging the letters of a word [..] to produce a new word [..], using all the original letters exactly once” (Wikipedia). This is a perfect match for `std::is_permutation`:

```cpp
bool is_anagram(std::string const& lhs, std::string const& rhs)
{
  return std::is_permutation(begin(lhs), end(lhs), begin(rhs), end(rhs));
}
```

### Equality but for the ordering

A more interesting use case of `std::is_permutation` is to check that two collections are equal but for the ordering. In other words, that they contain the exact same elements. This is a pretty **common use case inside Unit Tests**.

For instance, we may want to test a function `eval_accounting_entries` that produces a vector of accounting entries for the current period. The contract says nothing about the order of the accounting entries produced.

In order to avoid testing more than the actual contract of `eval_accounting_entries`, increasing the precision and resilience of our unit test, we could use `std::is_permutation` instead of `std::equal`.

```cpp
std::vector<AccountingEntry> expected = ...;
std::vector<AccountingEntry> result = eval_accounting_entries(...);
	
ASSERT_TRUE(
  std::is_permutation(
    begin(expected), end(expected),
    begin(result), end(result)));
```

This check of equality without the ordering might be useful as well inside a GUI. We might be interested in notifying some listening processes if the user modifies some accounting entries, but not if the modification is a simple rearrangement.

## Testing an unstable sort

Time to describe my little misadventure. Let us say that we just implemented a new unstable sort algorithm and want to make sure that it is correct. How can we test that our implementation is bug free?

We could do some example based tests for starter, before adding some Property Based Testing to increase our level of confidence. The plan is to:

* Find a good invariant that, if it holds, proves the sort is correct
* Generate a bunch of random vectors
* Call our sort routine on each of these vectors
* Check that the invariant hold on the outputs of the sort

### Finding a good property to check

If our sort was stable, we could check the result of our sort against `std::stable_sort`. That would make it a perfect property. But **our sort is unstable**. So we cannot compare it against a reference sort to prove that it works. Sure, `std::sort` is unstable too, but the unstable swaps do not have to be the same.

Fortunately for us, sorting a collection is performing a permutation of the elements of a range such that the resulting order of the elements complies with the increasing order relative to some ordering relation R.

It means that we can check that our unstable sort works fine by simply checking whether the resulting collection answers true to both `std::is_permutation` and `std::is_sorted`.

### Implementing the property

We can port this definition into code. The `check_sort_property` function returns true if the result iterable is a correct sort of the input iterable:

```cpp
template<typename InputIterable, typename ResultIterable, typename Less>
bool check_sort_property(InputIterable const& input,
                         ResultIterable const& result,
                         Less less)
{
  return
    std::is_permutation(begin(input), end(input), begin(result), end(result))
    && std::is_sorted(begin(result), end(result), less);
}

template<typename InputIterable, typename ResultIterable>
bool check_sort_property(InputIterable const& input,
                         ResultIterable const& result)
{
  auto less = std::less<typename InputIterable::value_type>();
  return check_sort_property(input, result, less);
}
```

_Note: Asserting that `std::is_sorted` returns true would not be enough: there could be missing elements. Asserting that the number of elements is the same would not be enough either: it could be the same element repeated as many time as needed. We really need the permutation check to ensure our sort works correctly._

### Generating vectors

We then need to generate random vectors to perform our tests. We define a function `make_vector_gen` that will return a lambda which we can call to generate random vectors for us.

* The max_size allows us to parameterise how big the vector can be.
* The value_generator allows us to generate the content of the vector

With this small piece of code, we are able to generate random vectors made of random elements of any chosen type:

```cpp
template<typename ValueGenerator>
auto make_vector_gen(int max_size, ValueGenerator value_generator)
{
  return [=](std::mt19937& gen) {
    std::uniform_int_distribution<int> distribution (0, max_size);
    std::vector<decltype(value_generator(gen))> out(distribution(gen));
    for (auto& val: out) val = value_generator(gen);
    return out;
  };
}
```

_Note: alternatively, we could rely on [RapidCheck](https://github.com/emil-e/rapidcheck). This is the choice to make if it was production code. But as the generator is simple enough, we can do it by hand here._

### Generating tuples

To test our our sort on our random vector, we must first be able to generate some random elements inside our vector. We will settle for tuples of integers.

Why tuples and not simple integers? The motivation is to test our sort routine with a “less” function that orders by the second value of the tuple. This will allow us to exercise the “unstable” part of our algorithm (sorting integers would show no differences between stable and unstable sorts).

We design our tuple generator generically. The following code allows to generate arbitrarily large tuples, makes of arbitrarily chosen types:

```cpp
template<typename... Generators>
auto make_tuple_gen(Generators... generators)
{
  return [=](std::mt19937& gen)
  {
    return std::make_tuple(generators(gen)...);
  };
}
```

### Composing and testing our generators

We now have the ability to generate vectors of tuples of arbitrary types. We can create a last generator of random integers:

```cpp
auto int_gen(int max_size) {
  return [=](std::mt19937& gen) -> int {
    std::uniform_int_distribution<int> distribution(0, max_size);
    return distribution(gen);
  };
}
```

We can now easily combine all of our generators to create a vector of tuples of integers. We can quickly check that it behaves correctly:

```cpp
auto int_vector_gen = make_vector_gen(10, int_gen(1000));
auto int_pair_gen = make_tuple_gen(int_gen(100), int_gen(1000));
auto int_pair_vector_gen = make_vector_gen(20, int_pair_gen);

// Example code
std::mt19937 gen = ...;

int_vector_gen(gen);
//Outputs: [171,614,15,346,263,44]

int_pair_vector_gen(gen);
//Outputs: [(1, 280),(47, 218),(42, 867),(32, 119),(48, 396),(58, 82),(41, 576),(63, 108),(16, 108),(50, 216)]
```

### Three, two, one... test!

Everything is ready. We can now test that our `custom_sort` routine does its job correctly by running some random test inputs and check that our sort property always holds.

Let us start small, with one single example of a vector holding up to 50,000 tuples of integers, which we will sort by their second component:

```cpp
auto int_pair_gen = make_tuple_gen(int_gen(100), int_gen(1000));
auto inputs_gen = make_vector_gen(50000, int_pair_gen);
auto less_on_second = ordering_by_index<1>();

//Generate a vector of up to 50k elements
auto input = inputs_gen(gen);

//To test our sorting routine
auto sorted = input;
custom_sort(begin(sorted), end(sorted), less_on_second);

//Check the property of unstable sort
check_sort_property(input, sorted, less_on_second);
```

Now, this is where it hurts. Depending on the generating inputs:

* The actual sorting lasts less than 1 milliseconds
* The property check might last more than 500 milliseconds

So much for the fast feedback of unit tests… With 100 test samples (the default of most property based testing libraries), we have to wait almost a minute to get some feedback. What is happening?

## Hopelessly inefficient std::is_permutation

The reason why the property check is so slow is because the implementation of `std::is_permutation` has worst case quadratic complexity. This behaviour is documented in both [cplusplus.com](http://www.cplusplus.com/reference/algorithm/is_permutation/) and [en.cppreference.com](http://en.cppreference.com/w/cpp/algorithm/is_permutation).

_Note: obviously, I should have read the manual. But had I read the manual, I would have chosen not to use `std::is_permutation`. This is not very satisfactory._

### Quadratic by design

Given the definition of the `std::is_permutation` algorithm, there is no other way it could be better. Having only access to an equality predicate, the algorithm cannot do much better than trying (in the worse case) all the combinations.

By asking for a little more than an equality predicate, it could do a much better job. We will explore such alternatives later in the post.

### Surprisingly slow

Most algorithm design courses teach us that quadratic complexity is something we should not accept unless we have no other choice (*). The big problem here is that there are much more efficient algorithms available for us to check for `std::is_permutation`.

Most developers do know about these better algorithms, as they are frequently asked as interview questions. This makes the quadratic implementation of `std::is_permutation` **counter-intuitively slow**, increasing the the chance of introducing a performance issue in the software.

_(*): The rationale is that quadratic algorithms do not scale. Linear algorithms scale with the hardware. Throwing a multiplicative logarithm does not change this result that much: with twice as much as resources, such algorithm can handle nearly twice as big inputs._

### What about Range-V3?

[Range V3](https://github.com/ericniebler/range-v3) seems to follow the same algorithm as the one used for std::is_permutation. It also implements a quadratic algorithm to do the job. You can have a look at [permutation.hpp](https://github.com/ericniebler/range-v3/blob/master/include/range/v3/algorithm/permutation.hpp) and read the code by yourself.

## Some possible alternatives

There are plenty of alternatives to implement `std::is_permutation` much more efficiently. Which alternative to select depends on the concepts available on the type inside the iterables provided to `std::is_permutation`.

### Potential solutions

We list below some of these most obvious algorithms we could use to implement `std::is_permutation`, in decreasing order of preference:

For types with **few inhabitants**, like char, we can number each inhabitant. We can create an array, indexed by each inhabitant, and count the number of characters for both strings. We then check that the number of occurrences of each character is the same in both strings. The resulting algorithm is **O(N)**.

For types with **hash functions**, we can follow the same principle, but with a hash map. We count the frequency of each term in each collection and compare the result. We then check if the keys and their associated values (the number of occurrences of the key) match in both hash maps. The resulting algorithm is slower, but still **O(N)**.

For types with a **strict total ordering**, we can sort both collections and compare them for equality. If their size differ, we can early return with a negative answer. We can also sort one of the collection, and binary search each element of the other collection in it (the usual trick is to **sort the smallest collection**), but that would require all elements to be unique in order to work. The resulting algorithm is **O(N * log N)** in both cases.

### Drawbacks of the alternatives

Each of these alternative solutions are much faster than the `std::is_permutation` current implementation. They will scale properly.

The problem with each of these algorithms is that they require extra memory to perform their work. The first two alternatives requires either an array or an hash map, and the last one will have to create a vector to avoid mutating its inputs (and also, binary search requires a random access container to be efficient, which is not a given).

One additional problem with each of these algorithms is that they ask for more than the standard **Regular Type** requirements on which the STL is based. On the other side, the `std::is_permutation` implementation is perfectly satisfied by the Regular Type requirements.

## Conclusion and what's next?

We went over a use case that show how penalising the quadratic algorithmic complexity of `std::is_permutation` can be. We discussed why the algorithm could not be made better with the requirements it makes on the types contained in the ranges it takes as inputs.

We went over some alternative, in which the algorithm complexity could be vastly improved by requiring just a bit more on the types manipulated inside the algorithm.

The next post will discuss a bit more these additional requirements and whether they can be justified as basic assumption on types manipulated by the STL algorithms.

