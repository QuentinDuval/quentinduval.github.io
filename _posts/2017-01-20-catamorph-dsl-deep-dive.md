---
layout: post
title: "Catamorph your DSL (Deep Dive)"
description: ""
categories: [functional-programming]
tags: [Functional-Programming, Haskell]
---

This post is a second of the series of post dedicated to notion of Catamorphisms and its application to build Domain Specific Languages (DSLs).

Our last post introduced an Arithmetic DSL that allowed to build expression composed of operations like addition and multiplication integer constants and integers variables. On top of this DSL, we built several interpreter functions:

* prn to pretty print our Arithmetic expressions
* eval to compute the value of an expression, given the value of all variables
* optimize to simplify and optimize our expression
* partial to partially evaluate our expression, given the value of some variables
* dependencies to retrieve not yet resolved variables from our expression

At the end of the last post, we noticed that eval could be written in terms of partial and optimize. The impossibility to compose these interpreters efficiently showed us the limits of our model. Our design that coupled the traversal and the transformation on the Abstract Syntax Tree (AST) was simply not good enough.

The post of today will introduce the notion of Catamorphisms as a nice way to:

* Decouple the traversal from the actions to perform
* Get efficient composition of our interpreters back

### Open type recursion

One of the first step to do in order to get to Catamorphisms is to deconstruct the recursion. Practically, it means getting rid of the recursive nature of the data structure by introducing a template.

```
type Id = String
data OpType = Add | Mul deriving (Show, Eq, Ord)

data Expr
  = Cst Int
  | Var Id
  | Op OpType [Expr]
  deriving (Show, Eq, Ord)
```

Becomes the following, where recursion on _Expr_ is replaced by the templated type _r_:

```
type Id = String
data OpType = Add | Mul deriving (Show, Eq, Ord)

data ExprR r
  = Cst Int
  | Var Id
  | Op OpType [r]
  deriving (Show, Eq, Ord)
```

This looks innocent, but this is a very powerful trick. We already covered a similar trick for functions in my very first blog post when we were concerned about dynamic programming.

Back then, open recursion allowed us to plug memoization to our recurrence formula. Deconstructing our type recurrence relation will allow us to separate the recurrence from the action to perform during it.

If you read this previous post, you know the drill. Our first step is to get back our naive recurrence by computing the fix point of our recurrence. For types, this is a bit more complex (as we cannot create infinite types) and is achieved using the following trick:

```
newtype Fix f = Fix { unFix :: f (Fix f) }

-- Analog to Expr = ExprR Expr
type Expr = Fix ExprR
```

Fortunately, there is a Haskell package on hackage that help you automating this, in the [data-fix package](https://hackage.haskell.org/package/data-fix-0.0.3/docs/Data-Fix.html).

### Building an expression

To construct an _Expr_ instance, we need to adapt our “smart constructors” (Haskell’s name for factory functions) introduced in our previous post:

```
cst = Fix . Cst
var = Fix . Var
add = Fix . Op Add
mul = Fix . Op Mul
```

Having introduced this layer of abstraction allows us to keep the exact same syntax as before to construct our expressions:

```
> let e = add [ cst(1)
              , cst(2)
              , mul [cst(0), var("x"), var("y")]
              , mul [cst(1), var("y"), cst(2)]
              , add [cst(0), var("x") ]]
```

Although our expression type got more complex by the introduction of a template, we can still instantiate it as easily as we used to, as we do not need to know the exact constructors needed at construction.

### Getting to catamorphisms

Let us start by demystifying the big word. A Catamorphism is a just a name for a recursion scheme. A kind of design pattern to decouple the traversal of a recursive data structure from the work to do inside the traversal.

Our cataphormism implementation will make use of the open recursion we introduced in _ExprR_ to implement a kind of post-order depth first search traversal.

#### Functor instance

Let us start by the first building block of the Catamorphism, making ExprR a Functor instance. It allows to apply a function on the type parameter r for the ExprR r. The Functor basically allows us to transform the type that represents the recurrence (for example to remove the recurrence).

DerivingFunctor could have generated the Functor instance for us as follows, but doing manually is really instructive:

```
instance Functor ExprR where
  fmap _ (Cst c) = Cst c
  fmap _ (Var v) = Var v
  fmap f (Op opType xs) = Op opType (map f xs)
```

Now, if we consider a function with a prototype _ExprR a -> a_, we see that providing it to fmap will get us a function _ExprR (ExprR a) -> ExprR a_. Somehow, using _fmap_ is like navigating between levels of recursion.

But we are not done yet. There is an unbound number of levels of recursions to deal with. Moreover, _fmap_ does not quite fit our recursion: we have to take into account the _Fix_ constructor wrapping each _ExprR_.

#### FMAP at each level

So we need to build a function that will apply our transformation through fmap, at each level of recursion. This function is the catamorphism function below which morphs a function _ExprF a -> a_ to a function _Expr -> a_:

```
cataExpr :: (ExprR a -> a) -> Expr -> a
cataExpr algebra =
  algebra
  . fmap (cataExpr algebra)
  . unFix
```

In simpler terms, it provides a way to lift a local transformation (that applies to one stage of the Abstract Syntax Tree) into a global transformation on the whole expression.

It proceeds in three steps:

* unFix allows to extract the ExprF from the Fix constructor
* fmap (cataExpr algebra) recurs on the sub-trees: each sub expression becomes a a
* The resulting ExprR a is given to the function algebra to finish the job

#### Generalization to all Functors

We can generalize this cataExpr function to work on any fixed point of an open recursive data structure that is an instance of Functor. It means we can replace our ExprR by a Functor instance, as follows:

```
cata :: Functor f => (f a -> a) -> Fix f -> a
cata algebra =
  algebra
  . fmap (cata algebra)
  . unFix
```

This is a bit abstract, so do not worry if you do not quite get this one. Just remember that applied to our ExprR, this function will resolve to the same cataExpr function we saw earlier.

### Revisiting pretty printing

To best understand how catamorphisms work, examples are key. Let us start by reworking our first interpreter, the pretty printer, in terms of cata.

One cool thing about using this recursion scheme is that we can reason locally: we only need to care about what happens at one given stage (one node of our AST). Namely, we know that pretty printing a:

* Constant is calling show on the numeric value
* Variable is returning the name of variable
* Addition is concatenating the sub-expression textual representation with +
* Multiplication is concatenating the sub-expression textual representation with *

This translates almost directly into the following code. We can observe that algebra function is only concerned about implementing the local transformations listed above. The cata function takes entirely care of the recursion:

```
prn :: Expr -> String
prn = cata algebra where
  algebra (Cst n) = show n
  algebra (Var x) = x
  algebra (Op Add xs) = "(+ " ++ unwords xs ++ ")"
  algebra (Op Mul xs) = "(* " ++ unwords xs ++ ")"
```

To convince ourselves it works, we can test our function:

```
> let e = add [ cst(1)
              , cst(2)
              , mul [cst(0), var("x"), var("y")]
              , mul [cst(1), var("y"), cst(2)]
              , add [cst(0), var("x") ]]

> prn e
"(+ 1 2 (* 0 x y) (* 1 y 2) (+ 0 x))"
```

### Evaluation and dependencies

Let us continue our exploration of catamorphisms by rewriting our two next most straightforwards evaluators, eval and dependencies.

Our eval function is based on an algebra function that only has to deal with integers:

* The evaluation of constant and variables is just as before
* Addition is a simple call to sum on the sub-expression results
* Multiplication is a call to product on the sub-expression results

```
eval :: Env -> Expr -> Int
eval env = cata algebra where
  algebra (Cst n) = n
  algebra (Var x) = env Map.! x
  algebra (Op Add xs) = sum xs
  algebra (Op Mul xs) = product xs
```

The dependencies function is based on an algebra function that is only concerned in joining sets of variables identifiers:

* Variables terms have only one dependency: the variable itself
* Operations have for dependencies the total dependencies of their sub-expressions

```
dependencies :: Expr -> Set.Set Id
dependencies = cata algebra where
  algebra (Cst _) = Set.empty
  algebra (Var x) = Set.singleton x
  algebra (Op _ xs) = Set.unions xs
```

In both cases, and because cata takes care of the recursion, the implementation almost reflects the specification textually!

### Optimized combination of optimizations

Let us now move on the fun part, decomposing our optimization functions bit by bit, before recombining them with great efficiency. We will adapt the optimize function we introduced in our previous post, but with two big differences:

We extracted the algebra from the recursion: to actually run the optimizations, these functions should be given as argument to cata.

Optimizations for Add and Mul are separated: this is to reflect possible real situations in which two teams (and two sets of functions) are specialized and focused on dealing one complex optimization each.

```
optimizeAdd :: ExprR Expr -> Expr
optimizeAdd op@(Op Add _) = optimizeOp op 0 (+)
optimizeAdd e = Fix e

optimizeMul :: ExprR Expr -> Expr
optimizeMul op@(Op Mul xs)
  | not (null (dropWhile (/= cst 0) xs)) = cst 0
  | otherwise = optimizeOp op 1 (*)
optimizeMul e = Fix e

optimizeOp :: ExprR Expr -> Int -> (Int -> Int -> Int) -> Expr
optimizeOp (Op optType xs) neutral combine =
  let (constants, vars) = partition isCst xs
      constantsVal = map (\(Fix (Cst x)) -> x) constants
      sumCst = foldl' combine neutral constantsVal
  in case vars of
      []  -> cst sumCst
      [y] | sumCst == neutral -> y
      ys  | sumCst == neutral -> Fix (Op optType ys)
      ys  -> Fix (Op optType (cst sumCst : ys))
```

We already know how to run these optimizations inefficiently in two tree traversals. Thanks to our previous labour though, we can do much better. Because the algebra are not coupled to the traversal, we combine two algebra into a single algebra before running one single traversal.

We introduce below the means to compose our algebras:

* _comp_ combines two algebra by function composition with _unFix_
* _compAll_ combines several algebra by reducing with comp over them

```
type Algebra f = f (Fix f) -> Fix f

comp :: Algebra f -> Algebra f -> Algebra f
comp f g = f . unFix . g

compAll :: Foldable t => t (Algebra f) -> Algebra f
compAll fs = foldr1 comp fs
```

Using these composition functions, we can write an efficient optimize function:

```
optimize :: Expr -> Expr
optimize = cata (optimizeMul `comp` optimizeAdd)
```

We did manage to combine several actions in a single traversal. This is a quite important feature which might remind you of our traversal combinators for simple ranges in this post.

This ability to compose easily will allow us to build modular yet efficient software as we will see in the next section.

### Partial + Optimize = Eval

Let us take back from where we failed in the last post. We will now build an efficient evaluation function from three different building blocks:

* A partial evaluation algebra to replace the known variables by constants
* Our addition optimization and multiplication optimization algebra
* Our dependencies interpreter that identifies remaining unknowns

Our partial evaluation algebra function will be named replaceKnownVars. It replaces variables bound in its environment argument by their value:

```
replaceKnownVars :: Env -> ExprR Expr -> Expr
replaceKnownVars env = go where
  go e@(Var v) =
    case Map.lookup v env of
      Just val -> cst val
      Nothing -> Fix e
  go e = Fix e
```

To finish up the work, we just need to assemble the pieces together:

* _partial_ combines the replacement of known variables with the optimization steps
* _eval_ runs a partial evaluation, checks whether the expression was reduced to a constant and reports an error with remaining dependencies if it was not

```
partial :: Env -> Expr -> Expr
partial env = cata (compAll [optimizeMul, optimizeAdd, replaceKnownVars env])

eval :: Env -> Expr -> Int
eval env expr =
  case partial env expr of
    (Fix (Cst n)) -> n
    e -> error $ "Missing vars: " ++ show (dependencies e)
```

That’s it! Three steps in one traversal in case of success, and a second traversal in case of error, all obtained from modular and independently testable pieces of software.

_Note: you can notice there another advantage of working with catamorphisms: pattern matching does not have to be concerned with every possible cases. Since the recursion is handled outside, algebra that transform the AST can make use of the default case._

### Conclusion and what’s next

This concludes our introduction of catamorphisms in Haskell.

We went through the mechanisms needed to introduce this recursion scheme, and demonstrated how it could be applied to combine several tree recursions into a single traversal.

We discovered how using this technique might improve the modularity and testability of our code, with minimum expenses in terms of efficiency.

The next posts will focus on discussing the limits of this technics, some of its trade-offs and show how it translates in some other languages.
