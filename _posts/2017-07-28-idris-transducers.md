---
layout: post
title: "Implementing Clojure-like Transducers in Idris: definitions & concepts"
description: ""
excerpt: "Let's implement a small transducer-like library in Idris, for efficient and composable algorithmic transformations."
categories: [functional-programming]
tags: [Functional-Programming, Idris]
use_math: true
---

The [1.0.0 of Idris](https://www.idris-lang.org/idris-1-0-released/) has been released just a few months back. We will continue our series on post on Idris, by implementing a small transducer library (the Idris package is available in this [GitHub repo](https://github.com/QuentinDuval/IdrisReducers)).

Our goal in this post will be to provide Idris with a library efficient composable algorithmic transformations. This is something we quickly feel the need to when playing with Idris, due to the strict (non-lazy) nature of the language.

## Motivation

Let us first start by discussing why transducers are an interesting library to introduce in Idris.

### Lazy vs Strict

As mentioned in the post listing [10 differences between Haskell and Idris](({% link _posts/2017-06-14-idris-improvements.md %})), Idris is strict by default, while Haskell is lazy by default. Whether or not laziness is a good default is a common subject of discussion in the Haskell community.

On the bad side, laziness is often complained about for its tendency to introduce non obvious memory leaks or other performance defects in Haskell code. On the good side, laziness allows to separate processes that produce data, from the one transforming or consuming it (*).

_(*) John Hughes writes in lot more details about the benefits of laziness in terms of software modularity in its great paper [Why functional programming matters](https://www.cs.kent.ac.uk/people/staff/dat/miranda/whyfp90.pdf)._

### Composing algorighms

Thanks to laziness, we can easily compose `map`, `filter` and `folds` together in Haskell and not worry to much about intermediary list creation. But this is not the case in Idris.

The following two lines of Idris code will not execute at the same speed: `map (+1)` will need to first realize the whole intermediary list before take extracts only the 10 first elements:

```haskell
take 10 $ map (+1) [1..20]   -- Faster
take 10 $ map (+1) [1..1000] -- Slower
```

In Idris, we need to implement ways to emulate laziness in order to get back the efficient composition of algorithm which is native in Haskell.

### Why transducers?

Transducers are one way to implement efficient composition of algorithms. Transducers comes from the Clojure world and have been built with efficiency and reuse in mind:

* They eliminate intermediary sequence or containers when composing algorithms
* They decouple the transformation from the data source and destination

This makes transducers a great tool to build generic pipe-lines of algorithmic transformation, reusable in quite different contexts, and which can be composed very efficiently.

## Step functions & Reducing functions

We will start with some definitions and define their associated types. The vocabulary is directly inspired from Clojure, with some minor twists. Defining this vocabulary will guide us through the main concepts of pipelines of data transformation.

### Step functions

A stateless **step function** is a function whose purpose is to be used in a left `fold` (also known as `reduce` in Clojure or `std::accumulate` in C++).

It combines an element coming from a data source (a container for instance) with some intermediary result, to compute the next intermediary result. Put differently, a step function represents a task to be performed in one iteration of a pure loop (without states and side-effects).

* We call **accumulator** (or acc) the result being computed
* We call **element** (or elem) an element of the data source to read

We can define a type alias that will help us formalise this concept in the code:

```haskell
StatelessStep : (acc, elem: Type) -> Type     -- List of type parameters for the alias
StatelessStep acc elem = acc -> elem -> acc   -- Define the alias to the step function
```

For instance, the type alias `StatelessStep Int String` represents a step function that consumes strings to produce a result of type integer. It could be a function that sums the length of strings (in a fold):

```haskell
sumLength : StatelessStep Int String
sumLength res str = res + fromNat (length str)

foldl sumLength 0 ["abc", "de", "", "fghij"]
> 10
```

### Adding state

A **stateful step function** is a step function that needs some additional state to do its job properly. For instance, splitting a collection into chunks of equal size requires some kind of state: elements have to be kept in store until the completion of a chunk.

Because we are in a pure language, we will avoid relying on side-effects to track the state of our step function. Instead, we will rely on the `State` Monad:

```haskell
Step : (state, acc, elem: Type) -> Type
Step state acc elem = acc -> elem -> State state acc
```

For instance, the type alias `Step Bool Int String` represents a step function that consumes strings to produce a result of type integer, and maintains a state of type boolean. It could be a function that sums the length of strings, ignoring one of every two strings it encounters:

```haskell
sumLengthOfEveryOddStrings : Step Bool Int String
sumLengthOfEveryOddStrings totalLength str = do
  doSum <- get                        -- Get the state (should we sum the length?)
  modify not                          -- Modify the state (alternate True and False)
  pure $ if doSum
    then sumLength totalLength str    -- Sum the length in case of True
    else totalLength                  -- Otherwise return the length unchanged
```

### Adding early termination

We will also want to express early termination for algorithms such as take that do not need to consume the whole data source. To do so, we will enhance our stateful step function to return a result decorated by a status:

```haskell
data Status a = Done a | Continue a

Step : (state, acc, elem: Type) -> Type
Step state acc elem = acc -> elem -> State state (Status acc)
```

* The step can return `Done` to indicate early termination of the loop
* The step can otherwise return `Continue` to proceed with the rest of the loop

For instance, the type alias `Step () Int String` represents a step function that consumes strings to produce a result of type integer, and maintains no state. It could be a function that sums the length of strings, stopping when the sum exceeds a given value:

```haskell
sumLengthUntil : Int -> Step () Int String
sumLengthUntil maxValue totalLength str =
  pure $ if totalLength <= maxValue
    then Continue (sumLength totalLength str)
    else Done totalLength
```

_Note: this ability to return early combines especially well with state (for instance to build functions such as take or drop). But we just saw that it also makes sense without state._

### Reducer (or reducing function)

A **reducer** (or **reducing function**) is what we get when we bundle a stateful step function with its state. The following Idris code creates a type `Reducer` that contains a field state and two functions `runStep` and `complete`:

```haskell
record Reducer st acc elem where  -- Defines a reducer in terms of
  constructor MkReducer           
  state : st                      -- * A piece of state
  runStep : Step st acc elem      -- * A (stateful) step function
  complete : st -> acc -> acc     -- * A completion function
```

The `runStep` function is doing the heavy lifting. It takes as input the current state, the current value of the accumulator, and an element. It returns the a new accumulator and the state to use for the next iteration. It represents the content of a loop.

The `complete` function is there to deal with the remaining piece of state. It represent the termination of the loop, the last thing to perform once the source of data is consumed. It allows to discharge remaining pieces of state at termination.

### Example of reducer

Going back to our previous example, we can package our `sumLengthOfEveryOddStrings` step function (which sums the length of strings, ignoring one of every two strings it receives as input) with its state (a boolean).

Having packaged the stateful step with its state, we get a reducer that is self sufficient. It can be plugged into reduce to execute it on an input collection:

```haskell
let reducer = MkReducer True sumLengthOfEveryOddStrings (const id)

reduce reducer 0 ["abc", "de", "", "fg", "hij"]
> 6
```

We have not introduced the `reduce` function yet, but you can see it as a generalised `fold` which manage both state and early termination.

### Transducer

A **transducer** is a transformation of reducers. It takes as input a reducer, and returns another reducer. The new reducer includes additional functionality. In the process of adding new functionality, a transducer can both:

* Change the type of the state (_s1_ to _s2_), usually to track some more state
* Change the type of the element (_elem1_ to _elem2_) being accumulated

The type of the accumulator, on the other hand, cannot be modified (the result we expect from a loop is fixed). This leads to the following type alias for transducers:

```haskell
Transducer : (acc, s1, s2, elem2, elem1: Type) -> Type
Transducer acc s1 s2 elem2 elem1 =
  Reducer s1 acc elem1      -- Input reducing function
  -> Reducer s2 acc elem2   -- Output reducing function
```

For instance, the type alias `Transducer Int () Bool String Char` represents a transducer that transforms:

* A stateless reducer accumulating strings into a result of type integer
* To a reducer accumulating characters into result of type an integer
* And adding a boolean state in the output reducer

### Transducer composition

Because transducers are just functions from one reducer to another reducer, they can be composed together just like normal function can. Composing transducers lets us:

* Gradually add features to pipe-line of data transformation
* Avoid realizing intermediary containers: we only compose recipes

For instance, we can compose a transducer than adds a filtering behaviour on a reducer to a transducer that maps each element to its square:

```haskell
filtering odd   -- Added behavior: keep only odd numbers
mapping square  -- Added behavior: square each element

-- When composed: keep only odd numbers and square them afterwards
filtering odd . mapping square
```

Because of this, we can see transducers as pipe-lines, each each step is responsible for its own transformation, and forwards resulting elements to the next transducer in the line.

_Note: You may have noticed that the type alias for transducer had `elem1` and `elem2` reversed. This is because transducers compose from right to left (the direction of composition), but the data flows from left to right (the direction of pipeline)._

## Running the loop

Reducing functions provides us with recipes to consume a stream of value and summarize it as a single result. Transducers allows us to build such recipes out of smaller recipes. But to make these recipes useful, we need to way to execute them.

In this section, we will dive into the implementation of `reduce` and `transduce`, two function that will allow us to execute the recipe on some data. You can skip this section if you are not interested in the implementation, and jump to the next one for example of transducers.

### Reduce

Our library provides a function named `reduce` that we used already in this post. It executes a provided reducing function (a recipe), given:

* An initial value for the accumulator
* A data source we can fold over to retrieve a stream of values

It will consume elements of the data source, executing the recipe at each iteration, until the stream is totally consumed or until the recipe asks for early termination. Then it will call the completion function on the result before returning it.

```haskell
reduce : (Foldable t) => Reducer st acc elem -> acc -> t elem -> acc
reduce step acc =
  uncurry (complete step)                 -- 4) Call the completion on the result
    . (\(acc, s) => (s, unStatus acc))    -- 3) Remove the wrapping status
    . (flip runState (state step))        -- 2) Remove the state monad
    . runSteps (runStep step) acc         -- 1) Consume as many value as needed
```

This implementation relies on `runSteps` to run the loops until either the stream of values is totally consumed, or the reducing function asks for early termination (returning `Done`).

### Running the steps

The implementation of `runSteps` relies on [Continuation Passing Style](https://en.wikipedia.org/wiki/Continuation-passing_style) to support the early termination.

```haskell
runSteps : (Foldable t) => Step st acc elem -> acc -> t elem -> State st (Status acc)
runSteps step acc elems =
  foldr stepImpl (pure . id) elems (Continue acc) -- Using continuation passing style
  where
    stepImpl _ nextIteration (Done acc) = pure (Done acc)
    stepImpl e nextIteration (Continue acc) = step acc e >>= nextIteration
```

* `foldr` builds a chain of functions, one for each element of the source
* Each of these functions represents one iteration of the loop
* Each iteration and is provided with the next one: `nextIteration`
* If `nextIteration` is not called, the next iteration is not run and so the loop stops

### Transduce

The second key function named `transduce` is only a thin but useful layer above reduce. It reduces (pun intended) verbosity for the most common use cases.

Indeed, a lot of pipe-lines of data transformation ends up with a stateless step function (like a sum, a concatenation, etc.). In such cases, using transduce instead of reduce removes a bit of noise. Here is an example in which we sum the square of odd numbers:

```haskell
-- with transduce
transduce (filtering odd . mapping square) (+) 0 [1..10]
> 165

-- with reduce
reduce (filtering odd . mapping square $ terminal (+)) 0 [1..10]
> 165
```

One other great advantage is that it allows to use the word transduce in our code, making your Idris code almost as cool as Clojure code.

## Building our own transducer

It is time to build our very own transducers. We will keep it simple in this post, and focus on stateless transducers. We will also use a helper function `statelessTransducer` (available in the library) to abstract away some of the details of the construction of a transducer.

The goal is to build an intuition for those who are foreign to the concept. The next post will deal with stateful transducer and unveil the details behind the helper functions.

### Mapping

Mapping consists in using a function from **a** to **b** to transform an input reducer operating on element of type **b** to a reducer operating on elements of type **a** (we see the continuation passing style formulation here).

It takes elements of type **a** and sends elements of type **b** to the next element of the pipe-line. In the code below, `next` represents the step function of the next transducers in the pipe-line, while `fn` represents the mapping function from a to b.

* By applying the function `fn` on an element of type `a`, we get an element of type `b`
* We can give this element to next, which consumes elements of type `b`

```haskell
||| Given a mapper of type (a -> b), it transforms:
||| - a step function taking values of type `b`
||| - into one taking values of type `a`
mapping : (a -> b) -> Transducer acc s s a b
mapping fn = statelessTransducer $
  \next, acc, a => next acc (fn a)
```

Here is an example in which we sum the length of a bunch of strings, multiplying each length by 2 along the way:

```haskell
transduce (mapping length . mapping (*2)) (+) 0 ["abc", "", "de"]
> 10
```

Thanks to transducers and ability to compose in terms of recipe, we get the following equivalence by construction: `mapping f . mapping g = mapping (g . f)`. Note that `f` and `g` are inverted compared to the usual `Functor` law because of the left to right flow of data.

### Filtering

Filtering consists in transforming an input reducer operating on element of type a to a reducer operating on only those elements a that satisfy the a given predicate.

In the code below, `next` represents the step function of the next transducers in the pipe-line, while `pf` represents the predicate function from `a` to `Bool`.

By calling `next` on only these elements that satisfy the predicate, we effectively filter these elements from the input source data: the rest of the pipeline will not see them.

```haskell
||| Given a predicate of type (a -> Bool), it transforms:
||| - a step function taking values of type `a`
||| - into one taking the values of type `a` that fulfills the predicate
filtering : (a -> Bool) -> Transducer acc s s a a
filtering pf = statelessTransducer $
  \next, acc, a =>
    if pf a
      then next acc a
      else pure (Continue acc)
```

Here is an example in which we sum the length of strings that do not start with the letter ‘a’:

```haskell
transduce (filtering (\s => not (startsWith 'a' s)) . mapping length) (+) 0 ["abc", "de", "fgh"]
> 5
```

### Concat Mapping

Concat mapping is just like mapping, to the exception that the function provided to the `catMapping` function may return several `b` for one `a`.

In the code below, `next` represents the step function of the next transducers in the pipe-line, while `fn` represents the mapping function from a to a collection of b.

By calling next on all the the elements output by `fn`, `catMapping` takes elements of type a from and the left, and transmits elements of type b to pipe-line on the right. The main difference with mapping is that the number of `bs` is not necessarily the same as the number of `as`.

```haskell
||| Given a mapper of type (a -> Foldable b), it transforms:
||| - a step function taking values of type `b`
||| - into one taking values of type `a`
catMapping : (Foldable t) => (a -> t b) -> Transducer acc s s a b
catMapping fn = statelessTransducer $
  \next, acc, a => runSteps next acc (fn a)
```

Here is an example in which we sum the length of strings, counting each of them twice:

```haskell
transduce (catMapping (replicate 2) . mapping length) (+) 0 ["abc", "", "de"]
> 10
```

## Conclusion, and what’s next

In this post, we went over basics to define transducers in Idris. We defined the concepts and our main types, detailed the main function `reduce` and `transduce`, and built very simple transducers.

In the next post, we will see how to define more interesting transducers, such as `take`, `takeWhile`, `interspersing` or `groupingBy`. We will also look at some helper functions that helps building these transducers more easily.

The library is available in this [GitHub repository](https://github.com/QuentinDuval/IdrisReducers). Any suggestions for improvements is obviously welcomed.

