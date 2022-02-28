---
layout: post
title: "Transforming data structures into types: an introduction to dependent typing and its benefits"
description: ""
excerpt: "A showcase of how dependent typing can help both to increase type-safety and reduce to zero the cost of change when introducing type-safety in legacy code."
categories: [functional-programming]
tags: [Functional-Programming, Haskell]
---

The [1.0.0 of Idris](https://www.idris-lang.org/idris-1-0-released/) has been released just a few months back, just enough to start trying out the language and some of the possibilities dependent typing offers.

Back in July, we implemented a [type safe bowling kata]({% link _posts/2017-07-01-idris-bowling-kata.md %}) in which we could not create a bowling game that would not satisfy the rules of the game. This was a show-off of the strong typing capabilities of [dependent types](https://en.wikipedia.org/wiki/Dependent_type).

Today’s post is not a extensive show-off of the capabilities of Idris. Instead, it is inspired from a real and recent use case, in which I wish I had a dependently typed language to support me. Through a practical use case, we will discover that dependent typing:

* Is not really more complex to use despite its sophistication
* Avoids some of the nasty headaches with less advanced type systems
* Limits the risk of rewrites when going the strong typing road

_Disclaimer: The post features Idris. This is an introduction post. As such, it is written so that the knowledge of this language is not a prerequisite, but it helps if you know a bit about Haskell-like syntax._

## The (adapted) test case

Let us say that we have a list of event to celebrate, and we want to build a small program to automatically publish some message when one of these events occurs.

### What's an event?

Each event has a **title** such as “Birthday”. This title is a short name for the event, which allows to categorize it and identify it more easily. It is represented as a simple string.

Each event also has an associated **message specification**. This is the template of the announcement of the event. It can be viewed as a string with placeholders for missing values. For instance, the message specification _“Happy Birthday: {int} years old, {string}”_ contains two placeholders.

The **placeholders each have an associated type** such as integer or string. Only a value of the corresponding type should be interpolated at this position. This avoids emitting strange messages such as _“Happy Birthday: John years old, 37”_ instead of _“Happy Birthday: 37 years old, John”_.

### Requirements

Our goal is to support the following needs:

* Pretty printing an event (for documentation) with its title and message specification
* Storing our events in a list to scan and automate tasks such as pretty printing
* Parsing the message specification as a string, such as “Happy {int} birthday {string}”
* Interpolating values inside the message specification to produce a valid announcement message

We would also like to support these needs **in a type-safe way**: it should not be possible to interpolate values of the wrong types inside a message specification. And if possible, we would like this to fail at compile time.

_Note: Without going into much details, the original use case is about constraining the schema of a JSON-like data structure, based on an identifier associated to this particular structure._

## First, catching errors at runtime

Let us start simple. We will focus on having a simple API which will catch the interpolation type errors inside the message specification at runtime. The next sections will deal with moving these checks at compile time.

### Message specification

We start by defining some types that will allow us to describe a message specification. We will call this type `Schema`. It consists in a list of schema elements which each represents part of the message.

Each **schema element** is either some known text, or a placeholder for an integer or string value. For instance, here is how we could decompose the message specification _“{int} years old, {string}”_:

```
Placeholder        Placeholder
     |                  |
     v                  v
  "{int} years old, {string}"
        ^^^^^^^^^^^^
         Known text
```

Translated in Idris, a `Schema` is an alias for a list of `SchemaElem` which can be either:

* `IntVar`: a placeholder for an integer value
* `StrVar`: a placeholder for a string value
* `StrCst`: a wrapper around a known string

```haskell
data SchemaElem
  = IntVar
  | StrVar
  | StrCst String

Schema : Type
Schema = List SchemaElem
```

To create a message specification, we just build a list of schema elements. Here is the code that corresponds to the message specification _”{int} years old, {string}”_:

```haskell
-- Equivalent of: "{int} years old, {string}"
[IntVar, StrCst " years old, ", StrVar]
```

Since it is much more convenient to describe the message specification as text, we also provide a function `parseMessageSpec` to parse the textual description into `Schema` value:

```haskell
> parseMessageSpec "{int} years old, {string}"
[IntVar, StrCst " years old, ", StrVar]
```

We now have a way to define a message specification quite succinctly.

### Event definition

An event contains both a title and a message specification. This is easily translated into an Idris record, which is analog to a data structure with automatically defined accessors on it:

```haskell
record Event where
  constructor MkEvent
  title : String
  messageSpec : Schema
```

Instantiating an event is pretty simple. We just use the constructor `MkEvent` followed by a string that describe the title, and a message specification. Here are two examples of such events:

* The birthDay message specification is described by writing the Schema manually
* The newYearEve message specification is parsed from a string

```haskell
birthDay : Event                                                            -- Declaration
birthDay = MkEvent "Happy BirthDay" [IntVar, StrCst " years old, ", StrVar] -- Definition

newYearEve : Event
newYearEve = MkEvent "New Year's Eve:" (parseSchema "Happy new year {int}")
```

Finally, accessing the members of an event instance is done by calling the accessors functions on it:

```haskell
> title birthDay
"Happy BirthDay"

> messageSpec birthDay
[IntVar, StrCst " years old, ", StrVar]
```

### Pretty printing an event

Our requirements include the need to pretty print the definition of an event. This means printing the title, followed by pretty printing the message specification.

We will use the `Show` interface of Idris (an interface to specify show the equivalent of Java’s `toString`) and provide two implementations for both a schema element and an event:

```haskell
implementation Show SchemaElem where
  show IntVar = "{int}"
  show StrVar = "{string}"
  show (StrCst s) = s

implementation Show Event where
  show e = title e ++ ": " ++ concatMap show (messageSpec e)
```

* We make use of pattern matching on SchemaElem to distinguish placeholder from know text
* We use concatMap to transform each schema element into a string and concatenate them

Going back to our birthDay event definition, we can now call show on it to pretty print it:

```haskell
birthDay : Event
birthDay = MkEvent "Happy BirthDay" [IntVar, StrCst " years old, ", StrVar]

-- In the REPL:
> show birthDay
"Happy BirthDay: {int} years old, {string}" -- Output
```

### Interpolation of values

A message specification contains **placeholder** for integer and string values. To emit an announcement message, we first need to **interpolate** integer and string values in place of the placeholders.

We will start by defining a type to represent values that can be used to fill a placeholder, which we will name `PlaceholderValue`. It contains either an integer or a string:

```haskell
data PlaceholderValue
  = IntVal Int
  | StrVal String
```

To interpolate these placeholder values inside the message specification, the `logEvent` event function will take as input an event (containing a message specification) as well as a list of placeholder values.

It will try to match the placeholders in the order in which they appear in the message specification, with the placeholders values in the order in which they appear in the list.

```
"Happy birthday: {int} years old, {string}"
                   ^                 ^
                   |                 |
                   v                 v
              [IntVal 32, StrVal "Quentin"]
```

Since this process might fail (due to arguments of the wrong type, or missing arguments), `logEvent` will return an optional value (`Maybe` in Idris) to signal it.

```haskell
-- Takes an Event + a List of PlaceholderValue as argument
-- Returns an optional String as a result
logEvent : Event -> List PlaceholderValue -> Maybe String
logEvent = ?missingImplementation
```

The implementation of this function is not that important (it is mainly pattern matching the placeholder expected type against the placeholder value) but provided [here](https://gist.github.com/deque-blog/4ab18f24db8a201a8d65344b6a5a2462) as reference.

### Examples of interpolation

Below are some examples of using our logEvent function:

```haskell
birthDay : Event
birthDay = MkEvent "Happy BirthDay" [IntVar, StrCst " years old, ", StrVar]

-- In the REPL:

> logEvent birthDay [IntVal 32, StrVal "Quentin"]    -- Values match the placeholder expected types
Just "Happy BirthDay: 32 years old, Quentin"

> logEvent birthDay [StrVal "Quentin", IntVal 32]    -- Wrong placeholder types (inverted)
Nothing

> logEvent birthDay [IntVal 32]                      -- Missing placeholder value
Nothing
```

It is safe because **wrong types for placeholder will lead to Nothing being returned** as an answer (to indicate a failure) at runtime. So we cannot construct and therefore emit bad messages.

But we could argue that these type and missing values errors are programming errors, not user input errors, and could therefore be moved at compile time. We will see how to do this in the next section.

## Evaluating the costs

Before we proceed into making our API type-safe in such as way that interpolation errors are caught at compile time, it is worth to consider the cost of doing so.

### The cost of implementation

Our needs are pretty simple. A non-technical person would expect it to be done quite quickly. Yet, and depending in the language we use, making our API type-safe might be quite expensive.

There is indeed a hidden challenge here. Ensuring type safety, while still allowing message specifications to be parsed from a string, is no easy programming task.

Take C++ for instance. Type-safety would go through hoisting our message specification to the type level. Parsing the message specification will therefore have to be done at compile-time too. This would require using either template meta programming or constexpr functions.

This can be done. [Ben Deane](https://twitter.com/ben_deane) and [Jason Turner](https://twitter.com/lefticus) showed how to parse JSON at compile time in their [“constexpr ALL the things!”](https://www.youtube.com/watch?v=HMB9oXFobJc) talk. But it is not for the feint of heart, and certainly not without a cost.

The question then becomes whether it is worth it. It is not if the cost of developing a fully type-safe implementation is higher than the cost of dealing with or correcting the errors.

### The cost of change

We also need to evaluate the cost of migrating our previously developed API into this new compile-time type safe API. Think about the refactoring needed to accommodate a single new requirement, type safety. Think about the potential cost of migrating potential existing users.

Can we even keep a single function from the previous API?

Now, it turns out that **in Idris, we do not need to change anything** of our previous API. To make it fully type safe, we **only have to add two new functions**. And the use of the word add instead of change is important here: we do not need to break anything.

## Now, with compile time checks

In our current design, interpolation errors can only be caught at runtime. Let us see how we catch them at compile time instead, without breaking our previous code.

### Overall Design

We want a way to interpolate placeholder values inside a message specification such that any missing or invalid placeholder values is caught at compile time.

To get an idea of where we want to go, let us write some client code of the API first. Here is one possible such client code, where `logEvent` is a function which:

* Takes as first argument the event to be announced
* Takes as subsequent arguments its placeholder values

```haskell
birthDay : Event
birthDay = MkEvent "Happy BirthDay" [IntVar, StrCst " years old, ", StrVar]

-- In the REPL:
> logEvent birthDay 32 "Quentin"
"Happy BirthDay: 32 years old, Quentin"
```

Now, it is important to notice that **the number and the types of the arguments** of this `logEvent` function **depend on the value (not the type) of the first argument**. A perfect match for dependent types.

### How we will proceed

Thanks to [currying](https://en.wikipedia.org/wiki/Currying), we know that a function that takes N arguments can be seen as a function that takes one argument, and returns a function which expects N-1 arguments.

So we can see the `logEvent` as a function which takes an event and returns a new function, which we will name `f` here. This function `f` expects as many arguments as there are placeholders in the event, each with the appropriate type. It will return the resulting announcement message as a string.

To illustrate this, we can use `:t` (in the REPL) to query the type of the function `f` returned by the partial application of logEvent on an event:

```haskell
birthDay : Event
birthDay = MkEvent "Happy BirthDay" [IntVar, StrCst " years old, ", StrVar]

-- In the REPL:
> :t logEvent birthDay    -- Query the type of logEvent applied to the birthDay event
Int -> String -> String   -- The type of a function expecting an integer and a string to produce a string
```

Writing `logEvent` consists in finding how to write this function `f`. We will proceed in two phases:
* We will first see how to compute the type of this function f, provided a message specification (ultimately the one of the event).
* Then we will complete the implementation.

### Computing types

So how do we compute the type of this mysterious `f` function? And what does it mean to compute a type?

In Idris, it means nothing in particular. A perfectly normal function can take a type as input, or return a type as output. Computing a type simply means calling a function which returns a type.

We will now write a function `SchemaType` to compute the type of `f` from the schema of a message specification. We recall that a `Schema` is a list of schema element, where each element is either a known text fragment or a placeholder:

```haskell
data SchemaElem
  = IntVar
  | StrVar
  | StrCst String

Schema : Type
Schema = List SchemaElem
```

We can use recursion on the list of schema element to slowly build the type of a function. At each recursive call, we will pattern match on the list, until we reach the empty list

```haskell
-- Function which takes a Schema and returns a function Type
SchemaType : Schema -> Type
SchemaType [] = String
SchemaType (IntVar :: xs) = Int -> SchemaType xs
SchemaType (StrVar :: xs) = String -> SchemaType xs
SchemaType (_ :: xs) = SchemaType xs
```

* If the list starts with an `IntVar` placeholder, we recur and add a `Int` argument in first position
* If the list starts with an `StrVar` placeholder, we recur and add a `String` argument in first position
* If the list starts with known text, we recur, not adding any new argument

We can try our SchemaType function in the REPL to convince ourselves that it works:

```haskell
> SchemaType [IntVar, StrCst " years old, ", StrVar]
Int -> String -> String
```

### The type of logEvent

Until now, we carefully avoided to define the type signature of the `logEvent` function. Now, that we defined `SchemaType`, we can finally do it.

To do so, we extract the message specification from the event given as first argument of the function and call `SchemaType` on it, **inside the type signature of the function**:

```haskell
logEvent : (event: Event) -> SchemaType (messageSpec event)
logEvent e = ?unspecified
```

There are important characteristics of Idris to notice here:

* Idris allows to refer to other arguments’ value by name in any type signature
* We can call functions such as `messageSpec` in the type signature of a function
* We can call functions such as `SchemaType` to compute part of the type signature

In our specific case, the return type of `logEvent` refers to the value of its first argument, `event`. We then extract the message specification out of it, in order to compute the rest of the type of the function.

Let's check that we get what we expect:

```haskell
> :t logEvent birthDay
Int -> String -> String -- A function that expects an integer and a string to produce the message
```

### Finishing up the implementation

Now that we have the type signature of `logEvent`, it is only a [small matter of programming](https://en.wikipedia.org/wiki/Small_matter_of_programming) to complete the implementation of the function. We just use the same trick we already used: recursion.

Each recursion level looks at a schema element, and builds a lambda which takes one placeholder value (the syntax `\argument => body_of_the_lambda`). Inside the lambda, we simply concatenate the result of serializing the placeholder value with the rest of the message.

```haskell
logEvent : (event: Event) -> SchemaType (messageSpec event)
logEvent e = loop (messageSpec e) (title e ++ ": ")
  where
    loop : (messageSpec: Schema) -> String -> SchemaType messageSpec
    loop [] out = out
    loop (IntVar :: rest) out = \i => loop rest (out ++ show i)
    loop (StrVar :: rest) out = \s => loop rest (out ++ s)
    loop (StrCst s :: rest) out = loop rest (out ++ s)
```

We can try it and check that it works fine when correctly used and that it fails when incorrectly used:

```haskell
> logEvent birthDay 32 "Quentin"
"Happy BirthDay: 32 years old, Quentin"

> logEvent newYearEve 2017
"New Year's Eve: Happy new year 2017"


> logEvent birthDay "Quentin" 32
".. Fails to compile ..."

> logEvent birthDay 32 32
".. Fails to compile ..."
```

We know have a fully type-safe interpolation of values inside the message specification. And we did not break our previous implementation. In fact, both versions can co-exist.

## Conclusion

In today’s post we showed some important feature of Idris type system, and of dependent typing in general: being able to compute types based on values.

We applied it to a use case, inspired from real a real one. We showed that it allows to build a stronger type-safety than what is allowed with more traditional type systems.

In addition of stronger type-safety, dependent types also fill the gap between values and types. They allow us to move between runtime and compile time checks in a smooth way. This helps removing some of the costs of using strong types in statically typed languages.

In particular, **the risk of having to rewrite entirely an API is reduced substantially** (as well as the risk to adapt or migrate its client code) upon changes in the requirements.
