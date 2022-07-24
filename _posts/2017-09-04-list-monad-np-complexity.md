---
layout: post
title: "List Monad connection with Non-deterministic Polynomial time complexity"
description: ""
excerpt: "A problem is in NP if it can be solved by an algorithm that runs in polynomial time on a non-deterministic Turing Machine, a computer that can “guess” which path to take toward the right answer when given a choice."
categories: [functional-programming]
tags: [Functional-Programming, Haskell]
use_math: true
---

A problem is in NP (Non-deterministic Polynomial time complexity) if it can be solved by an algorithm that runs in polynomial time on a non-deterministic Turing Machine, a computer that can “guess” which path to take toward the right answer when given a choice.

I never quite built a good intuition on what it means. What does it mean to guess? If such a powerful computer could guess, why could it not guess the right answer to the problem right from the start? Why would it have to go through a polynomial number of steps?

This short post will show connect the List Monad with non-deterministic Turing machines, to build some intuition on NP, how a non-deterministic Turing machine behaves, and it would feel to program one such computer, thanks to Haskell.

## Motivation

I recently found this [great video](https://www.youtube.com/watch?v=eHZifpgyH_4) from [Erik Demaine](http://erikdemaine.org/), a short but quite entertaining overview of P, NP and NP-completeness complexity classes.

At [8 minutes 50 seconds](https://www.youtube.com/watch?v=eHZifpgyH_4#t=8m50s) in the video, he explains why the [Boolean Satisfiability Problem](https://en.wikipedia.org/wiki/Boolean_satisfiability_problem) (SAT) is in NP. Here is the content for his board after the explanation:

```
guess: x1 = True or False
guess: x2 = True or False
...
guess: xN = True or False
check if formula on (x1, x2 ... xN) is True or False
```

Now, if you know about the List Monad, you should see it right there, in front of you. This insight might give you an idea of what “a non-deterministic computer guess” could be. If not, this post might be of interest to you.

## Non-deterministic computation = List Monad

A lot of Monad tutorials start with the Maybe and List monads. Most good tutorials will also mention that the List Monad models **non-deterministic computations**. It sounds like it must somehow be connected to the NP non-deterministic Turing machine. Let us see how.

### I see a Monad in your SAT proof

To understand the similarities between the board of Erik Demaine and the List Monad, let us translate the SAT NP proof from Erik Demaine into Haskell code. Thanks to the List monad, the translation is pretty simple.

Here is the original board he wrote:

```
guess: x1 = True or False
guess: x2 = True or False
...
guess: xN = True or False
check if formula on (x1, x2 ... xN) is True or False
```

Below is our Haskell version. We added a function name, introduced an or to stop at the first result, but mostly, our task was to replace occurrences of “guess” by the bind `<-` operator:

```haskell
satExample :: Bool
satExample = or $ do
  x1 <- [True, False] -- "guess" is replaced by bind (<-)
  x2 <- [True, False]
  {- ... -}
  xN <- [True, False]
  pure (formula x1 x2 {- ... -} xN)
```

So our List Monad bind operator is analog to the “guess” of a non-deterministic Turing Machine.

### Back to List Monad tutorial

Now that we have seen the similarities, let us look for an online Monad tutorial and check the definition it provides for non-deterministic computation. If we pick this one from [FP complete](https://www.schoolofhaskell.com/school/starting-with-haskell/basics-of-haskell/13-the-list-monad), we can read:

<blockquote> [..] non-deterministic computation. It’s the kind of computation that, instead of producing a single result, might produce many.</blockquote>

The connection with the definition of NP is not direct, but pretty close. A non-deterministic computation returns several results.

A non-deterministic Turing Machine is able to magically pick a “good” result among theses results, by guessing which one will lead to one “yes” outcome (there might be many).

### From exponential to non-deterministic polynomial complexity

Let us go back to our translation of the SAT NP proof in Haskell:

```haskell
satExample :: Bool
satExample = or $ do
  x1 <- [True, False] -- "guess" is replaced by bind (<-)
  x2 <- [True, False]
  {- ... -}
  xN <- [True, False]
  pure (formula x1 x2 {- ... -} xN)
```

Executed on a conventional computer, this function will explore the search space of all potential variable assignments for `x1`, `x2` up to `xN`, and stop at the first one for which the formula returns a positive answer. This code has therefore an exponential time complexity.

But on a non-deterministic computer, each guess can be performed in constant time. And these guesses correspond to the bind operator of the `List` Monad. So, provided formula runs in linear time, the non-deterministic complexity of our `satExample` algorithm is linear.

### Estimating the non-determistic complexity of an algorithm

Generalising to all decision problems (problems whose answer is either True or False) and to non deterministic computations, we can reason that:

_Given a non-deterministic computation `f` occurring inside the decision problem `D`, and returning a set of result `R`, a non-deterministic Turing machine is able to pick a value from `R` which leads to a “yes” answer for `D` (if it exists) in constant time._

_In the `List` Monad, the non-deterministic algorithmic complexity of the statement `y <- f x` is therefore equal to the non-deterministic algorithmic complexity of f, however many results `f` may return._

This gives us a way to estimate the non-deterministic complexity of an algorithm: we refactor the algorithm in Haskell in such as way that the `List` Monad bind operators becomes visible. Then we derive its complexity, by counting each bind operator as constant time assignments: O(1).

We can do this for SAT, as well as any other algorithm. Take your favorite search-like algorithm, write it using the `List` Monad, then you can visually estimate its complexity on an fantasized non-deterministic Turing machine of the future. I am not sure it is terribly useful, but hey, it is still fun.

### Example of SAT

Let us look at the code of a complete (and completely naive!) SAT solver (*). Our solver takes as input an instance of a SAT problem `pb`, and returns if there exists an assignment of the variables that makes the formula True.

Our implementation has been refactored to make each bind operator of the List Monad visible:

```haskell
sat :: SAT -> Bool
sat pb = or $ do
  assignment <- forM (allVars pb) $ \var -> do
    val <- [True, False]
    pure (var, val)
  pure (evalSat assignment pb)
```

* `allVars` extract all the variables from the SAT instance (normalised as a conjunction of disjunction)
* `evalSat` checks if the SAT instance is verified for a given assignment of the variables

Assuming the algorithmic complexity of `allVars` and `evalSat` are linear, and counting each bind operation as O(1) assignments, we can clearly see that the non-deterministic complexity of the sat algorithm is linear as well (there is a linear number of calls to bind).

The full code for this naive SAT evaluator is available at the end of the post.

_(*): A real SAT solver would explore the search space by cutting branches early. It would use a clever ordering for the variable assignment, would use constant propagation to identify dead-end as soon as possible, as well as other clever tricks such as [unit propagation](https://en.wikipedia.org/wiki/Unit_propagation)._

_The goal of all these tricks is to reduce the exponential exploration space. Interestingly, if we could ever have a non-deterministic Turing machine, we would not need such tricks. In fact, they would only slow down the algorithm._

## Conclusion

The expressivity of Haskell and it's do-notation gives us the ability to express different types of computations. Thanks to this, we can use Haskell to experience how programming would look like in a different world, in which we had a different kind of hardware to run our software.

Thanks to this, we can better understand how computation would work on a non-deterministic Turing machine from our home, and therefore understand NP complexity better.

## Full code of SAT evaluator

```haskell
module SAT where

import Control.Monad
import qualified Data.Map as Map
import qualified Data.Set as Set


type Var = String
type Assignment = [(Var, Bool)]

data Term
  = Pos { getVar :: String }
  | Neg { getVar :: String }

newtype SAT = SAT { conjunctions ::  [[Term]]}

sat :: SAT -> Maybe Assignment
sat pb = headSafe $ do
  assignment <- forM (allVars pb) $ \var -> do
    val <- [True, False]
    pure (var, val)
  guard (evalSat assignment pb)
  pure assignment

allVars :: SAT -> [Var]
allVars = Set.toList . Set.fromList . map getVar . concat . conjunctions

evalSat :: Assignment -> SAT -> Bool
evalSat assignment =
  let env = Map.fromList assignment
  in and . map (any (evalTerm env)) . conjunctions

evalTerm :: Map.Map Var Bool -> Term -> Bool
evalTerm env (Pos var) = env Map.! var
evalTerm env (Neg var) = not (env Map.! var)

------------------------------------------------------------

headSafe :: [a] -> Maybe a
headSafe [] = Nothing
headSafe xs = Just (head xs)
```
