---
layout: post
title: "Accumulating merge sort (C++)"
description: ""
excerpt: "Implementing a merge sort in terms of a std::accumulate (left to right scan) is possible. Here is how to merge sort using only ForwardIterators."
categories: [modern-cpp]
tags: [Functional-Programming, C++]
use_math: true
---

Back in September 2016, I was lucky enough to attend a talk from [Ben Deane](https://twitter.com/ben_deane) entitled [std::accumulate: Exploring an Algorithmic Empire](https://www.youtube.com/watch?v=B6twozNPUoA). In this talk, he presents the unknown possibilities of `std::accumulate`, an algorithm also known as fold or reduce depending on the language.

In his talk, he announced he had implemented almost all the algorithms of the STL based on `std::accumulate` (41st minute in the [video](https://www.youtube.com/watch?v=B6twozNPUoA)), including `std:stable_sort`, which was a bit surprising to me.

So I searched [his blog](http://www.elbeno.com/blog/) and could not find a hit on how he did it. So I decided to write this post to describe the solution I found.

The subject of today’s post is therefore to:

* Implement this algorithm in C++
* Make it work on the input container directly
* Make it work for more than just lists

In the process, we will build a `std::stable_sort` variant that works for [ForwardIterators](http://en.cppreference.com/w/cpp/concept/ForwardIterator), whereas std::stable_sort only works for [RandomAccessIterator](http://en.cppreference.com/w/cpp/algorithm/stable_sort).

## Small ranges and binary counters

The solution relies on using a **binary counter** like data-structure:

* Our binary counter will hold at each level $L$ a range of size $2^L$. The binary counter will thus be limited in size to $\log N$ if $N$ is the number of elements of the collection to sort.
* We populate the binary counter by successively adding ranges of size one to the counter.
* Each time we have a collision of “bit” (range of the same size), we merge them and consider it as a “carry”.
* We keep propagating the carry up the levels, until we reach an empty slot.

This is best explained by an example:

```
Initially
Counter = [[1, 2], [3, 4, 5, 6]]
 
Adding element 7
Counter = [[7], [1, 2], [3, 4, 5, 6]]
 
Adding element 8
Counter = [merge [8] and [7], [1, 2], [3, 4, 5, 6]]
Counter = [merge [7, 8] and [1, 2], [3, 4, 5, 6]]
Counter = [merge [1, 2, 7, 8] and [3, 4, 5, 6]]
Counter = [[1, 2, 3, 4, 5, 6, 7, 8]]
 
Adding elements 9, 10 and 11
Counter = [[11], [9, 10], [1, 2, 3, 4, 5, 6, 7, 8]]
```

At this point, we are almost done, but not quite. We still need to collapse all the remaining “bits” of our counter into one answer. We do this by accumulating over these “bits” with a merge operation.

```
[[11], [9, 10], [1, 2, 3, 4, 5, 6, 7, 8]]
[[9, 10, 11], [1, 2, 3, 4, 5, 6, 7, 8]]
[[1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11]]
```

We are done! Let us try to implement this in C++.

## Impementing the binary counter

Following a quick youtube search, I was able to find an episode of the [A9 Stepanov lectures](https://www.youtube.com/playlist?list=PLHxtyCq_WDLXryyw91lahwdtpZsmo4BGD) that dealt with the implementation of a binary counter in C++.

The following code is greatly inspired (almost copy-pasted, except for the `std::find_if`) from these amazing lessons:

* The `add_to_counter` function handles the carry propagation, and returns the carry if it ran out of bits
* The `reduce_counter` allows to collapse all the remaining bits at the end of the accumulation
* The `binary_counter` class maintains the vector of bits, and is responsible for gluing the algorithms together

```cpp
template<typename Iterator, typename Value, typename BinaryOp>
Value add_to_counter(Iterator first, Iterator last,
                     Value carry, Value const &zero,
                     BinaryOp op)
{
    assert(carry != zero);
    for (; first != last; ++first) {
        if (*first == zero) {
            *first = carry;
            return zero;
        }
        carry = op(*first, carry);
        *first = zero;
    }
    return carry;
}

template<typename Iterator, typename Value, typename BinaryOp>
Value reduce_counter(Iterator first, Iterator last,
                     Value const &zero, BinaryOp op)
{
    first = std::find_if(first, last, [&zero](auto &v) { return v != zero; });
    if (first == last)
        return zero;

    Value result = *first;
    for (++first; first != last; ++first) {
        if (*first != zero)
            result = op(*first, result);
    }
    return result;
};


template<typename BinaryOp, typename Value>
class binary_counter {
public:
    binary_counter(BinaryOp const &op, Value const &zero)
        : m_bits(), m_zero(zero), m_op(op) {}

    void add(Value carry) {
        carry = add_to_counter(begin(m_bits), end(m_bits), carry, m_zero, m_op);
        if (carry != m_zero)
            m_bits.push_back(std::move(carry));
    }

    Value reduce() const {
        return reduce_counter(begin(m_bits), end(m_bits), m_zero, m_op);
    }

private:
    std::vector<Value> m_bits;
    Value m_zero;
    BinaryOp m_op;
};
```

As for the Haskell implementation in our [previous post]({% link _posts/2016-12-22-folding-merge-sort.md %}), the C++ implementation of the binary counter is able to deal with any Monoid. It only requires:

* An associative binary operation
* That admits a neutral element

This implementation also shows a great usage of object orientation to structure a program: the class is used to combine some algorithms with the data their operate on (to guaranty invariants), while the algorithms are kept outside of the class for greater re-use.

## The merge binary operation

Now that we have a `binary_counter` that works on any Monoid, we need to define the equivalent of the `SortedList` Monoid instance on which to apply it.

* The neutral element will be the empty range
* The binary operation will be something close to `std::merge`.

Why something close and not `std::merge` directly? Because we need some kind of auxiliary buffer to perform the merge.

This is the implementation I came to:

```cpp
template<typename Iterator>
struct merger
{
    using Value = typename std::iterator_traits<Iterator>::value_type;
    using SortedRange = std::pair<Iterator, Iterator>;

    SortedRange operator()(SortedRange const &lhs, SortedRange const &rhs) const {
        assert(lhs.second == rhs.first);
        std::vector<Value> tmp(lhs.first, lhs.second); //Copy the left range
        std::merge(
                std::begin(tmp), std::end(tmp), // Left container copy (source)
                rhs.first, rhs.second,          // Right container (source)
                lhs.first);                     // Left container (destination)
        return SortedRange(lhs.first, rhs.second);
    }
};
```

I found this implementation more tricky than I first thought it would be. In particular, we need to ensure that the two ranges we merge are always:

* Next to each other (one range end is the beginning of the other)
* In the right order (left range is before the right range)

I claim these two propositions will always hold if we accumulate our container from left to right. The argument is based on the implementation of `add_to_counter` and `reduce_counter` and the properties of Monoids.

Indeed, `reduce_counter` initializes its result from the lower level “bit” (which is the latest arrived in the counter) and combines it on the right with `op(*first, result)`. So the latest arrived range is always provided as right argument of the merge operation.

The same goes for `add_to_counter` that combines the carry on the right side of the binary operation. Again, the latest arrived range is provided as right argument of the merge operation.

This property of the `binary_counter` algorithm is crucial. The Monoid property guaranties our operation is associative but not necessarily commutative. So the `binary_counter` must have this property to be correct, and as a result we can rely on it.

## Accumulating with iterators

It is now time to combine our different elements to build our std::stable_sort based on a `std::accumulate`. But there is a catch…

As observed by Ben Deane in [std::accumulate: Exploring an Algorithmic Empire](https://www.youtube.com/watch?v=B6twozNPUoA), we cannot really use the std::accumulate of the STL here. It deals with values while we need ranges here, which means iterators.

An easy way to get past this is to create our own `accumulate_iter` algorithm, that provides an iterator to the accumulating function (instead of a value):

```cpp
template<typename Iterator, typename Value, typename Accumulator>
Value accumulate_iter(Iterator first, Iterator last,
                      Value val, Accumulator fct)
{
    for (; first != last; ++first)
        val = fct(val, first);
    return val;
};
```

Based on this algorithm, we are now able to assemble the different parts needed to build a `std:stable_sort` working on `ForwardIterators`:

```cpp
template<typename ForwardIterator>
void stable_sort_forward(ForwardIterator first, ForwardIterator last)
{
    using Range = std::pair<ForwardIterator, ForwardIterator>;
    using Counter = binary_counter<merger<ForwardIterator>, Range>;
  
    Counter c { merger<ForwardIterator>{}, { last, last } };
    accumulate_iter(first, last, &c,
                    [](auto c, auto it) {
                        c->add({it, std::next(it)});
                        return c;
                    })->reduce();
}
```

Now we are truly done. You can convince yourself that it does stable sort a container by trying it on some examples.

## Wrapping it up

We managed to write a `std::stable_sort` using a variant of `std::accumulate` that sorts a container provided as parameter. I do not know if this is the same solution Ben Deane used to implement his own, but it is definitively doable.

One interesting consequence is that our stable sort algorithm also works for `ForwardIterator`. So you can sort a `std::list` and even a `std::forward_list` with it.

Now, I would not be offended if you told me you found `accumulate_iter` usage neither beautiful, nor concise, nor simple. If you do, you might be more interested in this version of the algorithm, where we use a raw loop:

```cpp
template<typename ForwardIterator>
void stable_sort_forward(ForwardIterator first, ForwardIterator last)
{
    using Range = std::pair<ForwardIterator, ForwardIterator>;
    using Counter = binary_counter<merger<ForwardIterator>, Range>;

    Counter c { merger<ForwardIterator>{}, { last, last } };
    for (; first != last; ++first) {
        c.add(std::make_pair(first, std::next(first)));
    }
    c.reduce();
}
```

It gives the same results, so it is mostly a matter of taste here. Indeed, C++ may not be the best language to play with fold algorithms. You might prefer the imperative style when programming in C++. Ultimately, deciding which tool and which paradigm to use is a choice you and your team have to make.