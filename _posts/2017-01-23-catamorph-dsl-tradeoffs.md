---
layout: post
title: "Catamorph your DSL (Trade-offs)"
description: ""
categories: [functional-programming]
tags: [Functional-Programming, Haskell]
---

This post is a third of the series of post dedicated to notion of Catamorphisms and its application to build Domain Specific Languages (DSLs).

Our first post introduced an Arithmetic DSL that allowed to build expression composed of operations like addition and multiplication integer constants and integers variables. We built several interpreters on top of it, to print, evaluate or optimize our arithmetic expressions. But then we faced an issue: our AST manipulation function could not be composed efficiently.

Our second post introduced the notion of Catamorphism as a way to solve the composability problem we identified in our first approach. Catamorphism allowed us to decouple our operations on the AST from the recursion. This powerful technic allowed us to specify our AST transformation and combine them before traversing our AST.

We have seen some of the benefits of using Catamorphism. Today’s post will discuss the trade-offs associated with it.

## Complexity through un-familiarity

The first obvious drawback to the introduction of catamorphism is the increase in complexity they bring.

The addition of a template parameter surely makes our expression noisier. The fixed point of a recursion and the notion of Catamorphism themselves are clearly not straightforward.

However, we could argue that this complexity is mainly due to the pattern not being familiar. Object Oriented programming and design patterns like the Visitor are not straightforward either for most developers going out of university.

If on the contrary, we consider simplicity as defined by Rich Hickey, that means as a measure of complected-ness, Catamorphisms do simplify things quite a bit:

* The cata function is only concerned about the recursion
* The algebra function is only concerned about the transformation
* Efficient composition favours modular and testable code

## Post walk does not fit all recursions

Catamorphisms do not fit all of the recursions you might want to perform on your expression. They only deal with one recursion pattern, depth first search post-order traversal, in exchange of what they provide their benefits.

One such constraint is that catamorphisms do force to visit the whole AST of an arithmetic expression. This might not be always necessary.

For example, inside our optimize function, we might not need to visit the "(+ 1 x 2)" branch inside the expression "(* 0 (+ 1 x 2))" to deduce the output of the expression.

The laziness of Haskell might help to reduce this overhead in some particular cases, but it is important to realize that like most algorithms, Catamorphisms are specialised in solving a particular kind of problem.

This should not neither be surprising nor considered a drawback though. Although map (std::transform in C++) is more specialized than fold (a.k.a. reduce or std::accumulate), we use extensively. Using a more specialized algorithm declares our intent more effectively. The next developer might even thank you.

## Termination guaranty

One nice thing about catamorphisms is that this recursion scheme is guaranteed to terminate for non-pathological cases (pathological cases include passing to cata a non terminating algebra or an infinite expression to traverse).

On the one side, this is quite an interesting property. We can forget about messing up the termination condition of our recursion and contributing excessively to global warming. The recursion is indeed managed by the catamorphism in a very safe way.

On the other side, it also sets some strict limits to what we can express with this recursion scheme. Some interpreters for some DSL might need to run something indefinitely.

For example, a LISP interpreter might need to be able to express things like a server loop. This is one limitation to keep in mind when choosing a recursion scheme.

## Missing tail recursion

Another limitation of Catamorphisms is that this recursion scheme is not tail recursive. In particular, the implementation we provided will consume one additional layer of stack at each recursion.

This problem can however be mitigated by using _continuation passing style_ in our implementation of cata. The advantage of having factored the recursion out is that we can tweak at will and affect all its usage, instead of having to chase down each recursion in our software to fix it.

Here is one implementation of catamorphism using continuation passing style:

```
import Control.Monad.Cont

cataCps :: (Traversable f) => (f a -> a) -> Fix f -> a
cataCps algebra expr = runCont (recur algebra expr) id

recur :: (Traversable f) => (f a -> a) -> Fix f -> Cont a a
recur algebra (Fix expr) = do
  sub <- sequence $ fmap (recur algebra) expr
  return (algebra sub)
```

The Control.Monad.Cont monad will get rid of the stack consumption, by chaining functions instead of recursing. But our expression now needs to be an instance of the Traversable typeclass, a strictly more demanding requirement than Functor.

This is not a problem though. We can ask GHC to generate an instance for us:

```
{-# LANGUAGE DeriveFunctor, DeriveFoldable, DeriveTraversable #-}

data OpType = Add | Mul deriving (Show, Eq, Ord)

data ExprR r
  = Cst Int
  | Var Id
  | Op OpType [r]
  deriving (Show, Eq, Ord, Functor, Foldable, Traversable)
```

In the end, I do not think tail recursion is a real drawback of catamorphisms. Having extracted and isolated the recursion allows us to make sure we have an efficient of data that does not need to be tail recursive. Ensuring the same level of quality would be much harder if all recursion where done manually.

## Conclusion and what’s next

Through this post, we discussed some of the tradeoffs related to the use of the catamorphism recursion scheme.

The main point to be aware of is that Catamorphisms embodies one specific recursion scheme that might not fit our problem. As a general rule, we should refrain from using a solution before having identified the problem. In particular, Catamorphisms allow to easily express terminating interpreters, which is both an interesting guaranty and a hard limitation.

The next and final two posts will conclude our study of Catamorphism by looking at how this concept translates in two other languages: Clojure and C++.
