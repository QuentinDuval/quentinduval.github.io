---
layout: post
title: "Catamorph your DSL (Clojure)"
description: ""
categories: [functional-programming]
tags: [Functional-Programming, Clojure]
---

In the previous three posts, we went over the process of building a DSL for arithmetic operations and we introduced the concept of Catamorphism as a way to decouple the traversal of the AST of our DSL from he operations we want to perform on it.

We saw how it could help us compose operations before traversing the AST of our DSL, leading to more efficient composition and better testability.

But Haskell being quite unique, it is quite interesting to take some distance and take time to investigate at how applicable is this concept in other languages.

In this post, we will focus on Clojure, and will:

* Build a small arithmetic DSL in Clojure
* Look at how catamorphism translates in Clojure
* Implement the same interpreters as in Haskell
* Marvel at the beauty of coding in Clojure

In the process, we will illustrate that the concept of Catamorphism is related to post-order depth first search traversal of trees.

## Growing our arithmetic DSL

Our first task will be to choose a representation for our DSL.

Clojure encourages to use simple data structures as much as possible. We will follow the philosophy of the language and represent our DSL using nested vectors, and two keywords for addition and multiplication.

```
[:add 1 2
 [:mul 0 "x" "y"]
 [:mul 1 "y" 2]
 [:add 0 "x"]]
```

Using simple data structures does not forbid introducing abstractions. In the same spirit as in Haskell, we will introduce “smart constructor” (factory functions) to build our expression without having to know its internal structure:

```
(defn cst [n] n)
(defn sym [s] s)
(defn add [& args] (into [:add] args))
(defn mul [& args] (into [:mul] args))
```

The following code shows how to construct an arithmetic DSL from these constructors:

```
(def expr
  (add
    (cst 1) (cst 2)
    (mul (cst 0) (sym "x") (sym "y"))
    (mul (cst 1) (sym "y") (cst 2))
    (add (cst 0) (sym "x"))))
```

To complete our arithmetic expression, and abstract away from its representation, we also offer some query and extraction functions on it. In particular:

* _op?_ returns whether the top expression is an addition or multiplication.
* _rator_ extracts the operator of an operation (addition or multiplication)
* _rands_ extracts the operands of an operation (addition or multiplication)

```
(defn rator [e] (first e))
(defn rands [e] (rest e))

(defn cst? [n] (number? n))
(defn sym? [v] (string? v))
(defn op?  [e] (vector? e))
(defn add? [e] (and (op? e) (= (rator e) :add)))
(defn mul? [e] (and (op? e) (= (rator e) :mul)))
```

These query functions will allow us to implement the equivalent of the pattern matching logic of Haskell in our Clojure code.

## Introducing Clojure Walk

We already discussed in the previous posts that catamorphisms relate to post-order depth first search traversal of the AST of our DSL. Clojure offers a really good implementation of tree traversals in its [clojure.walk namespace](https://clojure.github.io/clojure/clojure.walk-api.html).

Our code will rely on this namespace, as well as some additional standard namespaces inclusions. You can find them below:

```
(ns arithmetic
  (:require
    [clojure.string :as string]
    [clojure.set :as set]
    [clojure.walk :as walk]
    ))
```

In particular, _clojure.walk/postwalk_ will be of great interest to us. It visits all nodes in tree, apply a transformation on it, before visiting their father node recursively, until it reaches the root node.

Let us see how it works by tracing the visit of all nodes along the traversal:

```
(walk/postwalk
  #(do (print (str % " ")) %)
  [:add 1 [:mul "x" 2]])

:add 1 :mul x 2 [:mul "x" 2] [:add 1 [:mul "x" 2]]
```

* The walk starts by the leftmost element and goes as deep as it possibly can
* Each node is scanned and transformed exactly once (we used the identity here)
* The father of a group of node is visited right after all its children
 
Note that there is one big difference with the Catamorphism we had in Haskell: because we visit every single node, we will also visit the keywords :add and :mul of our DSL. This is something we will have to take this into account.

## Pretty print walk

Let us put the postwalk function to use by translating in Clojure our most basic interpreter, the pretty printer.

* _walk/postwalk_ plays the role of the cata function in Haskell
* The first argument of _postwalk_ is the algebra (named for clarity only)
* The _:else_ handles the case of the keywords _:add_ and _:mul_

The code below is the full Clojure implementation:

```
(defn print-expr [expr]
  (walk/postwalk
    (fn algebra [e]
      (cond
        (cst? e) (str e)
        (sym? e) e
        (add? e) (str "(+ " (string/join " " (rest e)) ")")
        (mul? e) (str "(* " (string/join " " (rest e)) ")")
        :else e))
    expr))
```

Using the function results in the result we expect:

```
(def expr
  (add
    (cst 1) (cst 2)
    (mul (cst 0) (sym "x") (sym "y"))
    (mul (cst 1) (sym "y") (cst 2))
    (add (cst 0) (sym "x"))))

(print-expr expr)
=> "(+ 1 2 (* 0 x y) (* 1 y 2) (+ 0 y))"
```

Except for the matching of the operation keywords, the Clojure implementation looks really much like the Haskell implementation of our previous post:

```

prn :: Expr -> String
prn = cata algebra where
  algebra (Cst n) = show n
  algebra (Var x) = x
  algebra (Op Add xs) = "(+ " ++ unwords xs ++ ")"
  algebra (Op Mul xs) = "(* " ++ unwords xs ++ ")"
```

Although it leverages very different means, Clojure achieves the same kind of simple, decoupled, and close-to-specification code than Haskell. Beautiful, right?

## Evaluate and dependencies

We can as easily translate to Clojure our next two most straightforward interpreters, _eval_ (renamed evaluate since eval already exists in Clojure) and _dependencies_.

The evaluate function below needs an environment holding the value of the variables in the arithmetic expression. To make it simple, we used a Clojure hash map (which we can consult using the _get_ function).

```
(defn evaluate [env e]
  (walk/postwalk
    (fn [e]
      (cond
        (sym? e) (get env e)
        (add? e) (reduce + (rest e))
        (mul? e) (reduce * (rest e))
        :else e))
    e))
```

The implementation of _dependencies_, which lists all the variables of the expression, is also quite small and simple to follow. It makes use of Clojure standard sets to gradually accumulate all variable names from an arithmetic expression:

```
(defn dependencies [e]
  (walk/postwalk
    (fn [e]
      (cond
        (cst? e) #{}
        (sym? e) #{e}
        (op? e) (apply set/union (rands e))
        :else e))
    e))
```

## Composable optimisations

After the appetizers, the main dish: the implementation of the arithmetic expression optimization functions.

To show how easy composition of tree walking functions, we will provide two separate functions to optimize the addition and the multiplication:

```
(defn optimize-add [e]
  (if (add? e)
    (optimize-op e + 0)
    e))

(defn optimize-mul [e]
  (if (mul? e)
    (if (some #{(cst 0)} e)
      (cst 0)
      (optimize-op e * 1))
    e))
```

These two functions make use of the optimize-op helper function that contains their common parts, which you can consult here.

Because our solution in Clojure did not have to include a mysterious Fix type to create an infinite type (as it was required in Haskell), composition of our “algebra” is the standard function composition:

```
(defn optimize [e]
  (walk/postwalk
    (comp optimize-mul optimize-add)
    e))
```

It almost looks too simple to be true. Let us try it out to check if we are not dreaming in high definition:

```
(print-expr expr)
=> "(+ 1 2 (* 0 x y) (* 1 y 2) (+ 0 x))"

(print-expr (optimize expr))
=> "(+ 3 (* 2 y) x)"
```

## Partial application

Our last remaining challenge is to implement the partial evaluation of an arithmetic expression, which we will name partial-eval to avoid naming conflicts with existing Clojure functions.

To do so, we will create a replace-var function whose goal is to replace variables bound in the evaluation environment with their respective value. We can combine this function with our optimization functions to obtain a fully fledged partial application that simplifies expression in one single tree traversal.

```
(defn replace-var
  [env x]
  (if (string? x) (get env x x) x))

(defn partial-eval
  [env e]
  (walk/postwalk
    (comp optimize-mul optimize-add #(replace-var env %))
    e))
```

Let us see if it works. We will partially apply our initial expression with “y” bound to zero:

```
(print-expr expr)
=> "(+ 1 2 (* 0 x y) (* 1 y 2) (+ 0 x))"

(print-expr (partial-eval {"y" 0} expr))
=> "(+ 3 x)"
```

Interestingly, if we partially evaluate our function with a environment holding all the variables appearing in the expression, the result is identical to a call to _evaluate_:

```
(print-expr (optimize expr))
=> "(+ 3 (* 2 y) x)"

(evaluate {"x" 1 "y" 2} expr)
=> 8

(partial-eval {"x" 1 "y" 2} expr)
=> 8
```

## Conclusion and what’s next

In a few lines of code, we managed to build an arithmetic DSL, with all the interesting features we built in Haskell. The code is about the same length, as simple and composable as in Haskell.

The main difference to me is the efforts we had to put in to get the same result. There was no need to invent anything complicated, like parameterized recursion, to get to this result.

But on the other hand, we were able to count on the amazing clojure.walk namespace, and we lost almost all the type safety we had in Haskell.

As for myself, I enjoyed implementing both of these solutions equally well. I hope you enjoyed the ride too!

This concludes our study of how catamorphism translates into Clojure. The next post will be dedicated in trying to do the same exercise in C++, a much more mainstream language than our previous candidate.
