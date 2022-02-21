---
layout: post
title: "Folding merge sort (Haskell)"
description: ""
excerpt: "Implementing a merge sort in terms of a fold (left to right scan) is possible. Here is how in Haskell."
categories: [functional-programming]
tags: [Functional-Programming, Haskell]
use_math: true
---

Back in September 2016, I was lucky enough to attend a talk from [Ben Deane](https://twitter.com/ben_deane) entitled [std::accumulate: Exploring an Algorithmic Empire](https://www.youtube.com/watch?v=B6twozNPUoA). In this talk, he presents the unknown possibilities of `std::accumulate`, an algorithm also known as fold or reduce depending on the language.

In his talk, he announced he had implemented almost all the algorithms of the STL based on `std::accumulate` (41st minute in the [video](https://www.youtube.com/watch?v=B6twozNPUoA)), including `std:stable_sort`, which was a bit surprising to me.

So I searched [his blog](http://www.elbeno.com/blog/) and could not find a hit on how he did it. So I decided to write this post to describe the solution I found. I hope you will find it as elegant as I do.

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

We are done! Let us try to implement this in Haskell.

## Haskell implementation

The implementation in Haskell follows pretty closely the description we gave of the solution in the previous paragraph.

The main difference lies in the use of an auxiliary integer to keep the length of the list. Had we not do that, our solution would have been much less efficient: computing the length of a list in Haskell is linear in the size of the list.

```haskell
foldMergeSort :: (Ord a) => [a] -> [a]
foldMergeSort =
  foldl1 (flip merge) . map snd . foldl addToCounter []
  where
    addToCounter counter x = propagate ((1::Int,[x]) : counter)
    propagate [] = []
    propagate [x] = [x]
    propagate counter@(x:y:xs) -- x arrived last => combine on right
      | fst x == fst y = propagate ((fst x + fst y, merge (snd y) (snd x)) : xs)
      | otherwise      = counter
```

* The head of the counter list is the lowest level bit
* The function `propagate` implements the carry propagation
* The function `addToCounter` add a “bit” in the counter
* The `map snd` allows to get rid of the integer holding the list length
* The `foldl1 (flip merge)` implements the collapse of the counter

To ensure our merge sort is stable, ranges that came last must be passed as right parameters to the `merge` function. This is the reason why we collapse our counter with `flip merge` instead of `merge` (which would also type check but result in unstable sort).

This implementation assumes the existence of a `merge` function that combines two sorted lists into one bigger sorted list. You can implement it as follows (it requires a bit of care to make it stable):

```haskell
merge :: (Ord a) => [a] -> [a] -> [a]
merge xs [] = xs
merge [] ys = ys
merge l@(x:xs) r@(y:ys)
  | y < x     = y : merge l ys -- Keeps it stable
  | otherwise = x : merge xs r
```

As `map` (`std::transform` in the STL) can be itself implemented in terms of fold (`std::accumulate` in the STL), this is one way to implement a “out of place” stable sort in Haskell based only on folds and composition of functions.

## Generalizing to Monoids

Looking back at the code of our solution, we can observe that this algorithm would work for any Monoid. So we can make our algorithm more general, and fold any Monoid following a tree like structure.

```haskell
foldMonoidTree :: (Monoid a) => [a] -> a
foldMonoidTree =
  foldl1 (flip (<>)) . map snd . foldl addToCounter []
  where
    addToCounter counter x = propagate ((1::Int,x) : counter)
    propagate [] = []
    propagate [x] = [x]
    propagate counter@(x:y:xs) -- x arrived last => combine on right
      | fst x == fst y = propagate ((fst x + fst y, snd y <> snd x) : xs)
      | otherwise      = counter
```

To get our stable merge sort back, we can create a type for sorted lists, and make it an instance of Monoid. Here is one possible implementation:

```haskell
newtype SortedList a = SortedList { unSortedList :: [a] }

instance (Ord a) =>  Monoid (SortedList a) where
  mempty = SortedList []
  mappend (SortedList a) (SortedList b) = SortedList $ merge a b

foldMergeSort :: (Ord a) => [a] -> [a]
foldMergeSort = unSortedList . foldMonoidTree . map (\x -> SortedList [x])
```

## Conclusion and what's next

I was quite astonished how elegant the solution was and how simple the resulting code. But somehow, you should feel unsatisfied with the performance.

I cheated. I did not use C++ as the original presentation did. And this implementation of stable_sort creates a new list. Haskell did not give me the choice.

So next time we will have a look at an implementation in C++ that stable sorts the container provided as input.
