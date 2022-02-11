---
layout: post
title: "Code your own Quick Check"
description: ""
categories: [functional-programming]
tags: [Functional-Programming, Haskell]
---

In the next posts, we will go through a small implementation challenge. The goal will be to implement our own limited version of [QuickCheck](https://hackage.haskell.org/package/QuickCheck), the famous generative testing framework of Haskell.

Whether or not you do know about the concept of generative testing, I advice you to have a look at the [original publication of QuickCheck](http://www.cs.tufts.edu/~nr/cs257/archive/john-hughes/quick.pdf). This is indeed a very good and quite enlightening read.

Our goal will be to reproduce something close to the features described in this publication, all on our own. Through this exercise, we will gain some better understanding of the magic that happens inside QuickCheck.

* The goal of today is to implement the basic features of QuickCheck
* The next post will be focused on testing higher order functions
* Our last post will be focused on shrinking counter examples

We will name our own generative testing framework RapidCheck to avoid confusion when referring to the real QuickCheck.

## Introduction

QuickCheck is a framework that allows to test properties that we think should hold on our functions. It is pretty versatile and can be used in many different contexts:

* To test that the code your wrote does behave correctly
* To investigate on the behaviour of some code you are reading
* To test specifications against a working code

QuickCheck is not a proof system and is not a replacement for example based testing either. QuickCheck is more like a nasty user which will try everything he can to break your software.

Romeu Moura did a [great talk](https://www.youtube.com/watch?v=5pwv3cuo3Qk) explaining what property-based testing is and is not about. I encourage you to have a look at it.

## Our goal for today

Our goal for today will be to implement a subset of our RapidCheck that supports checking some basic properties. We will keep the topic of testing higher order function for our next post.

More practically, we give ourselves the goal to test the two following properties on the Greatest Common Divisor with RapidCheck, all by the end of this post.

The `prop_gcd` is a valid property. Multiplying the Greatest Common Divisor of a and b with the Lowest Common Multiple of a and b should be equal to the produce of a and b. We expect this test to pass.

```
prop_gcd :: Integer -> Integer -> Bool
prop_gcd a b = a * b == gcd a b * lcm a b

rapidCheck prop_gcd
> Success
```

The `prop_gcd_bad` is an invalid property. The Greatest Common Divisor of two numbers might be equal to 1 if these numbers are coprime. We expect this test to fail (with high probability) and we expect RapidCheck to provide us a counterexample.

```
prop_gcd_bad :: Integer -> Integer -> Bool
prop_gcd_bad a b = gcd a b > 1

rapidCheck prop_gcd_bad
> Failure {seed = -1437169021,
           counterExample = ["1076253199","40866101"]}
```

How does the `rapidCheck` function knows how to test these properties? How does it recognize which kind of input to generate? These are the questions that will be answered throughout this post.

We will use the same vocabulary used inside the original publication of QuickCheck. Our implementation will differ on several aspects from the one of the publication, in the spirit of simplifying things a bit.

## Selecting our outputs

Let us start by addressing the simplest of our needs: returning useful results as output to the tests. Needless to say, because Property Based Testing is based on generating random inputs, returning “Test Failed” is not really helpful. The output should at the very least contain enough information to replay the failing scenario.

We will select two useful pieces of information in case of failure:

* The counter example itself (the failure inputs) as strings
* The random seed used to generate the counter example

This translates into the following Result type:

```
data Result
  = Success                     -- In case of success, no additional information
  | Failure {
    seed :: Int,                -- The seed used to generate the counter example
    counterExample :: [String]  -- The counter example (failing inputs to string)
  } deriving (Show, Eq, Ord)    -- Useful instances to print and compare Results
```

To help dealing with the Failure case more easily, we create the following helper function `overFailure`. It takes a function, applies it on the Result if it is a Failure, and returns the updated value.

```
overFailure :: Result -> (Result -> Result) -> Result
overFailure Success _ = Success
overFailure failure f = f failure
```

We also provide a Monoid instance for our result type. A Monoid instance is a type offering a default value, `mempty` in Haskell, and a binary associative operation, `mappend` in Haskell.

For Result, we will select a meaning that will make it easier for us to collapse several test results into one single result:

* `mempty`: running no test case always results in Success
* `mappend`: if at least one test case fails, the whole test fails

```
instance Monoid Result where
  mempty = Success
  mappend lhs@Failure{} _ = lhs
  mappend _ rhs = rhs
```

Note that our implementation is left biased: it will report the leftmost failure if both test cases fail. Applied to a list of several test cases, it will therefore report the first error found in the list.

## Generating random inputs

In order to be able to test properties over our functions, RapidCheck needs to be able to generate some random values. We model a generator of random input of type `a` by the type `Gen` parameterized on the type it generates.

This `Gen` a is a wrapper around a function that takes a pseudo-random generator (an instance of `StdGen` from the [System.Random](https://hackage.haskell.org/package/random-1.1/docs/System-Random.html) module) and returns an element `a`.

We also provide an typeclass (a kind of interface) named `Arbitrary` for types for which a standard generator is defined.

```
newtype Gen a = Gen {
  runGen :: StdGen -> a
}

class Arbitrary a where
  arbitrary :: Gen a
```

Note that the function wrapped in Gen is pure. Calling it twice with the same random generator will return the same result twice. It means that the caller of the function is responsible for providing an updated pseudo random generator to the function to get different results.

This might seem awkward, but we have to remember that our goal is to output reproducible test cases for the API user. Being able to replay a failing scenario deterministically is an important feature. A pure function is the safest way for us to guaranty this property.

## Property

The goal of our RapidCheck library is to check to properties on our code. Properties are predicates we can run generative test against, to verify if they hold. We identify several characteristics of a Property:

* We need to test it, and generate a Result from it
* This result is based on randomly generated inputs

Presented differently, a Property is a computation that return a Result through a pseudo-random generation process. It means we can model a property as a wrapper around `Gen Result`:

```
newtype Property = Property {
  getGen :: Gen Result
}
```

To avoid systematic unwrapping of the `Property`, we provide the following helper function `runProp`. It just calls the generator function with the provided pseudo random generator `StdGen`.

```
runProp :: Property -> StdGen -> Result
runProp prop rand = runGen (getGen prop) rand
```

This `Property` type might seem incomplete. In particular, there is no mention of the random inputs that drives the generation of the `Result`. Surely, this is a problem!

Not if we think of a `Property` as the result of a construction process that integrates the generation of the random inputs together with the predicate we want to verify. The actual “construction process” of a Property is the subject of the next section.

## Testable inputs

The purpose of this section is to define the inputs of our rapidCheck function. Before we go further, let us go through an inventory phase, in which we will review what we already know and express on paper what we really want.

### Inventory

Our ultimate goal is to test any property expressed as a function that returns something we could interpret as a Result. From now on, we will refer to such function as testable function. In particular, any function returning a Bool or a Result (like our prop_gcd function) should qualify.

```
prop_gcd :: Integer -> Integer -> Bool
prop_gcd a b = a * b == gcd a b * lcm a b
```

Each of the argument of a testable function should have a random generator associated with it. We already know how to express this constraint: all arguments should implement the `Arbitrary` type.

We also have a nice target abstraction for our testable functions, the type `Property` we introduced earlier. We want a way to transform such testable functions into a `Property`.

For this transformation to make sense, the generator wrapped by the `Property` should represent the combined effect of the generators of each of the arguments of the original function.

### Our problem

Our problem is that there are theoretically no limits to the number of arguments this function could take. We could try to take a shortcut and provide overloads until we reached a maximum number of arguments, but we can do better.

The solution is to break down our problem and think in terms of mathematical induction. This means finding a base case, a set of function with no arguments which we can convert to a Property, and identifying a induction step to grow the number of arguments supported one at a time.

We define `Testable` as the typeclass that represents the functions that can be transformed into a Property (our testable function, or alternatively, something we can generate a Result from):

```
class Testable a where
  property :: a -> Property
```

Our goal is identify a base case of known Testable instances, and implement an induction step to grow this set of instances.

### Base case

Based on our inventory, we know that functions with no arguments (constants) that return a result or a boolean are convertible to a property.

These constants do not have any random inputs influencing them. As a consequence, a `Property` obtained from one such function should always produce the same consistent result. This is expressed in the code below:

```
-- Property is trivially convertible to Property
instance Testable Property where
  property = id

-- Result is convertible to Property by creating
-- a generator that always return this same result
instance Testable Result where
  property r = Property (Gen (const r))

-- Bool can be converted to Result
-- Result can be converted to a Property
-- By composition we get that Bool is convertible to Property
instance Testable Bool where
  property = property . toResult where
      toResult b = if b then Success
                        else Failure { seed = 0, counterExample = []}
```

### Induction step

The inputs of our induction step are an instance of a `Testable` function, named `testable` and a printable argument implementing `Arbitrary`. Our desired output is a testable instance for the function taking the additional argument as parameter.

Let us write this desire, with the implementation left undefined:

```
instance
  (Show a, Arbitrary a,        -- Given a type `a` supporting random generation
   Testable testable)          -- And an already existing testable function
  => Testable (a -> testable)  -- The function (a -> testable) is also testable
  where
    property = undefined       -- With the provided implementation
```

Now, let us assume the existence of a function named `forAll` that takes a generator and our function `a -> testable` and return a new testable instance.

Then we could define our instance as follows, by providing the generator of our argument a to the `forAll` function:

```
instance
  (Show a, Arbitrary a,
   Testable testable)
  => Testable (a -> testable)
  where
    property f = forAll arbitrary f

forAll :: (Show a, Testable testable) => Gen a -> (a -> testable) -> Property
forAll = undefined
```

We just pushed the problem a bit further down the road. But the forAll really needs its next section for it alone.

## Implementing forAll

Before diving into the implementation of the forAll function, let us note that it is clearly more general than our Testable instance.

It does not require a type to implement Arbitrary and instead asks for a generator of argument. This function is both at the heart of QuickCheck and a very important tool when dealing with random generation of inputs.

This allows to define specialized random generator for the same argument. This in turns allows to test some properties that hold in specific equivalence classes only.

Now, to the implementation.

The code will rely on the use of the `split` function of the module [System.Random](https://hackage.haskell.org/package/random-1.1/docs/System-Random.html) module. This function allows to create two `StdGen` pseudo random generator from a single one. It is important to note it is a pure function, meaning its result is deterministic.

The `Property` created by `forAll` is a wrapper around a generator: it takes as input a `StdGen`, named `rand`, to output a `Result`. Inside the returned `Property`, we will split this pseudo random generator in two, named `rand1` and `rand2`.

* We use `rand1` to generate the new argument `arg` to the testable function
* We use `arg` to get `subProp`, which is already an instance of testable (by our induction hypothesis and the type of `forAll`)
* We use `rand2` to run `subProp` and generate our result
* In case of failure, we enrich the counter-example with the value of arg

You can find the full implementation below:

```
forAll :: (Show a, Testable testable) => Gen a -> (a -> testable) -> Property
forAll argGen prop =
  Property $ Gen $ \rand ->             -- Create a new property that will
    let (rand1, rand2) = split rand     -- Split the generator in two
        arg = runGen argGen rand1       -- Use the first generator to produce an arg
        subProp = property (prop arg)   -- Use the `a` to access the sub-property
        result = runProp subProp rand2  -- Use the second generator to run it
    in overFailure result $ \failure -> -- Enrich the counter-example with `arg`
        failure { counterExample = show arg : counterExample failure }
```

## Implementing rapidCheck

The goal of rapidCheck is to take a `Testable` instance as parameter, and try its associated `Property` with different values of seeds, in an attempt to find a counter-example.

We will first define a function `rapidCheckImpl` that takes as parameter a starting seed for our pseudo random generator, and a maximum number of attempts to try to find a counter-example:

* `runOne` will trigger one such attempt, with a seed provided as parameter. Upon failure, it will complete the error report with the seed.
* `runAll` will run all attempts and exploit the Monoid instance for Result to collapse these results together

```
rapidCheckImpl :: Testable prop => Int -> Int -> prop -> Result
rapidCheckImpl attemptNb startSeed prop = runAll (property prop)
  where
    runAll prop = foldMap (runOne prop) [startSeed .. startSeed + attemptNb - 1]
    runOne prop seed =
      let result = runProp prop (mkStdGen seed)
      in overFailure result $ \failure -> failure { seed = seed }
```

We can note that, once again, this function is side effect free. It will return a deterministic result.

We finally introduce the functions `rapidCheck` to get the non-determinism needed to solve our problem. This function produces a side-effect to get the initial seed to feed `rapidCheckImpl`. We also add `rapidCheckWith`, as a variation of `rapidCheck` that accept the maximum number of attempts as parameter.

```
rapidCheck :: Testable prop => prop -> IO Result
rapidCheck = rapidCheckWith 100

rapidCheckWith :: Testable prop => Int -> prop -> IO Result
rapidCheckWith attemptNb prop = do
  seed <- randomIO
  return $ rapidCheckImpl attemptNb seed prop
```

There they are, the only two functions with side effects of our RapidCheck module. It is quite remarkable how far we can push side effects away from the core implementation of a module whose main purpose is to generate random inputs to find counter-examples.

## Enjoying rapidCheck

Now that we did all this work, it is time to enjoy. We will run our initial test cases and have a look at how it behaves.

Both of our test cases takes two integers and returns a boolean. We know from the previous sections that a function with no parameters (a constant) returning a boolean implements Testable. What is missing for the whole function to be Testable is an Arbitrary instance for our Integer type:

* We use the pure function next of System.Random to generate a Int
* We convert this Int to an Integer (an arbitrary large number)

Because we generated a Int, the random generator will not cover all the spectrum of values an Integer could have. We will ignore this here (*), as the only reason why we use an Integer was to avoid overflows:

```
instance Arbitrary Integer where
  arbitrary = Gen $ \rand ->
    fromIntegral (fst (next rand))
```

We expect our first test case to pass as it verifies a valid property of our GCD:

```
prop_gcd :: Integer -> Integer -> Bool
prop_gcd a b = a * b == gcd a b * lcm a b

rapidCheck prop_gcd
> Success
```

Our second test case should fail with high probability. RapidCheck should be able to exhibit a counter example based on two coprime numbers. And it does:

```
prop_gcd_bad :: Integer -> Integer -> Bool
prop_gcd_bad a b = gcd a b > 1

rapidCheck prop_gcd_bad
> Failure {seed = -1437169021,
           counterExample = ["1076253199","40866101"]}
```

Note that if we had used Int instead of Integer, RapidCheck should be able to find a counter example to our valid prop_gcd properties. Because an Int has not an infinite precision, we expect RapidCheck to find integer overflow issues. And it does:

```
instance Arbitrary Int where
  arbitrary = Gen $ \rand -> fst (next rand)

prop_gcd_overflow :: Int -> Int -> Bool
prop_gcd_overflow a b = a * b == gcd a b * lcm a b

rapidCheck prop_gcd_overflow
> Failure {seed = -881134321,
           counterExample = ["171542757","1235104953"]}
```

_(*) In real code, we should avoid defining Arbitrary instances with bad statistical properties (what we did for the Integer). We always have the option to define custom generators and use forAll explicitly.

We can also note that writing a random generator for arbitrary long integers is not that easy. Thankfully, QuickCheck integrates a great deal of already defined Arbitrary instances. Practically, we do not need to worry about writing a generator of Integer._

## Conclusion and what's next

In this post, we went over the basic building block of the QuickCheck library, by implementing our own Property Based Testing framework, RapidCheck.

We used the same vocabulary as QuickCheck to better understand how the real library works. The details of implementations are not the exact reproduction of the original library though:

* The `Gen` type is not a Monad in RapidCheck, to avoid the topic entirely. It also support less feature than the QuickCheck version.
* The `Result` type is much simpler in RapidCheck to concentrate on the principles rather than trying to be exhaustive.
* As a consequence, both `forAll` and `rapidCheck` functions, although quite similar in purpose, are quite different in terms of implementation.
 
But we are not done yet. The next post will go into another magic property of QuickCheck. We will add to our RapidCheck the ability to do something quite remarkable: generating function to test higher order functions.
