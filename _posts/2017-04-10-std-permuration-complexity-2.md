---
layout: post
title: "How to improve std::is_permutation complexity"
description: ""
excerpt: "Exploring alternatives to the hopelessly inefficient std::is_permutation."
categories: [modern-cpp]
tags: [Functional-Programming, C++]
---

In our [previous post]({% link _posts/2017-04-04-std-permuration-complexity.md %}), we talked about the `std::is_permutation` algorithm and its algorithm complexity issue and wee went over several use cases that seems like perfect matches for this algorithm for which we cannot really use it because of its complexity issues.

Indeed, because of its quadratic complexity, we made the argument that std::is_permutation is almost impractical: its costs is not worth the trade-off of manually coding the alternative.

Finally, we discuss three alternative of more efficient implementations, that each require some extra memory and some additional requirements on the inputs.

In this post, we will discuss these implementations further, and the additional requirements they add on the inputs. The goal is to explore whether these additional requirements make sense and could be used as basis for an alternate implementation. We will end up by a proposal of evolution for `std::is_permutation` and some other parts of the STL as well.

## Alternate implementations

Since the standard implementation of `std::is_permutation` is too slow to be usable in practice, we have to find some more efficient implementations for it.

Our previous post listed three such alternatives. These alternatives depend on the operations available on the type std::is_permutation iterates over:

* An array based approach for small vocabulary sizes (like ASCII strings)
* An hash map based approach in case std::hash is available
* An sort based approach in case a strict total ordering is available

Since the array based approach is at the same time very specialized and very similar to the hash map based approach, we will only cover the approaches 2 and 3 below.

### Hash based implementation

If we have a hash available on the types we iterate over, the implementation of is_permutation can leverage it to perform fast searches inside a hash map. The implementation consists in:

* Counting the number of occurrences of each elements in the left iterable
* Counting the number of occurrences of each elements in the right iterable
* Checking that these elements and occurrence counts match up

We can be a bit more clever than this in our implementation, and **use a single `unordered_map` instead of two**. We count plus one for each element of the left iterable, and count minus one for the elements of the right iterable.

```cpp
template <
  typename LhsForwardIterator,
  typename RhsForwardIterator,
  typename Equal = std::equal_to<typename LhsForwardIterator::value_type>,
  typename Hash = std::hash<typename LhsForwardIterator::value_type>
>
bool is_permutation_hash(
  LhsForwardIterator lhs_first, LhsForwardIterator lhs_last,
  RhsForwardIterator rhs_first, RhsForwardIterator rhs_last,
  Equal equal = Equal(),
  Hash hash = Hash())
{
  using value_type = typename LhsForwardIterator::value_type;
  std::unordered_map<value_type, int, Hash, Equal> frequencies;
  
  for (; lhs_first != lhs_last; ++lhs_first) frequencies[*lhs_first] += 1;
  for (; rhs_first != rhs_last; ++rhs_first) frequencies[*rhs_first] -= 1;
  
  return std::all_of(begin(frequencies), end(frequencies),
                     [](auto const& p) { return p.second == 0; });
}
```

This listing is naive in many ways, but has the merit not to require any scrolling. We can improve the solutions in many ways:

* Early returning in case the size of both iterable do not match
* Reserving the space inside the `unordered_map` to limit resizes
* Aborting the second scan in case the key to decrement is not found

These ideas are implemented in the following [Gist GitHub](https://gist.github.com/deque-blog/dab4ccc71aadd9e2bdc41713d197ebe6). And there are for sure many more refinements we could do, optimizations as well as cosmetic changes (like using ranges), but this is not the point of this post.

_Note: There is also a problem of packaging of the function `is_permutation_hash`. It takes up to six arguments, which is a first problem. The default arguments are the second problem: both the equal and the hash have to be provided or none should be. Both these problems should be dealt with in production code._

### Ordering based implementation

If we have a strict total ordering available on the types we iterate over, the implementation of is_permutation can leverage it to sort both collections and compare them equal afterwards. Here is a naive implementation of such a strategy:

```cpp
template <
  typename LhsForwardIterator,
  typename RhsForwardIterator,
  typename Less = std::less<typename LhsForwardIterator::value_type>
>
bool is_permutation_less(
  LhsForwardIterator lhs_first, LhsForwardIterator lhs_last,
  RhsForwardIterator rhs_first, RhsForwardIterator rhs_last,
  Less less = Less())
{
  using value_type = typename LhsForwardIterator::value_type;
  std::vector<value_type> lhs(lhs_first, lhs_last);
  std::vector<value_type> rhs(rhs_first, rhs_last);
  if (lhs.size() != rhs.size())
    return false;
  
  std::sort(begin(lhs), end(lhs));
  std::sort(begin(rhs), end(rhs));
  return lhs == rhs;
}
```

### Performance improvements

Due do their lower algorithmic complexity, both these alternatives perform quite faster than their std::is_permutation counterpart.

For 50,000 elements, the standard implementation lasts around 1.7 seconds in average on my laptop. Both alternatives are systematically below the 10 milliseconds mark. This is more than 2 order of magnitude faster.

These implementations are naive and could be made much faster: minimal changes to the hash based implementation leads to go down the 5 milliseconds. Further improvements could be reached by using more appropriate data structures. For instance, a hash map implementation based on [linear probing](https://en.wikipedia.org/wiki/Linear_probing) would be much more efficient since we do not delete any keys in such use case.

## Requirement on types

All the implementations listed above add some additional constraints on the underlying types of the iterables provided to the `is_permutation` algorithm. The concern is whether these constraints would make sense in the general case.

In this section, I will try to convince you that these constraints are justified in many cases. In particular, I will argue that implementing `std::hash` should be workable for most types, and as a consequence, that we should be able to rely on its presence in all STL algorithms manipulating value types.

### Value types

We can safely assume that the two iterables provided to `is_permutation` will contain what is usually called value types.

There are plenty of definitions available for value types. [John Lakos](https://www.youtube.com/watch?v=W3xI1HJUy7Q) refers to them as matching some ethereal types. [Eric Evans](https://www.amazon.fr/Domain-Driven-Design-Tackling-Complexity-Software/dp/0321125215) refers to them as objects whose characteristics are derived from their attribute values.

In any case, it would not make much sense to call `std::is_permutation` on iterables containing something else than value types. For instance, it makes no sense on the two other object categories distinguished by Eric Evans, Entity types and Services.

### Regular types

In his quite famous paper [Fundamentals of Generic Programming](http://stepanovpapers.com/DeSt98.pdf), Stepanov defines the concept of regular type as a type satisfying a set of constraints that makes it suitable for generic programming.

These constraints happens to match pretty closely what we expect from a value type. The following [Stack Overflow](http://stackoverflow.com/questions/13998945/what-is-a-regular-type-in-the-context-of-move-semantics) thread features a great answer from Sean Parent that adds some more constraints after the release of C++11.

Among the list of constraints first listed by Stepanov stands the requirement for a total ordering. Among the additional suggested by Sean Parent stands the requirement for `std::hash` to be properly defined for the type.

So the constraints required by both our alternative implementations do make some sense for value types. Let us now discuss which of strict total ordering and hash function would make more sense.

### Hash is easier to implement

For the simplest use cases, defining a hash function or an operator less on a type are both pretty simple.

The lexicographic comparison works great to define the operator less on `std::tuple`, `struct` or `class`. Hash functions can be implemented in terms of of the hashes of the attributes and the use of `boost::hash_combine` to combine their result (and again, we can selection the lexicographic ordering for the order of combination).

But there are cases in which **defining a strict total ordering gets much harder**. For instance, how would you efficiently define the operator less on an `unordered_map`? The order of the elements of a hash map is unspecified, making the lexicographic comparison unworkable.

Defining a hash function does not require such lexicographic ordering to exist. We can define the hash of a `unordered_map` by summing the hash values of each pairs stored inside the map (this is in fact what a [valid Java implementation](http://grepcode.com/file/repository.grepcode.com/java/root/jdk/openjdk/8u40-b25/java/util/AbstractMap.java#AbstractMap.hashCode%28%29) does). There are other definitions possible: any associative and commutative function could replace plus.

I have no formal proof to back it up, but I would argue that providing a std::hash function should be workable in every places in with we can define `operator==` (while operator less is more problematic). For instance, Clojure has no problem providing a hash function by default for all its value types.

### Hash complements equal

The other reason why `std::hash` is easier to add as a type requirement is its absence of meaning. The hash function is **only an implementation trick to improve the efficiency of our algorithms**. We do not have to bother with its meaning, we only need to make sure it is consistent with `operator==`.

This is not true for the operator less. Let us consider an example in which this distinction is important. Here is a very naive modelization of a color in RGB encoding:

```cpp
struct rgb_color
{
  int m_red;
  int m_green;
  int m_blue;
};
```

We can easily define the `operator==` such that two colours are equal if all their RGB components are equal. This definition makes sense in the context of the domain.

We can as easily define `std::hash`, by combining the hash of the red, green and blue integers. This definition does not have to make sense in the domain, it is merely an implementation detail.

We can also define the ordering rather mechanically, using `std::tie`:

```cpp
bool operator< (rgb_color const& lhs, rgb_color const& rhs)
{
  auto as_tuple = [](rgb_color const& c) {
    return std::tie(c.m_red, c.m_green, c.m_blue);
  };
  return as_tuple(lhs) < as_tuple(rhs);
}
```

**But it does not make much sense to define such ordering**. Is there really a notion of order between colours? Why this one in particular?

Sean Parent advices in this same [Stack Overflow answer](http://stackoverflow.com/questions/13998945/what-is-a-regular-type-in-the-context-of-move-semantics) to **reserve the operator less for when there is a natural ordering**. And there seems to be no natural ordering in this context.

### Value composition

**Composition allows decomposition**. There is no point is breaking down a problem into pieces if we cannot plug back the pieces together. Compositions is therefore one of the most (if not the most) important characteristic to seek in everyday software.

This applies to value types. We get tremendous leverage if we can aggregate values into values and preserve their value-like concepts transparently (such as equality and hash) along the the way as we compose them.

For instance, if an unordered_map (or vector) of values is a value, we can transparently replace a std::string by an arbitrary collection of std::string inside an STL algorithm (or container) leveraging hash, and get the job done with the same efficiency.

Since the `std::hash` could be defined mechanically in most case, there is therefore a great value in supporting it for our most common ways to aggregate values (tuples, vectors and maps for instance) and to provide helpers to help us define it on our own types (via `std::tie` for instance).

## Dear std::hash needs some love

The STL does not seem to love `std::hash` that much. For reasons that I am unaware of, this lack of affection seems to be all over the Standard Template Library.

### Proofs of un-affection

For instance, `std::tuple`, `std::vector` and `std::map `define both equality and equivalence relations automatically. But none of them define `std::hash`, while they could rather easily.

The same goes for `std::unordered_map`: it defines equality (but not operator less, probably for the reasons we listed above) but does not provide the associated specialization for `std::hash`.

### Standard Tuple

This lack of definition for `std::hash` is especially troubling in the context of `std::tuple`: defining `std::hash` on `std::tuple` would automatically offer a helper function to define it on our user defined types.

Using `std::tie` could get us an implementation for our custom data types very succinctly. For our rgb_color, it could look like this:

```cpp
size_t operator()(rgb_color const& c) const
{
  auto as_tuple = std::tie(c.m_red, c.m_green, c.m_blue);
  return std::hash<decltype(as_tuple)>()(as_tuple);
}
```

### Why not fix it?

For all the reasons listed above and in the previous section (composition), I strongly believe that `std::hash` should be defined for the following types:

* `std::tuple`: this is the most important one for it is an enabler
* Sequential containers: for consistency with `std::string` (also a container)
* Associative containers: very useful since less is hard to define on hash maps

Adding these would make for a good addition to the features of C++ (*) and would probably greatly help a lot of C++ developers in their daily job.

_(*) I did not go through all the standard to check if this was already proposed for C++20 (or already integrated in C++17). Maybe it already is._

## Back to std::is_permutation

Let us assume for this section that the arguments above convinced you, and you are ready to believe that `std::hash` should be a given for value types. Let us discuss what it would imply for `std::is_permutation`.

### Use hash in std::is_permutation

If `std::hash` becomes available for most of our types, we could improve `std::is_permutation` such that:

* It makes it use `std::hash` if available, making it O(N) [W.H.P](https://en.wikipedia.org/wiki/With_high_probability).
* It falls back to its current implementation otherwise

### What about memory usage?

The algorithms based on `std::hash` or operator less do need additional memory. Although we are most likely to suffer more from the quadratic complexity than from this O(N) space requirement, this drawback cannot just be ignored.

It could very well be a problem in some use cases in which memory is rare. The STL could provide a specific quadratic implementation for such cases, that could be named `std::inplace_is_permutation` to be explicit.

### Combining alternatives

The [last CppChat](https://www.youtube.com/watch?v=svA89XpMghE) asked whether we should try to always automatically select the best implementation, or whether the user should have a way to select. We could make so that both approaches can be combined.

For instance, `std::is_permutation` could be selecting its implementation based on some concepts or `constexpr if`. This selection process could take the form of a simple redirection to algorithms (with distinct names) that would implement the different policies (such as `std::inplace_is_permutation`).

The rationale is that the default selection policy (with priorities inside the selection in case several branches are available) might not be the best choice for everyone. Providing different functions allows the user to make a conscious choice if he desires to do so.

## Summary of the proposal

We discussed two different implementations that offers much better algorithmic complexity than the standard implement of std::is_permutation.

We choose the hash based implementation, based on the argument that it would make sense for most for all value types to support std::hash. Based on this assumption, we saw how we could improve std::is_permutation to take advantage of this.

Here is a list of the different proposals of changes for the STL, that I think would make a lot of sense and improve the life of C++ developers in the process:

* Add a std::hash specialization for std::tuple
* Add a std::hash specialization for all sequential containers
* Add a std::hash specialization for all associative containers
* Fix std::is_permutation to take advantage of std::hash if available
* Provide a std::inplace_is_permutation as alternative implementation

In particular, I think that the first proposal, the specialization of std::hash for std::tuple, would be both feasible and really useful. It would help developers define their own std::hash for custom data types.

## Final words

This is it. I described a series of changes I would love to see in the STL. These were only opinions of mine, which I do not expect everyone to share.

I have no doubt some C++ developers have put a lot more thoughts on thinking about the place of std::hash in the STL and might have come to other conclusion. I would be very interested in hearing about these different opinions. My best wish it that it might at least trigger some interesting discussions.
