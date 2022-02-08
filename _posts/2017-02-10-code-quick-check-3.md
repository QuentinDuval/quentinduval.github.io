---
layout: post
title: "Code your own Quick Check (Shrink)"
description: ""
categories: [functional-programming]
tags: [Functional-Programming, Haskell]
---

In the previous posts, we started to implement our own QuickCheck in Haskell, which we named RapidCheck, based on the [original publication on QuickCheck](http://www.cs.tufts.edu/~nr/cs257/archive/john-hughes/quick.pdf).

The first post went over the basic concepts needed to build such a module, while our last post focused on how we can write generator for arbitrary functions. You might want to read them before continuing:

* [Code you own QuickCheck]({% link _posts/2017-02-03-code-quick-check-1.md %})
* [Code you own QuickCheck (Higher Order Functions)]({% link _posts/2017-02-06-code-quick-check-2.md %})

In today’s post, we will complete the implementation of our RapidCheck module, by adding the ability to shrink counter examples to make them more manageable.

This post will present two different strategies to implement this shrinking process. The first attempt will show the simplest solution that might come into mind. Although it will achieve the desired shrinking, we will explain in what sense it is badly designed. Identifying these defects will lead us to a second, much better solution to the problem.

# Motivation

In our first post, we went over the process of building the basic blocks of our RapidCheck implementation. One of our success criteria was for our implementation to successfully find a counter example to following invalid property:

```
prop_gcd_bad :: Integer -> Integer -> Bool
prop_gcd_bad a b = gcd a b > 1

rapidCheck prop_gcd_bad
> Failure {seed = -1437169021,
           counterExample = ["1076253199","40866101"]}
```

While this counter example is perfectly valid, it is unnecessarily complex. This property could have failed with much smaller numbers:

```
gcd 1 1
> 1

gcd 0 0
> 0
```

The goal for today is to modify the implementation of our RapidCheck implementation to implement shrinking, one of the most amazing feature of QuickCheck.

At the end of this post, our implementation should be able to come up with much smaller counter examples for prop_bad_gcd:

```
rapidCheck prop_gcd_bad
> Failure {seed = 1034882204061803680,
           counterExample = ["1","0"]}
```

# Meaning of shrinking

Our goal is to provide simpler counter examples to the user of the RapidCheck module, ideally a minimal test case. But what do we mean by “simpler” or “minimal”?

The general notion has to do with information. We want to remove noise from the test case. We want to get rid of artefacts that do not participate to the error. We want our arguments to contain only the information needed to make our test case fail.

We can therefore understand shrinking as the process of removing information from our arguments until we get to the point where all the information contained in these arguments is necessary for the test case to fail.

# Enhancing Arbitrary

The first step toward being able to provide simpler counter examples is to figure out a way to reduce the amount of information of each of the arguments that lead to the property to fail.

To know where to start, we will enumerate some of the things we know and want about this shrinking process:

* Quantifying information, thus reducing it, highly depends on the considered type
* It involves search: there is no known universal solution to find a minimal test case
* Shrinking might not make sense for all values (zero) or types (functions)
* We would like equal values to shrink in a similar deterministic way

To sum up, we need a way to express an optional process that is neither random nor generic, and involves non-determinism. This is exactly what the shrink function of the Arbitrary type class has to offer:

```
type Shrinker a = a -> [a]

class Arbitrary a where
  arbitrary :: Gen a
  shrink :: Shrinker a
  shrink = const []
```

* The list allows non-determinism: each output represents a different “path”
* If the test case is already minimal, we can return an empty list
* The default implementation helps for types that do not support shrinking

# The shrink tree

To find the simplest counter arguments, several sequential calls to shrink might be needed. These recursive applications of shrink build a tree, whose root is the initial value that led to a counter example.

We will assume that this tree is built such that the children are ordered in such a way that it maximize the chances to find the smallest counter example. This assumption means we must be very careful in the ordering of the elements returned by shrink.

If this assumption holds, we can therefore explore the tree by finding the first children that makes the property fail, and dive deeper into this branch. If no children makes the property fail, the visit stops and we return our smaller counter example.

# Arbitrary example

Now that we know what is expected from the shrink function, we can enrich the Arbitrary Integer instance to provide an implementation for it. Our implementation will stay very close to the one provided in [Test.QuickCheck.Arbitrary](https://hackage.haskell.org/package/QuickCheck-2.9.2/docs/Test-QuickCheck-Arbitrary.html):

* We first try to remove the sign of negative integers
* Then we try zero, the simplest possible integer value
* Finally, we proceed with a right recursive dichotomy

```
instance Arbitrary Integer where
  arbitrary = Gen $ \rand -> fromIntegral $ fst (next rand)
  shrink n
    | n == 0 = []
    | otherwise = [abs n | n < 0] ++ 0 : rightDichotomy where
      rightDichotomy =
            takeWhile
              (\m -> abs m < abs n)
              [ n - i | i <- tail (iterate (`quot` 2) n)]
```

The behavior of this Arbitrary Integer instance is better explained through examples:

```
shrink 2048
> [0,1024,1536,1792,1920,1984,2016,2032,2040,2044,2046,2047]

shrink (-2048)
> [2048,0,-1024,-1536,-1792,-1920,-1984,-2016,-2032,-2040,-2044,-2046,-2047]
```

# Plugging the shrinking in Testable

We know our `Arbitrary` type class has a new member shrink. We would like this ability to shrink to be automatically used inside the `Testable` properties that are made of shrinkable arguments.

From our first post, we know that the `Testable` properties are defined in terms of an induction on the number of arguments. We will need to enrich this induction to include shrinking.

To do so, we add a shrinker argument to `forAll`, the function that implements the induction step. For now, we keep the implementation unchanged:

```
forAll :: (Show a, Testable testable)
          => Gen a -> Shrink a -> (a -> testable) -> Property
forAll argGen shrink prop = ...
```

We can then adapt the `Testable` induction step to call `forAll` function one more argument:

```
instance
  (Show a, Arbitrary a, Testable testable)
  => Testable (a -> testable)
  where
    property = forAll arbitrary shrink
```

We now reached the point where we need to implement the forAll function to plug all the pieces together. The next sections will present two different implementations:

* We will start by the simplest implementation (and make it work)
* We will then go for a better implementation (and make it work faster)

# First try: visiting the shrink tree inside forAll

Our first implementation will consist in slightly adapting our `forAll` function to handle the shrinking. For reference, the current implementation of this function is listed below:

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

We will enrich the lambda given to `overFailure` and inside this lambda, we will shrink the value arg of the initial counter example.

#### `Shrinking` implementation

Our shrinking process will take a function `a -> Result` to test the property against an argument of type `a`. This a will take different value as we visit the shrink tree.

Our `shrinking` function will also need the root of the shrink tree and the shrink function, to get the children of a given node. We get the following prototype:

```
shrinking :: (Show a) => Shrink a -> a -> (a -> Result) -> Result
shrinking = undefined
```

From a list of potential counter-example, we can write a search function that will return the first one that makes the property fail:

```
findFailing :: [a] -> (a -> Result) -> Maybe (a, Result)
findFailing smaller runSub =
  let results = map runSub smaller
  in find (isFailure . snd) (zip smaller results)
```

Applied to the output of shrink, the first match will provide us with the next branch of the shrink tree to explore. With that in mind, we can finish the implementation of the shrinking function:

```
shrinking :: (Show a) => Shrink a -> a -> (a -> Result) -> Result
shrinking shrink arg runSub =
  let children = shrink arg                 -- Get the children of the current branch
      result = findFailing children runSub  -- Look for the first failure
  in case result of
      Nothing -> Success
      Just (shrunk, failure) ->             -- In case a failure is found
        shrinking shrink shrunk runSub      -- Try to shrink further the child
        <>                                  -- OR (in case it fails)
        addToCounterExample shrunk failure  -- Add child to the counter example
```

#### Plugging `shrinking` in `forAll`

We want the shrunk arguments to be used in place of the original randomly generated argument, and not trigger a full random test case again. So we must use the same random generator used to initially run the sub-properties for the shrinking.

Inside forAll, we can therefore add the shrinking inside the lambda given to overFailure. The shrinking uses the same runSub function (bound to the same seed) to search for a smaller counter example:

```
forAll :: (Show a, Testable testable)
          => Gen a -> Shrink a -> (a -> testable) -> Property
forAll argGen shrink prop =
  Property $ Gen $ \rand ->             -- Create a new property that will
    let (rand1, rand2) = split rand     -- Split the generator in two
        arg = runGen argGen rand1       -- Use the first generator to produce an arg
        runSub = evalSubProp prop rand2 -- Factorize a runner for the sub-property
        result = runSub arg             -- Run the sub-property with value `arg`
    in overFailure result $ \failure -> -- In case of failure,
        shrinking shrink arg runSub     -- Attempt to shrink the counter example
        <>                              -- OR (in case the shrinking failed)
        addToCounterExample arg failure -- Add the argument to the counter example
```

This implementation makes use of the `evalSubProp` function, to get the (a -> Result) function required to explore the shrink tree:

```
evalSubProp :: Testable t => (a -> t) -> StdGen -> a -> Result
evalSubProp prop rand = (`runProp` rand) . property . prop
```

#### Resulting behavior

This implementation works in the sense that it will shrinking the counter example as we expect it would:

```
rapidCheck prop_gcd_bad
> Failure {seed = 1034882204061803680,
           counterExample = ["1","0"]}
```

But our implementation of `forAll` is deceptively simple. It looks like it tries to shrink the arguments sequentially, starting from the last argument of the property. But it is not what happens.

Our `forAll` is part of the induction process that builds a Property from a list of arguments from the last to the first of a `Testable` property. Since our `forAll` implementation now includes visiting the shrink tree, finding a counter example for an argument (this includes shrinking this same argument) also involves visiting the shrink tree of all subsequent arguments.

To illustrate this, let us take the following invalid property:

```
prop_gcd_bad :: Integer -> Integer -> Bool
prop_gcd_bad a b = gcd a b > 1
```

* Finding an initial counter example for a involves evaluating the sub-properties `prop_gcd_bad` is built from. This includes shrinking argument b.
* Shrinking a also involves re-evaluating the sub-properties `prop_gcd_bad` is built from. This also automatically includes shrinking the argument b.

So we shrunk argument b twice, and this will only grow with the number of arguments participating in the property.

So this is quite a waste. And there is nothing we can do about it: it is a natural consequence of a design that integrates visiting the shrink tree inside forAll.

# Second try: visiting the shrink tree outside `forAll`

We know from our previous design that we need to search for better counter example outside of the `forAll` function. In this second design, the `forAll` function will only be responsible to build a search tree, for the rapidCheck function to explore it.

#### Result tree

To achieve this, we will have to modify our `Property` type to return a `Result` tree instead of a single `Result`.

```
data Tree a = Tree
  { treeVal :: a
  , children :: [Tree a] }
  deriving (Functor)
  
newtype Property = Property { getGen :: Gen (Tree Result) }
```

In this design, our `rapidCheck` function is responsible for navigating the tree and seeking a better counter example (in case of failure). The only modification needed in `rapidCheckImpl` is a call to `visitResultTree`:

```
rapidCheckImpl :: Testable prop => Int -> Int -> prop -> Result
rapidCheckImpl attemptNb startSeed prop = runAll (property prop)
  where
    runAll prop = foldMap (runOne prop) [startSeed .. startSeed + attemptNb - 1]
    runOne prop seed =
      let result = visitResultTree (runProp prop (mkStdGen seed))
      in overFailure result $ \failure -> failure { seed = seed }
```

Implementing the visitResultTree function is quite straightforward. We find the first children that preserves the failure, and dive deeper into it:

```
visitResultTree :: Tree Result -> Result
visitResultTree (Tree Success _) = Success
visitResultTree (Tree failure children) =
  let simplerFailure = find (isFailure . treeVal) children
  in maybe failure visitResultTree simplerFailure
```

#### Building the result tree

Since a Property now returns a result tree, we need to adapt our forAll function accordingly to return a result tree instead of a single result.

Each argument inductively added by forAll to the Property will return a tree to the “father” forAll. For forAll to output a tree, we need a way to collapse a tree of tree of results into a tree of results.

This collapse must be very carefully designed to prioritize the shrinking of the outer argument over the shrinking of the inner arguments. Otherwise, the outer arguments would almost never be visited and thus shrunk. This is the job of joinTree:

```
joinTree :: Tree (Tree Result) -> Tree Result
joinTree (Tree (Tree innerArgResult innerArgShrinks) outerArgShrinks) =
  Tree innerArgResult
       (map joinTree outerArgShrinks ++ innerArgShrinks)
```

The creation of the tree of tree of results involves a bit of code:

* We first need to create a shrink tree of all outer argument values
* For each outer argument value, we evaluate the sub-property result tree
* We enrich the sub-property result tree with the outer argument value
* At the end, we join our tree of result tree into a result tree

```
resultTree :: (Show a, Testable t) => Shrink a -> a -> (a -> t) -> Property
resultTree shrinker arg prop =
  Property $ Gen $ \rand ->
    let shrinkTree = buildTree shrinker arg   -- Build the shrink tree
        resultTree = fmap toResult shrinkTree -- Transform it to a result tree
        toResult x =                          -- To compute a result tree
          addCounterExample x $               -- Add the outer arg to all failures
            runProp (property (prop x)) rand  -- Inside the sub result tree
    in joinTree resultTree                    -- At the end, join the result tree
```

This code makes use of the following helper functions:

* `buildTree` creates a potentially infinite shrink tree from a root value
* `addCounterExample` add a counter example across a whole result tree

```
buildTree :: Shrink a -> a -> Tree a
buildTree shrinker = build where
  build x = Tree x (map build (shrinker x))

addCounterExample :: (Show a) => a -> Tree Result -> Tree Result
addCounterExample arg = fmap (\r -> overFailure r addToFailure)
  where addToFailure f = f { counterExample = show arg : counterExample f }
```

### Wrapping up

To finish up the implementation, we only need to adapt our `forAll` to do the necessary plumbing:

* Split the pseudo random generator in two: `rand1` and `rand2`
* Use `rand1` to generate the root value of the shrink tree of the outer argument
* Use `rand2` to evaluate the next result tree of the chain

```
forAll :: (Show a, Testable t) => Gen a -> Shrink a -> (a -> t) -> Property
forAll argGen shrink prop =
  Property $ Gen $ \rand ->               -- Create a new property that will
    let (rand1, rand2) = split rand       -- Split the generator in two
        arg = runGen argGen rand1         -- Use the first generator to produce an arg
        tree = resultTree shrink arg prop -- Enrich the sub-property result tree
    in runProp tree rand2 
```

This is it: we know have an implementation that will shrink the outer arguments of a given property first, before proceeding with the inner arguments.

# Enjoying shrinking

Now that our implementation works, we can play a bit with it. We will run it on our invalid property, and check that the results are satisfying:

```
rapidCheck prop_gcd_bad
> Failure {seed = 1034882204061803680,
           counterExample = ["1","0"]}
```

We can add traces in our property to check how it behaves. The following is an excerpt of log traces of the a and b values used to find a counter example. The full file is only 63 lines long:

```
925396436 234647012
925436450 1835767207
0         1835767207
462718225 1835767207
0         1835767207
231359113 1835767207
...
4         1835767207
0         1835767207
2         1835767207
0         1835767207
1         1835767207
0         1835767207
1         0
```

# Conclusion

In this post, we went over how we could add shrinking to provide better counter examples to invalid properties.

We completed the code of our RapidCheck module with a shrinking mechanism and tried two different implementations of it. The mistakes done in the first implementation allowed us to understand the second implementation better. The full code of our RapidCheck module is available in this [GitHub Gist](https://gist.github.com/deque-blog/78c7d575505a25dbd3bcb34816e00a4d).

This second implementation is not quite the same as the one used in [QuickCheck](https://hackage.haskell.org/package/QuickCheck), due to the numerous additional features it supports. But this journey should still guide (the most adventurous of) you quite a bit in understanding the code behind the fabulous QuickCheck.

I hope you enjoyed the journey as much as I did, and that you learned something reading these last few posts.
