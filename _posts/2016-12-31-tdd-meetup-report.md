---
layout: post
title: "Meetup Report: Better tests Kata night"
description: ""
excerpt: "How to design better tests? How to code using Test Driven Development? My takeways of a very instructive meetup with Romeu Moura."
categories: [software-design]
tags: [Functional-Programming, Haskell]
---

Two days ago, I went at a [Crafting Software Meetup](https://www.meetup.com/Crafting-Software/events/236460542/) at Arolla. The goal was to join other developers and improve our skills as a group by practicing kata. The chosen Kata was the simple Fizz Buzz kata.

Together with my pair, we developed it using Haskell, following a TDD approach. Then went on improving our test coverage by using Property Based Testing. The night ended with [Romeu Moura](https://twitter.com/malk_zameth) doing a quite amazing demonstration on how he did tests and TDD.

I found the lessons valuable and this post will attempt at retranscribing them.

## Our initial solution for the Kata

Fizz Buzz is a simple program that takes a positive integer as input, and outputs a string. It should:

* Return “0” if the input is 0
* Return “Fizz” if the input is a multiple of 3
* Return “Buzz” if the input is a multiple of 5
* Return “FizzBuzz” if the input is a multiple of 3 and 5
* Echo its input as a string otherwise (example “7” for 7)

Here is the implementation of Fizz Buzz we ended with:

```haskell
fizzBuzz :: Int -> String
fizzBuzz 0 = "0"
fizzBuzz n =
  let res = fizzBuzzImpl [newRule 3 "Fizz", newRule 5 "Buzz"] n
  in if res == ""
      then show n
      else res

type Rule = Int -> String

newRule :: Int -> String -> Rule
newRule divisor out n
  | isMultiple n divisor = out
  | otherwise = ""

fizzBuzzImpl :: [Rule] -> Int -> String
fizzBuzzImpl rules n = concatMap ($ n) rules

isMultiple :: Int -> Int -> Bool
isMultiple n divisor = mod n divisor == 0
```

Here are the tests we ended up with to test this implementation:

```haskell
testCases :: [(Int, String)]
testCases = [ (0, "0"), (1, "1"), (3, "Fizz"), (5, "Buzz")
            , (7, "7"), (15, "FizzBuzz"), (100, "Buzz")]

tests :: Test
tests = TestList $ map createTestCase testCases
  where
    createTestCase (input, expected) =
      let label = "FizzBuzz of " ++ show input
      in TestCase $ assertEqual label expected (fizzBuzz input)
```

After that, we went a bit crazy and did our best to leverage property based testing to check some properties of our algorithm. We managed not to replicate the implementation of the fizz buzz in the properties (which is an error we can easily fall into). You can check the result of this experiment at the [following link](https://gist.github.com/deque-blog/868b78ec7a5272259e20c84eecdc7ade) if your are curious.

## Great lesson on How To Test

Writing test is not easy. Deriving good practices on how to test is not easy either. This section is about restituting some of the great advices from the amazing demonstration of TDD done by Romeu.

### Tell a story

Use naming to make your tests such as they read like a story. Use the name of the Fixture, of the tests, and even the parameters to help you. The ordering of the tests inside the Fixture might also be important.

For example, in Java, you could have something like this:

```java
class FizzBuzzShould {
 
  @Test
  void fizz_buzz_for(int multiple_of_3_and_5) {
    assertThat(fizzBuzz(multiple_of_3_and_5)).isEqualTo("FizzBuzz");
  }
 
  @Test
  void otherwise_fizz_for(int multiple_of_3) {
    assertThat(fizzBuzz(multiple_of_3)).isEqualTo("Fizz");
  }
 
  @Test
  void otherwise_buzz_for(int multiple_of_5) {
    assertThat(fizzBuzz(multiple_of_5)).isEqualTo("Buzz");
  }
 
  @Test
  void otherwise_echo_its(int input) {
    assertThat(fizzBuzz(input)).isEqualTo(Integer(input).toString());
  }
}
```

### Use bad faith

After adding tests, write the least amount of work needed to make the test pass. This include writing code that no-one would on purpose: special “if” cases.

*Why would we do that?* The goal is to show that the tests actually test something and are able to catch errors. If a new test is never going red, how can we ensure it is really a useful tests?

*Where does it stop?* Writing special case code can go on for quite a long time, so we need something to stop the cycle. One such rule is the rule of three which states that upon 3 code repetition, the code must be factorized.

### Make it work, then make it right

Refactor only if all tests are green. Mixing refactoring and fixing bugs might make you loose time if something bad happens.

For example, to end up the “bad faith cycle”, start by making the 3 repetitions to make the test pass. Once green, you can get rid of this DRY violation.

### Lazy naming

Delay the correct naming of your tests as late as you possibly can. Use inaccurate name at first, then revisit them at the end.

Naming your test accurately at the beginning is quite hard. Naming early might lead you to use names too close from the implementation. It might also lead you to rely on badly conceived notions that will be very hard to get rid of later.

As a consequence, naming your test “fizz_buzz_0” is fine when building the test suite: most of the tests will disappear in the end.

### Fix something

Resist the temptation of writing a very generic test that take a list of (input, output) pairs. The problem with this approach is that it does not provide much of the intent behind each the test cases. As a consequence, when a test will fail, the reader will have a hard time figuring out why.

If you look at our tests in the initial part, this is a mistake we did: our tests take a list of pairs of (input, output) and run them all.

Instead, try to fix either a given input or output, as it will help giving meaning to your test cases. A good approach is to divide by **equivalence class**. For example, all inputs that produces “Fizz” could end up in one parameterised test.

### Show the rules

Instead of writing a test that asserts that fizzBuzz of 30 is equal to “FizzBuzz”, explicit what 30 is.

For example, 30 is a shortcut for 2 * 3 * 5. Using the shortcut in the test means that the next reader will spend time figuring out the rules. Providing the rules explicitly will diminish the mental load needed to understand the test.

### Keep tests small

Avoid conditional statements and looping constructs in your tests. The test should be as simple as a single assertion.

Use parameterised testing to get rid of the loops on the inputs. Rework your design to leverage value types to keep the number of assertions limited.

## Putting these lessons to practice

The initial test suite we developed with my pair was clearly not up to the level of quality that Romeu described. We violated several of the rules:

* Our test suite is very generic and does not show its intent.
* We used shortcuts like 30 instead of 2 * 3 * 5 that show the rules
* Our test cases do not tell a story at all

Here is an attempt (that might maybe go a bit too far) to make the tests readable:

```haskell
tests :: Test
tests =
  let multiples_of_3_and_5 = [3 * 5, 2 * 3 * 5, 3 * 3 * 5]
      multiples_of_3 = [3, 1 * 3, 2 * 3, 3 * 3]
      multiples_of_5 = [5, 1 * 5, 2 * 5, 3 * 5]
      other_numbers = [0, 1, 2, 4]
  in TestList
    [ fizzBuzz `should_equal` "FizzBuzz" `for_all` multiples_of_3_and_5
    , fizzBuzz `should_equal` "Fizz" `for_all` multiples_of_3
    , fizzBuzz `should_equal` "Buzz" `for_all` multiples_of_5
    , fizzBuzz `should_do_like` show `for_all` other_numbers
    ]
```

It requires some helpers to be defined for us (fortunately, these are quite generic function we could put in a separate file):

```haskell
should_equal :: (Show t, Show a, Eq a) => (t -> a) -> a -> t -> Test
should_equal underTest expected input =
  let label = "Expects " ++ show expected ++ " for " ++ show input
  in TestCase $ assertEqual label expected (underTest input)

should_do_like :: (Show t, Show a, Eq a) => (t -> a) -> (t -> a) -> t -> Test
should_do_like underTest refFct input
  = should_equal underTest (refFct input) input

for_all :: (a -> Test) -> [a] -> Test
for_all fct inputs = TestList (map fct inputs)
```
