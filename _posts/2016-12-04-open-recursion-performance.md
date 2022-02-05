---
layout: post
title: "Open Recursion (Performance)"
description: ""
categories: [functional-programming]
tags: [Functional-Programming, C++]
---

In the two previous posts, we used open recursion to provide efficient solutions to dynamic programming problems, keeping the following aspects decoupled:

* The recurrence relation: the solution to our problem
* The memoization strategy: indexing into a vector
* The order in we compute the sub-solutions

As a reminder, here is the open recurrence relation we based our examples upon:

```
template<typename Recur>
long long bst_count(Recur recur, int n)
{
   if (n <= 1) return 1;

   long long sub_counts = 0;
   for (int i = 0; i < n; ++i)
      sub_counts += recur(i) * recur(n-1-i);
   return sub_counts;
}
```

However, we observed that the memoization strategy was still depending on the sub-solutions space needed to compute the solution. The vector we used had to be of the right size.

Today, we will explore a general memoization strategy that does not require much knowledge about the content of the recurrence formula to memoize. In particular, it does not require prior knowledge of the sub-solution space.

### Using hash map as generic memoization strategy

The simplest way to be decoupled from the sub-solutions space we need to compute, is to use a hash map. This hash map:

* Will start empty
* Will be filled gradually with new sub-solutions
* Will be searched at each recursion

Here is a C++ implementation of this strategy, based on open recursion

```
static long long lazy_map(std::unordered_map<int, long long>& memo, int n)
{
   auto& computed = memo[n];
   if (computed) return computed;
   auto recur = [&](int k) { return lazy_map(memo, k); };
   return computed = bst_count(recur, n);
}

long long bst_count_map(int n)
{
   std::unordered_map<int, long long> memo;
   return lazy_map(memo, n);
}
```

With this implementation the decoupling is nearly total. The only requirement left is that the sub-solutions form a DAG, a basic assumption when dealing with Dynamic Programming problems.

In particular, the memoization strategy does not have to care about:

* The order of evaluation of the sub-problems
* The number of sub-problems to compute

But by doing so, we went from a tightly packed vector to a more memory consuming and less local data structure. Surely there is a price to pay.

### Performance implications

Rich Hickey said that nowadays “Programmers know the benefits of everything and the tradeoffs of nothing”. So we must ask ourselves: did our successive decoupling refinement had an impact on performance?

_Note: The following measures were performed on the code we show previously. And we know this result is wrong (long long do overflow for N=1000): they only serve as approximation, only to have some idea of what we are talking about._

I measured the performance of each of these solutions, for 100 runs with N=1000, and got the following results:

* Eager vector (coupling with evaluation order): 0.047s
* Lazy vector (no coupling with evaluation order): 0.308s
* Hash map (no coupling with sub-problem range): 2.306s

So there is approximately an order of magnitude performance drop at each refinement we did.

What about an implementation without open recursion, in which everything is coupled together? I tried benching the following C++ implementation:

```
long long bst_count_classic(int n)
{
   if (n <= 1) return 1;
    
   std::vector<long long> memo(n+1, 0);
   memo[0] = memo[1] = 1;
   for (int k = 2; k <= n; ++k)
      for (int i = 0; i < k; ++i)
         memo[k] += memo[i] * memo[k-i-1];
   return memo.back();
}
```

Sure enough this implementation is faster than our “Eager vector” implementation: it takes 0.034s instead of 0.047s (a 28% improvement).

(All of my benchmarks were performed on Clang with -02)

### No benefits without trade-offs

Open-recursion is a nice technique that allows to provide quick and easily readable solution to dynamic programming problems. It is also a nice cheat: I used it repeatedly in Hackerrank or TopCoder to get a fast working implementation of my algorithms.

But even in a language like C++, coined as “zero cost abstraction language” by its creator, there are tradeoffs to almost each abstraction mechanism we use.

Does it means that the different techniques we saw are useless? On the contrary, it means that each of these techniques has its specific spot. Here is a quick rule of thumb for C++ regarding when to use each of the techniques we walked through:

* Use hash map when there is no simple indexing strategy (or when it leads to sparse vectors)
* Use hash map when knowing which sub-solutions to compute in advance is hard (here is a nice example)
* Otherwise, use the open recursion technique with vector-based memoization (eager if the order of evaluation is known)
* Finally, if all sub-problems solutions do not need to stay in memory (example of fibonacci), or if you absolutely need the ultimate performance, write an custom algorithm

These rules are obviously not covering all cases, but they served me well until now.

### Conclusion

This post concludes the series focused on open recursion technique, which I think is a really nice cheat for solving dynamic programming problems.

Here are some nice resources you may be interested about:

* [An absolute gem from Edward Kmett](http://stackoverflow.com/questions/3208258/memoization-in-haskell)
* [Haskell Data.Function.Memoize package](https://hackage.haskell.org/package/memoize-0.8.1/docs/Data-Function-Memoize.html)
* [Clojure memoize function](https://clojuredocs.org/clojure.core/memoize)

