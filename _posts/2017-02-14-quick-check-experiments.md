---
layout: post
title: "Experimenting with QuickCheck on a DSL"
description: ""
categories: [functional-programming]
tags: [Functional-Programming, Haskell]
author_profile: false
---

After having spent the last three posts exploring how to implement our own QuickCheck, the goal of the next two posts will be to play with it, to test and discover properties on our code and have fun as well.

The target of these experiments will be the Arithmetic DSL we built several posts back in January:

* [Catamorph you DSL: Introduction]({% link _posts/2017-01-17-catamorph-dsl-intro.md %})
* [Catamorph you DSL: Deep Dive]({% link _posts/2017-01-20-catamorph-dsl-deep-dive.md %})

After a quick refresher on our Arithmetic DSL and what it is made of, we will use QuickCheck to conduct some experiments on it.

Today’s post will focus on creating the appropriate arithmetic expression generator and put them into use to unit test our code. The next post will deal with the more exotic (and less serious) usage we can make of QuickCheck.

## Our arithmetic DSL

Our Arithmetic DSL is made of 4 primitives: integer constants, variables, addition and multiplication operations. In addition to these types, we introduced the notion of environment, a mean to retrieve an integer value from a variable identifier.

The following code snippet depicts all the types involved:

```hs
type Id = String    -- Variable identifier
type Env =          -- Environment of evaluation
  Map Id Int

data OpType         -- Types of operations
  = Add             -- * Addition
  | Mul             -- * Multiplication
  deriving (Show, Eq, Ord)

data ExprR r        -- Open recursive expression type
  = Cst Int         -- * Constant
  | Var Id          -- * Variable
  | Op OpType [r]   -- * Operation
  deriving (Show, Eq, Ord)

-- The type of arithmetic DSL (fixed point of ExprR)
type Expr = Fix ExprR
```

Our arithmetic DSL comes with a five different interpreters. We listed them with their description below:

* `prn`: pretty prints an Arithmetic expression
* `eval`: computes the value of an expression, given an Env
* `optimize`: optimizes an expression by using known constants
* `partial`: partially computes an expression, given an Env
* `dependencies`: retrieve all variables names from an expression

The following short REPL interaction illustrate each of these interpreters:

```hs
let e = add [ cst(1)
            , cst(2)
            , mul [cst(0), var("x"), var("y")]
            , mul [cst(1), var("y"), cst(2)]
            , add [cst(0), var("x") ]
            ]             

-- Pretty printing an expression
prn e
> "(+ 1 2 (* 0 x y) (* 1 y 2) (+ 0 x))"

-- Optimization of an expression
prn (optimize e)
> "(+ 3 (* 2 y) x)"

-- Partial evaluation of an expression
prn $ partial (Map.fromList [("y", 0)]) e
> "(+ 3 x)"

-- Getting the list of variable names in an expression
dependencies e
> fromList ["x","y"]

-- Full evaluation of an expression
eval (Map.fromList [("x", 1), ("y", 0)]) e
> 4
```

We will stop there for the refresher. If you are curious and want to know how each of these interpreters is implemented, you are welcome to read the series on catamorphism linked in the introduction.

We will now dive into QuickCheck and play with it on our DSL.

## Generator of arithmetic expressions

To play with QuickCheck on our DSL, our first task will be to implement the required generators of arithmetic expressions.

### The wrong way

The initial temptation is to create a new `Arbitrary` instance for our `Expression` and stick all the code inside it. We will resist this temptation.

Indeed, and except for very few rare cases, we will likely need specialized generators in order to test different properties on our code. The arguments of a function often have distinct equivalence classes.

For instance, our arithmetic DSL makes emerge at least one very interesting class of expression: constant expressions, which are the expression that do not contain any variables.

Each equivalence classes will likely need to a specific generator. If we move all the implementation into the arbitrary function, we loose a lot of flexibility by having fused everything together. **`Arbitrary` is better used to designate a default generator for a type.**

### A better way

We will define functions returning `Gen Expr` generators. Using standard function will allow us to mix and match them to get appropriate generators for our equivalence classes. We can then optionally select one of these generators as our default generator by instantiating `Arbitrary Expr`.

An interesting rule of thumb when dealing with Algebraic Data Types is to **create one generator by constructor of our type**. This is by no mean a rule, but it happens to give good results in general.

Let us start with constants and variables:

* Our constant generator `genCst` use the standard integer generator
* Our variable generator `genVar` will be letters from “a” to “z”

```hs
genCst :: Gen Expr
genCst = fmap cst arbitrary

varNames :: [String]
varNames = [[v] | v <- ['a'..'z']]

genVar :: Gen Expr
genVar = fmap var (elements varNames)
```

From these, we can define another useful generator that creates expressions made of only variable or constants with equal probability. We call these simple terms (as opposed to operations which are compound terms):

```hs
genSimpleTerm :: Gen Expr
genSimpleTerm = oneof [genVar, genCst]
```

Our generator of operation will take as parameter a generator for simple terms as well as a size n. The bigger the n, the more complex the generated expression:

* With a probability 1/n, we generate a simple term
* Otherwise, we create an operation with m<=n sub-expressions
* We recursively generate sub-expressions for size n/m

```hs
opsGen :: Gen Expr -> Int -> Gen Expr
opsGen simpleTermGen = go where
  go n = do
    m <- choose (0, n)
    if m == 0
      then simpleTermGen
      else elements [add, mul] <*> replicateM m (go (div n (m + 1)))
```

Since our operation generator can be parameterized by a simple term generator, we have the flexibility to defined two specialized generators:

* `genExpr`: a generator of expression that may contain variables
* `genCstExpr`: a generator of expression that contains no variable

```hs
genExpr :: Int -> Gen Expr
genExpr = opsGen genSimpleTerm

genCstExpr :: Int -> Gen Expr
genCstExpr = opsGen genCst
```

These generators will be quite handy in the following.

### Environment generators

As well as generating arbitrary arithmetic expressions, we will need to generate environment of evaluation to test our eval and partial interpreters.

Again, we will likely need different generators, so we define the following functions instead of rushing for `Arbitrary` (*):

* `makeEnvWith` allows us to create an environment holding random values for some predefined variable identifiers
* `genTotalEnv` creates an environment associating a value to all possible variable identifiers used in our tests

```hs
makeEnvWith :: Set.Set String -> Gen Env
makeEnvWith deps = do
  let n = Set.size deps
  values <- replicateM n arbitrary
  return $ Map.fromList (zip (Set.toList deps) values)

genTotalEnv :: Gen Env
genTotalEnv = makeEnvWith (Set.fromList varNames)
```

_(*) QuickCheck can be quite helpful at showing us design defects in our code. We cannot create a new Arbitrary instance for our Env type because it is a type synonym of a Map, which already has one such instance defined. QuickCheck points to us the need for stronger type._

## Verifying and discovering properties

Now that we have quite a few generators of expressions available, we can use them in combination with QuickCheck to verify some interesting properties on our Arithmetic expressions.

### Identifying good properties to check

Before we start, we should remind ourselves that QuickCheck is no replacement for all our example based tests. QuickCheck is really great at ensuring that some invariants hold, but will not handle very specific test cases.

For example, checking that our eval function does output the right number cannot be easily done, except by replicating most of the logic of eval inside the test. This would somehow defeat the purpose of our test.

What we can do however is to check relations between our different interpreters. There are quite a few that should pop in our head. We selected some representative examples below.

### Optimize should not change the value of expressions

This property points at an interesting relation between the optimize function and the eval function.

Evaluating an expression should yield the same value as evaluating the optimised expression in the same environment. This should hold for every environment that contains the variables appearing in the non-optimised expression (*).

```hs
prop_optimize_eval :: Expr -> Property
prop_optimize_eval e =
  forAll genTotalEnv $ \env ->
    eval env e == eval env (optimize e)

quickCheck prop_optimize_eval
> +++ OK, passed 100 tests.
```

_(*) The “non-optimized” qualifier is necessary. Our last test will show us some interesting twists which make this distinction important._

### Partial evaluation of constant expressions should yield a number

In the post describing our Arithmetic DSL, we pointed out that eval could be implemented in terms of partial evaluation. If the result of a partial evaluation is a constant term, we just had to unwrap it and get the value of the expression.

This argument was based on the assumption that the partial evaluation is able to fully optimize a constant expression into a single constant term. Since partial is based on optimize, this property should also hold for the later.

```hs
prop_optimize_constant :: Property
prop_optimize_constant =
  forAll (sized genCstExpr) (isCst . optimize)

prop_partial_constant :: Property
prop_partial_constant =
  forAll (sized genCstExpr) (isCst . partial Map.empty)
```

The `sized` function is a quite helpful QuickCheck helper that allows us to inject the size of the input into our generator `genCstExpr`.

### Dependencies should return the variables needed for evaluation

This property points out at an interesting relation between our evaluation functions and the dependencies. If each variable identifier returned by dependencies appears in the environment, partial should be able to fully reduce an expression to a constant term.

```hs
prop_dependencies_allow_eval :: Property
prop_dependencies_allow_eval =
  forAll (sized genExpr) $ \e ->
    forAll (makeEnvWith (dependencies e)) $ \env ->
      isCst (partial env e)
```

We may have the feeling the converse should also hold: if a single variable from an dependencies does not appear in the environment, partial should not be able to fully evaluate an expression.

```hs
makePartialEnv :: Set.Set Id -> Gen Env
makePartialEnv deps = do
  v <- elements (Set.toList deps)
  makeEnvWith (Set.delete v deps)

prop_missing_dependencies_forbid_eval :: Property
prop_missing_dependencies_forbid_eval =
  forAll (sized genExpr) $ \e ->
    let deps = dependencies e
    in Set.size deps > 0 ==>
        forAll (makePartialEnv deps) $ \env ->
          not (isCst (partial env e))
```

But actually, our property fails:

```hs
quickCheck prop_missing_dependencies_forbid_eval
> *** Failed! Falsifiable (after 4 tests): 
> (* g x)
> fromList [("x",0)]

quickCheck prop_missing_dependencies_forbid_eval
> *** Failed! Falsifiable (after 4 tests): 
> (* 0 b q)
> fromList [("b",2)]
```

And indeed, `(* g x)` does not necessarily require all variables to be bound to get a value out the expression. If either `g` or `x` is equal to zero, the expression is trivially equal to zero. The second counter example featuring `(* 0 b q)` is even more telling.

We discovered that our partial function was in fact able to evaluate expressions that do not have all their variable bound in the environment. QuickCheck discovered this knowledge for us, with only 4 tests.

Using QuickCheck, we can quickly encounter many similar experiences. **QuickCheck is quite impressive at proving us wrong**. Doing so, it participates rather effectively at increasing the knowledge on our own code.

### Optimize does not necessarily preserve the dependencies

From the previous experiment, we can deduce that our optimize function does not systematically preserves the dependencies of an expression.

**We can express the negation** (that it should keep the dependencies of an expression unchanged), and indicate to QuickCheck we expected the property to fail:

```hs
prop_optimize_preserves_dependencies :: Property
prop_optimize_preserves_dependencies =
  forAll (sized genExpr) $ \e ->
    dependencies e == dependencies (optimize e)

quickCheck (expectFailure prop_optimize_preserves_dependencies)
> +++ OK, failed as expected. Falsifiable (after 8 tests): 
> (* g r e m (* 0))
```

Going back to our first test, we now understand why the distinction “non-optimised” in the property definition was quite an important one.

## Conclusion and what's next?

We took as example our Arithmetic DSL we build in earlier posts, and used QuickCheck to build some generators and check some properties on it.

Through these experiments, we discussed strategies to build generators and discovered several interesting use cases of QuickCheck:

* Its ability to reveal some design defects
* Its ability to test high level relations between functions
* Its ability to increase our knowledge by proving us wrong
* Its ability to check that some property can be falsified

Next post will be dedicated to more “exotic” and maybe be less “serious” use cases for QuickCheck.
