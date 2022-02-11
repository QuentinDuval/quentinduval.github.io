---
layout: post
title: "Quickcheck is fun, deal with it"
description: ""
categories: [functional-programming]
tags: [Functional-Programming, Haskell]
---

In our [previous post]({% link _posts/2017-02-14-quick-check-experiments.md %}), we played with QuickCheck on an arithmetic DSL and used it to check and discover properties on its associated interpreters.

Through these experiments, we explored some of the classic usage of QuickCheck and demonstrated through some examples:

* Its ability to reveal some design defects
* Its ability to test high level relations between functions
* Its ability to increase our knowledge by proving us wrong
* Its ability to check that some property can be falsified
 
It does not seem fair however to associate QuickCheck with serious testing business only. QuickCheck is really a fun library to use. You might have already found out that we can leverage its generators for many other purpose than just Property Based Testing.

This post is dedicated to do justice to QuickCheck by exploring some of those more exotic use cases of QuickCheck. Do not expect anything serious in this post: we will embrace the absurd world of generating random stuff.

_Note: This post is built on top of our [previous post]({% link _posts/2017-02-14-quick-check-experiments.md %}). The experiments we will conduct with QuickCheck will feature the same arithmetic DSL. You might want to read this previous post first._

## Generating random valid code

Generating random stuff is always quite a bit of fun. Why not break the boredom of doing serious test, by leveraging our generators to do stupid random things?

You surely noticed our `prn` interpreter was pretty printing our arithmetic expression in a LISP like syntax. Generating random Clojure function is not very far away:

* We have a `genExpr` function to create random arithmetic expressions
* Our `prn` interpreter can pretty print an expression in Clojure
* Our `dependencies` interpreter can list the bound variables

We can easily create a generator of function names which satisfy good code convention: not too short, not too long, and (most importantly) not too understandable.

```
genName :: Gen String
genName = do
  n <- elements [5..10]
  replicateM n (elements ['a'..'z'])
```

We can assemble our function by pretty printing the arithmetic expression, print the dependencies inside brackets (for the parameters for the function), prefixed by a function name, and surround the whole with parentheses:

```
toClojureFunction :: String -> Expr -> String
toClojureFunction name e =
  unwords
    ["(defn", name,
     "[" ++ unwords (Set.toList $ dependencies e) ++ "]",
     prn e, ")"]
```

We now have everything we need to generate Clojure functions containing optimized (we would not like to produce inefficient code) arithmetic expressions:

```
clojureFunctionGen :: Int -> Gen String
clojureFunctionGen size =
  toClojureFunction
    <$> genName
    <*> fmap optimize (genExpr size)

generate (clojureFunctionGen 30)
> "(defn ptvkegely [c j m n t u y] (+ -64 m c t y n j u) )"
```

We can check that our random generated function is valid Clojure code by firing up a Clojure REPL and play with it.

```
(defn ptvkegely [c j m n t u y] (+ -64 m c t y n j u))
=> #'user/ptvkegely

(ptvkegely 1 2 3 4 5 6 7)
=> -36
```

Did you know that 64, the square of 8, negated and added to the sum of numbers from 1 to 7, was equal to 36, the square of 6? The proof that QuickCheck can be both fun and instructive.

_Note: if your company pays you by the line, you know have a way to create an incredible amount of valid code in a very short amount of time._

## Ending the infix VS prefix debate

Since we are already on the LISP subject, we can take this opportunity to solve the long lasting developer debate. Which is better, infix or prefix notation?

Since we have arithmetic expressions generators at hand, we can devise a fair experiments based on statistics:

* We create a infix pretty printer of arithmetic expression `prnInfix`
* We use QuickCheck to generate a bunch of arithmetic expressions
* We serialize each of these expressions with `prn` and `prnInfix`

Because every developer’s life goal is to golf his code, the winner will be the format that results in the shortest string representation in average.

### A first infix pretty printer

Let us start by creating our `prnInfix` pretty printer. We need to be careful with the parentheses. We would not want our pretty printer to modify the meaning of our arithmetic expressions.

As a first approximation, we could use the following strategy:

* Systematically surround the arguments of a multiplication with parentheses
* Do not add parentheses for the arguments of an addition

This lead us to the following implementation in which we use catamorphisms for concision (if you are not familiar with this recursion scheme, you can read [our series dedicated to it]({% link _posts/2017-01-17-catamorph-dsl-intro.md %})).

```
prnInfix :: Expr -> String
prnInfix = cata infixAlg where

  infixAlg (Op Add xs) = concat (intersperse " + " xs)
  infixAlg (Op Mul xs) = concat (intersperse " * " (map parens xs))
  infixAlg (Cst n) = show n
  infixAlg (Var v) = v
  
  parens x = "(" ++ x ++ ")"
```

We can try our implementation on a complex enough example, to see if it leads to acceptable results:

```
let e = add [ cst(1)
            , cst(2)
            , mul [cst(0), var("x"), var("y")]
            , mul [cst(1), var("y"), add [cst(2), var("x")]]
            , add [cst(0), var("x") ]]
  
prn e
> "(+ 1 2 (* 0 x y) (* 1 y (+ 2 x)) (+ 0 x))"

prnInfix e
> "1 + 2 + (0) * (x) * (y) + (1) * (y) * (2 + x) + 0 + x"
```

Quite disappointing to say the least. With that much noise induced by unnecessary parentheses, the infix notation clearly stands no chance. We will need to improve on our heuristic to have an entertaining fight.

### Improving the infix pretty printer

To witness a fair fight, we need our `prnInfix` to use much less parentheses. We note that a multiplication only needs to add parentheses around sub-expressions that correspond to additions.

Catamorphisms are not a powerful enough recursion scheme to do this. We need to turn to Paramorphisms, which are Catamorphisms with context information. The context information will provide us with the sub-expression we need to make the right parenthesizing choice:

```
para :: (ExprR (a, Expr) -> a) -> Expr -> a
para alg = alg . fmap (para alg &&& id) . unFix
```

Based on this construct, we can define our pretty printer to leverage the context to only add parentheses around parameters that correspond to addition sub-expressions:

* The parensPlus handles the parentheses around multiplication sub-expressions
* The addition discards the additional context (and never adds parentheses)
* The simple terms are computed the same as before

```
prnInfix :: Expr -> String
prnInfix = para infixAlg where

  infixAlg (Op Add xs) = concat (intersperse " + " (map fst xs))
  infixAlg (Op Mul xs) = concat (intersperse " * " (map parensPlus xs))
  infixAlg (Cst n) = show n
  infixAlg (Var v) = v

  parensPlus (s, Fix (Op Add _)) = "(" ++ s ++ ")"
  parensPlus (s, _) = s
```

We can test our new improved pretty printer and check the improvement. In our test expression, there is no unnecessary parentheses left for us to see:

```
let e = add [ cst(1)
            , cst(2)
            , mul [cst(0), var("x"), var("y")]
            , mul [cst(1), var("y"), add [cst(2), var("x")]]
            , add [cst(0), var("x") ]]
  
prn e
> "(+ 1 2 (* 0 x y) (* 1 y (+ 2 x)) (+ 0 x))"

prnInfix e
> "1 + 2 + 0 * x * y + 1 * y * (2 + x) + 0 + x" 
```

Our infix pretty printer is now fairly equipped for a fair contest.

### Game, set and match

Our challengers are ready. The crowd is waiting. We only need to implement our contest. Here is how the game will be structured:

* We generate a thousand of random expressions using QuickCheck
* Each expression is optimised before being pretty printed
* We return the sum of sizes for both `prn` and `prnInfix` (their scores)

A lower score means the pretty printer produced smaller string representations for the expression. The lowest score will thus designate our winner.

To finish up, our contest function will take as parameter the complexity of the arithmetic expressions to pretty print:

```
data ContestResult = ContestResult {
    prefixScore :: Int ,
    infixScore  :: Int }
  deriving (Show)

runContest :: Int -> IO ()
runContest size = do
  expressions <- generate (replicateM 1000 (genExpr size))
  let run printer = sum $ map (length . printer . optimize) expressions
  print $ ContestResult (run prn) (run prnInfix)
```

The results shows the quite stunning performance of the prefix notation, winner by quite a large margin:

```
runContest 30
> ContestResult {prefixScore = 31951, infixScore = 42293}

runContest 100
> ContestResult {prefixScore = 87662, infixScore = 125822}
```

* 25% characters less for `prn` for expressions of complexity 30
* 30% characters less for `prn` for expressions of complexity 100

We would not want to conclude too fast and declare that prefix notation completely owns the infix notation. Instead, we hope the reader will come to this same conclusion by himself/herself.

## This madness is over

We are done: these random stupid experiments are over (for now). I cannot know if you found such experiments funny or even remotely enjoyable.

But I thought it was useful to cover one of the aspects of programming that maybe does not appear enough at work: we can have great fun doing crazy things. The experiments listed above were done in-between more serious work, just to try stupid things.

We must realize we have the luxury of having both powerful hardware and software available for us, at relatively cheap prices. We can use them for fun as much as for work. And who knows, we might even learn something in the process.

With this post, we concluded our series of five posts on QuickCheck. We tried to give a board spectrum of what QuickCheck and Property Based testing is about, how it can be implemented, and how we can use it for both serious tests and less serious experiments.
