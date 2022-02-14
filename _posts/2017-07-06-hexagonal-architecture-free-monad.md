---
layout: post
title: "Hexagonal Architecture and Free Monad: Two related design patterns?"
description: ""
excerpt: "In today’s post, we will explore the hexagonal architecture through an example, describe it, and relate it to Free Monads, a well-known and pretty popular pattern in typed functional programming languages to build Domain Specific Languages."
categories: [functional-programming]
tags: [Functional-Programming, Haskell]
---

_You do **not** need to understand Monads to follow this post. This is not a Monad tutorial either, but it might help you getting some intuition on what is a Monad. Some knowledge of Haskell and C++ might help but are not pre-requisite._

There is a lot of talk about [Domain Driven Design](https://en.wikipedia.org/wiki/Domain-driven_design) lately. This approach of design advocates that most of the software complexity we face today is due to our software not being aligned with the domain it attempts to model.

If the software does not properly decompose the problem it was built to solve, or solves another problem, or simply does not speak the language of the problem, chances are high that the customer will not be happy with it.

Domain Driven Design also comes with some design patterns (often referred as tactical pattern in DDD). One of those patterns that is often associated to Domain Driven Design is the [Hexagonal Architecture](http://alistair.cockburn.us/Hexagonal+architecture).

In today’s post, we will explore this architecture through an example, describe it, and relate it to Free Monads, a well-known and pretty popular pattern in typed functional programming languages to build Domain Specific Languages.

## Introduction & motivation

Paris developers got lucky lately. The capital of France hosted a bunch of high quality presentations featuring the Hexagonal Architecture:

* The [Alistair in the Hexagon](https://www.meetup.com/DDD-Paris/events/240715351/?eventId=240715351) meetup with [Alistair Cockburn](https://twitter.com/TotherAlistair)
* A [full Afternoon dedicated to DDD](https://www.meetup.com/DDD-Paris/events/238799479/?eventId=238799479) and refactoring toward DDD.

These presentation were great. Technically, they largely focused on delivering design patterns for OOP languages. Some attendees got the feeling that paradigms such as FP could not really offer the same domain modeling as OOP.

This is obviously not true. On the contrary, most Functional Programming languages are great at **modelling domain specific problems** by supporting the definition of embedded Domain Specific Languages (DSL) quite naturally.

But it is true is that different paradigms lead to different solutions. Languages such as Haskell are not that good at applying patterns from the OOP world. Instead, they have patterns of their own: such as the Free Monad in place of the Hexagonal Architecture.

In today’s post, we will explore the relations and differences between the Free Monad and the Hexagonal Architecture.

## Motivating example

Any proper discussion about design needs a good example as illustration. We will use the same example as the [Domain Driven Design afternoon](https://www.meetup.com/DDD-Paris/events/238799479/?eventId=238799479) presentation and build the domain logic of a [train reservation application](https://github.com/emilybache/KataTrainReservation).

### Business rules

The goal is to implement the domain of train reservation. We should offer an API to reserve a given number of seats at a given date.

The rules of the reservation of a train are the following:

* We cannot reserve seats in a train if it bumps up the occupancy over 70%
* All the reserved seats should be in the same coach (we cannot separate families)
* Preferably, we should avoid bumping the occupancy of a coach over 80%

### External services

Our application is only responsible for providing this business logic. Some other services already implement the booking and browsing of the trains:

* One service offers the capability to search for trains by date, and access the typology of the train (available seats, by coaches)
* One service offers the capability to book a bunch of seats in a coach in a given train. This service might deny the request (due to race conditions)

Our application will have to integrate to an monitoring infrastructure. We will simulate this by adding an external logging service.

### Architecture constraints

We want our application to separate the domain logic from the technicalities of the communication with the external services and the infrastructure:

* The business logic cannot depend on the real services
* A wrapping layer should take care of the plumbing to the real services

These rules are deliberately built to be in line with the tenets of Domain Driven Design and the Hexagonal Architecture, the topic of the next section.

## Hexagonal Architecture

In this section, we will introduce the basics of the Hexagonal Architecture. We will keep it minimal and the curious reader is encouraged to look at the [original article](http://alistair.cockburn.us/Hexagonal+architecture) and additional resources.

### Dual goals

The Hexagonal Architecture has two connected goals.

From a DDD perspective, it aims at structuring the code such that it offers a shell in which the code speaks the language of the domain and is not immediately bothered by technicalities. These technical aspects are kept separated.

From a evolution perspective, it aims at structuring the code such that it makes it safe and easy to switch technology stacks by minimising the amount of rework of the code that implements the business logic.

### Principles

The core business domain code is kept isolated from the implementation of the service on a particular technology stack.

The domain code communicates with the *real world* by using **ports** (interfaces in OOP). All the services on which the core business domain depends are abstracted by these ports, named after the capabilities they offer from the domain point of view.

There are no layers below or above the Hexagon. Instead, there is a layer around the hexagon consisting in adapters and code plugging these **adapters** to the ports the hexagon defines (dependency injection).

## Applying the Hexagonal Architecture

We will now illustrate how the Hexagonal Architecture translates in terms of code to implement our service. To keep things short, we will only focus on the architecture of the application. The full code will be provided at the end.

### Service providers

As described in the previous section, the core domain depends on service providers. We abstract each of them behind dedicated interfaces, named after the capability they offer from the point of view of the core domain.

We name these interfaces using the naming scheme advocated by [Thomas Pierrain](http://tpierrain.blogspot.fr/): we will use the **I** of the interface name to form a sentence that explains the capability offered by the service.

* IReserveTrains allows to reserve a set of seats in a coach of a given train.
* ISearchTrains allows to:
  * Retrieve all the trains departing around a provided date-time
  * Get the typology of a given train (coaches and seats available)

Here is how it would translate in OOP (using C++):

```cpp
struct ISearchTrains : Interface<ISearchTrains>
{
    virtual std::vector<TrainId> trains_around(DateTime const&) = 0;
    virtual TrainTypology get_typology(TrainId const&) = 0;
};

struct IReserveTrains : Interface<IReserveTrains>
{
    virtual std::optional<ConfirmedReservation> reserve(ReservationAttempt const&) = 0;
};
```

We can abstract the logging service the exact same way, by providing an interface that models this capability:

```cpp
struct ILog : Interface<ILog>
{
    virtual void log(std::string const&) = 0;
};
```

### Service provided (API)

As described in the previous section, our hexagon also offers a port for the service the core domain offers (basically, what our application does). We will modeled this service with an interface `IFindTheBestPlaces`:

```cpp
struct IFindTheBestPlaces : Interface<IFindTheBestPlaces>
{
    virtual std::optional<ConfirmedReservation> reserve(ReservationRequest const&) = 0;
};
```

Clients of our applications will only depend on this interface. The implementation of the service will stay hidden in the Hexagon, behind the adapter layer.

Instantiating this implementation require implementations for all the service providers it depends on (ISearchTrains, IReserveTrains, ILog). These implementations are taken as argument of the constructor:

```cpp
class FindTheBestPlaces : public IFindTheBestPlaces
{
public:
    FindTheBestPlaces(IReserveTrains&, ISearchTrains&, ILog&);
    std::optional<ConfirmedReservation> reserve(ReservationRequest const&) override;
    
private:
    IReserveTrains& _iReserveTrains;
    ISearchTrains& _iSearchTrains;
    ILog& _iLog;
};
```

### Plumbing

The plumbing is done inside the factory function that creates the `IFindTheBestPlaces` service. It injects the real implementation of the service providers (`ISearchTrains`, `IReserveTrains`, `ILog`) inside the `FindTheBestPlaces` concrete API implementation.

All the details of this plumbing are therefore hidden from the user:

```cpp
std::unique_ptr<IFindTheBestPlaces> find_best_places_service()
{
    ISearchTrains& iSearchTrains = ...;
    IReserveTrains& iReserveTrains = ...;
    ILog& iLog = ...;
    return std::make_unique<FindTheBestPlaces>(iReserveTrains, iSearchTrains, iLog);
}
```

A similar plumbing goes into the unit tests of the core business logic. Each time a port is used inside a code under test, a mock, fake, stub must be provided.

A similar plumbing also happens in the code living inside the Hexagon. `ISearchTrains`, `IReserveTrains` and `ILog` must be carried to all classes and functions that need them (outside of the context of using a dependency injection framework).

## Drawbacks of the Hexagonal Architecture

Applying the Hexagonal Architecture to our train reservation application allowed us to decouple the implementation of the service providers from the core business logic.

The code in the Hexagon is less likely (there are limits to everything) to need rework upon changing the technology stack or communication protocols used to exchange with the service providers. It solves our initial problem.

But there are obviously drawbacks in using this architectural pattern. These drawbacks should not be understood as advice against the Hexagonal Architecture. But we have to be aware of the costs of techniques we use before applying them.

### Somestimes heavy

The Hexagonal Architecture is essentially a pattern of dependency injection, a pretty heavy technique, which leads to mocks in tests and the multiplication of interfaces.

The overhead of having to inject the interfaces is there. It can sometimes represent a good chunk of the code or the unit tests. Each class that needs to access the services will need to be injected the implementation of these services.

### Not declarative

The hexagonal shell is not visible in the code. There is no clear indication in the code that some code is inside the hexagon. It is only a matter of convention between developers.

As a result of having only conventions, there is no strong insurance against a developer using the dependencies directly, instead of going through the ports, or adding a new dependencies without the associated port (it happens, we all have seen it).

### Other caveats

Abstracting the services providers in the business logic does not abstract away the knowledge of know how many of these services there are.

Having `IReserveTrains`, `IGetTypologies` and `ILog` as ports shows part of the infrastructure inside the business logic code (the Hexagon). This is not much but it represents a source of coupling still. It can be solved by reworking the interfaces.

## Embedded DSL: a more declarative approach

We have seen how the Hexagonal Architecture allowed us to separate the code that implements the domain core logic, from the code that deals with the infrastructure or the technicalities of the communication with other services.

We will now describe another approach, more adapted to typed functional programming languages, that aims at tacking the same problem, with different trade-offs.

### Being declarative by selecting our primitives

The [Structure and Interpretation of Computer Programs](https://en.wikipedia.org/wiki/Structure_and_Interpretation_of_Computer_Programs?oldid=737448219) describes the basic building blocks of a programming language in the first sentences of the first chapter:

* **Primitives**: the simplest elements a language is built upon
* **Means of combination**: used to build compound elements from simpler ones
* **Means of abstraction**: used to abstract compound elements as single unit

Most often, the primitives of the language of our business domain are **not** the primitives our general purpose programming language. This is why we write well-named functions, classes or interfaces: the build the right abstractions.

But sometimes, it is not enough.

Creating an Embedded Domain Specific Language (EDSL) in a language such as Haskell will allow us to keep much of the powerful means of combination and abstraction of our language, but will allow us to **pick our own primitives**.

Picking our own primitives will allow us to build our domain language at a much deeper level than function and classes. We will get a more declarative and safer equivalent of the Hexagonal Architecture, to decouple our business logic from the “real world”.

### Selecting our primitives, but how?

It depends. In some host language (the language in which we build our embedded DSL), some approaches are more idiomatic and appropriate than others. Data and macros are the way to go in Clojure, while Monads will be prefered in Haskell.

Because we choose Haskell in this post, we will select an approach based on the concept of Monad (and Free Monad in particular). It is especially effective in languages having a very expressive type system such as Haskell, Idris or Scala.

### A high level definition of Monads

We will not go into the conceptual definition of a Monad. It is an abstraction for a recurring pattern in Category theory, much like an interface or a design pattern in a programming language, but we will not this here.

Instead, we can consider a Monad as a way to define a small language inside our programming language. Inside this language, we can choose to **modify the rules of composition, evaluation and the available primitives**.

In short, it is like a customized environment of execution. Inside a Monad, the rules of the host language can be changed as we see fit. This allows us to speak a different tongue, closer to the language of the domain we are trying to model, and decoupled from the technicalities of the implementation.

## Using Monads to select our primitives

In this section, we will explain how Monads can be used to offer a more declarative alternative to the Hexagonal Architecture. The next section will then show how to implement such a Monad.

### Functions without side effects

Let us start with **pure** functions, which are function that lives outside of any Monad. You can find below the prototype of a function computing the length of a string:

```hs
-- Compute the length of a `String` and returns it as an `Int`
length :: String -> Int
```

This Haskell function, by the very declaration of its type, is limited to **pure computations**, with no side-effects allowed such as:

* Sending a request to the Search Train service
* Calling a function that would itself call the Search Train service

In other words, the primitives the length function has access to, are limited to those that do not perform side-effects. This seems to align particularly well with our need to create a functional core in the Hexagonal Architecture, but as we will see, this is too restrictive.

### Functions with IO side effects

In Haskell, we can tag a function as living in the `IO` Monad, by prefixing its return type by `IO`. Inside this Monad, a function is allowed to do any side effect it wants (printing something, accessing a file, sending a HTTP request, etc):

```hs
-- Cannot do anything but pure computations
reserve :: ReservationRequest -> ConfirmedReservation

-- Can do anything it wants, such as sending a HTTP request
reverseWithIO :: ReservationRequest -> IO ConfirmedReservation
```

In other words, the primitives the `reserveWithIO` function has access to, contain all those that perform arbitrary side-effects.

Tagging a function with a Monad allows to **select or discard the primitives available inside a piece of code, declaratively**.

### Defining our own language (Monad)

Inside the `IO` Monad, we can do anything. This is **too permissive**. We would like to forbid direct access to the implementation of the service providers in the Hexagon.

Outside of any Monad, we cannot do any side effects, directly or indirectly. This is too restrictive. Our business logic needs to access the service providers.

The solution is to define our own Monad, our own environment of execution with our own rules, which we will name `ReservationExpr`:

```hs
-- The `ReservationExpr` is a Monad with the appropriate restrictions:
reserve :: ReservationRequest -> ReservationExpr ConfirmedReservation
```

Inside this Monad, we will offer only the side effects needed to connect to the external services we want, by defining our own primitives to do it. By construction, *these primitives will be the only way to access the service provider*, the real world.

This effectively defines a DSL that is **declarative** (the Monad is clearly visible) and **safe** (we cannot bypass the primitives). We will now see how it also allows to solve our dependency injection concerns.

### Free your Monad

As in the Hexagonal Architecture, we want to decouple the **intent** to use a service from its actual implementation inside our DSL. In other words, the primitives of our new language will **only express what we want to do, and not how**.

By varying the how behind the what, we can make our primitives do different things, much like an interface allows to vary the implementations. Our primitives will for instance connect to external services in production settings, but not during tests.

To do this, we will build our `ReservationExpr` Monad such that the code that lives inside it will **emit abstract instructions** upon evaluation, describing the connection to these external services instead of directly connecting to them.

For instance, the code inside our Monad will emit instructions such as:

```
SearchTrains at 11/06/2017-11:00
=> Bind result to variable `trainId`

RetrieveTypology of Train with ID `trainId`
=> Bind result to variable `trainTypology`

Call pure function `belowOccupancyThreshold` on `trainTypology`
=> Bind result to `isValid`
```

We call this kind of Monad a Free Monad, as it only describes a sequence of operations, **free of any hardcoded interpretations**.

### Interpreters

Functions tagged with our ReservationExpr Monad will just emit abstract instructions (an [Abstract Syntax Tree](https://en.wikipedia.org/wiki/Abstract_syntax_tree) to be exact) upon evaluation, instead of executing the code they contain. This effectively buys us one level of indirection.

These instructions can then be **interpreted** differently in different contexts. Inside a production context, we can make them trigger connections to the real world. Inside a test context, we can make them use an in-memory test database.

We do this by defining several functions that each transforms these abstract instructions into different real instructions of our host language (yes, much like compilers do for assembler). We call these functions **interpreters**.

We can build as many interpreters as we wish. We just pick the one we want depending on the context. For instance, for the same abstract instruction “SearchTrain at datetime D” and depending on the interpreter, we could:

* Send an HTTP request (on a production environment)
* Lookup inside an in-memory DB (for integration tests)
* Return a fixed value (for a specific unit test)
* Log the call to the function (for simulations)

### You know the basics

If you understood this section, you understand how Free Monads help defining EDSLs, which are declarative and controlled environment to express domain specific logic.

Inside the EDSL, the business logic is decoupled from a lot of technical concerns such as external dependencies, or even the evaluation strategy. This is done by relying on primitives of a language that we developer have control on.

The rest of the post will demonstrate how **simple and concise** it is to define our own EDSL, by using Free Monads in Haskell.

## Free Monad: Example implementation

This section explains how to create our `ReservationExpr` Monad easily and concisely. It is described in such a way that only a small amount of Haskell knowledge is needed.

If you are not interested in Haskell or never touched it, you should skip this section.

### Step 1: Identify the abstract instructions

The first step to define our EDSL is to identify its primitives, the low level abstractions needed to express our business logic. We identify six of them:

* `SeachTrain T`: searching for trains around the datetime T
* `GetTypology T`: retrieving the typology for the train with reference T
* `Reserve T C S`: reserving the seats S in the coach C of the train T
* `Log M`: logging the message M (represented as a string)
* `Pure C`: running a pure computation C (to implement our business logic)
* `Bind I C`: binding the result of instruction I to the next computation C (*)

The last two instructions `Pure` and `Bind` are the two instructions needed in every Free Monad. They basically allow you to use the pure part of Haskell inside the EDSL.

_(*): For non Haskell, do not worry if you do not understand Bind. Just think of it as a kind of glorified C++ / Java / C# semicolon and you will be fine._

### Step 2: Define the type of the instructions

The abstract instructions have a semantic that could be encoded by functions. Searching for the trains at date-time T could be implemented by a `searchTrain` function living in the `ReservationExpr` Monad:

```hs
-- Search for a train:
-- * At a given DateTime (example: 2017-07-01 14h38)
-- * Returning a list of TrainId
searchTrain :: DateTime -> ReservationExpr [TrainId]
```

Similarly, every single of our primitives could be encoded as functions:

```hs
searchTrain :: DateTime -> ReservationExpr [TrainId]
getTypology :: TrainId -> ReservationExpr (Maybe TrainTypology)
reserve :: Reservation -> ReservationExpr (Maybe Reservation)
log :: String -> ReservationExpr ()
```

Now, these functions cannot contain direct calls to external services. Instead, they should emit abstract instructions (for an interpreter to translate them later).

This is easy enough: we just take the functions above, capitalise them (first letter to upper case) to get type constructors, and group these constructors into a type named after our Monad: `ReservationExpr`.

```hs
data ReservationExpr a where
  SearchTrain :: DateTime -> ReservationExpr [TrainId]
  GetTypology :: TrainId -> ReservationExpr (Maybe TrainTypology)
  Reserve :: Reservation -> ReservationExpr (Maybe Reservation)
  Log :: String -> ReservationExpr ()
  Pure :: ta -> ReservationExpr ta
  Bind :: ReservationExpr ta -> (ta -> ReservationExpr tb) -> ReservationExpr tb
```

`ReservationExpr` a is the abstract type for an instruction which, upon evaluation, will return a value of type a. It consists of primitive instructions such as `SearchTrain`, which will return a list of train ids upon evaluation.

Thanks to `Pure` and `Bind`, the primitive instructions can be composed into bigger computations. So `ReservationExpr a` is also the abstract type of computations inside the `ReservationExpr` Free Monad, which return a value of type `a`.

### Step 3: Make it a Monad

Haskell will need a bit of boilerplate code to be satisfied and make our `ReservationExpr` a recognized Monad.

```hs
instance Functor ReservationExpr where
  fmap fn expr = expr >>= Pure . fn

instance Applicative ReservationExpr where
  pure = Pure
  fExpr <*> aExpr = fExpr >>= \f -> fmap f aExpr

instance Monad ReservationExpr where
  (>>=) = Bind
```

If you do not understand these lines, it is fine. It is enough to copy-paste them and search and replace in them to define your own Free Monad.

### Step 4: Write your business logic

We now are all set. We can define our business logic in our EDSL based on the abstract instructions we defined above. Here is an example of such a code:

```hs
reserve :: ReservationRequest -> ReservationExpr ReservationResult
reserve request = do
  trains <- SearchTrain (_dateTime request) -- Search for trains at date-time
  forM trains $ \train ->                   -- Loop on all the trains
    typology <- GetTypology train           -- Get the typology of a train
    ...                                     -- Implement the reservation rules
    Log ("Confirming reservation")          -- Logging stuff
    confirmed <- Request reservation        -- Trying to reserve the train
    ...                                     -- More stuff
```

Upon evaluation, this code will translate into a tree of instructions, ready to be read by an interpreter. Therefore, all the code that lives inside the `ReservationExpr` Monad is itself **free of side effects**. It **only builds an AST**.

### Step 5: Write your interpreters

A `ReservationExpr a` is an Abstract Syntax Tree that upon evaluation will produce a value of type `a`. Any interpreter we define on this AST, whatever its implementation, will have to do exactly that.

This means that our `ReservationExpr` Monad, through its types, forces the interpreters to comply with the rules expressed inside the DSL. The type system will make sure that *SearchTrain must return a list of train ids* or else will reject the code.

This has two important consequences:

* You get tons of help by the type system to write the interpreters
* You can encode very powerful invariant in the DSL, using the type system

We can therefore write our production settings interpreter by (almost) just pattern matching on the AST and following the type hints of the Haskell compiler.

```hs
evalReservation :: ReservationExpr ty -> IO ty
evalReservation = evalCmd
  where
    evalCmd :: ReservationExpr ty -> IO ty
    evalCmd (Log msg) = putStrLn msg
    evalCmd (Pure val) = pure val
    evalCmd (Bind val next) = evalCmd val >>= evalCmd . next
    evalCmd (SearchTrain time) = searchTrainAt time
    evalCmd (GetTypology trainId) = getTypologyOf trainId
    evalCmd (Reserve command) = confirmCommand command
```

This `evalReservation` **transforms abstract instructions into real world IO** calls. It effectively fills the role of <ins>dependency injection</ins> in the Hexagonal Architecture.

But we are not limited to this interpreter. We can build an interpreter to run our instructions into a in-memory database model (the `State` monad):

```hs
-- Run the computation using a InMemoryDb of trains and typologies
-- * Simulates real world behaviors with fake data
-- * Can simulate the behavior of successive interacting requests
evalWithFakeDB :: ReservationExpr ty -> State InMemoryDb ty
```

And we can do more. We can write interpreters to log the function calls, return constant values, and more… There is no limits but your imagination.

### Step 6: Wrap up in a nice API

As in the case of the Hexagonal Architecture, we can provide a wrapper around our API to hide the details of the implementation of the service.

In the case of the Free Monad, that would be hiding the construction of the AST (done in `reserveImpl`) and the interpreter we use, all behind a single function call:

```hs
-- Wrapper around the reserveImpl function, living in ReservationExpr
reserve :: ReservationRequest -> IO ReservationResult
reserve = evalReservation . reserveImpl
```

Similarly, we can provide wrapper for the in-memory DB interpreter, to help the writing of our integration tests:

```hs
reserveWithFakeDb :: ReservationRequest -> State InMemoryDb ReservationResult
reserveWithFakeDb = evalWithFakeDB . reserveImpl
```

### Step 7: Enjoy & Celebrate

There is nothing much to do. Following this pattern, we provided a safe and decoupled EDSL for our business logic code.

Code expressed in this EDSL is truly decoupled from the external world by the Free Monad, hidden from the external world by the wrapping layer around the API, and incapable of directly communicate with the external world.

## Free Monad vs OOP Hexagonal Architecture

Having gone through this post, the similarities between the Hexagonal Architecture and the Free Monad pattern should be quite clear.

Both approaches rely on creating a *“context bubble”*, decoupled from the real world by an abstraction, and plugged back to the real world by an adaptation layer. But there are some important differences too.

### Complexity

I think there is little argument that the Free Monad is a more complex pattern than the Hexagonal Architecture. Interfaces are a much easier concept to grasp than Monads for the vast majority of developers, which makes it easier to build the initial architecture.

But on the other hand, the Free Monad provides a better guidance to the developers: it is much more explicit and declarative. Developers do not have to rely on documentation (if any) to find their way in the architecture, making their life easier.

It ultimately boils down to what is more idiomatic. Interfaces are pretty common in OOP, while Monads and Free Monads are a pretty classic pattern in Haskell.

### Declarative-ness

The Free Monad is by its very nature much more declarative than the Hexagonal Architecture. I think this is a no-contest.

Any code written inside a Free Monad is only providing a recipe (the what). How the recipe will be translated in terms of real world instruction is completely decoupled from the recipe and left for the interpreter to decide.

The interpreter can choose to evaluate the code differently than how it looks. It might bulk the request to external services, cache them, parallelise some evaluation safely (knowing it is pure) and more (see Haxl for a great example of this by Facebook).

Technical aspects are therefore more decoupled in the Free Monad than in the Hexagonal Architecture. The Hexagonal Architecture is still bound by the standard rules of evaluation of the language and can only abstracts the real world calls.

### Rigidity vs Safety

Both pattern aim at creating an environment where code is more constraint than usual code, allowed to perform any kind of side effects (such as code living in the IO Monad).

The difference lies in how each pattern enforce this. The Hexagonal Architecture does not enforce any guaranty, while the Free Monad does. In truth, both approaches have their benefits.

Being more constraint makes the Free Monad able to offer stronger guaranties (the code must comply with the architecture, and the interpreter can therefore leverage this) but also makes the architecture more rigid.

As a result, jumping right on the Free Monad from the start of a project, while still exploring the problem, is likely not a good idea. But jumping too soon on the Hexagonal Architecture is also a risk (I learned it some years ago).

Both patterns erect walls between the real world and the domain logic. This is what they do. And it should be quite obvious that this is not a very good idea to try to erect walls too soon, before even knowing what we want to protect.

### Domain modelling

The Free Monad offers a different kind of domain modelling than the Hexagonal Architecture, by separating the following aspects of the domain:

* The recipe describing the business logic
* The rules the recipe must follow, encoded with types in the DSL
* The transcription of the recipe to the real world

This separation allows us to profit more from domain modelling. We can for instance enforce some invariants of the domain directly inside the type system of the DSL.

We can also abstract some technical aspects some more. For instance, error handling can be done inside the interpreter, stopping the evaluation of an expression without having to rely on exception going through the business logic code.

## Final words

The Free Monad is a kind of Hexagonal Architecture on steroid. More declarative and stronger at enforcing the separation of concern, the Free Monad is also more rigid and a potentially riskier choice at the start of project.

The Free Monad shines by providing a stronger decoupling between the recipe describing the business logic and the actual evaluation of the recipe, allowing different evaluation of the code and domain specific optimizations.

Ultimately though, both Hexagonal Architecture and Free Monad are patterns whose complexity is tied to the idioms of the languages they are used in.

* The Hexagonal Architecture is a better fit in OOP
* The Free Monad is likely a better fit for Haskell
* For languages such as Scala, both approaches are available

If anything, I hope this post showed how Functional Programming languages such as Haskell have their own pattern, which make them great at modelling a problem.

You can find the code in Haskell in the [following Git Hub repository](https://github.com/QuentinDuval/HaskellTrainReservationKata).

<hr>

Acknowledgments to the speakers of the [Afternoon dedicated to DDD](https://www.meetup.com/DDD-Paris/events/238799479/?eventId=238799479): [Thomas Pierrain](https://twitter.com/tpierrain), [Jérémie Grodziski](https://twitter.com/jgrodziski) and [Bruno Boucard](https://www.meetup.com/DDD-Paris/members/40215522/) whose Kata was a great inspiration for this post.

Acknowledgments to the speakers of the [Alistair in the Hexagon](https://www.meetup.com/DDD-Paris/events/240715351/?eventId=240715351) meetup: [Thomas Pierrain](https://twitter.com/tpierrain) and of course [Alistair Cockburn](https://twitter.com/TotherAlistair) for their great and illustrated explanation of the Hexagonal Architecture.