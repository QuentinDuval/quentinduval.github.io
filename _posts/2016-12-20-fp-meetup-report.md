---
layout: post
title: "Meetup Report: The philosophy of FP"
description: ""
excerpt: "My meetup report on a talk by Arnaud Lemaire introducing the philosophy behind functional programming."
categories: [functional-programming]
tags: [Functional-Programming, Haskell]
---

Yesterday, I attended a [meetup](https://www.meetup.com/Crafting-Software/events/236062267/) at [Arolla](http://www.arolla.fr/), where Arnaud LEMAIRE ([@lilobase](https://twitter.com/Lilobase)) did a wonderful talk on the philosophy of Functional Programming. You can find the slides of the presentation [here](https://speakerdeck.com/lilobase/philosophie-fonctionnelle-meetup-craftingsw-paris-2016).

The presentation was an introduction to the concepts behind Functional Programming. It went over the pillars of FP, before concluding with practical advices on how to use this knowledge in our (most probably object-oriented) code base.

Arnaud summarized complex ideas into simple definitions and managed to find analogies and make interesting parallels that made these concepts easily understandable.  So I thought it would be a good idea to write down here what notes I took, and share it with you in this post.

## What is a state?

The result of combining a value with time. This is how you get variables and overall, this is a good recipe to get bugs.

The functional paradigm language is about getting rid of (or more realistically controling) this complexity by getting rid of time (or more realistically introducing an **explicit model of time**), and play with immutable values instead. The trick is to avoid variables in large portion of our programs, and create new values for each *modification* we would do in imperative programming style.

Creating these new values can be done very efficiently. Persistent data structures are based on structural sharing, the ability to re-use part of the old data value to construct the new one.

But **persistent data structures** do not make the cost of creating new values disappear entirely. They do consume more memory than their respective counterparts. They also add indirections that may cost in terms of computation time. This cost can often be offset by the whole range of possible optimization they make possible in exchange. In particular:

* Much easier use of parallel or distributed computations
* No need to use defensive locking mechanisms

## What are pure functions?

Pure functions can be seen as **delayed values**: they just miss some arguments to be able to deliver their value. Give them the same arguments twice, and they output the same value. This has important consequences:

**Memoization**: Instead of recomputing the values of a pure functions called with the same argument, we can keep the result in store for the next time it is called. Pure functions make caching almost trivial.

**Lazy evaluation**: We can emulate the lazy evaluation of languages like Haskell by wrapping a computation inside a lambda that takes no arguments. Forcing the computation is just calling that closure with no arguments.

**First class functions**: If functions are like values, there is no reason you could not manipulate them as values. In particular, it means having functions returning functions, and functions taking functions as parameters, both of which are called higher-order functions.

## Objects and closures

OOP is based on smart data structures, classes that carry with them the functions that operate on them. Functional Programming is based on simpler data structures: functions that operate on the data structure are moved outside of it.

The FP view makes for easier composition of functions, especially if you see the functions as transformation between two sets of values. Chaining functions becomes function composition, and can be done without utilizing side effects.

Arnaud ended this sections with a wonderful comparison of Closure and Objects. He pointed out that **Closures can be seen as the inverse of Objects**:

* Objects are data that carry some functions with them
* Closures are functions that carry some data with them

## On type systems

Functional programming is usually based on structural typing, whereas Object Oriented Programming is usually based on nominative typing.

[Structural typing](https://en.wikipedia.org/wiki/Structural_type_system) checks if types are equivalent by matching their structure. If two values have a similar structure, they can be considered of the same type.

[Nominative typing](https://en.wikipedia.org/wiki/Nominal_type_system) checks if types are equivalent by matching their “name”. If two values have the name class name, they are considered of the same type.

As an example of structural typing, if a coordinate is made of two doubles, and if a function expects a coordinate, providing it two doubles will work fine. In the case of nominative typing, this would not type-check: you would need an explicit coordinate data structure to be instantiated.

Structural typing is more loose and flexible than nominative typing. It makes it easier to build generic functions that can apply on different data structures. But it comes at a price: nominative type systems are better at ensuring correctness of your program.

In reality, this dichotomy is not as strict as it may sound. Some languages reinforce the structural typing by putting a name on the structure. This ensures the same correctness guaranty as nominative type systems, but provides a greater expressiveness.

## Pattern matching and Polymorphism

Pattern matching in Functional Programming can be seen as the opposite of polymorphism in Object Oriented Programming:

* Interfaces in OOP are closed under the addition of new functions, but not on a addition of new types. The dispatch is based on the name of the type.
* Algebraic data types are closed under addition of new types, but open in terms of addition of new functions. The dispatch is based on the structure.

We can see there a reference to the [Expression Problem](https://en.wikipedia.org/wiki/Expression_problem) coined by [Philip Wadler](https://en.wikipedia.org/wiki/Philip_Wadler). Both use cases exists: for example in Haskell, you can switch to *typeclasses* when you need to be open to new types and closed to new functions.

This is why Arnaud talked about not being a zealot and advised against the strict segregation of paradigms.

## Practical advices

The talk ended by a collection of practical advices for all of us that do not program into a natively functional programming language. These pieces of advice were especially useful since most of attendees did not have access to languages like Haskell or Clojure at their workplace:

* Try to use immutability as much as you can: use const everywhere you possibly can. This will catch a lot of bugs.
* Use algorithms like map, filter and reduce to avoid states in your looping constructs. It makes the code more declarative and less error prone.
* Try to use Algebraic Data Types. They are available in most languages: JS, Python, `std::variant` in C++, etc.
* To handle concurrency, try to rely on concepts like actors to avoid the synchronization nightmares (dead-lock and other performances issues).
* Use concepts like Functional Core, Imperative Shell to make your business logic code as pure as possible.

