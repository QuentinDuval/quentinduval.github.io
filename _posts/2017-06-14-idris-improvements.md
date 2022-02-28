---
layout: post
title: "10 things Idris improved over Haskell"
description: ""
excerpt: "The 1.0.0 of Idris has been released just a few months back, just enough to start trying out the language and some of the possibilities dependent typing offers"
categories: [functional-programming]
tags: [Functional-Programming, Haskell]
---

The [1.0.0 of Idris](https://www.idris-lang.org/idris-1-0-released/) has been released just a few months back, just enough to start trying out the language and some of the possibilities dependent typing offers.

But this post is not about dependent typing. There is already a really good book that came out this year, named [Type Driven Development with Idris](https://www.manning.com/books/type-driven-development-with-idris), exploring the benefits (and trade-offs) of dependent typing. I highly recommend you to read it.

Instead, this post describes some of the pleasant surprises you get trying out Idris, coming from the Haskell world. These pleasant surprises have nothing to do with the dependent typing features. They are simple yet impacting modifications, which improve the developer experience substantially.

I listed my top 10 in the hope they will convince you to give a try at Idris.

## 1) Idris strings are not lists

It does not seem like much, but Haskell Strings are such a pain to deal with, that this improvement got the first place.

Haskell String are lists of characters. This big mistake in terms of efficiency is corrected by two very good packages: [Data.Text](https://hackage.haskell.org/package/text-1.2.2.2/docs/Data-Text.html) and [Data.ByteString](https://hackage.haskell.org/package/bytestring-0.10.8.1/docs/Data-ByteString.html). Nevertheless, this is still a source of pain and accidental complexity:

* Newcomers to Haskell almost always face the inefficiency of Strings
* Strings are not first class, and forces use the [Overloaded Strings](https://ocharles.org.uk/blog/posts/2014-12-17-overloaded-strings.html) extension
* The code gets obscured by successive by pack and unpack calls

Idris learned from the mistakes of Haskell and made Strings first class. You can call pack and unpack strings to go back an forth to a List Char representation. This is illustrated by this simplistic implementation of the [caesar cipher](https://deque.blog/2017/06/14/10-things-idris-improved-over-haskell/caesar%20cipher) algorithm:

```haskell
caesar_cipher : Int -> String -> String
caesar_cipher shift input =
  let cipher = chr . (+ shift) . ord
  in pack $ map cipher (unpack input)

caesar_cipher 1 "abc"
=> "bcd"

caesar_cipher 0 "abc"
=> "abc"
```

## 2) Overloading without Type Classes

Overloading of function has also been a subject of numerous complains in the Haskell community. Of course, we can workaround the issue by using Type Classes, but this has some drawbacks as well.

Idris learned from these complains as well, and offers a kind of overloading. You can import the same name from different modules and Idris will not care, as long as it can deduce which one to use.

Even better, Idris introduce the notion of **namespace**, that we can use inside a module to introduce overloads. For instance, we can create a plus operator that works for both scalars and lists in the same module.

```haskell
infixr 5 +.

namespace Scalar
  (+.) : Int -> Int -> Int
  x +. y = x + y

namespace Vector
  (+.) : List Int -> List Int -> List Int
  xs +. ys = zipWith (+) xs ys
```

As long as Idris can deduce which overload to use, the developer does not have to prefix the symbol with the namespace:

```haskell
3 +. 5
=> 8 : Int

[1, 2, 3] +. [4, 5]
=> [5, 7] : List Int
```

This is a very nice improvement over Haskell rules that forbid any kind of overloading, and has nice benefits on the code:

* It improves composability of software (by limiting conflicts of names)
* It avoids using awkward operators to avoid ones used by other libraries
* It remains safe and clear, by supporting explicit namespace prefix

## 3) Record fields are namespaced

The notion of namespace of Idris really shines when it comes to records. The fields of a record live in the namespace of their respective record. The net effect is that they do not collide with the names of the other records as they do in Haskell.

So we can have several records with the same field name. For instance, we define below an `Account` and `Customer` record, each having an `address` field:

```haskell
record Account where
  constructor MkAccount
  accound_id : String
  address : String

record Customer where
  constructor MkCustomer
  name : String
  address : String
  account : Account
```

As for the overloading rules and namespace, Idris will try to infer which function is being referred to when using the name `address`. Most often, the developer does not have to provide any hint.

Here is an example of function that uses both the customer address and account address to check if they both match:

```haskell
custom_billing_account : Customer -> Bool
custom_billing_account customer =
  address customer /= address (account customer)
```

This type-checks correctly. In case of ambiguous calls, you can always add the namespace as prefix to help Idris figuring your intent:

```haskell
custom_billing_account : Customer -> Bool
custom_billing_account customer =
  Customer.address customer /= Account.address (account customer)
```

## 4) Record update and access syntax

Updating values in nested record does come at the cost of verbosity in Haskell. There are packages that help navigating data structure, such as lenses, but a lot of Haskell users recognise it as a pretty big dependency.

Idris offers a lighter update syntax for the fields of nested records. We can update the address of the account of a customer that way:

```haskell
update_billing_address : Customer -> String -> Customer
update_billing_address customer address =
  record { account->address = address } customer
```

Idris also offers the possibility to apply a function over a field in a nested records. For instance, we can complete (by concatenation) the address of the account of the customer with `$=`:

```haskell
concat_billing_address : Customer -> String -> Customer
concat_billing_address customer complement =
  record { account->address $= (++ complement) } customer
```

We can try both our previous function in the REPL to convince ourselves that they work properly (and they do):

```haskell
john : Customer
john = MkCustomer "John" "NYC" (MkAccount "123" "NYC")

update_billing_address john " LND"
=> MkCustomer "John" "NYC" (MkAccount "123" "LND") : Customer

concat_billing_address john " LND"
=> MkCustomer "John" "NYC" (MkAccount "123" "NYC LND") : Customer
```

This does not replace the need for lenses (which are much more general). But it is nice to count of some basic support in core Idris when we do not need that much power.

## 5) Monad and Functor got fixed

The Haskell Monad `return` is absent of Idris, and so is the fail method, which most Haskell developers recognise as being a design mistake.

In Idris, Monad extends the `Applicative` (as it does now in Haskell, but did not for quite a long time) and so `pure` makes sure that `return` is not needed. Since the naming was conflicting with the *return statement* of many programming languages, this is one less distinction to explain to newcomers to Idris when compared to Haskell.

You can also notice the presence of `join` in the interface of the Monad, allowing to define whichever of `bind` or `join` is more intuitive depending on the Monad instance:

```haskell
Idris> :doc Monad
Interface Monad

Parameters:
    m

Constraints:
    Applicative m

Methods:
    (>>=) : Monad m => m a -> (a -> m b) -> m b
    join : Monad m => m (m a) -> m a
```

Looking at the `Functor` type class, `fmap` is renamed `map` in Idris. This also improves over Haskell, by avoiding the awkward explanation to newcomers to Haskell of why map is in fact named fmap for legacy reasons:

```haskell
Idris> :t map
map : Functor f => (a -> b) -> f a -> f b
```

## 6) Laziness is opt-in, not opt-out

There are plenty of good reasons for laziness, there is no denying it. Reading the amazing [Why Functional Programming matters](https://www.cs.kent.ac.uk/people/staff/dat/miranda/whyfp90.pdf) shows how much it helps modularising and decoupling our software, by allowing the separation of processes producing data from the one consuming it.

But there are plenty of drawbacks to laziness as well. Some are technical (space leaks) and some are psychological (non strict evaluation is not intuitive when coming from other languages).

A bit like Clojure and other FP languages did before, Idris embraces opt-in laziness, with a strict evaluation by default. Streams are the lazy equivalent of Lists and the language features both. You can notice the different pretty easily, when playing with infinite and finite ranges:

```haskell
Idris> :t [1 .. 10]
enumFromTo 1 10 : List Integer

Idris> :t [1 ..]
enumFrom 1 : Stream Integer
```

## 7) A smaller Num class

For all of those who likes to play with DSL in Haskell, the fact that the [Haskell Num class](http://hackage.haskell.org/package/base-4.9.1.0/docs/Prelude.html#t:Num) is split into smaller bits in Idris is a small but nevertheless enjoyable improvement:

```haskell
Idris> :doc Num
Interface Num
    The Num interface defines basic numerical arithmetic.

Parameters:
    ty

Methods:
    (+) : Num ty => ty -> ty -> ty
    (*) : Num ty => ty -> ty -> ty
    fromInteger : Num ty => Integer -> ty

Child interfaces:
    Integral ty
    Fractional ty
    Neg ty
```

The Haskell version often leads to dummy implementations of the `negate`, `abs` or `signum` methods. This should not occur anymore in the Idris version of the `Num` class.

## 8) The Cast type class

Idris removed the `Read` type class and replaces it with the more general `Cast` type class. The `Read` type class allowed to parse a String into a type `a`, effectively a transformation from a String to any other type. The `Cast` class is parameterized on the input type as well, allowing to transform between any two types implementing the type class.

```haskell
Idris> :doc Cast
Interface Cast
    Interface for transforming an instance of a data type to another type.

Parameters:
    from, to

Methods:
    cast : Cast from to => (orig : from) -> to
```

We can read a string into an integer like follows (you can note that reading an invalid string returns a default value instead of throwing an exception):

```haskell
the Int (cast "abc")
=> 0 : Int

the Int (cast "123")
=> 123 : Int
```

The generalisation allows to avoid the multiplication of functions to convert between the numeric types. For instance, we can use it to convert (with losses) a double into an integer value:

```haskell
the Int (cast 1.1)
=> 1.0 : Int
```

## 9) Clear separation of evaluation and execution

Let us write a simple function that reads the input from the command line, interprets it as a number, double this number, before printing it on the screen:

```haskell
double_it : IO ()
double_it = do
  input <- getLine
  let n = the Int (cast input)
  printLn (n * 2)
```

If you evaluate a call to this function in the Idris REPL, you will get the following big pile of Abstract Syntax Tree, which correspond to the recipe to execute the IO expression described by `double_it`:

```haskell
[*src/InOut> double_it
io_bind (io_bind prim_read
                 (\x =>
                    io_pure (prim__strRev (with block in Prelude.Interactive.getLine', trimNL (MkFFI C_Types String String)
                                                                                              (prim__strRev x)
                                                                                              (with block in Prelude.Strings.strM (prim__strRev x)
                                                                                                                                  (Decidable.Equality.Bool implementation of Decidable.Equality.DecEq, method decEq (not (intToBool (prim__eqString (prim__strRev x)
                                                                                                                                                                                                                                                    "")))
                                                                                                                                                                                                                    True))))))
        (\input =>
           io_bind (prim_write (prim__concat (prim__toStrInt (prim__mulInt (prim__fromStrInt input) 2)) "\n"))
                   (\__bindx => io_pure ())) : IO ()
```

To execute the function and not just evaluate it, you need to say so explicitly, by using :exec as prefix to the expression to execute:

```haskell
[*src/InOut> :exec double_it
123 -- My input
246 -- Output
```

Why considering this an improvement over Haskell. If you are a Lisp adept, you can but appreciate the **reification** of the `IO` actions into a proper AST. It may just be personal taste, but I just love it.

## 10) The Do-notation without Monads

I kept the best part of Idris for the last. Before of its support for overloading, Idris allows the developer to define a bind operator out of the Monad class, and the `do` notation is based on the sole presence of this operator.

This is truly an awesome idea. It allows the developer to profit from the nice syntax of the do notation, without having to comply to the interface of the Monad `bind` operator.

* It avoids forcing awkward design just to fit the Monad type class requirements
* It opens up the notation for numerous types that could not be made proper instance of Functor, Applicative or Monad, because they are not parameterised
* It allows to keep the do notation for dependent types, which much rarely conform to the interface of Monad

It also allows us to abuse even more the do notation. More power means more changes to do horrors as well. Here is an example of abuse that shows that we do not even need a parameterized type for the do notation anymore:

* It describes a not very useful StringBuilder DSL
* Evaluating an expression of this DSL with `eval_string` yields a String
* Defining the bind operator allows us to describe our DSL expression with do

```haskell
data StringBuilder
  = Pure String
  | (>>=) StringBuilder (String -> StringBuilder)

eval_string : StringBuilder -> String
eval_string (Pure s) = s
eval_string (x >>= f) = eval_string (f (eval_string x))
```

Here is an example of usage of this DSL, in which we create an StringBuilder expression named john, using our beloved and dreaded do notation:

```haskell
build_john : StringBuilder
build_john = do
  john <- Pure "John"
  let john_doe = john ++ " Doe" ++ concat (replicate 10 "!")
  Pure john_doe
```

Evaluating our StringBuilder expression shows us the AST, and calling `eval_string` on it interprets it as a String:

```haskell
[*src/DoNotation> build_john
Bind (Pure "John") (\john => Pure (prim__concat john " Doe!!!!!!!!!!")) : StringBuilder

[*src/DoNotation> eval_string build_john
"John Doe!!!!!!!!!!" : String
```

## Conclusion

Idris offers some pleasant surprises for the Haskell programmers, even without considering the dependent typing support. There are many other pleasant surprises I got, such as the distinction between `print` and `printLn`, which is now aligned with `putStr` and `putStrLn`, and many others.

These changes mostly show how helpful it is to be able to start all over again from scratch and fixing these pesky mistakes we did at the beginning.
