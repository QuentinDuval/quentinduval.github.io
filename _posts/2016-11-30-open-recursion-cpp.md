---
layout: post
title: "Open Recursion (C++)"
description: ""
tags: [Functional-Programming, C++]
---


## Open Recursion (C++)

In the [previous post](2016-11-27-open-recursion-haskell.md), we explored how we could leverage open recursion to solve a dynamic programming problem, while keeping the following aspect decoupled:

* The recurrence relation: the solution to our problem
* The memoization strategy: indexing into a vector
* The order in we compute the sub-solutions

Today we will see how to apply this trick in a more mainstream language, C++.

### Open recursion in C++

Our first step will be to translate the recurrence formula into a naive C++ and very inefficient implementation of our solution to the [counting binary search trees problem](https://www.hackerrank.com/challenges/number-of-binary-search-tree):

```
long long bst_count(int n)
{
    if (n <= 1) return 1;
 
    long long sub_counts = 0;
    for (int i = 0; i < n; ++i)
        sub_counts += bst_count(i) * bst_count(n-1-i);
    return sub_counts;
}
```

Adding open recursion results in a pretty simple change for our C++ implementation:

* Adding a template parameter “recur” as first argument
* Replace each recursive call by a call to “recur”

Following this recipe leads to the following C++ implementation:

```
template<typename Recur>
long long bst_count(Recur recur, int n)
{
­    if (n <= 1) return 1;
 
    long long sub_counts = 0;
    for (int i = 0; i < n; ++i)
        sub_counts += recur(i) * recur(n-1-i);
    return sub_counts;
}
```

Similar to what we did in Haskell, we can get back back the naive algorithm by introducing a wrapper function handling the two step recursion:

```
long long best_count_naive(int n)
{
   auto recur = [](int k) { return best_count_naive(k); };
   return bst_count(recur, n);
}
```

This function still has the same terrible performance. It is time to improve our algorithm, by leveraging Dynamic Programming techniques.

### Adding memoization

You know the drill: we can now exploit this open recursion to insert some memoization in the middle in the recursion.

We will use the same memoization strategy we used previously: indexing into a vector to seek the results of our previously computed sub-solutions.

```
long long bst_count_memo(int n)
{
    std::vector<long long> memo_table(n+1);
    for (int i = 0; i <= n; ++i)
        memo_table[i] = bst_count([&](int k) { return memo_table[k]; }, i);
    return memo_table.back();
}
```

Even if we ignore the integer overflow issue (let us imagine we use a unbounded integer representation), there is still a big difference between this C++ implementation and the corresponding implementation in Haskell:

```
memoBstCount :: Int -> Integer
memoBstCount n = Vector.last memo
  where
    memo = Vector.generate (n+1) (bstCount (memo Vector.!))
```

In C++, the order of evaluation now has a direct impact on the correctness of our solution. Had we computed the sub-solutions in the wrong order, we would have got a completely erroneous solution.

So although we applied the same idea, we achieve less decoupling in C++ than in Haskell: the implementer of the memoization must still care about the insides of the recurrence formula. Can we do better?

### Adding laziness

To get rid of the coupling to the order of evaluation, we will take some of the ideas from Haskell, and introduce laziness into our solution. We just need a helper function that will:

* Index into the vector to check for a previously computed value
* Return this value if it could be found
* Perform the computation otherwise, and store it in the vector

This gives us to the following C++ implementation:

```
static long long lazy_impl(std::vector<long long>& memo, int n)
{
    if (memo[n]) return memo[n];
    auto recur = [&](int k) { return lazy_impl(memo, k); };
    return memo[n] = bst_count(recur, n);
}
 
long long bst_count_lazy(int n)
{
    std::vector<long long> memo(n+1);
    return lazy_impl(memo, n);
}
```

You might wonder if we could have used a lambda instead of introducing a function. Unfortunately no, since a lambda has no name, it cannot refer to itself in its body.

The following would indeed not compile:

```
long long bst_count_lazy(int n)
{
    std::vector<long long> memo(n+1);
    auto lazy_impl = [&](int n) {
        if (memo[n]) return memo[n];
        auto recur = [&](int k) { return lazy_impl(memo, k); };
        return memo[n] = bst_count(recur, n);
    }
    return lazy_impl(memo, n);
}
```

We now have the same level of decoupling as with our Haskell solution.

### Conclusion

We proved that the open recursion trick known from the Haskell world can be very easily brought to C++. Using it, we can decouple the memoization strategy of a Dynamic Programming solution from the recurrence relation.

Although it might seem that we achieved a total decoupling between the recurrence relation and the memoization part, the implementer of the memoization strategy still needs to care about which sub-solutions to compute.

Can we do something about it? We will look into this subject in the next post
