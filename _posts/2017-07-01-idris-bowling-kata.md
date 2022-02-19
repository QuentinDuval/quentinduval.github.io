---
layout: post
title: "Idris dependent typing challenge: Bowling Kata"
description: ""
excerpt: "Let's implement an entirely type safe bowling game, where invalid games cannot even be instantiated, and see how it goes and if it is even desirable."
categories: [functional-programming]
tags: [Functional-Programming, Haskell]
---

The [1.0.0 of Idris](https://www.idris-lang.org/idris-1-0-released/) has been released just a few months back, just enough to start trying out the language and some of the possibilities dependent typing offers.

In this post, we will look at implementing a type safe version of the [Bowling Kata](http://codingdojo.org/kata/Bowling/) in Idris, to see how far we can go with the Idris type system.

## The rules

The goal of the original [Bowling Kata](http://codingdojo.org/kata/Bowling/) is to implement a function that allows to compute the rules of the [Ten-Pin Bowling game](https://en.wikipedia.org/wiki/Ten-pin_bowling). Reading the problem description, we see it does **not** include any validity checks:

* _"We will not check for valid rolls"_
* _"We will not check for correct number of rolls and frames"_
* _"We will not provide scores for intermediate frames"_

We will extend this kata by adding the validity checks, and try to implement them at the type level. Our goal will be to make **invalid bowling game not representable**.

Here are the constraints we will implement, all at the type level:

* A **roll** cannot knock down more than 10 pins.
* A **frame** cannot knock down more than 10 pins.
* A valid game contains exactly **10 frames**.
* There are exactly 2 bonus rolls for a **strike**, 1 for a **spare**, and 0 otherwise
* Rolling a strike in the first **bonus roll** gives another 10 pins to roll against
* Outside of a strike, a roll cannot knock down more than 9 pins

Let's do this as a challenge, to see where it leads us. We will discuss at the end whether or not going that far was a good idea.

## Describing a valid frame

A game of Ten-Pin Bowling rules consists of 10 frames. At each frame, a player attempt to knock down the 10 pins.

* Should he succeeds in 1 roll, he gets a strike.
* Should he succeeds in 2 rolls, he gets a spare.

In this section, we will try to capture the nature of a frame in terms of types. Our goal will be to make an invalid frame not representable

### One roll or two rolls

The first distinction we have is that a frame consists either of 1 roll (for a strike) or 2 rolls in any other case. We will represent this using a sum type.

```haskell
data Frame : Type where
  TwoRolls : (x, y : Nat) -> Frame
  Strike : Frame
  
roll : (x, y: Nat) -> Frame
roll x y = TwoRolls x y

strike : Frame
strike = Strike
```

* We use `Nat` to make sure the pins knocked down are positive
* We added _smart constructors_ to decoupled the instantiation (potentially exporting the smart constructors and not the data constructors)

We add the notion of a spare though a predicate on our frame:

```haskell
PinCount : Nat
PinCount = 10

isSpare : Frame -> Bool
isSpare (TwoRolls x y) = x + y == PinCount
isSpare _ = False
```

This implementation is however **not** type safe: any positive number will do. We might create a frame in which a player knocked 25 pins in one single roll.

### Important properties involve multiple fields

To make sure we can only construct valid `TwoRolls` frames, we need a pre-condition that relates both numbers given to the data constructor.

Constraining each number individually is not enough. The best we could do is making sure that both numbers do not exceed 9 (included) individually. But it would not prevent the creation of frame with two rolls knocking 9 pins each.

In general, **the most interesting properties involve several arguments** (or fields). Focusing on single value type-safety only checks the most trivial properties, for sometimes only a deceiving **illusion of safety** (yet this is the focus of most type systems).

### Proving a frame is valid

To make sure we can only create valid Bowling frame, we will need to express a property on both the rolls of the `TwoRolls` type constructor:

```haskell
data Frame : Type where
  TwoRolls : (x, y : Nat) -> { auto prf : ValidFrame x y } -> Frame
  Strike : Frame
```

This syntax is asking Idris to find an implicit proof (the `auto prf` syntax where `prf` is the variable name of the proof), that both our numbers `x` and `y` obey the property `ValidFrame` (described below).

A `ValidFrame` is such that both rolls are below 10 (_excluded_, or else it would be a strike) pins knocked down, and that the sum of pins knocked down is below 10 (_included_, to take into account the possibility for spares).

### Propositions as types

Prooving something in the sense of Idris, is being able to create an instance of a given type (if you are unclear why being able to instantiate a type is proving the proposition defined by the type, you should watch [“Propositions as Types” by Philip Wadler](https://www.youtube.com/watch?v=IOiZatlZtGU)).

In Idris, the type that represents “lower than” is `LT` and the type that represents “lower than or equal” is `LTE`. So a `TwoRolls x y` is valid if we can instantiate the following types:

* `LT x 10`: the first roll is strictly lower than 10
* `LT y 10`: the second roll is strictly lower than 10
* `LTE (x + y) 10`: the sum of both rolls is lower or equal than 10

We need to express the conjunction of all these propositions. To do so, we need to prove that we can construct all the types that represent these propositions.

A **tuple represents a conjunction**: to instantiate a tuple, we need to instantiate all the types it contains. We can therefore express our proof that a frame is valid as follows:

```haskell
PinCount : Nat
PinCount = 10

ValidFrame : (x, y: Nat) -> Type
ValidFrame x y = (x + y `LTE` PinCount, x `LT` PinCount, y `LT` PinCount)
```

Now, any attempt at constructing an invalid Bowling frame should be rejected by the type system. Here are some example in the REPL:

```haskell
roll 10 0
-- COMPILE ERROR: "Can't find a value of type (LTE 10 10, LTE 11 10, LTE 1 10)"

roll 9 9
-- COMPILE ERROR: "Can't find a value of type (LTE 18 10, LTE 10 10, LTE 10 10)"

roll 0 0 -- Fine
=> TwoRolls 0 0 : Frame

roll 5 5 -- Fine
=> TwoRolls 5 5 : Frame
```

## Describing a valid game

A game of Ten-Pin Bowling rules consists of 10 frames, with the last frame being somewhat special. In case the last frame of the game is a spare, the player gets one additional roll, if it is a strike, the layer gets two additional rolls.

### Subtelties of the strike bonus

In the case of a strike, there is some additional complexity. In case the first bonus roll is a strike, we get another 10 pins to roll against. In case your first bonus roll is not a strike, the second roll only has the remaining available pins to knock down.

Some examples of valid and invalid bonus rolls following a strike at the last frame:

```
Valid: [9, 1]
Invalid: [9, 2]

Valid: [2, 8]
Invalid: [2, 9]

Valid: [10, 8]
Invalid: [8, 10]
```

This shows that encoding the bonus rolls as a vector of positive integer bounded by 10 (the type `Fin 11` in Idris) will not cut it. We need to do a better than that.

### Specifying the bonus

We will start by writing a function that returns the number of bonus rolls given the last frame. It returns 2 for a strike, 1 for a spare and 0 otherwise:

```haskell
bonusRolls : Frame -> Nat
bonusRolls Strike = 2
bonusRolls rolls = if isSpare rolls then 1 else 0
```

We can use this function to compute type of the vector of bonus rolls. `BonusRollType` returns the type of the bonus roll vector with the appropriate size:

```haskell
BonusRollType : Frame -> Type
BonusRollType f = Vect (bonusRolls f) (Fin (S PinCount))
```

* The `Vect` is a list parametized by its length: `Vect 2 Int` contains exactly 2 integers
* `Fin (S PinCount)` represent positive integers bounded by `PinCount` (10) included

Used on the last frame of a bowling game, this function will compute the type of the vector of bonus rolls. This will make sure the number of bonus rolls will match the rules of the bowling game.

But this is not be enough. We need to forbid sequences such as 9 followed by 2. We will therefore need to add some preconditions.

### Properties on the bonus

The bonus rolls of a strike are valid if either the sum of the rolls is lesser or equal than 10, or if the first roll is a 10. We can express this property in types:

```haskell
ValidBonuses : Vect n (Fin (S PinCount)) -> Type
ValidBonuses {n = Z}      bonuses = LTE 0 0     -- 1) No bonuses, no constraints
ValidBonuses {n = (S _)}  bonuses =             -- 2) In case of bonuses:
  Either                                        -- Either:
    (finToNat (head bonuses) = PinCount)        -- * The first roll is 10
    (sum (map finToNat bonuses) `LTE` PinCount) -- * Or the sum is less than 10
```

* `Either` express a disjunction of proposition: you can construct a `Either a b` instance if you can instantiate either a `a` or a `b`
* `finToNat` allows to convert from a bounded positive integer `Fin Bound` to a non bounded positive integer `Nat` (`LTE` only works on `Nat`)

The conversions between `Fin` (bounded natural numbers) and `Nat` (unbounded natural numbers) make this this quite dense. Another interesting design would have been to get rid of `Fin` and encode the bounds inside `ValidBonuses`.

### Specifying the game

It is now time to wrap up our specification for a valid bowling game.

* A game is made of 10 exactly frames. We can enforce this using a `Vect 10 Frame`
* We use `BonusRollType` to compute the bonus roll vector type from the last frame
* The `ValidBonuses` precondition makes sure the strike bonus rolls are valid

```haskell
FrameCount : Nat
FrameCount = 10

data BowlingGame : Type where
  MkBowlingGame :                              -- Create a bowling game from:
    (frames: Vect FrameCount Frame)            -- A vector of 10 frames
    -> (bonuses : BonusRollType (last frames)) -- The bonus rolls of the last frame
    -> {auto prf: ValidBonuses bonuses}        -- The bonus rolls precondition
    -> BowlingGame
```

We used dependent types extensively here. We see that the BowlingGame type depends heavily on its content.

### Checking our type checking

Time to make sure we did a fine work. We will fire up a REPL and try to instantiate some valid and invalid games:

```haskell
MkBowlingGame (replicate 10 strike) [10, 10] -- OK: The perfect game
MkBowlingGame (replicate 10 strike) [10, 1] -- OK: Almost perfect game
MkBowlingGame (replicate 10 strike) [9, 1] -- OK: Too bad for the last rolls

MkBowlingGame (replicate 10 strike) [10]
-- FAIL: Missing a last bonus roll (there should be 2)

MkBowlingGame (replicate 10 strike) [9, 2]
-- FAIL: Can't find a value of type Either (9 = 10) (LTE 11 10)

MkBowlingGame (replicate 10 (roll 5 5)) [10] -- OK: only spares
MkBowlingGame (replicate 10 (roll 5 5)) [9] -- OK: only spares

MkBowlingGame (replicate 10 (roll 5 5)) [10, 10]
-- FAIL: Only spares but two bonus rolls instead of 1
```

Note that testing that invalid instances are rejected is interesting, but it is even more important to test that valid instances pass the type checking. We would not want to forbid valid use cases.

## Implementating the scoring

Pretty interestingly, the code to implement the scoring of a Bowling game (which is the focus of the original [Bowling Kata](http://codingdojo.org/kata/Bowling/)) ends up being shorter than the code needed to implement the rigorous type safety.

### View: a series of rolls

To solve the puzzle elegantly, we need two views on our data: one as a sequence of frames and one as a sequence of rolls. The latter will be used when dealing with strikes and spares.

We therefore need a function gameRolls to transform our game to a list of rolls:

```haskell
knockedPins : Frame -> List Nat
knockedPins (TwoRolls x y) = [x, y]
knockedPins Strike = [PinCount]

gameRolls : BowlingGame -> List Nat
gameRolls (MkBowlingGame frames bonuses) =
  concatMap knockedPins frames ++ toList (map finToNat bonuses)
```

Aside from being helpful for the computation of the score, this function is useful in a more general context. We illustrate how it behaves below:

```haskell
gameRolls $ MkBowlingGame (replicate 10 (roll 5 5)) [9]
[5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 9] : List Nat
```

### Scoring a game

Implementing the score function makes use of both our frames-based view and our rolls-based view. We recur on both these view at the same time, dropping one frame and one or two rolls at each iteration.

```haskell
score' : Nat -> Vect n Frame -> List Nat -> Nat
score' current [] _ = current
score' current (f :: fs) rolls =
  let frameRollNb = length (knockedPins f)      -- Nb of rolls in the frame
      scoreRollNb = frameRollNb + bonusRolls f  -- Nb of rolls to sum
      frameScore = sum (take scoreRollNb rolls) -- Sum of the rolls to sum
  in score'                       -- Recur:
       (current + frameScore)     -- Add the frame score to the current score
       fs                         -- Advance the the next frame
       (drop frameRollNb rolls)   -- Advance to the next rolls

score : BowlingGame -> Nat
score game@(MkBowlingGame frames bonus) = score' 0 frames (gameRolls game)
```

We can check that our code works correctly by implementing a series of unit tests. I only listed two of them below:

```haskell
should_be_300_for_perfect_game : IO ()
should_be_300_for_perfect_game =
  assertEq 300 $ score $ MkBowlingGame (replicate 10 strike) [10, 10]

should_be_150_for_5_pins_only_game : IO ()
should_be_150_for_5_pins_only_game =
  assertEq 150 $ score $ MkBowlingGame (replicate 10 (roll 5 5)) [5]
```

Note that we did not have to perform any defensive checks in our scoring function. Our type-safety has already got rid of any kind of badly formed inputs. The code instead focuses on the **nominal case**.

## Conclusion

Using Idris, we managed to build a type-safe variant of the [Bowling Kata](http://codingdojo.org/kata/Bowling/), making sure than invalid Bowling games are not representable. The full code is available in this [GitHub repo](https://github.com/QuentinDuval/IdrisBowlingKata/tree/master), in the file [Bowling.idr](https://github.com/QuentinDuval/IdrisBowlingKata/blob/master/Bowling.idr).

Now let's take a step back.

### The good parts

Idris allows to check quite complex properties at the type level. These properties can capture quite a lot of our **domain invariants**.

Where most type systems are limited to single argument or field validity, Idris is not. We can use this ability to ensure lots of invariants, and not only the most trivial ones (as type systems such as the one of Java do).

Also, since Idris blurs the limit between the type system and run-time values, our work at the type level can often be reused in our run-time code. For instance, `bonusRolls` is used to both compute the score and the type of the vector holding the bonus rolls.

### The price to pay

First, we have to pay long **compilation times**. The simple module here takes several seconds to compile on my laptop. And I could observe things getter slower as I added more properties on the bowling game types.

The second issue is the cost of dependent typing in terms of **flexibility**. The scoring is so coupled to the notion of valid game that we cannot use it to compute the intermediary scores anymore (the bowling kata does not deal with intermediary score either).

Finally, and for all cases in which the inputs are coming from the real world (like a bowling game inputs), making invalid states not representables does not dispense us to implement the run-time checks of validity of our inputs (*). Dependent types however make sure that these checks take place.

_(*) These run-times checks will differ from the usual run-time checks in that they will have to provide a proof. A simple boolean will not do._

### Strong typing is not for everything

Due to the additional efforts needed and the potential inflexibility implied by that much type safety, a valid question is whether or not dependent typing is worth it.

Richard Eisenberg arguments about reserving them to the core business logic in this [Haskell Cast](http://www.haskellcast.com/episode/014-richard-eisenberg-on-dependent-types-in-haskell). It looks like a real good advice: dependent types are not for everything, abusing them certainly can lead to code that is unnecessarily rigid and more complex.

The good news is that **Idris does not forces us to dependently type everything**. Ultimately, this is our responsibility to decide on the appropriate use of those features.

### Consider alternatives

Last but not least, we should also note that there are other ways than dependent typing to get very strong type safety, some of these being less expensive. For instance, we could use [phantom types](https://wiki.haskell.org/Phantom_type) to **force a workflow** in which we would validate our Bowling Game before scoring it.

Overall, **data-flow typed-enforced designs** seem more appropriate for cases like the bowling game: we know that the inputs will come from the real world, we know they will require validation, so having a design that matches the real world acquisition process seems much more robust.
