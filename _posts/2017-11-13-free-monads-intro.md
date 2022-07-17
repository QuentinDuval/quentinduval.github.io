---
layout: post
title: "Free Monads: from the basics to composable and effectful stream processing"
description: ""
excerpt: "Let's embark in a journey, from Free Monad basics to more advanced examples such as stream processing as done in IdrisPipes."
categories: [functional-programming]
tags: [Functional-Programming, Haskell]
use_math: true
---

In this post, we will embark in a journey from the basics of Free Monads and interpreters, to more advanced examples, up to how to implement the core of [IdrisPipes](https://github.com/QuentinDuval/IdrisPipes). Through this exercise, we will illustrate some aspects that makes the Free Monad interesting.

_This post does not require any previous knowledge of Free Monads. We will start simple, with a first simple example of Free Monad, and slowly introduce more advanced concepts, to finally dive into the internals of [IdrisPipes](https://github.com/QuentinDuval/IdrisPipes)_

## Free Monads - Prelude & Motivation

We will start with a quick recap on Monads, and a first introduction to what Free Monads are and what makes them special.

### Monads as environment of execution

In Haskell and Idris, Monads are a idiomatic way to establish a specific environment of execution for our code, by allowing us to modify the rules of the languages inside the Monad:

* Changing the rules of composition (the Monad itself)
* Changing the available primitives (selecting available effects)
* Allowing for more than one interpretation of a computation(*)

Free Monads allow us to get one step further, and allow us to transform, optimize or combine our computations by manipulating the instructions they are made of.

_(*) By using Monad typeclasses instead of concrete Monads. One implementation can interact with the production environment (a real database, etc.), while another can interact with test components (in-memory database, etc.), as described in our previous article about [Hexagonal Architecture]({% link _posts/2017-07-06-hexagonal-architecture-free-monad.md %})._

### Free Monads & AST transformations

Free Monads are special kinds of Monads which produce an **Abstract Syntax Tree** upon evaluation, a data structure for a recipe to execute. Instead of performing side-effects, they only emit instructions.

Nothing really happens until an interpreter reads these instructions to interact with the world (or further transform them until we reach a point where we interact with the world).

This offers us a great flexibility in terms of evaluating the Abstract Syntax Tree. The recipe is completely decoupled from the evaluation of the recipe: we can switch back-ends, writing several interpreters to produce different effects out of the same AST.

This also offer us a great flexibility in terms of consultation or modification of the Abstract Syntax Tree. We can modify our Abstract Syntax Tree, transform it, remove or add some instructions inside it, or even **combine several trees together** (which is key for building a pipe-like library as we will see).

## Free Monads - The Basics

To get a first feel for Free Monads, let's implement a small password guessing function and refactor it to use a Free Monad instead of the IO Monad directly.

_Note: The examples below are all in Idris, but are almost portable in Haskell as-is with some minor syntax differences. We use Idris here for convenience: it supports pretty printing lambdas, allowing us to see the AST produced more easily (see [10 things Idris improved over Haskell]({% link _posts/2017-06-14-idris-improvements.md %}))._

### A password guessing game

We will consider the following function, named `password`. It asks a user for a password in the console, and fails after reaching the maximal number of attempt allowed:

```haskell
password : Nat -> (String -> Bool) -> IO Bool
password maxAttempt valid = recur 1 where
  recur n = do
    putStrLn "Password:"     -- Ask the user for a password
    attempt <- getLine       -- Read the password entered by the user
    if valid attempt         -- In case of valid password:
      then do
        putStrLn ("Successful login after " ++ show n ++ " attempt(s).")
        pure True
      else do                -- In case of invalid passord:
        putStrLn ("Login failed: " ++ show n ++ " attempt(s).")
        if n < maxAttempt
          then recur (n + 1) -- * Try again with one less attempt
          else pure False    -- * Stop after reaching maximal number of attempt
```

We can run this function inside the REPL. Here is a trace of the console, in a scenario in which we successfully log in the system at the second attempt, with the password “Scotty”:

```text
Password:
Spock
Login failed: 1 attempt(s).
Password:
Scotty
Successful login after 2 attempt(s).
```

Now, this function is rather rigid and hardly testable. Our goal will be to transform it to use a Free Monad, in order to separate the recipe from the evaluation (and later on manipulate its AST).

### Defining the Free Monad AST

The first step in defining a Free Monad is to define the structure of its Abstract Syntax Tree. To do so, we need to identify a set of instructions which allows us to write a function such as password.

As for all Monads, all **pure computations are automatically supported** inside a Free Monad. The AST only needs to encode the instructions which corresponds to the additional desired effects. If we look at our function above, there are only two of them:

* Printing a message in the console
* Reading a line from the console

Here is one such AST, named `IOSpec`, which supports these two instructions:

```haskell
data IOSpec a
  = ReadLine (String -> IOSpec a)
  | PrintLine String (IOSpec a)
  | Pure a
```

* `ReadLine` represents the instruction of reading a line. It contains a continuation which represents the set of instructions to execute after getting the string from the console.
* `Writeline` represents the instruction of writing a line. It contains the string to print and the set of instructions to execute after the printing is done.
* `Pure` is the marker for the end of the computation. It carries inside the result of the overall computation.

### Examples

Below are some examples of `IOSpec` Abstract Syntax Tree and their corresponding semantic. This should help you better understand how the instructions of our AST work together:

```haskell
-- Read a line and return it
ReadLine (\x => Pure x)

-- Print a line "Hello" and return 5
WriteLine "Hello" (Pure 5)

-- Read a line and print it twice, and return 5
ReadLine (\x => WriteLine x (WriteLine x (Pure 5)))
```

Now, obviously, writing an AST by hand is not such a great experience. To make it feasible to write reasonable sized function producing such an AST, we will need two more ingredients:

* Functions to factorize the creation of such instructions
* A Monad instance to sequence the instructions: a Free Monad

### Emitting instructions

We will start with the first ingredient, i.e. functions to easily create fragments of our AST. We need two here, one for each of our basic instructions, printing a line and writing a line:

```haskell
readLine : IOSpec String
readLine = ReadLine (\s => Pure s)
 
prnLine : String -> IOSpec ()
prnLine s = WriteLine s (Pure ())
```

These two functions create `ReadLine` and `PrintLine` instructions whose respective continuations do the simplest thing that makes sense:

* `readLine` produces a computation that reads a line and returns it unchanged (the continuation wraps the acquired string inside `Pure`)
* `prnLine` produces a computation that prints a line and returns an empty tuple afterwards (the continuation returns the empty tuple wrapped in `Pure`)

### Free Monads - Chaining instructions

The second ingredient is to make it easy to combine two AST into a single one, sequencing the instructions of one AST after the other. For instance, in order to print the line returned by `readline`, we need a `PrintLine` instruction to be injected in the continuation of the `ReadLine` instruction:

```haskell
-- Reading a line (readLine)
ReadLine (\s => Pure s)

-- Printing a line (\x => prnLine x)
\x => WriteLine x (Pure ())

-- Chaining them together (printing the line read)
ReadLine (\x => WriteLine x (Pure ()))
```

We can implement a `Monad` instance for `IOSpec` that will do this sequencing of instruction for us. This will let us profit from the convenient do-notation of Haskell and Idris. The implementation is quite simple (*):

* It looks for the last instruction of the left operand (a `Pure` instruction)
* And chains to it a new set of instruction that depends on the value `Pure` contains

```haskell
implementation Monad IOSpec where
  (>>=) ma f = recur ma where
    recur (Pure a) = f a
    recur (ReadLine cont) = ReadLine (recur . cont)
    recur (WriteLine s next) = WriteLine s (recur next)
```

We can play in the REPL to get a better feel of how it works:

```haskell
-- Read a line and print it immediately
REPL> readLine >>= \x => prnLine x
ReadLine (\x => WriteLine x (Pure ()))

-- Read a line and print it with an exclamation mark
REPL> readLine >>= \x => let y = x ++ "!" in prnLine y
ReadLine (\x =>
  WriteLine (prim__concat x "!") (Pure ()))
```

_(*) This implementation is incomplete and requires a `Functor` and `Applicative` to be defined, as well as some trickeries with assert_total (for Idris). These are [provided here](https://gist.github.com/deque-blog/c0fb76fe69a91596715f3b08e08aaa1a) as reference. There are also some issues in terms of efficiency, which will not be addressed here._

### Refactoring to use our Free Monad

Equipped with our factories for instructions, and a Free Monad to combine them, we can refactor our password function to emit an IOSpec AST instead of performing `IO` actions directly.

This refactoring does not require much changes:

* We replace `IO` by `IOSpec`
* We replace `putStrLn` by `prnline`
* We replace `getLine` by `readLine`

And this is it. Here is the resulting function after transformation:

```haskell
password : Nat -> (String -> Bool) -> IOSpec Bool
password maxAttempt valid = recur (S Z) where
  recur n = do
    prnLine "Password:"  -- `prnLine` instead of `putStrLn`
    attempt <- readLine  -- `readLine` instead of `getLine`
    if valid attempt
      then do
        prnLine ("Successful login after " ++ show n ++ " attempt(s).")
        pure True
      else do
        prnLine ("Login failed: " ++ show n ++ " attempt(s).")
        if n < maxAttempt
          then recur (n + 1)
          else pure False
```

Now, and upon evaluating this function in the REPL, we get an enormous AST out of it, which correspond to the recipe of our `password` checking logic ([linked here](https://gist.github.com/deque-blog/d32977edcf4100a448260ff582cf6161#file-passwordeval-idr) as reference). The last remaining part is to write some interpreters for this AST.

### Interpreter

We refactored a function performing `IO` directly, into a function that emits an AST containing the assembly instructions that matches our level of abstraction. For this sequence of instructions to be useful, we need an interpreter to translate it into a lower level language (*).

To complete our example, we will write an `interpreter` that transforms a computation expressed in the `IOSpec` Free Monad into a `IO` computation to execute it. Our interpreter will simply:

* Transform each `ReadLine` into a `getLine` call
* Transform each `WriteLine` into a `putStrLn` call
* Replace continuations with a `IO` Monad bind operator (i.e. sequence them)

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

_(*) We can define other kind of interpreters as well, some to test our function and feed it with fake inputs, some to transform it into another data structure. Interpreters do not even have to evaluate the instructions per say, they can also simply transform the data structure._

## Free Monads - Transforming computations

Until know, nothing we did was specific to Free Monads. As mentioned before, it is perfectly possible to have several interpretations of the same computation with regular Monads (for instance, using typeclasses).

We will now look at some additional nice things that we can do with Free Monads, leading us one step closer to understanding how to build a pipe library in Haskell or Idris.

### A demonstration robot

Let us say that we want to do a nice live demonstration of our program, in which we have a bot that types characters in the console instead of us, feeding our `password` function with inputs (and giving us back control after the bot is out of instructions to execute).

We can represent the commands our bot is allowed to do as follows:

```haskell
data TestBotAction
  = Typing String
  | Thinking String

data TestBot = MkTestBot (List TestBotAction)
```

* `Typing` represents our bot typing a line in our console application
* `Thinking` represents our bot thinking out loud, halting and waiting for a notification to continue

Here is an example of instructions for our bot, in which it tries “Spock” as a first password, then thinks, then tries “Scotty”, before giving up on guessing the password:

```haskell
botSequence : TestBot
botSequence = MkTestBot
   [ Typing "Spock"
   , Thinking "Trying again... (press enter to continue)"
   , Typing "Scotty"
   , Thinking "Nore more ideas. I give up."]
```

We will now see how we can combine this sequence of instruction with our `password` function, yielding another sequence of instructions.

### Integrating our Bot Commands

Free Monads allow us to work on a computation by playing with its AST. So we can alter a computation expressed in the IOSpec monad (such as our password function) by inserting into it new IOSpec instructions corresponding our bot commands.

Here is one way to integrate our bot commands into an IOSpec computation:

* We will match a `Typing` bot command with a `ReadLine` instruction and transform it into
    * A `PrintLine` instruction to print the text entered by the bot
    * A fixed text value to feed into the continuation of `ReadLine`
* We will transform a `Thinking` bot command into a:
    * A `PrintLine` instruction to print the thoughts of the bot
    * A `ReadLine` instruction used to wait for notification (and discarding the value read)

The following pipe operator implements this transformation (understanding the whole code is not essential – you can focus on understanding the principle):

```haskell
(.|) : TestBot -> IOSpec r -> IOSpec r
(.|) (MkTestBot actions) prog = pull actions prog where
  mutual -- To define mutually recursive functions
 
    pull : List TestBotAction -> IOSpec r -> IOSpec r
    pull actions (Pure r) = Pure r
    pull actions (WriteLine s next) = do
      prnLine s          -- Print the line to display (such as "Password:")
      pull actions next  -- Keep on traversing the IOSpec computation
    pull actions (ReadLine cont) =
      push actions cont -- Give control to bot actions
 
    push : List TestBotAction -> (String -> IOSpec r )-> IOSpec r
    push [] cont = ReadLine cont
    push (Typing x::xs) cont = do
      prnLine x         -- Display the value typed by the bot
      pull xs (cont x)  -- Send `x` to the main program, and give it control back
    push (Thinking x::xs) cont = do
      prnLine x         -- Print the thoughts of our bot
      readLine          -- Wait for a line (notification to continue)
      push xs cont      -- Keep going
```

* `pull` traverses an `IOSpec` computation until it meets a `ReadLine` instruction, at which point it yields control to the push to insert some bot commands
* `push` integrates bot commands until it meets a `Typing` bot command, at which point it gives the bot’s string to the `IOSpec` computation and resumes it

The overall implementation produces a new `IOSpec` computation, based on the initial `IOSpec` computation, and transformed to replace `ReadLine` instructions by the values provided by the bot.

### Examples

Since combining our bot commands with an `IOSpec` computation results in a new `IOSpec` computation, we can use our previous `interpreter` to evaluate the resulting AST (*).

```haskell
runBot : IO Bool
runBot = interpret (botSequence .| password 3 (== "Scotty"))
view raw
```

Here is the console output that correspond to running this `runBot` function:

```text
Password:
Spock
Login failed: 1 attempt(s).
Password:
Trying again... (press enter to continue)

Scotty
Successful login after 2 attempt(s).
```

It is interesting to note that the bot waits for us to press enter (the empty line after “Trying again”. It would also have given us back control for the last password attempt would “Scotty” been invalid.

It is also interesting to look at the resulting AST after the transformation. You will note that it does not contain any traces left of `ReadLine` instructions: it has been transformed into a AST consisting only of print line instructions.

_(*) The closure property (staying in the same type) is a powerful way of abstraction, as we already talked about in our previous articles about [Monoids]({% link _posts/2017-09-13-monoids.md %})._

### Other ideas of transformation

Playing with our bot is just one example of transforming an AST generated by a Free Monad to enrich, transform or combine computations. There are plenty of other transformations we can do (with limits in terms of practicality) such as:

* Combining two computations, like injecting some instructions of one computation in another
* Compiling the AST of a high-level computation into a lower-level AST
* Aspect Oriented Programming, like adding logs around specific instructions automatically (*)

As we will see in the next section, implementing a pipe library such as [IdrisPipes](https://github.com/QuentinDuval/IdrisPipes) will involves making use of Free Monads and the transformation (1) listed above.

_(*) Note though, that some of these transformation might just be much easier to do by playing with Monad type-classes instead. But Free Monads can be abstracted behind Monad type-classes as well, and is in fact recommended in many articles such as [this one](http://degoes.net/articles/modern-fp) from [@jdegoes](https://twitter.com/jdegoes)._

## Using Free Monads - Example of IdrisPipes

We covered the basics of Free Monads, and showed how we could play with an AST to transform and enrich a computation. We are now ready to explore a more involved example: the implementation of a stream processing library such as [IdrisPipes](https://github.com/QuentinDuval/IdrisPipes).

### The link with streaming

In our previous example, we combined an `IOSpec` computation with the commands of a bot, replacing some of the occurrence of the `ReadLine` with other instructions (such as printing a line). But there are plenty of other transformation we could have done as well.

For instance, we could also combine two `IOSpec` computations x and y together to build a new x .| y computation, by plugging the `PrintLine` instructions of x to the `ReadLine` instructions y.

* A `ReadLine` instruction in y would ask (or `await`) for a string coming from x
* A `PrintLine` instruction in x would send (or `yield`) a string to y‘s `ReadLine` instruction

```haskell
data IOSpec a
  = ReadLine (String -> IOSpec a) -- Await a value
  | PrintLine String (IOSpec a)   -- Yield a value
  | Pure a
```

Now, if we rename `ReadLine` by `Await`, `PrintLine` by `Yield` and `IOSpec` by `Pipe`, we get the basic idea behind the implementation of a streaming library:

```haskell
data PipeLight a
  = Await (String -> PipeLight a)
  | Yield String (PipeLight a)
  | Pure a
```

This PipeLight AST only allows for strings to be streamed, it does not support additional side-effects, terminations and so on, but gives the idea behind the implementation of [Haskell Conduit](https://hackage.haskell.org/package/conduit), [Haskell pipes](https://hackage.haskell.org/package/pipes) or [IdrisPipes](https://github.com/QuentinDuval/IdrisPipes).

### IdrisPipes instruction set

The actual instruction set of [IdrisPipes](https://github.com/QuentinDuval/IdrisPipes) and its corresponding AST is obviously a bit more complex than the one described above. We need to integrate effects, the ability to await or yield different types of values, handle early termination, and more:

```haskell
data PipeM : (a, b, r1 : Type) -> (m : Type -> Type) -> (r2 : Type) -> Type where
  Pure    : r2 -> PipeM a b r1 m r2                                 -- Lift a value into the pipe
  Action  : m (Inf (PipeM a b r1 m r2)) -> PipeM a b r1 m r2        -- Interleave an effect
  Yield   : Inf (PipeM a b r1 m r2) -> b -> PipeM a b r1 m r2       -- Yield a value and next continuation
  Await   : (Either r1 a -> PipeM a b r1 m r2) -> PipeM a b r1 m r2 -- Yield a continuation (expecting a value)
```

The instruction set is slightly enriched:

* `Await` expects an `Either` value to support early termination of the upstream pipe
* `Action` allows us to perform effects of type m inside the pipe

The type parameters of `PipeM` are also a bit more involved:

* `a` is the type of the values flowing in from the upstream (left pipe)
* `b` is the type of the values flowing out to downstream (right pipe)
* `m` is the Monad the pipe runs in
* `r2` is the type of the returned value of the pipe
* `r1` is the type of the upstream pipe return value (left pipe)

Overall, the syntax may look a bit different (due to the use of Inf and the use of Generalized Algebraic Data Types), but the basic idea is the same.

### Emitting, Sequencing, Combining

As we saw through the example of `IOSpec`, we need to add a few ingredients to make the construction of an AST practical. We need factory functions for the common instructions, and a Monad instance for sequencing instructions. In [IdrisPipes](https://github.com/QuentinDuval/IdrisPipes):

* `yield`, `await` and `lift` (from the Monad transformation type-class) are the factory functions
* The Monad instance of `PipeM` is available in [Pipes.Core](https://github.com/QuentinDuval/IdrisPipes/blob/master/src/Pipes/Core.idr)

Finally, the `.|` pipe operator allows us to combine two pipes into one, by plugging together the `Yield` of the left pipe to the `Await` of the right pipe:

```haskell
(.|) : (Monad m) => PipeM a b r1 m r2 -> PipeM b c r2 m r3 -> PipeM a c r1 m r3
(.|) = pull where
  mutual
 
    pull : (Monad m) => PipeM a b r1 m r2 -> PipeM b c r2 m r3 -> PipeM a c r1 m r3
    pull up (Yield next c) = Yield (up `pull` next) c         -- Yield downstream
    pull up (Action a) = lift a >>= \next => up `pull` next   -- Produce effect downstream
    pull up (Await cont) = up `push` cont                     -- Ask upstream for a value
    pull up (Pure r) = Pure r
 
    push : (Monad m) => PipeM a b r1 m r2 -> (Either r2 b -> PipeM b c r2 m r3) -> PipeM a c r1 m r3
    push (Await cont) down = Await (\a => cont a `push` down)   -- Await upstream
    push (Action a) down = lift a >>= \next => next `push` down -- Produce effect upstream
    push (Yield next b) down = next `pull` down (Right b)       -- Give back control downstream
    push (Pure r) down = Pure r `pull` down (Left r)            -- Termination, send pipe result downstream
```

* `pull` traverses the downstream `PipeM` computation until it meets an `Await` instruction, at which point it gives the control to the upstream pipe (via `push`) to insert its own instructions
* `push` inserts the upstream instruction until it meets a `Yield` instruction, at which point it connects it to the downstream pipe’s `Await` instruction and gives back control downstream

You can note that the type signature of the `.|` operator makes sure that we cannot plug together pipes that would not connect together in terms of values exchanged.

### Running a pipe - Interpreter

At this point, the last missing piece is the interpreter, the function that runs a pipeline, and execute the streaming recipe it represents. As described in our previous article on [IdrisPipe](https://github.com/QuentinDuval/IdrisPipes), we can only running a complete pipeline, i.e. a self sufficient pipe which does not await or yield anything.

[IdrisPipes](https://github.com/QuentinDuval/IdrisPipes) defines a type alias to represent such pipes. An `Effect` is an alias over `PipeM`, where `Void` is used in place of the type of the values flowing in and out of the pipeline (*):

```haskell
Effect : (m: Type -> Type) -> (r: Type) -> Type
Effect m r = PipeM Void Void Void m r
```

Because we cannot construct a value of type Void, the interpreter of a complete pipeline only has to deal with two instructions:

* An `Action` to execute some side effect, followed by a continuation
* The `Pure` instruction for the termination of the computation
* While the other instructions lead to an absurd statement

```haskell
runPipe : (Monad m) => Effect m r -> m r
runPipe (Pure r) = pure r                         -- Done executing the pipe, return the result
runPipe (Action a) = a >>= \p => runPipe p        -- Execute the action, run the next of the pipe
runPipe (Yield next b) = absurd b                               -- Absurd
runPipe (Await cont) = runPipe $ Await (either absurd absurd)   -- Absurd
```

_(*) The `Source` and `Sink` are similarly defined, with `Void` in place of the type for the values flowing in for the `Source`, and the values flowing out for the `Sink`._

## Conclusion and what's next

At this point, you should now enough about Free Monads to understand the concept and even build your own. You should have a basic idea of what Free Monads are about, what an interpreter is, and why it might be interesting for you to look further at Free Monads.

You should also know enough about the implementation of [IdrisPipes](https://github.com/QuentinDuval/IdrisPipes) to read its implementation in full, understand it, and maybe even contribute to it if you are interested.
