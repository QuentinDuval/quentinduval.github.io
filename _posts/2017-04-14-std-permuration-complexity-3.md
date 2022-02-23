---
layout: post
title: "Lost in std::is_permutation complexity (reddit answer)"
description: ""
excerpt: "Answering comments on the usefulness of property based testing to ensure correctness of unstable sort."
categories: [modern-cpp]
tags: [Functional-Programming, C++]
---

In this short post, I would like to answer an interesting comment that was added in the [Reddit thread](https://www.reddit.com/r/cpp/comments/63e6a1/lost_in_stdis_permutation_complexity/) asssociated to my [lost in permutation complexity previous post]({% link _posts/2017-04-04-std-permuration-complexity.md %}). Answering this comment on Reddit directly would make for a big wall of text, hence this post, which aims at providing a comprehensive answe

## Context

The [original post]({% link _posts/2017-04-04-std-permuration-complexity.md %}) described a necessary and sufficient property to test that an unstable sort was doing its job correctly.

This property is based on the fact that an unstable sort is a **permutation** of a collection such that the resulting collection is sorted after the permutation. We named this property `check_sort_property` and translated it into code:

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

We used this property inside a **Property Based Test** which consist in four distinct phases:

1. Generating random inputs (random vectors of pair of ints in our case)
2. Call our unstable sort routine on each of these vectors
3. Check that the property holds on the output of the unstable sort
4. Shrinking the failing random input to find a simpler counter example

To summarize, the goal of the `check_sort_property` property is to describe succinctly and precisely a condition that makes such a test pass or fail, knowing that the test is performed on random inputs.

## The Reddit comment

The `check_sort_property` property got some attention. One such comment mentioned that the usage of `std::is_permutation` was not justified here, and that there were another ways to unit test the unstable sort:

> […] the given problem is unit testing an unstable sorting algorithm. Compare the output with your expected result, and check that corresponding items in output and expected are equal (or neither less than the other). Using is_permutation to compare two sorted ranges is misguided.

The comment (as I understand it) arguments that it would be much easier to test the unstable sort by comparing the output of the unstable sort against an expected result, for instance, using GTEST:

```cpp
std::vector<std::tuple<int, int>> inputs = ...;
std::vector<std::tuple<int, int>> expected = ...;

custom_sort(begin(inputs), end(inputs));
ASSERT_EQ(expected, inputs);
```

We will now go through some rationales that justify the usage of properties (such as `check_sort_property`) and **property based testing** to verify the correctness of an algorithm, instead of comparing the result of a function call with an expected output as we would do with GTEST. I believe these arguments do apply for both property based tests and example based tests.

## Missing in action: Expected

There are cases in which we cannot test an algorithm against a fixed expected result. This happens when the expected result is not easy to craft.

### Random inputs

In most cases, it is very difficult to check our input against expected results when the inputs are random. Generating random inputs is precisely what we do in the context of property based testing as this allow us to test much more and potentially find edges cases we would not have thought about.

In some cases, despite random inputs, it is however still possible to get the exact expected result. For instance, if we were implementing a stable sort, we have a reference algorithm to test against.

In our case however, there is no algorithm that would match exactly what we want to test.

### Big inputs

One second use case for properties in when the inputs are huge. In such cases, the “expected” result is not easily accessible or downright impactical. For instance, we could try to test our unstable sort on a vector of thousands or more elements.

An hand-crafted expected result (matching a hand-crafted input) will be very awkward to create and expensive to maintain. Similarly, understanding the output of a failed test in such cases is hard and it is much easier to understand the output of a property (if this property correspond to a nice abstraction such as "not a valid permutation").

Tradional unit tests on big inputs becomes so tedious that most developers will just copy-paste the result of their algorithm and set it as “expected” output. This somehow defeats the purpose of testing in the first place.

And when code evolves, these “unit tests” on big inputs often transform into “regression tests”. These tests do not ensure correctness as much as they ensure that nothing changes. When software changes (as it invariably does), these tests will become red in non-meaninful ways (and either be commented, ignored, or corrected by copy pasting the new results as expected results).

### Non deterministic algorithms

One last reason, and not the least, for avoiding hardcoded expected results to our unit tests is that we sometimes have to deal with non deterministic algorithms.

We could for instance envision a distributed sorted algorithm, which, for efficiency reason, would be unstable and non-deterministic. Based on how fast each of the workers process their respective tasks, the output would be correctly sorted but elements in the same equivalence class could change their respective orders.

## The dangers of over-testing

Property based testing allows to test predicates, while hardcoded expected results typically test much more than those properties. They tend to test all the implementations details of the algorithm (in the case of our stable sort, they would test the "exact unstable way" in which the sorting algorithm is unstable)

As a rule of thumb, we should avoid testing too much of a function, and in particular, we should avoid to expand the tests past the contract of the function. Why?

### Freedom to improve

The purpose of implementing an unstable sort instead of a stable sort is to get a degree of freedom on the output of the algorithm. This degree of freedom can be leveraged to implement a faster algorithm.

This freedom extends in the time axis too: it should be fine to get different results for our stable sort across releases. If we discover a faster implementation in the future, we want to be in position to implement it. This might change the relative ordering of equivalent elements: this is fine.

### Improving developers productivity

If instead of relying on a property to check the correctness of our unstable sort, our unit tests matched against whole expected output, improvement to our implementation would be made more difficult.

Improving the algorithm could make such tests fail. This fail status could be because we broke the algorithm. But it could also be because the test relied on the implementation details of the algorithm (the specific instability).

The developer will have to check the result to make sure the failed status represents a real issue or if the test should be adapted. This manual work could be avoided by making sure our unit tests test the interface, not the implementation details.

For that reason, testing the output of an algorithm using a property might have a big positive impact on a software flexibility and developers productivity. Tests that only test the contract of a function will only fail if the function gets broken.

_Note: This argument for testing the contract and not the implementation is in contradiction with some mechanical applications of Test-driven development. The inflexibility of locking everything in place by testing implementation details will likely hurt productivity._

## Conclusion

I hope this post answers the valid concerns that were raised inside the [Reddit post](https://www.reddit.com/r/cpp/comments/63e6a1/lost_in_stdis_permutation_complexity/) and did clarify why we went for the implementation of a property making use of `std::is_permutation` and `std::is_sorted` to verify the correctness of our unstable sort.

In this process, I hope I did good justice to **property based testing** and why it is useful and sometimes preferable to **example based testing** (testing against fixed inputs with fixed expected outputs), in particular to test the contract of a function and avoid testing too much of its implementation details.
