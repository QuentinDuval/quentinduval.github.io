---
layout: post
title: "Code your own Quick Check (Higher Order Functions)"
description: ""
categories: [functional-programming]
tags: [Functional-Programming, Haskell]
---

In the previous post, we started to build our own QuickCheck implementation in Haskell, which we named RapidCheck. We went over the basic concepts needed to build such a module, based on the [original publication on QuickCheck](http://www.cs.tufts.edu/~nr/cs257/archive/john-hughes/quick.pdf).

In particular, we explained in details and implemented the following concepts from QuickCheck:

* `Gen a`: a generator of value of type `a`
* `Result`: a type to represent the outcome of a test (success or counter example)
* `Arbitrary`: the class of types with a default `Gen` associated to it
* `Property`: a testable predicate, from which we can derive a `Gen Result`
* `Testable`: the class of types we can convert to a `Property`

In this post, we will add the support for generators of functions, which is critical to test properties expressed as higher order functions, while the next post will focus on adding the support for argument shrinking, which is critical to provide better counter examples for failing properties.

## Motivating Example

In a functional language such as Haskell, we often modularize and factorize our code through the use of Higher Order Functions. Testing these functions with generative testing requires QuickCheck to support the generation of functions.

Our goal for today will be to enrich our RapidCheck implementation with everything needed to test the following properties.

The `prop_partition` is a valid property. It verifies that partition correctly does it jobs:

* It checks that all elements on the left match the predicate
* It checks that no elements on the right matches the predicate
* It checks that no elements were lost in the process

We expect this property to pass:

```hs
prop_partition :: [Integer] -> (Integer -> Bool) -> Bool
prop_partition xs p =
  let (lhs, rhs) = partition p xs
  in and
      [ all p lhs
      , not (any p rhs)
      , sort xs == sort (lhs ++ rhs) ]
 
 rapidCheck prop_partition
 > Success
```

The prop_distributive is an invalid property. It asserts that every single functions from integer to integer is distributive over the operator plus. We expect this property to fail, and RapidCheck to provide us with a counter example:

```hs
prop_distributive :: Integer -> Integer -> (Integer -> Integer) -> Bool
prop_distributive a b f = f (a + b) == f a + f b

rapidCheck prop_distributive
> Failure {seed = 7720540227186278723,
           counterExample = ["1716301762","55093645","<function>"]}
```

You might feel quite disappointed by this counter example. The “function” string counter example is not that helpful. While this is true, understanding how to circumvent this issue goes beyond the scope of this post.

There is a [brilliant presentation](https://www.youtube.com/watch?v=CH8UQJiv9Q4) you can have a look at to better understand how the real QuickCheck manages to provide useful outputs for counter examples on functions.

## Toward generating functions

How can we even generate functions? Functions can operate on a potentially infinite number of values, and it seems hard to conceive a way to generate such a complex structure.

Fortunately, the original publication on QuickCheck gives us a nice trick to understand where to start. The trick is based on playing with types.

Our goal is to be able to create a generator of functions from `a` to `b`: `Gen (a -> b)`. We know this `Gen (a -> b)` type is just a wrapper around a function `StdGen -> (a -> b)`. By getting rid of the parentheses and reordering the parameters of this function, we get the following prototype: `a -> StdGen -> b`.

We recognize in this last expression a generator of value of type b. This means we can rewrite `Gen (a -> b)` as `a -> Gen b`.

This play with types makes use realize that we can build a generator of functions from a function returning a generator, by merely moving the application of the function inside the generator.

We name this transformation `promote`:

```hs
promote :: (a -> Gen b) -> Gen (a -> b)
promote f =
  Gen $ \rand a ->
    runGen (f a) rand
```

## Generators perturbation

The `promote` helper requires us to give it a function: `a -> Gen b`. Finding a way to instantiate such a function is the focus of this section.

We can reason that implementing a function that instantiate a generator from a value of a completely different type is not conceivable unless this function is the result of a partially applied function. So let us add an argument in front of it to see how it could help us.

By adding `Gen b`, we get the following prototype: `Gen b -> a -> Gen b`. One way to see this is as a perturbation on a random generator driven by a value of type `a`. We capture this requirement on a with the `CoArbitrary` type class:

```hs
class CoArbitrary a where
  coarbitrary :: Gen b -> a -> Gen b
```

The `coarbitrary` function is very close to what the function `promote` needs to create a generator of function. We just need to provide it a generator as first parameter.

To sum this up, it means we can make an `Arbitrary (a -> b)` instances from the two following requirements:

* `Arbitrary b`: the existence of a standard generator for the type b
* `CoArbitrary a`: the ability for the type a to perturb another generator

Then we only need to provide the generator to coarbitrary to feed it to promote:

```hs
instance
  (CoArbitrary a, Arbitrary b)
  => Arbitrary (a -> b)
  where
    arbitrary = promote (coarbitrary arbitrary)
```

## Implementing perturbations

We saw that generating a function requires each of the argument of the function to implement the `CoArbitrary` type class.

We know that for a type a to be an instance of `CoArbitrary` it needs to be able to perturb generators of any other types. Let us now look at how we can implement such a perturbation.

### General pattern

To influence a generator `Gen`, we can act on its `StdGen` parameter. This impact on the random generation process needs to be linked with the value of the type implementing `CoArbitrary`.

Applied to the integer data type, this translates into the following code, where perturb is yet to be defined:

```hs
instance CoArbitrary Integer where
  coarbitrary gen n =
    Gen $ \rand ->
      runGen gen (perturb n rand)
```

This is the general pattern. We can use it to define any `CoArbitrary` instances: the only difference will be the implementation of the `perturb` function.

### Implementing perturb

The implementation of `perturb` is critical to have good statistically properties. Matching the function prototype alone is not enough.

As an extreme example, let us imagine an implementation of `perturb` that would return its input generator unmodified. The resulting function generator would produce functions that systematically ignore all their parameters and return the same result for any inputs. Applied to our partition property, it would generate predicates that either return True on all inputs, or False on all inputs. Not that great.

To get good statistical properties, the implementation of `perturb` should instead try to map each possible value of a to a different perturbation on the generator `Gen b`.

This is the approach we have chosen for our `CoArbitrary Integer` instance, for which we provide different chains of `split` calls for each different number we get:

* The sign of the number is taken into account
* The value zero is not forgotten and does impact the generator
* The decomposition of the number into digits further drives the perturbation

```hs
perturb :: (Integral n) => n -> StdGen -> StdGen
perturb n rand0 =
  foldl
    (\rand b -> vary b rand)  -- Vary generator based on digit value
    (vary (n < 0) rand0)      -- Vary generator based on sign
    (digits (abs n))          -- Decompose a positive number in digits
  where
    vary digit rand =
      (if digit then snd else fst)
      (split rand)
    digits =
      map ((== 0) . (`mod` 2))
      . takeWhile (> 0)
      . iterate (`div` 2)
```

Although we must be careful, we are not limited: there are in practice many different valid implementations possible.

### Leveraging

We can use the pattern we saw above in combination with our implementation of `perturb` to easily build more instances of `CoArbitrary`. Since we have a way to build a perturbation from an integer, we can use it as an intermediary step.

We can map the values of a type to an integer value to quickly get a `CoArbitrary` instance for it. We can also use perturb several times in a row:

```hs
instance CoArbitrary [Int] where
  coarbitrary gen xs =
    Gen $ \rand ->
      runGen gen (foldr perturb (perturb 0 rand) xs)
```

The bottom line is that we do not need to re-invent the wheel each time we want to define a `CoArbitrary` instance.

In particular, QuickCheck offers a lot of helpers for building both `Arbitrary` and `CoArbitrary` in its [Test.QuickCheck.Arbitrary](https://hackage.haskell.org/package/QuickCheck-2.9.2/docs/Test-QuickCheck-Arbitrary.html) module. This is a good place to look before starting implementing a new instance of either of those typeclasses.

## Showable functions

We now have everything we need to run our tests cases… except for one small but important detail. Our code will not compile, due to the requirements associated with the `Testable` typeclass:

```hs
instance
  (Show a, Arbitrary a,
   Testable testable)
  => Testable (a -> testable)
  where
    property f = forAll arbitrary f
```

We need functions to implement `Show` for higher order functions to be valid instances of `Testable`. We will take a shortcut here and import the [Text.Show.Functions](https://hackage.haskell.org/package/base-4.9.1.0/docs/Text-Show-Functions.html) module which defines it for us.

The problem with this approach is that we get “function” inside our counter example, when the property does not pass. This is clearly not very informative.

QuickCheck offers the [Test.QuickCheck.Function](https://hackage.haskell.org/package/QuickCheck-2.9.2/docs/Test-QuickCheck-Function.html) to deal with this issue. It requires to transform the declaration of properties to use `Fun a b` instead of `a -> b`. In exchange for this small change in syntax, it is both able to show and shrink functions, improving our user experience by that much.

## Conclusion and what's next

In this post, we went over how we could generate functions to test properties expressed as higher order functions.

We completed the code of our RapidCheck library with all that was needed to make our original test cases pass. You can find the current state of the code of our RapidCheck available in this [GitHub Gist](https://gist.github.com/deque-blog/a498b6573e637e2d8d23f7230184bd2b).

Adding this feature allowed us to test properties on an algorithm such as partition by generating predicates for it.

Our next post will aim at finishing the work and add one last important features of Property Based Testing, the ability to shrink counter examples.
