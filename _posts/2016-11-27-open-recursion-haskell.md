---
layout: post
title: "Open Recursion (Haskell)"
description: ""
tags: [Functional-Programming, Haskell]
---

## Open Recursion (Haskell)

There is a large class of problems that we can solve by nice mathematical recurrence relations.

The recurrence is usually simple to read, to reason about, and describes the solution with concision. But a naive translation of this recurrence into code will very often lead to a very inefficient algorithm.

When the sub-problems of the recurrence relation form a DAG, the usual trick is to use Dynamic Programming to speed up the computation. But it often results in a complete change of the code, hiding the recurrence, sometimes to the point we cannot recognize the problem anymore.

Thankfully, with open recursion, can have the best of both worlds!

### Counting Binary Search Trees

Before introducing the technique, let us first take a nice example whose solution can be described by a simple recurrence relation: [counting binary search trees](https://www.hackerrank.com/challenges/number-of-binary-search-tree).

We observe that we can cut any collection [X1, X2 .. Xn] in two parts, around one of its element Xi. Then, we can:

* Recursively count all BST on the left part [X1 .. Xi).
* Recursively count all BST on the right part (Xi .. Xn]
* Multiply these values to get the count of BST rooted in Xi.

As we have to do this for all i in [1..N], the recurrence relation becomes:

```
bstCount :: Int -> Integer
bstCount n
  | n <= 1 = 1
  | otherwise = sum [bstCount i * bstCount (n-1-i) | i <- [0..n-1]]
```

But this algorithm is terribly inefficient: we keep recomputing the same sub-solutions over an over again.

We know the next step, which is to memoize the solutions to the sub-problems. However, as our original goal was to do it without compromising the code readability, let us first introduce open recursion.

### Adding Open Recursion

Open recursion consists of avoiding direct recursion by adding an extra layer of indirection. It typically means transforming our recurrence relation to take a new parameter, a function that will be called instead of recurring.

By doing so, the recurrence formula looses its recursive nature. Here is how it would translate into Haskell:

```
bstCount :: (Int -> Integer) -> Int -> Integer
bstCount rec n
  | n <= 1 = 1
  | otherwise = sum [rec i * rec (n-1-i) | i <- [0..n-1]]
```

How can we get our recurrence relation back? By introducing a wrapper function that will give itself to the “bstCount” recurrence. Instead of having a direct recursion, we have a two-step recursion. This is best explained by example:

```
bstCountNaive :: Int -> Integer
bstCountNaive = bstCount bstCountNaive
```

By simple renaming, we can see that it can be expressed as: x = f x. So the naive recursive algorithm we had earlier is effectively the fixed point of the open recurrence relation. Which can be written in Haskell as:

```
import Data.Function(fix)
 
bstCountNaive :: Int -> Integer
bstCountNaive = fix bstCount
```

### Adding memoization

We can now exploit this open recursion to insert some memoization in the middle of the recursion. In our specific case, the sub-problems exhibit a simple structure:

* We can compute a vector of the results of each sub-problem 0..N
* Then recurring part consist in indexing into this vector in O(1)

So instead of triggering a recursion, we search the sub-solution result into our memoization vector, which translates into the following Haskell code:

```
memoBstCount :: Int -> Integer
memoBstCount n = Vector.last memo
  where
    memo = Vector.generate (n+1) (bstCount (memo Vector.!))
```

How can it even work? You are witnessing here the magic of Haskell: laziness helps us refer to the item we are computing inside its own computation.

Following this change, at each N we now have N-1 steps to perform, each taking constant time, thanks to memoization. This gives us a quadratic complexity (the result of the sum from 1 to N).

### Conclusion

Using open recursion in combination with Haskell laziness has effectively let us decoupled the following aspects:

The recurrence relation, solution to our problem
The memoization strategy, indexing into a vector
The order in which we compute these sub-solutions
As a result, we get all the benefits of having a simple recurrence relation, untainted by the implementation details required to get an efficient implementation.

In the next post, we will see how to apply this trick in a more mainstream language: C++.
