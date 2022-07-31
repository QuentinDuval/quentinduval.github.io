---
layout: post
title: "Implementing Clojure-like Transducers in Idris: advanced transducers"
description: ""
excerpt: "Part 2 of our implementation of transducers in Idris. This time we tackle stateful transducers and transducers with completion steps."
categories: [functional-programming]
tags: [Functional-Programming, Idris]
use_math: true
---

In the [previous post](({% link _posts/2017-07-28-idris-transducers.md %})), we started the implementation of a small transducer library for Idris. We went over the main concepts, defined types for each of them, and implemented *reduce* and *transduce*. We ended the post by building basic transducers such as *mapping* or *filtering*.

In today’s post, we will continue where we left off. We will first look at a helper function we used in our last post to build stateless transducers. We will then build some more complex transducers, ones with states and featuring completion steps. We will also build some more helper to define transducers more concisely and correctly.

*Disclaimer: This post builds on our previous post on transducers. You should read it if you want to be able to follow this one.*

## Quick summary of the last episode

Before going further in this post, let us review together the main concepts behind transducers. If any of these concepts is not clear to you, you can read my previous post on the subject.

### Vocabulary for reduction recipes

Let us start with the basic vocabulary:

* A **reduction** is a process that consumes a series of values to build a final result
* An **accumulator** is the name we give to this result under construction
* A **step function** represent the content of an iteration of such a reduction
* A **reducer** is a recipe for a reduction, with a step function, some state and a completion step
* A **transducer** is a transformation on a reducer, used to add functionality to a recipe

### Composing recipes

Reducers are recipes for algorithms operating on source of values (such as but not limited to [Foldable](https://wiki.haskell.org/Foldable_and_Traversable)), and transducers are the means to transform smaller recipes into bigger ones.

To build a complete recipe, we can compose transducers together the same way we compose function. Each transducers adds a new ingredient to the recipe:

```haskell
filtering odd   -- Added behavior: keep only odd numbers
mapping square  -- Added behavior: square each element

-- When composed: keep only odd numbers and square them afterwards
filtering odd . mapping square
```

In the example above, we compose two transducers together to build a more complex recipe out of simpler ones.

### Executing a recipe

Having defined how to build recipes, we now need to show how to run them. The functions `reduce` and `transduce` allow us to execute a recipe on a collection of values (any `Foldable`).

For instance, we can compute the sum of the squares of odd numbers on a list, by first defining a transducer pipe-line, and then running it on a list of numbers:

```haskell
-- with transduce
transduce (filtering odd . mapping square) (+) 0 [1..10]
> 165

-- with reduce
reduce (filtering odd . mapping square $ terminal (+)) 0 [1..10]
> 165
```

Now that we are done with the reminder, let us move on!

## Back to stateless transducers

So we gave some example by implementing some stateless transducers. To do so, we used an helper function named `statelessTransducer`.

In this section, we will explore how this helper function is implemented. The goal is to explain how to easily create your own transducers, by showing how this helper function works behind the curtain.

### Back to the mapping transducer

The first transducer we defined was named `mapping`. It takes a function from `a` to `b` and applies it to each element that goes through the transducer:

* It receives elements of type `a` from the previous transducer in the pipe-line
* It forwards elements of type `b` to the next transducer in the pipe-line

In the implementation of `mapping`, `next` represent the next step function of the pipe-line, `acc` represents the accumulator, and `a` represent the input flowing through the pipe-line.

```haskell
mapping : (a -> b) -> Transducer acc s s a b
mapping fn = statelessTransducer $
  \next, acc, a => next acc (fn a)
```

Now, if we go back to the definition of a transducer, it is a transformation of a reducer into another reducer. So a transducer can act upon the three components that together forms the reducer it transforms: a state, a step function and a completion function.

The `statelessTransducer` helper function allows to only deal with the transformation of the step function, when states and completion functions do not need any transformation. This is why the implementation of our mapping function is so tiny.

We will now look behind the curtain and see how `statelessTransducer` works.

### Defining of statelessTransducer

By definition, a stateless transducer does not need to modify the state of the pipe-line it transforms, since it does not track any additional state.

We can further reason that the only operation that makes sense for a stateless transducer is to transform the *step* function. Indeed, the completion function is mainly there to deal with the remaining piece of state upon termination of the data source. If there is no state to flush, there is no completion step needed either.

This leads us to define `statelessTransducer` as a factory function that creates a transducer from the specification of a transformation of a step function:

```haskell
statelessTransducer : (Step s acc b -> Step s acc a) -> Transducer acc s s a b
statelessTransducer onStep next =
  MkReducer
    (state next)              -- Do not add to state of next transducer
    (onStep (runStep next))   -- Modify the step of the next transducer
    (complete next)           -- Keep the completion of the next transducer
```

This function creates a transducer that takes elements of type a and outputs elements of type b from:

* The definition of `Step` receiving elements of type `a` (*)
* Built from of `Step` receiving elements of type `b`

_(*) Keep in mind that transducers compose to the left, while data flows to the right. A transducer that maps elements from a to b therefore transforms a step function on b‘s to a step function on a‘s_

### Using statelessTransducer

The `statelessTransducer` function takes as argument a function from a step function to a new step function. Yet, our mapping function is defined in terms of a function taking three arguments:

```haskell
mapping : (a -> b) -> Transducer acc s s a b
mapping fn = statelessTransducer $
  \next, acc, a => next acc (fn a)
```

To understand this, we need just need to substitute the second occurrence of the type alias `Step` inside the prototype of the function `onStep` given to `statelessTransducer`:

```haskell
-- Type of the `onStep` argument provided to `statelessTransducer`
onStep : Step s acc b -> Step s acc a

-- Type of `Step` (quick reminder)
Step state acc x = acc -> x -> State state (Status acc)

-- Type of `onStep` (substituded)
onStep : Step s acc b -> acc -> a -> State s (Status acc)
```

Put differently, the type of `onStep` correspond to a step function that takes an additional parameter as first argument: the next step function in the pipe-line. Hence the three arguments to `mapping`.

_Note: pragmatically, there is not much a stateless transducer can do with this next function except calling it once (mapping), several time (concat mapping) or not calling it (filtering). It still allows to build interesting transformations as the next section will show._

### More examples

Using this helper function, we can define stateless transducers in a very few lines of code.

For instance, we can define `takingWhile`, which consumes a source of data as long as its element satisfy a given predicate. The definition is short and quite close to the specification of what the transformation does:

```haskell
takingWhile : (a -> Bool) -> Transducer acc s s a a
takingWhile p = statelessTransducer $
  \next, acc, a => do
    if p a                  -- If the element a satisfies the predicate
      then next acc a       -- * Then forwards it to the rest of the pipe-line
      else pure (Done acc)  -- * Else trigger an early termination
```

Here is a quick example of how it can be used to consume a stream of values lazily:

```haskell
transduce (takingWhile (<= 10)) (+) 0 [1..100]
> 55

foldl (+) 0 [1..10]
> 55
```

_Note: interestingly, while we can define `takingWhile` as a stateless transducer, you cannot define `droppingWhile` (which ignores elements until one that satisfies the predicate is encountered) without tracking some state._

## Toward stateful transducers

We defined transducers as arbitrary transformation on reducers, taking as input a reducer and returning as output a potentially completely different reducer.

In practice, the transformation is not as arbitrary as this may sound: a lot of transformation do not make sense. We can therefore offer some helper functions that will help us define more easily a reduced set of transducer that we will call **sound transducers**.

In this section, we will define these helpers function to use them in the next section. If you need examples to make sense of these functions, try to switch back and forth between this section and the next one.

### Sound & unsound transducers

A **sound transducer** is one that performs only additive changes to the reducer it acts upon. It might add some state but will not remove or modify any existing one. It might add a completion step but will not remove or impact existing ones. In addition to this, a sound transducer will not mess with other’s state.

We will therefore define a sound transducer as a transducer that:

* Only adds to the state of the pipe-line
* Only impacts its own state in its step function
* Does not remove or mess around with other’s completion step

Our next step is to define a factory function `makeTransducer`, which will enforce these constraints, and by restraining the problem will also offer a cleaner API.

### Sound state transducers

A sound transducer is only able to add to the state of the pipe-line. We can therefore create a transducer by only asking for the initial value of this additional piece of state. We can then combine the additional piece of state (added by the transducer) to the pipe-line state, by grouping them under a pair.

Practically, our factory for transducer will ask for a state `s’` and will return a transducer with a state `(s’, s)`. This lets us define the first pieces of our `makeTransducer` prototype:

```haskell
makeTransducer :
  s' -- Additional initial piece of state
  -> ??? -- Step transformation
  -> ??? -- Completion transformation
  -> Transducer acc s (s', s) a b -- Output: a transducer adding state s'
```

There is no loss of generality there, it is simply a matter of convention.

### Sound step function transformations

A sound transducer will not mess with the completion functions of the pipe-line it transforms, nor to the state of this pipe-line. So a step function transformation only needs to have access to the state its operates on, and to the next step function in the pipe-line:

```haskell
makeTransducer :
  s' -- Additional initial piece of state
  -> (Step s acc b -> Step s (s', acc) a) -- Step transformation
  -> ??? -- Completion transformation
  -> Transducer acc s (s', s) a b -- Output: a transducer adding state s'
```

To better understand the step transformation prototype, we will play the same game we played before and substitute the definition of `Step` in it:

```haskell
-- Type of step transformation
stepTf : Step s acc b -> Step s (s', acc) a

-- Type of `Step` (quick reminder)
Step state acc x = acc -> x -> State state (Status acc)

-- Type of step transformation (after substitution)
stepTf : Step s acc b -> (s', acc) -> a -> State s (Status (s', acc))
```

So basically, we specify a step function in terms of the previous state of the current state of the transducer, and return a new version of this state. The whole computation happens in the `State s` monad to be able to call the next step function.

### Sound completion transformation

A sound transducer will not mess with the completion functions of the pipe-line it transforms. It can only add a new completion function, whose goal is to handle any remaining state upon termination of the pipe-line.

The new completion function will need to have access to the next step function in the pipe-line. This allows to send some leftover elements to be processed before terminating (quite useful for transducers such as `chunksOf`). Because of this, this computation also has to happens in the `State s` monad:

```haskell
makeTransducer :
  s' -- Additional initial piece of state
  -> (Step s acc b -> Step s (s', acc) a) -- Step transformation
  -> (Step s acc b -> s' -> acc -> State s acc) -- Completion transformation
  -> Transducer acc s (s', s) a b -- Output: a transducer adding state s'
```

The prototype of the function `makeTransducer` is now complete. The full definition of the function is available at this [Gist GitHub](https://gist.github.com/deque-blog/02b4a43d6eed78f882cb5cb9dc724ae5). Most of it consists in plumbing to transform a `State s` computation into a `State (s’, s)` computation, and to handle the systematic call of the next completion function in the pipe-line.

### Now with the completion step

A large number of stateful transducers do not need to define a completion step. Their state does not need to be flushed upon the termination of the pipe-line.

We can therefore improve the API again by providing an additional `statefulTransducer` helper function, that looks like `makeTransducer` except that it does not require a completion function:

```haskell
statefulTransducer : s' -> (Step s acc b -> Step s (s', acc) a) -> Transducer acc s (s', s) a b
statefulTransducer initState stepTf = makeTransducer initState stepTf onComplete
  where
    onComplete next _ acc = pure acc
```

The implementation consists in a wrapper around `makeTransducer` that specifies a completion step that does nothing. It is not much, but it does clean the definition of some transducers as we will see.

## Example of stateful transducers

We built some helper function to make the construction of stateful transducers easier and safer. It is time to go through some example to show how to use them to build interesting transducers.

### Indexing elements

We will start by defining a simple but useful transducer, that associates to each element that goes through the pipe-line an associated index (a position) in the stream. We will call it `indexingFrom`.

```
Inputs:
['a', 'b', 'c']

Outputs of (indexFrom 1):
[(1, 'a'), (2, 'b'), (3, 'c')]
```

This transducer requires some piece of state, and integer, to track the next index to associate to an element. At each new element going through, it bumps this index by one.

The prototype of this transducer is therefore:

```haskell
indexingFrom : Int -> Transducer acc s (Int, s) a (Int, a)
indexingFrom startIndex = ???
```

* It adds an integer to the state `s` of the pipe-line
* It associates each `a` that goes through it with an integer

### Implementing IndexFrom

We can reason that `indexingFrom` does not need any completion step: the state does not need to be flushed at pipe-line termination. The implementation will therefore use our `statefulTransducer` function, which takes the initial state as first argument, and the step transformation as second argument.

The initial step is simply the index from which we want to start counting. The step function just sends a pair (index, element) to the `next` step function and increments the index value:

```haskell
indexingFrom : Int -> Transducer acc s (Int, s) a (Int, a)
indexingFrom startIndex = statefulTransducer startIndex stepImpl
  where
    stepImpl next (n, acc) a = withState (succ n) <$> next acc (n, a)
```

This function makes use of the `withState` helper function, provided in the library, to add the next piece of state to the accumulator returned by the function.

```haskell
withState : s -> Status acc -> Status (s, acc)
withState s = map (\acc => (s, acc))
```

We can play along with our new transducer to associate indexes to element at different stages of a filtering pipe-line.

```haskell
reverse $ into [] (filtering even . indexingFrom 0) [1..10]
> [(0, 2), (1, 4), (2, 6), (3, 8), (4, 10)] : List (Int, Integer)

reverse $ into [] (indexingFrom 0 . filtering (even . snd)) [1..10]
> [(1, 2), (3, 4), (5, 6), (7, 8), (9, 10)] : List (Int, Integer)
```

As we can see, the result is not the same depending on whether we add the indices before the filtering or after the filtering: before the filtering, some indices will get jumped over.

### Taking a fixed amount a values

We can combine state with the ability to early terminate from a computation. We will demonstrate it by defining the function `taking` that only consumes a fixed number of elements of an input stream (much like the function take would do).

This transducer needs to maintain a count down of elements to consume, and will stop the pipe-line when this counter reaches zero:

```haskell
taking : Nat -> Transducer acc s (Nat, s) a a
taking n = statefulTransducer n takeImpl
  where
    takeImpl next (Z, acc) a = pure (Done (Z, acc))
    takeImpl next (n, acc) a = withState (pred n) <$> next acc a
```

Again, we can play with this transducer and see how it interacts with the rest of a filtering pipe-line:

```haskell
transduce (taking 10) (+) 0 [1..100]
> 55 : Integer

transduce (filtering even . taking 10) (+) 0 [1..100]
> 110 : Integer

transduce (taking 10 . filtering even) (+) 0 [1..100]
> 30 : Integer
```

### And plenty more...

We can define plenty more transducers, all inspired from the their equivalent list-based algorithm. Here are some that you can find in the [transducer library](https://github.com/QuentinDuval/IdrisReducers):

* interspersing
* dropping
* droppingWhile
* deduplicating

If you are interested in having a look at their implementation, all these transducers are available in the Idris [Transducers.Algorithms](https://github.com/QuentinDuval/IdrisReducers/blob/master/src/Transducers/Algorithms.idr) module.

## Conclusion, and what's next?

In this post, we went over the creation of stateful transducers in Idris. We defined some helper functions that helped us defined sound transducers and illustrated how they could be used by building our first stateful transducers.

In the next post, we will see how to define stateful transducers that have completion step, such as `chunksOf` or `groupBy`. We will also look at some additional helper functions such as `into` and `conj` to deal with reductions assembling collections.

The library is available in this [GitHub repository](https://github.com/QuentinDuval/IdrisReducers). Any suggestions for improvements is obviously welcomed.

