---
layout: post
title: "IdrisPipes: a library for composable and effectful stream processing in Idris"
description: ""
excerpt: "Our goal here is to provide Idris with a library for composable and effectful production, transformation and consumption of streams of data."
categories: [functional-programming]
tags: [Functional-Programming, Haskell]
use_math: true
---

Let's implement a pipe library for Idris, inspired by the great [Haskell Pipes](https://hackage.haskell.org/package/pipes) and [Haskell Conduit](https://hackage.haskell.org/package/conduit) libraries. Our goal is to provide Idris with a library for composable and effectful production, transformation and consumption of streams of data.

In short, our [IdrisPipes](https://github.com/QuentinDuval/IdrisPipes) library is an Idris package that aims at providing the means to write:

* Effectful programs, with side effects such as IO
* Over a stream of data, potentially infinite
* Efficiently, by streaming data and controlling memory consumption
* In a composable way, allowing problem decomposition and reuse

In this first post, we will illustrate with examples the motivations behind this library, and give a quick overview of the features it offers. Future posts will be dedicated to explain how it works in more details, how to build your own pipes easily, as well as describing some of the difficulties I had implementing it in Idris.

## Motivation & Examples

To explain the goal of this library, we will first illustrate how to use it on a small example of effectful code, and see how we can use [IdrisPipes](https://github.com/QuentinDuval/IdrisPipes) to improve on it.

### A world without pipes

We will start with a small effectful function, which we will later enrich with additional requirements. This small function is named “echo”:

* It acquires some strings from the standard inputs
* Echo them back in the standard output
* And stops upon receiving the string “quit” as input

Here is one possible implementation of this need, without using pipes:

```haskell
echo : IO ()
echo = do
  putStr "in> "                 -- Write "in> " in standard output
  l <- getLine                  -- Read the standard input
  when (l /= "quit") $ do       -- Upon encountering something else than "quit"
    putStrLn ("out> " ++ l)     -- Echo the string in standard output
    echo                        -- Recurse to loop again
```

Let us run it into the REPL. Upon each line entered after the “in” prompt, it echos back the same string in the “out” prompt, and the “quit” string correctly terminates the execution of the function:

```haskell
in> toto   -- user input
out> toto  -- output
in> titi
out> titi
in> quit
```

Simple enough. Let us now enrich our “echo” function with an additional requirement.

### Adding concerns

Our `echo` function should not repeat the same output twice in a row anymore. In other words, if we write two times in a row the same string, we want it to ignore the second occurrence:

```haskell
in> toto   -- user input
out> toto  -- output
in> toto   -- same entry, do not repeat
in> quit
```

We can easily adapt our previous `echo` function to support this new requirement, with just a few additional lines of code:

```haskell
echo : IO ()
echo = loop (const True) where
  loop : (String -> Bool) -> IO ()
  loop isDifferent = do           -- `isDifferent` refers to the previous string
    putStr "in> "
    l <- getLine
    when (l /= "quit") $ do
      when (isDifferent l) $      -- If the string is different from the previous
        putStrLn ("out> " ++ l)   -- Echo the string in standard output
      loop (/= l)                 -- Track the last read string
```

While the number of lines did not increase that much, the overall complexity of the function did.

The loop has to maintain a state that allows us to identify whether the new value is different from the previous one. This state is visible from the whole loop, and comes with additional conditional branching. In short, the additional concern is coupled with the rest of the `echo` function.

## Refactoring with IdrisPipes

We will now see how to get rid of this coupling by refactoring the function using [IdrisPipes](https://github.com/QuentinDuval/IdrisPipes). We will start with our initial “echo” function and then add the additional requirement of deduplicating entries afterwards.

### Thinking in streams

[IdrisPipes](https://github.com/QuentinDuval/IdrisPipes) is based on the idea of building small pipes dedicated to one specific task to perform on a stream of data (like production, transformation or consumption). Each pipe is only concerned with dealing with awaiting for new inputs, transforming this piece of data, and yielding resulting outputs.

These pipes can be connected together to form more complex pipes: the outputs of one pipes become the inputs of the next pipe. Ultimately, these pipes are assembled as a complete pipeline, an `Effect` in the vocabulary of [IdrisPipes](https://github.com/QuentinDuval/IdrisPipes), a recipe for stream processing.

The key to our refactoring of `echo` to use [IdrisPipes](https://github.com/QuentinDuval/IdrisPipes) is thus to observe in what way the function corresponds to the processing of a stream of data.

### Refactoring

We can easily view our `echo` function as processing a stream of lines read from the standard input and flowing down to standard output. We can therefore use [IdrisPipes](https://github.com/QuentinDuval/IdrisPipes) to model the recipe matching it:

```haskell
stdinLn "in> "
  .| takingWhile (/= "quit")
  .| mapping ("out> " ++)
  .| stdoutLn
```

This recipe is made of several pipes, connected together with the `.|` pipe operator:

* `stdinLn` feeds the pipe with strings read from the standard input
* `takingWhile` forwards elements passing through it, interrupting the pipeline if it sees the string “quit”
* `mapping` transforms elements passing through it (here to prepend a prompt to each string)
* `stdoutLn` writes in standard output any string which goes through it

Assembled together, these pipes build an `Effect` that corresponds to the recipe of echoing back strings as long as the string read is not “quit”.

### Running a recipe

The `Effect` we just wrote is but the description for a recipe for processing a stream. This allow us to decouple the specification of the transformation from its execution (and therefore potentially offer several ways to execute a recipe).

To run the recipe in a sequential fashion, and start flowing some data (from left to right) inside it, we use the `runEffect` function:

```haskell
echo : IO ()
echo = runEffect $              -- Run the pipeline which
  stdinLn "in> "                -- * read the standard input
    .| takingWhile (/= "quit")  -- * as long as the user does not write "quit"
    .| mapping ("out> " ++)     -- * preprend "out>" to the string read
    .| stdoutLn                 -- * then write the string in standard output
```

The code above is equivalent to our initial “echo” function:

```haskell
echo : IO ()
echo = do
  putStr "in> "                 -- Write "in> " in standard output
  l <- getLine                  -- Read the standard input
  when (l /= "quit") $ do       -- Upon encountering something else than "quit"
    putStrLn ("out> " ++ l)     -- Echo the string in standard output
    echo                        -- Recurse to loop again
```

The big difference is that each concerns are kept separated: for instance, the acquisition of inputs (`stdinLn`) and the stop condition (`takingWhile`) are fully decoupled. As we will see, this will help us a lot when comes to some additional requirements in our program.

### Using pipes to decouple concerns

In our initial implement of “echo”, adding the requirement of deduplicating the input strings led to coupled concerns. Using [IdrisPipes](https://github.com/QuentinDuval/IdrisPipes), we can instead isolate this additional concern into a specific pipe, named deduplicating.

This pipe will **await** values, keep track of the last value read, and only **yield** those that are different from the previous one. The full code of deduplicating is shown below as reference (*). We will later explain how to build our own pipelines in more details:

```haskell
deduplicating : (Eq a, Monad m) => Pipe a m a
deduplicating = recur (the (a -> Bool) (const True)) where
  recur isDifferent =
    awaitOne $ \x => do
      when (isDifferent x) (yield x)
      recur (/= x)
```

This deduplicating logic is isolated, decoupled from the rest of the pipeline, and can now be used else where in the code. In particular, we can add this new pipe in our pipeline to support the new requirement, and no other parts of the pipeline need to know about it:

```haskell
echo : IO ()
echo = runEffect $
  stdinLn "in> "                
    .| takingWhile (/= "quit")  
    .| deduplicating            -- Remove consecutive repeating calls
    .| mapping ("out> " ++)     
    .| stdoutLn  
```

And we are done. We managed to add the requirement of deduplicated entries as a separate concern, decoupled from the rest of the pipeline.

_(*) This Pipe is also available in the library in [Pipes.Prelude](https://github.com/QuentinDuval/IdrisPipes/blob/master/src/Pipes/Prelude.idr), along with plenty other standard pipes. You should consider taking a look at the rich set of already available pipes before implementing your own._

## A Quick tour of IdrisPipes support for stream processing

Let us now do a quick tour of the design and model followed by [IdrisPipes](https://github.com/QuentinDuval/IdrisPipes) and the features it provides to build effectful streaming programs.

### A few concepts - One composition operator

A pipeline of data transformation typically consists of three kind of elements: a source that streams pieces of data, intermediary pipes to transform that data, and a sink that consumes the data to produce a single result. Each of these elements can produce some side effects as well.

[IdrisPipes](https://github.com/QuentinDuval/IdrisPipes) defines a type for each of these kinds of pipes. These types are listed below, with `m` standing for the Monad matching the desired side effects:

* A `Pipe i m o` awaits values of type `i` and yields values of type `o`
* A `Source m o` yields values of type `o`, and cannot await any values
* A `Sink i m r` awaits values of type `i` to build a result of type `m r`
* A `Effect m r` cannot yield or await, and produces `a m r` when executed

In short, an `Effect` represents the complete pipeline, starting with a `Source`, ending with a `Sink` and possibly made of intermediary `Pipes`, and which can be run using `runEffect`:

```haskell
echo : IO ()
echo = runEffect $
  stdinLn "in> "                -- Source
    .| takingWhile (/= "quit")  -- Pipe
    .| deduplicating            -- Pipe
    .| mapping ("out> " ++)     -- Pipe
    .| stdoutLn                 -- Sink
```

All these types compose together nicely using a single `.|` operator, to produce new pipes that automatically match their associated semantic model:

* A `Source m a` followed by a `Pipe a m b` is a `Source m b`
* A `Pipe a m b` followed by a `Sink b m r` is a `Sink a m r`
* A `Pipe a m b` followed by a `Pipe b m c` is a `Pipe a m c`
* A `Source m a` followed by a `Sink a m r` is an `Effect m r`

All these types are in fact type synonyms for the core data type `PipeM i o r1 m r2`, with some of the type variables replaced by `Void`. We will explore in future posts what this type represents in more details.

_Note: If you look at the API, you will also notice additional types such as `SourceM`, `SinkM`, `PipeM`. These types are more general than `Source`, `Sink` and `Pipe` and give full access to the return type of the pipe, something useful in particular cases. We will explore this in future posts._

### Two main primitives: yield and await

Defining a new pipe is rather easy and relies upon just a few ingredients: `yield` (to send a value downstream) and `await` (to receive a value from upstream). To complete the picture, we can use tail recursion to build stateful pipes, and the Monad `m` to produce side-effects.

For instance, we can define an infinite `Source` that `yield` the repeated application of a given function (known as the function `iterate` in Idris) rather easily:

```haskell
iterating : (Monad m) => (a -> a) -> a -> Source m a
iterating f = recur where
  recur a = do
    yield a     -- Yield the value downstream
    recur (f a) -- Recurse with (f a) as next seed
```

Similarly, we can easily define a `Sink` that folds over its inputs to build a result. It just awaits for inputs and combine them with an initial accumulator. Upon `await` returning `Nothing`, the stream does not have any more values to stream, and we can return the result:

```haskell
fold : (Monad m) => (a -> b -> b) -> b -> Sink a m b
fold f = recur where
  recur acc = do
    ma <- await           -- await for more inputs
    case ma of                   
      Just x => do        -- in case there is a value to process
        acc' <- f x acc   -- * combine it the current accumulator
        recur acc'        -- * and recurse with the new value
      Nothing => pure acc -- otherwise, return the result
```

We can plug these two pipes together to sum the integers from 0 to 4 as follows:

```haskell
runPure $                 -- Run the pipeline in the Identity Monad (pure)
  iterating (+1) 0        -- Generate the integers from 0 to infinity
    .| takingWhile (< 5)  -- Take the ones below 5
    .| fold (+) 0         -- And sum them
```

_Note: these pipes are already available in [IdrisPipes](https://github.com/QuentinDuval/IdrisPipes), although the actual implementation is more concise and general. The library also offer a function “awaitOr” that allows to capture the return type of the previous pipe._

### Return value and early termination

As demonstrated by the great [Haskell Pipes](https://hackage.haskell.org/package/pipes) and [Haskell Conduit](https://hackage.haskell.org/package/conduit) libraries, there is a huge design space for a pipe library such as [IdrisPipes](https://github.com/QuentinDuval/IdrisPipes).

In particular, there are many design choice we can make regarding the support of early termination inside the core of the library, or the support of return values at different stages of the pipeline.

Following the reading of this [great post on the flows of Haskell Pipes and Conduit](https://www.yesodweb.com/blog/2013/10/core-flaw-pipes-conduit) by [Michael Snoyman](https://twitter.com/snoyberg), I decided to follow the path of [Conduit](https://hackage.haskell.org/package/conduit) and integrate early termination inside the core of [IdrisPipes](https://github.com/QuentinDuval/IdrisPipes).

This way, you can define a `Sink` such as fold, or a pipe such as `groupBy`, which would not have been possible without the support for early termination, while still allowing for the pipes to return some values.

### Pull based stream model

[IdrisPipes](https://github.com/QuentinDuval/IdrisPipes) follows a pull-based streaming model:

* The `Sink` starts processing first
* Each pipe gives control to the pipe before it when it awaits a value
* Each pipe immediately releases control to the pipe after it when it yields a value

This allows keeping the memory consumption of the pipeline under control. You can tweak it, and exchange memory for speed in some cases, by implementing pipes that group individual pieces of data into chunks. In fact, an implementation of this pipe is available in [Pipes.Prelude](https://github.com/QuentinDuval/IdrisPipes/blob/master/src/Pipes/Prelude.idr), named `chunking`:

```haskell
chunking : (Monad m) => (n: Nat) -> {auto prf: GTE n 0} -> Pipe a m (List a)
chunking = ?hole -- Defined in Pipes.Prelude
```

One direct consequence of this pull-based streaming model is that the source might not be totally consumed. The pipeline will only consume as much as the source as needed.

It also means that you can use [IdrisPipes](https://github.com/QuentinDuval/IdrisPipes) to perform pure **lazy computation** (using `runPure`), which you can find useful as Idris is strict by default (there are other solutions available for you to do this though).

### Large collection of built-in algorithms

In order to avoid you having to re-invent the wheel, [IdrisPipes](https://github.com/QuentinDuval/IdrisPipes) also comes with a large collection of already existing pipes in [Pipes.Prelude](https://github.com/QuentinDuval/IdrisPipes/blob/master/src/Pipes/Prelude.idr), which will only grow as time goes by.

Here is a non-exhaustive list of existing pipes you can use:

* `mapping f` maps each element of the pipe with `f`
* `filtering p` only forward elements satisfying `p`
* `groupingBy p` build chunks of elements comparable by `p`
* `splittingBy p` splits in chunk at elements satisfying `p`
* `tracing f` runs an effectful function `f` on each element

The library also comes with some helper functions which help you building your own pipes. For instance, some helpers will take care of early termination and automatic forwarding of the return value of the previous pipe. In particular, the following function might prove useful to you:

* `awaitForever` helps building stateless pipes
* `awaitOne` helps building stateful pipes
* `each` helps yielding several values
* `idP` gives you an identity pipe

These helpers are available and documented in [Pipes.Core](https://github.com/QuentinDuval/IdrisPipes/blob/master/src/Pipes/Core.idr).

### Missing features

[IdrisPipes](https://github.com/QuentinDuval/IdrisPipes) does not yet support leftovers, or the prompt release of resources, as [Haskell pipes-safe](https://hackage.haskell.org/package/pipes-safe) and [Haskell conduit](https://hackage.haskell.org/package/conduit) allows you to do. These features might be integrated into the library in the coming months.

## Conclusion and what's next?

In this post, we went over the motivations and goals behind [IdrisPipes](https://github.com/QuentinDuval/IdrisPipes) as well as a quick overview of the package and the features it offers.

In future posts, we will zoom into the implementation details of the package to explain how it works, and discuss some of the design choices and some possible alternatives.

You can have a look at the full package in this [GitHub](https://github.com/QuentinDuval/IdrisPipes) repo.

