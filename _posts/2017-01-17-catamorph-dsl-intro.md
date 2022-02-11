---
layout: post
title: "Catamorph your DSL (Introduction)"
description: ""
categories: [functional-programming]
tags: [Functional-Programming, Haskell]
---

I recently had the pleasure to go to a Haskell workshop on Domain Specific Languages (DSL). The goal was to teach us the basics on how to build a DSL.

The workshop covered a lot of ground, too much to cover here. I will instead focus on one specific notion that this workshop made me discover (and which I found pretty useful and amazing): catamorphisms.

This is a vast subject that will require several posts to explore. Across the following posts, we will:

* Start by describing and designing a small toy DSL in Haskell.
* Introduce catamorphisms to improve its flexibility and performance
* Then discuss the trade-offs involved in introducing it in your DSL
* Conclude by looking at how the concept translates to other languages

The goal of today’s post is to tackle the first point: introducing a small DSL which will be used across all the following posts.

## A small arithmetic DSL

We will create a small DSL that resemble the DSL introduced in the Boost Variant Tutorial. This is a classic example illustrated in many other blogs like this post on Simplify C++, but it is quite instructive nevertheless.

The goal is to perform some basic integer arithmetic operations, like addition and multiplication, on both integer constants and integer variables.

Our DSL will be named Expr and is based on several possible sub-types (Haskell constructors):

* Cst to represent constant integer values
* Var to represent unknown integer values (variables)
* Op to represent an operation (addition or multiplication)

This translates into the following Haskell code:

```
type Id = String
data OpType = Add | Mul deriving (Show, Eq, Ord)

data Expr
  = Cst Int
  | Var Id
  | Op OpType [Expr]
  deriving (Show, Eq, Ord)
```

You can notice in the code above that the Op constructor is recursive: an operation is itself based on sub-expression of the same Expr DSL. This is something that will be very important when introducing catamorphisms.

## Creating an arithmetic expression

To be help testing early, our first tasks will be to facilitate the creation of an expression. We will do this by introducing what is called in Haskell “smart constructors”, basically factory functions:

```
cst = Cst
var = Var
add = Op Add
mul = Op Mul
```

This makes it slightly easier to create an expression. It also helps abstracting away the details of the construction. In general, factory functions buy us some precious degrees of liberty by removing the need to know the exact constructor we use.

Let us create our first expression:

```
add [ cst(1)
    , cst(2)
    , mul [cst(0), var("x"), var("y")]
    , mul [cst(1), var("y"), cst(2)]
    , add [cst(0), var("x") ]]
```

This is not as readable as we could make it by by leveraging standard type classes (like fromInteger). But this is not the focus of this post. Instead, we will build a first interpreter to pretty print an expression of our DSL.

The trick to implement an interpreter is simply to make it follow the structure of our data. We will call the printer interpreter prn and make it output a lispy representation of our DSL (to honor Clojure):

```
prn :: Expr -> String
prn (Cst n) = show n
prn (Var v) = v
prn (Op Add xs) = "(+ " ++ unwords (map prn xs) ++ ")"
prn (Op Mul xs) = "(* " ++ unwords (map prn xs) ++ ")"
```

We can try our interpreter on our previous expression:

```
> let e = add [ cst(1)
              , cst(2)
              , mul [cst(0), var("x"), var("y")]
              , mul [cst(1), var("y"), cst(2)]
              , add [cst(0), var("x") ]]

> prn e
"(+ 1 2 (* 0 x y) (* 1 y 2) (+ 0 x))"
```

## Evaluating and arithmetic expression

Our next move should be to evaluate our expression, given some values for the variables that appear in it. We will need a supplier for these variables, that we will call an environment. Four our purpose, a simple map from string to int will do.

To perform the evaluation itself, we will create a new interpreter. We will call this function “eval”. Again, the function just has to follow the recursive structure of its input:

```
type Env = Map.Map String Int

eval :: Env -> Expr -> Int
eval env (Cst n) = n
eval env (Var v) = env Map.! v
eval env (Op Add xs) = sum $ map (eval env) xs
eval env (Op Mul xs) = product $ map (eval env) xs
```

Let us try this interpreter with an environment binding x to 1 and y to 2:

```
> let e = add [ cst(1)
              , cst(2)
              , mul [cst(0), var("x"), var("y")]
              , mul [cst(1), var("y"), cst(2)]
              , add [cst(0), var("x") ]]

> let env = Map.fromList [("x", 1), ("y", 2)]

> prn e
"(+ 1 2 (* 0 x y) (* 1 y 2) (+ 0 x))"

> eval env e
8
-- (+ 1 2 (* 0 1 2) (* 1 2 2) (+ 0 1))
-- (+ 3 4 1)
-- 8
```

Note that we throw an key-not-found exception when the environment does not contain the appropriate variables. We will improve on this later.

## Optimizating our arithmetic expressions

We covered the basic interpreters we could define for our DSL. At this point there is not much interest in having built a DSL in the first place.

One interesting power DSLs offer is the ability to manipulate their Abstract Syntax Tree (AST). For example, we might be interested in optimizing our arithmetic operations to collapse the known terms once and for all.

We will design our optimizer to perform the following improvements:

* Collapse the constants pertaining to the same operation: (+ 1 2) becomes 3.
* Remove the constants equal to the neutral element: (+ 0 x) becomes x.
* Simplify the operations which only have one argument: (+ x) becomes x.
* Get rid of a multiplication operation if it contains any 0 argument.

The function optimize implements our interpreter and, again, follows the structure of the data. The function optimizeOp factorizes the similarities between the additional and the multiplication (both are Monoids):

```
optimize :: Expr -> Expr
optimize op@(Op Add ys) = optimizeOp Add (map optimize ys) 0 (+)
optimize op@(Op Mul ys)
  | not (null (dropWhile (/= cst 0) xs)) = cst 0
  | otherwise = optimizeOp Mul xs 1 (*)
  where xs = map optimize ys
optimize e = e

optimizeOp :: OpType -> [Expr] -> Int -> (Int -> Int -> Int) -> Expr
optimizeOp opType xs neutral combine =
  let (constants, vars) = partition isCst xs
      constantsVal = map (\(Cst x) -> x) constants
      sumCst = foldl' combine neutral constantsVal
  in case vars of
      [] -> cst sumCst
      [y] | sumCst == neutral -> y
      ys  | sumCst == neutral -> Op opType ys
      ys  -> Op opType (cst sumCst : ys)
```

We can try our optimizer on our initial expression to see how it performs:

```
> let e = add [ cst(1)
              , cst(2)
              , mul [cst(0), var("x"), var("y")]
              , mul [cst(1), var("y"), cst(2)]
              , add [cst(0), var("x") ]]

> prn e
"(+ 1 2 (* 0 x y) (* 1 y 2) (+ 0 x))"

> prn (optimize e)
"(+ 3 (* 2 y) x)"
```

Pretty nice!

But is this optimize function that useful? The user might have trivially optimized the expressions by hand when providing them as input.

It might be true in that case, but for more complex examples of DSL or more complex expressions, it might not be as easy. Besides, the next interpreter will show us a use case in which optimization is very useful.

## Partial application

Let us imagine we have a really complex arithmetic expression, featuring several variables, and we want to “fix” some of them? For example, we might want to replace all occurrences of “x” with the value 1.

This use case is quite common in practice. It is similar to partial application of functions. In the finance software industry, it could correspond to the fixing of some of the unknowns of a payoff depending on the EURIBOR interest rate.

Let us write a simple interpreter for this use case. As for the evaluation, we will need an environment holding values associated to variables. But instead of returning an integer, we will return another expression, in which known variable occurrences will be replaced by their associated value.

```
partial :: Env -> Expr -> Expr
partial env e@(Var v) =
  case Map.lookup v env of
    Nothing -> e
    Just n -> cst n
partial env (Op opType xs) = Op opType (map (partial env) xs)
partial env e = e
```

Let us try it with our initial expression and an environment binding “y” to 0:

```
> let e = add [ cst(1)
              , cst(2)
              , mul [cst(0), var("x"), var("y")]
              , mul [cst(1), var("y"), cst(2)]
              , add [cst(0), var("x") ]]

> let env = Map.fromList [("y", 0)]

> prn (partial env e)
"(+ 1 2 (* 0 x 0) (* 1 0 2) (+ 0 x))"
```

Disappointed by how stupid this expression looks? Now is the time for our optimizer to really shine and clean that expression:

```
> prn (partial env e)
"(+ 1 2 (* 0 x 0) (* 1 0 2) (+ 0 x))"

> prn (optimize (partial env e))
"(+ 3 x)"
```

I told you this optimize function was useful!

## Partial + Optimize = Eval

The last example of optimization might have raised some questions in you. What if our partial evaluation was in fact total? What if we tried to optimize an expression with no variables left? How is that different from evaluating our expression?

It is not different. In fact, we could implement implement eval in terms of partial followed by optimize. If the resulting optimized expression is a simple constant, the evaluation just returns it.
 
We could even leverage this to provide better error messages. We only need to create another interpreter that would list all dependencies to variables inside an expression. We could call it dependencies and it would return a set of variable names.

```
dependencies :: Expr -> Set.Set Id
dependencies (Var v) = Set.singleton v
dependencies (Op _ xs) = foldl1' Set.union (map dependencies xs)
dependencies e = Set.empty

eval' :: Env -> Expr -> Int
eval' env e =
  case optimize (partial env e) of
    Cst n -> n
    e -> error $ "Missing vars: " ++ show (dependencies e)
```

Let us try this error handling on evaluations that are bound to fail:

```
> let e = add [ cst(1)
              , cst(2)
              , mul [cst(0), var("x"), var("y")]
              , mul [cst(1), var("y"), cst(2)]
              , add [cst(0), var("x") ]]

> let env = Map.fromList [("x", 1)]

> eval' env e
*** Exception: Missing vars: fromList ["y"]

> eval' Map.empty e
*** Exception: Missing vars: fromList ["x","y"]
```

## The limits of our model

The previous section showed that we could in theory implement our eval function in terms of partial and optimize.

But it would be much less efficient to do it. There are indeed 2 traversals of our AST: one due to partial and one due to optimize (and three in case of missing variables in our environment).

The same goes with our partial application function: we would like to automatically have the optimizer triggered after fixing a variable to avoid creating expressions that look stupid.

We could simply couple the two features together, together in the same function. It would however make the test of the partial function much more complex, as it would depend on the content of the optimization step. So it is not such a good idea, but the alternative is to do two traversals of the tree, which is not ideal either.

What then should we do? When we are faced with such dilemma, the best approach is almost always to take some distance, and rethink our design.

## Easy composition favors decoupling

The problem with our design is that it makes it hard to compose several tree traversals together efficiently.

When composition is not efficient or not made easy, there is less incentive to decompose problems into separate responsibilities. It often leads to hand-crafted special traversals.

For example, if addition and multiplication were operations that required different expertise, it could make sense to have different teams working on each operation. It would be better then to split the optimization code in two as well, for example to simplify code ownership. But because it would lead to two traversals, keeping it as a single traversal would be more efficient and likely preferred.

## Conclusion and what’s next?

We built a DSL to perform arithmetic operations on both constant integer and variable integers. We did it using a simple design, in which function recursion follows the recursive structure of our data.

Playing and enriching this DSL we found out the limits of this simple and naive approach: a code coupled with the traversal of the tree, that does not favor composition of different operations on our AST.

The next post will introduce catamorphisms, as a way to decouple the traversal of our tree from the operations we perform on it, and solve our problems of composition.
