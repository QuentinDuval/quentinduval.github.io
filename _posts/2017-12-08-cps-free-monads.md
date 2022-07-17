---
layout: post
title: "Continuation passing style Free Monads and direct style Free Monads"
description: ""
excerpt: "Generalized Algebraic Data Types gives us the power to develop type-safe Free Monads, without having to rely on continuation passing style."
categories: [functional-programming]
tags: [Functional-Programming, Haskell]
use_math: true
---

_[Generalized Algebraic Data Types](https://en.wikibooks.org/wiki/Haskell/GADT) gives us the power to develop type-safe Free Monads, without having to rely on continuation passing style when using simple [Algebraic Data Types](https://en.wikipedia.org/wiki/Algebraic_data_type)._

In today’s post, we will revisit the first Embedded Domain Specific Language (EDSL) example of our previous [Free Monad tutorial]({% link _posts/2017-11-13-free-monads-intro.md %}) post. We will look at its Abstract Syntax Tree (AST) again, and show its similarities to the [continuation passing style](https://en.wikipedia.org/wiki/Continuation-passing_style) (CPS) of programming.

We will then explore another way we can define an AST for the same language, only this time in direct style, the opposite of continuation passing style, using [Generalized Algebraic Data Types](https://en.wikibooks.org/wiki/Haskell/GADT).

We will talk about the advantages of defining an AST in direct style, and conclude by giving a small recipe we can use to almost automatically create a Free Monad from a set of instructions we want to support in a Domain Specific Language (DSL).

## Quick summary of the last episode

This post is a follow up of our previous [Free Monad tutorial post]({% link _posts/2017-11-13-free-monads-intro.md %}) in which we first introduced the basics of Free Monads and interpreters, before discussing more advanced examples where Free Monads are especially interesting.

We will start with a summary of the last episode, just enough for our purpose here. If you read the previous post or are familiar enough with Free Monads, you should skip this section.

### Free Monads (in short)

Free Monads are special kinds of Monads which produce an Abstract Syntax Tree (AST) upon evaluation, a data structure for a recipe to execute. They do not perform side-effects directly, they only emit instructions. Nothing really happens until an interpreter reads these instructions to interact with the world.

As shown in our [previous post]({% link _posts/2017-11-13-free-monads-intro.md %}), it gives us a great flexibility in terms of evaluation of the AST. We can switch back-ends, and write **several interpreters** to produce different effects out of the same AST (recipe).

We also illustrated the great power it offers us in terms of consulting, **transforming** or even **combining** recipes before evaluating them with an interpreter.

### Free Monads in 3 steps

We previously showed in 3 steps how to decouple the recipe for an `IO` computation, asking the user for a password, from its execution.

```haskell
password : Nat -> (String -> Bool) -> IO Bool
password maxAttempt valid = do
  putStrLn "Password:"     — Ask the user for a password
  attempt <- getLine       — Read the password entered by the user
  if valid attempt         — In case of valid password:
  — More code
```

Our first step was to identify and isolate the side-effects needed to express this computation. We identified two of them: reading a string from and writing a string to the standard output. We then created an AST containing of the instructions needed to support the side-effect we identified.

Here is the AST we developped for our IOSpec Domain Specific Language (DSL):

```haskell
data IOSpec a
  = ReadLine (String -> IOSpec a)
  | PrintLine String (IOSpec a)
  | Pure a
```

* `ReadLine` represents the instruction of reading a line. It contains a continuation which represents the set of instructions to execute after getting the string from the console.
* `Writeline` represents the instruction of writing a line. It contains the string to print and the set of instructions to execute after the printing is done.
* `Pure` is the marker for the end of the computation. It carries inside the result of the overall computation.

The second and third steps were respectively to create some factory functions and to make our `IOSpec` AST a Monad. These two steps completed the construction of our Free Monad, making it easy to emit an `IOSpec` AST by writing imperative looking code that uses the do-notation.

### Intepreting our Free Monad

For a sequence of instructions emitted by a Free Monad to be useful, we need an interpreter to translate the AST instructions into a lower level language, such as `IO`.

We illustrated this by developing a simple interpreter which transforms a computation expressed in the `IOSpec` Free Monad into a IO computation to execute it:

```haskell
interpret : IOSpec r -> IO r
interpret (Pure a) = pure a
interpret (ReadLine cont) = do
  l <- getLine
  interpret (cont l)
interpret (WriteLine s next) = do
  putStrLn s
  interpret next
```

This was a short summary of the first part of our [previous post on Free Monads]({% link _posts/2017-11-13-free-monads-intro.md %}). You can have a look at the rest of the post if you are interested in transformation on Free Monads AST.

## Continuation Passing Style AST

Let us now explore in what aspects the `IOSpec` Abstract Syntax Tree we just defined resembles the continuation passing style (CPS) of programming.

### What is Continuation Passing Style?

The [Continuation passing style](https://en.wikipedia.org/wiki/Continuation-passing_style) of programming consists in explicitly passing to each function a continuation which is a function that represents the rest of the computation to perform.

In direct style, a function returns its result of type a directly. It just returns the value. This is the default style in most programming languages:

```haskell
parens : String -> String
parens s = "(" ++ s ++ ")"

— In the REPL:
parens "Hello"
> "(Hello)"
```

In continuation passing style, a function takes an additional function – named a continuation – from `a` to an arbitrary type `r` as parameter, and calls it on its result of type `a`:

```haskell
parensCps : String -> (String -> r) -> r
parensCps s cont = cont ("(" ++ s ++ ")")
```

We will not go in details over the advantages of switching to continuation passing style (which include for example turning stack recursion into heap recursion). Instead, we will look at the overall pattern, abstract it, and see how this abstraction relates to our `IOSpec` AST.

### Abstracting Continuation Passing Style

Looking back at the definition of our parensCps continuation passing style function, we can see a pattern. The return type of the function follows the pattern `(a -> r) -> r`, where:

* `a` is the result type the function would have in direct style
* `a -> r` is the type of the continuation
* `r` is the result type of the continuation

In the case of `parensCps`, a matches the string type (the return type of `parens` written in direct style), while our continuation has the type `String -> a`:

```haskell
parensCps : String -> (String -> r) -> r
parensCps s cont = cont ("(" ++ s ++ ")")
```

We can capture this pattern through the type (or type alias in our case) `Cont r a`, and adapt the type of `parensCps` accordingly:

```haskell
— Type alias in Idris (a function on types)
Cont : Type -> Type -> Type
Cont r a = (a -> r) -> r

parensCps : String -> Cont r String
parensCps s cont = cont ("(" ++ s ++ ")")
```

In the code above, the type `Cont r a` should be understood as “the type of a continuation passing style function returning a `a`“ with `r` being the return type of the continuation. Alternatively, we could see `Cont r a` as the type of a function that leads us one step closer to a final result of type `r`, where:

* `a` is an intermediary step on the way to compute the final result of type `r`
* The continuation `a -> r` represents the rest of the computation to get to the final result

This continuation pattern is the one we will try to find in `IOSpec`.

### IOSpec continuation passing style

Looking back at the AST of the `IOSpec` Free Monad, let us see in what aspects it resembles the continuation passing style of programming. We will start with the definition of the AST:

```haskell
data IOSpec a
  = ReadLine (String -> IOSpec a)
  | PrintLine String (IOSpec a)
  | Pure a
```

We can  refactor this Algebraic Data Type (ADT) to use the more general syntax of [Generalized Algebraic Data Type](https://wiki.haskell.org/Generalised_algebraic_datatype) (GADTs), yielding the following equivalent definition for `IOSpec`:

```haskell
data IOSpec : Type -> Type where
  ReadLine  : (String -> IOSpec a) -> IOSpec a
  PrintLine : String -> (() -> IOSpec a) -> IOSpec a
  Pure      : a -> IOSpec a
```

Now, if you look at this carefully, you should start to see that:

* `ReadLine` resembles a function written in continuation passing style, returning a string
* `WriteLine` resembles a function written in continuation passing style, returning an empty tuple

In fact, we can rewrite our AST using our `Cont` type alias:

```haskell
data IOSpec : Type -> Type where
  ReadLine  : Cont (IOSpec a) String
  PrintLine : String -> Cont (IOSpec a) ()
  Pure      : a -> IOSpec a
```

_Note: We cheated a bit. For `PrintLine`, we replaced an `IOSpec` computation by a function taking an empty tuple and returning an `IOSpec` computation. These things are however semantically equivalent._

## Toward direct style AST using GADTs

Continuation passing style is arguably a bit complex to understand. Thankfully, we can very often transform a CPS function into a function written in direct style, usually much easier to understand. The same way, we can often refactor an AST written in continuation passing style to an AST written in direct style.

### Using GADT for a direct style AST

[Generalized Algebraic Data](https://wiki.haskell.org/Generalised_algebraic_datatype) Types allow us to write down the type of the constructors. Each of our constructors is free to return a different type. We can use this to define an AST made of instructions that are not of the same type.

This allows us to do something rather nifty. We can turn any function into an instruction of our AST by lifting the name of the function into a type constructor:

```haskell
— Turning the following functions:
writeLine : String -> IOSpec ()
readLine : IOSpec String

— Into data constructors:
data IOSpec : Type -> Type where
  WriteLine : String -> IOSpec ()
  ReadLine : IOSpec String
```

As we did for our continuation passing style AST, we can hide the construction of the AST using factories functions. These functions are much easier to write than previously: they just map instructions back to their corresponding functions.

```haskell
readLine : IOSpec String
readLine = ReadLine

prnLine : String -> IOSpec ()
prnLine s = WriteLine s
```

Starting from this base, we can build a new Free Monad that encode the same language as our previous AST, by simply adding the support for the Monad interface.

### Making it a Monad

To transform our AST into a Free Monad, we can almost mechanically add two additional constructors for our AST, Pure and Bind, which mimic the Monad interface:

```haskell
data IOSpec : Type -> Type where
  Pure : a -> IOSpec a
  Bind : IOSpec a -> (a -> IOSpec b) -> IOSpec b
  WriteLine : String -> IOSpec ()
  ReadLine : IOSpec String
```

Now, the Monad implementation of `IOSpec` almost writes itself. It only consists in calling the appropriate constructors, and is much simpler to write than the Monad instance needed for our continuation passing style AST. It is [provided here](https://gist.github.com/deque-blog/7ba19ef39a12e55d190d5db45d2b0735) as reference.

### A specification for the interpreters

A value of type `IOSpec` a represents a computation that returns a result of type a. Our GADT-based AST therefore encodes (and enforces!) the following:

* Interpreting the `ReadLine` instruction should return a `String`
* Interpreting the `PrintLine` instruction should return empty tuple

Somehow, this same information was also encoded, although a bit differently, in our initial continuation passing style AST:

```haskell
data IOSpec a
  = ReadLine (String -> IOSpec a)
  | PrintLine String (IOSpec a)
  — Missing constructors
```

It however required to know a bit about continuations passing style in order to recognize the pattern, and notice it was effectively equivalent to:

```haskell
data IOSpec : Type -> Type where
  ReadLine  : Cont (IOSpec a) String
  PrintLine : String -> Cont (IOSpec a) ()
  — Missing constructors
```

Our direct style AST is therefore at the same time easier to derive (just lift function to constructors) and easier to read as a **specification for an interpreter** to translate it into lower-level code.

### Simpler interpretation

Because our AST now encodes the specification of its interpretation in a more direct way, writing an interpreter is also much easier. The translation in a lower level language is almost straightforward:

* `Pure` translates to `pure` in the lower level language
* `Bind` translates into `>>=` between two interpretations
* The other instructions translate to their corresponding lower level instructions

```haskell
interpret : IOSpec r -> IO r
interpret (Pure a) = pure a
interpret (Bind a f) = interpret a >>= interpret . f
interpret ReadLine = getLine
interpret (WriteLine s) = putStrLn s
```

Each instruction but Bind is translated into a non-compound expression. This is simpler than with our continuation passing style AST, whose [interpretation](https://gist.github.com/deque-blog/8cad537eb1b92f9a204c6c5954018946) involved both translating instructions and chaining their result with their corresponding continuation.

### Bind factors the continuation

The astute reader might have recognized some continuation passing style creeping in our GADT-based AST, through the `Bind` constructor.

```haskell
data IOSpec : Type -> Type where
  Bind : IOSpec a -> (a -> IOSpec b) -> IOSpec b

— Or, using our Cont type alias	
data IOSpec : Type -> Type where
  Bind : IOSpec a -> Cont (IOSpec b) a
```

This is no accident. We can see the same continuation in the very definition of a Monad (*):

```haskell
class Monad m where
  (>>=) : m a -> (a -> m b) -> m b

— Or, using our Cont type alias
class Monad m where
  (>>=) : m a -> Cont (m b) a
```

What our GADT-based AST does is in fact to factorize the continuation which used to appear in the ReadLine and PrintLine constructors, into a single `Bind` single constructor. This is partly what makes the AST simpler to understand and the interpreters easier to write.

(*) Basically, we could see the `bind` operator of a Monad as mapping between a Monadic value and a continuation. In addition, continuations form a Monad too. This connection between Monads and continuations is expressed in more details in [The mother of all Monads](http://blog.sigfpe.com/2008/12/mother-of-all-monads.html).

## The end of our journey

There are many different ways to write the same code. Similarly, there is more than one way we can write our Free Monad. In particular, there are two broad categories of Abstract Syntax Trees: continuation passing style ASTs, and direct style ASTs.

### Continuation Passing Style

While continuation passing style is rarely encountered in the wild world of programming, it is relatively more present in the context of Free Monads.

The reason is that constructors of an algebraic data type must all return the same type, such as `AST a` where `a` stands for the result type of the overall computation. Therefore, encoding the return type of each instruction is not possible unless:

* We rely on something like continuation passing style
* We enable Haskell type extensions such as GADTs

### Switching to Direct Style

Turning up the GADTs extensions gives us the choice not to dive into the tricky world of continuation passing style (which is arguably harder to understand), and get some additional benefits such as:

* An almost automatic way to build an AST from the instructions we want it to support
* An AST that acts as easily to understand specification for its interpreters
* Simpler interpreters, factory functions, and Monad instances

These benefits makes it much easier to introduce the concept of Free Monad to newcomers to Haskell or Idris, refer to [this article as an example]({% link _posts/2017-07-06-hexagonal-architecture-free-monad.md %}), and make GADT-based AST a great alternative to continuation passing style ASTs.

