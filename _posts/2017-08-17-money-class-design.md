---
layout: post
title: "A study of 4 Money class designs, featuring Martin Fowler, Kent Beck and Ward Cunningham designs."
description: ""
excerpt: "Let's look at the Money class implementations found in Kent Beck, Martin Fowler and Ward Cunningham work, and propose a new one based on dependent typing."
categories: [software-design]
tags: [Functional-Programming, Haskell]
---

I started to read the [Test Driven Development: by example](https://www.amazon.fr/Test-Driven-Development-Kent-Beck/dp/0321146530) book (by Kent Beck). It offers a perspective on TDD that is far from being the dogmatic perspective that is taught by some trainers. It is a really good back and I highly encourage you to read it.

But **this post is not about TDD**. There are plenty of blog posts already available online for this. This post is about the main example used by Kent Beck in this book, **the Money class**.

In this post, we will look at this Money class implementation and compare it with the other approaches available in the literature, such as the one of [Martin Fowler](https://twitter.com/martinfowler). Through this study, we will:

* Discuss the approach provided in [PEAA](https://www.amazon.com/Patterns-Enterprise-Application-Architecture-Martin/dp/0321127420/ref=sr_1_1?ie=UTF8&qid=1502550798&sr=8-1&keywords=Patterns+of+Enterprise+Application+Architecture) and some related implementation
* Unveil the hidden defect behind the implementation of [TDD: by example](https://www.amazon.fr/Test-Driven-Development-Kent-Beck/dp/0321146530)
* Provide a different implementation of the same idea which fixes this defect
* Show how dependent typing and Idris provide new ways to design the Money class

Our goal is to explore the trade-offs behind each of these 4 approaches. This journey will lead us to discuss about the **hole in our mainstream type systems** and how dependent typing solves it.

_This post features C++, Haskell and Idris. The post is written such that the knowledge of these languages is not a prerequisite, but you will likely learn a bit about these languages. Similarly, the discussion is not exclusive to these languages and is easily generalisable._

## Motivation

The Money class is one of these popular class that appears almost everywhere you look. Martin Fowler talks about it in [PEAA](https://www.amazon.com/Patterns-Enterprise-Application-Architecture-Martin/dp/0321127420/ref=sr_1_1?ie=UTF8&qid=1502550798&sr=8-1&keywords=Patterns+of+Enterprise+Application+Architecture), it is the main example of [Test Driven Development: by example](https://www.amazon.fr/Test-Driven-Development-Kent-Beck/dp/0321146530) and is often taken as example in a lot of [Domain Driven Design](https://en.wikipedia.org/wiki/Domain-driven_design) talks.

The goal is to **design a type that encapsulate an amount together with its currency**. We want to provide a safe way to do arithmetic on amounts, and avoid the kind of bug that arise when summing double values with different units (such as EUR and USD) and get angry customers.

Unlike units in the metric system though, the **conversion rates are constantly changing** and depend on external sources (such as rate curves on the stock exchange). Converting EUR in USD is therefore not as easy as converting kilometres in meters.

Given these constraints, how can we implement a class that represents an amount of money with its currency? Interestingly, the literature provides different solutions on it.

## Martin's Fowler money class

Our first candidate for a Money class is the one proposed by Martin Fowler in his book, [Patterns of Enterprise Application Architecture](https://www.amazon.com/Patterns-Enterprise-Application-Architecture-Martin/dp/0321127420/ref=sr_1_1?ie=UTF8&qid=1502550798&sr=8-1&keywords=Patterns+of+Enterprise+Application+Architecture). To keep things short, we will focus on the safe arithmetic part and ignore the *allocate* part of his design.

### How it works

We group a currency and an amount inside the same class / structure. We create methods to add moneys together, multiply them by a constant, and compare them for equality.

* Equality is easy: two Money instances are equal if both their amount and currency are equal
* Multiplying by a constant is easy: we multiply the amount by the provided constant
* Adding Money instances is tricky: we have to deal with the possibility of different currencies

The solution proposed in [PEAA](https://www.amazon.com/Patterns-Enterprise-Application-Architecture-Martin/dp/0321127420/ref=sr_1_1?ie=UTF8&qid=1502550798&sr=8-1&keywords=Patterns+of+Enterprise+Application+Architecture) by Martin Fowler is to assert that the two currencies are equal inside the implementation of the addition. The literature also contains variations on the same scheme (*).

In short, adding currencies of different currencies is forbidden by the API. It becomes a precondition of calling the add method, and so the client is responsible:

* To check that the currencies are equal before calling add
* To convert the amounts to the same currency if needed

In this design, **the type system will not help** the client. Performing these checks in on the client, and forgetting about them will result in a runtime error, hopefully caught at testing time.

_(*) For instance, the book [Patterns, Principles and Practices of Domain-Driven Design](https://www.amazon.com/Patterns-Principles-Practices-Domain-Driven-Design/dp/1118714709/ref=sr_1_1?ie=UTF8&qid=1502637607&sr=8-1&keywords=patterns+principles+practices) encourages to throw an exception if the currencies are different (instead of an assertion)._

### Sample implementation

Here is a possible implementation of this Money class design in Haskell.

We start by defining of the data structure Money. It contains two fields, currency and amount, and a default implementation of the == operator:

```haskell
data Money = Money {    -- Declares a Money class with:
  amount   :: Amount,   -- * an amount field
  currency :: Currency  -- * a currency field
} deriving (Show, Eq)   -- * and equality on all fields
```

We then write a function to add two Money instances together. If you are not familiar with Haskell, the prototype of the function below says “take two instances of Money and return a Money instance”:

```haskell
add :: Money -> Money -> Money
```

We can then complete the definition of the function. The implementation throws an exception when the currencies of the two Money instances `m1` and `m2` are different:

```haskell
add :: Money -> Money -> Money
add m1 m2 = 
  if currency m1 != currency m2                   -- If the two currencies are different
    then error "Currencies should be equal"       -- * Throws an exception
    else Money { amount = amount m1 + amount m2   -- * Else return a new Money instance
               , currency = currency m1 }         --   which is the sum of the two inputs
```

For reference, and if you are unfamiliar with Haskell and more accustomed to OOP, the following C++ implementation is the equivalent implementation:

```cpp
class Money {
  Amount m_amount;
  Currency m_currency;
	
public:
  Money(Amount const& amount, Currency const& currency)
    : m_amount(amount)
    , m_currency(currency)
  {}
		
  friend bool operator==(Money const& lhs, Money const& rhs) = default;
  friend bool operator!=(Money const& lhs, Money const& rhs) = default;

  Amount amount() const { return m_amount; }
  Currency currency() const { return m_currency; }
};

Money add(Money const& lhs, Money const& rhs)
{
  if (lhs.currency() != rhs.currency())
    throw std::logic_error("Currency should be equal");
  return Money(lhs.amount() + rhs.amount(), lhs.currency());
}
```

### Critical analysis

The pattern is quite simple and allows to group together a value and its currency (its unit). The complete pattern also uses a dedicated Amount type (like a big Decimal, instead of doubles, for better precision). As it stands, this pattern is already quite an improvement over using doubles (without unit) as money amounts (*).

Quite an improvement already, but it is not perfect.

The main critic on this design is the **poor declarative nature of the error handling**. The type signature of add does not indicate it can fail. The polymorphic nature of the operation is hidden too. As a result, the client of the API is left the burden of the checking the precondition manually, with **no help from the type system**.

The exception-based implementation can also be criticized for using exceptions to deal with a non-exceptional situation: adding currencies of different currencies is the most common case. It might lead to the client code catching the exceptions to implement its control flow: basically, as glorified gotos.

Overall, this implementation of Money encapsulates some of the concerns associated to the representation of amounts, but lets one concern leaks away: the safety of operations such as add.

_(*) Using doubles to represent amounts of money is still unfortunately found pretty much in the wild, especially in legacy applications, in which it becomes really hard to refactor._

### Improvements using optionals

We can make the error handling more explicit by making it appear in the type signature of add. We can even force the client code to do a check of validity of the operation.

Nowadays, most languages define an optional type that encodes the presence of absence of something. In C++, this type is called `std::optional`. In Scala, it is called `Option`. In Haskell, it is called `Maybe`.

We can use it to return an optional Money from add:

```haskell
add :: Money -> Money -> Maybe Money
add m1 m2 = 
  if currency m1 != currency m2             -- If the two currencies are different
    then Nothing                            -- * Return an empty result
    else Just $ Money {                     -- * Else return a new Money instance
          amount = amount m1 + amount m2,   --   which is the sum of the two inputs
          currency = currency m1 }
```

Now, the type signature correctly indicates that add may fail to return a new Money instance. Better yet, the client has to verify the return value and check whether the operation went gracefully or not.

Finally, we now avoid to use exceptions to deal with what is a non-exceptional situation, making it an explicit part of our domain logic.

### Still room left for improvement

Despite the improvements we did, the Money class as it stands still has some annoying defects. Our add operation is **now asymmetric**. It returns an optional Money instance. In some languages such as Haskell, it means we cannot use the operator +.

Another annoying issue is that the checks of validity of the add operations occur after add has been called. It would be more intuitive to force the verification of the precondition before calling the function.

We will see how to address these issues in the last section of this post.

## Kent Beck's money class

We will now look at the different design of the Money class, which comes directly from [Test Driven Development: by example](https://www.amazon.fr/Test-Driven-Development-Kent-Beck/dp/0321146530) (written by Kent Beck). This design takes a rather different stance than the previous design: instead of forbidding cross-currency addition, it embraces it.

### How it works

The simple yet powerful idea (*) that lies behind the design exposed by Kent Beck is to separate the specification of the addition from its evaluation. Adding money instance with different currencies does not sum them immediately: it **builds an expression that represents the sum**.

Building something that represents the sum is in fact the only reasonable way to proceed. Indeed, adding the two amounts eagerly by forcing a currency conversion would be terrible:

* We would have to arbitrarily choose a currency to convert into
* We would have to arbitrarily choose a rate curve (on which a market?) to get the rates
* We would introduce a side-effect is what looks to be a pure computation
* It would **destroy information**: 3 EUR + 5 USD is a different information than 9.15 USD (**)

Instead, in the design of Kent Beck, adding two amounts results in building an expression that represents the sum of the two amounts. For instance, adding 3 EUR and 5 USD results in the following abstract syntax tree (AST):

```
     (+)
    /   \
3 EUR   5 USD
```

To evaluate the AST in any destination currency we want, we build an **evaluator** that needs to have access to a source of data that provides the conversion rates between the different currencies involved. The evaluation is decoupled from the specification. So much decoupled in fact, that it can be performed later, or never, or even several times, with different rate curves, etc.

_(*) Often, truly powerful ideas are simple in their very essence as they touch to the core of the concepts we try to model._

_(**) Summing the amounts early has for effect to collapse the information at a given point in time. Delaying the addition keeps the information independent of time, allowing several evaluations at different points in time._

### Implementation (AST)

Our Haskell implementation will distinguish between two notions:

* **Money expression**: an un-realized abstract syntax tree (MoneyExpr in the code)
* **Money**: a known amount with a known currency, the result of the evaluation of a MoneyExpr

We will keep our Money data structure from the previous implementation:

```haskell
data Money = Money {    -- Declares a Money class with:
  amount   :: Amount,   -- * an amount field
  currency :: Currency  -- * a currency field
} deriving (Show, Eq)   -- * and equality on all fields
```

We then create a `MoneyExpr` type for a money expression, which defines our AST. For those unfamiliar with Haskell, the pipe operator `|` represents a disjunction (OR). The code below says that a MoneyExpr is either an known amount `KnownAmount`, or an addition of several sub-expression `MoneyAdd` or a multiplication of a sub-expression `MoneyMult`.

```haskell
data MoneyExpr                  -- An Money expression made of EITHER:
  = KnownAmount Money           -- * A known amount and currency (base case)
  | MoneyAdd [MoneyExpr]        -- * A sum of several money expression
  | MoneyMul MoneyExpr Double   -- * A money expression multiplied by a factor
  deriving (Show, Eq)
```

We then implement our function such that they build the AST instead of directly computing sums and products:

* Adding will create a new MoneyAdd node.
* Multiplying by a constant will create a MoneyMult node.

```haskell
-- Given these expressions
let a = money 30 "USD"
let b = money 25 "EUR"
let c = money 1000 "JPY"

-- Add the different currencies
add (add a (multiply b 2)) c

-- It returns the following AST:
MoneyAdd
  [ MoneyAdd
    [ KnownAmount (Money {amount = 30.0, currency = "USD"})
    , MoneyMul (KnownAmount (Money {amount = 25.0, currency = "EUR"})) 2.0]
  , KnownAmount (Money {amount = 1000.0, currency = "JPY"})]
```

Graphically, the resulting AST of the code above would look like this:

```
         (+)
        /   \
      (+)   1000 JPY
     /   \
30 USD   (*2)
          |
        25 EUR
```

The only missing piece now is the evaluator.

### Evaluator implementation

The `MoneyExpr` represents a sum of money amounts expressed in different currencies. At some point though, the client code will be interested in converting all the amounts in a target currency (for instance to compare it with another amount in this currency). This is the job of the evaluator.

The evaluator will need a way to access conversion rates from a source of data (such as a rate curve on a given market). We can abstract this behind a function that returns a rates from a currency conversion:

```haskell
-- Abstracts the access to a source of data which provides
-- the rate associated to a conversion between 2 currencies
Conversion -> Maybe Rate
```

The evaluator will also need as input the money expression to evaluate and the target currency to convert into. This gives us the following prototype for our evaluator, which we name of `evalMoneyIn`:

```haskell
evalMoneyIn ::                -- Evaluate an Money expression
  (Conversion -> Maybe Rate)  -- * Given a way to retrieve a rate
  -> MoneyExpr                -- * A money expression to evaluate
  -> Currency                 -- * And a destination currency
  -> Maybe Money              -- Returns a known amount (or fails)
```

Here is how the function can be used, where `findRate` represents the function that gives access to the rates, and `moneyExpr` is a money expression:

```haskell
-- Successful evaluation
evalMoneyIn findRate moneyExpr "USD"
> Just (Money {amount = 100.0, currency = "USD"})

-- Failing evaluation
evalMoneyIn findRate moneyExpr "USD"
> Nothing
```

The implementation details of the evaluator would require going into some Haskell core abstractions, and is outside the scope of this post, but is provided as reference in this [GitHub Gist](https://gist.github.com/deque-blog/fed5c7def19a4c75908d37377a8b8e31).

### Critics

It is important to notice that this design **encapsulates more concerns** than the previous design did. It handles cross currency addition and also handles the conversions between currencies in a declarative way.

As a functional programming enthusiast and a Lisp lover, this design has a lot of appeal for me. If you ever implemented a small Lisp interpreter, I bet you like this design as much as I do.

Unfortunately, **this design is flawed**. The equality operator is broken. Depending on the way you assemble the expressions, two equivalent expressions may have very different AST:

```haskell
let x = add (add a (multiply b 2)) c  -- (a + b * 2) + c 
let y = add a (add (multiply b 2) c)  -- a + (b * 2 + c)

x == y                                -- Testing equality
> False                               -- Not what we expect
```

Graphically, these two equivalent expression will have the following AST:

```
         (+)                     (+)
        /   \                   /   \
      (+)   1000 JPY       30 USD   (+)
     /   \                         /   \
30 USD   (*2)                    (*2)  1000 JPY
          |                       |
        25 EUR                  25 EUR
```

The problem with this implementation is that it **keeps too much information about the initial expression**. So much that it becomes sensible to factors such as operator precedence. We will fix this in the next section.

## Fixing the Money expression

Separating the specification from the evaluation of an addition of Money allowed us to support cross-currency addition, quite an interesting feature. But our previous implementation is broken with regards to equality. In this section, we will fix this defect, and go back to the old idea of Money Bag from [Ward Cunningham](https://twitter.com/WardCunningham).

### How it works

This implementation keeps the Kent Beck’s design we presented in the previous section. We keep the decoupling between the specification of an addition and its evaluation. We keep the evaluator.

The only difference is the data representation of a money expression. We drop the AST (which captured too much information as we have seen) and use an **associative container** (a map) instead. This map associates each currency to its corresponding amount.

Now, adding money expressions is a simple matter of **merging their associated map**. And multiplying a money expression consists in multiplying the amount of each currency.

Use associative contains correctly fixes our previous defect with equality: two money expressions are equal if their keys are the same and if each keys is associated to the same value.

### Implementation

The API remains mostly unchanged. The implementation details are quite different though. We will look at the main changes.

First, we drop the AST in favor of a map. We also rename the Money expression into `MoneyBag`, in honor of Ward Cunningham, but also because it is closer to what it now is:

```haskell
newtype MoneyBag                    -- A Money Bag is a wrapper around
  = MoneyBag (Map Currency Amount)  -- * A map from currency to amount
  deriving (Show, Eq)               -- Equality is automatically generated
```

Adding two money bags is implemented as the union of two maps:

```haskell
add :: MoneyBag -> MoneyBag -> MoneyBag
add (MoneyBag m1) (MoneyBag m2) = MoneyBag (unionWith (+) m1 m2)
```

Multiplying a money bag with a constant consists in mapping over each amount:

```haskell
multiply :: MoneyBag -> Double -> MoneyBag
multiply (MoneyBag m) f = MoneyBag (fmap (* f) m)
```

The evaluator is quite simple too (3 lines of code). It consists in collapsing the map into a one that only contains a single currency key, the one we want to convert too. Again, this is outside the scope of this post, but the implementation is nevertheless provided as reference in this [GitHub Gist](https://gist.github.com/deque-blog/9ee91432af94905cfbb5b3da4fbcb4e5).

### Critics

This design has basically the same advantages than the Kent Beck’s design. It also handles cross currency addition and handles the conversions between currencies in a declarative way.

But now it works with equality. It also offers a more compact data representation in most cases (same currencies are collapsed). The only drawback is that multiplication has to scan over all the currencies (*).

Compared to the Martin Fowler’s design, it remains heavier. It consumes more memory and more CPU, at the price of managing addition in a truly encapsulated way. This is a trade-off.

_(*) This could be fixed by adding the factor as a separate field, delaying its application until the very moment it is needed (evaluation time and merging of money bags)_

## Improving on Martin Fowler’s Money class with Idris

In the original Money class design of Martin Fowler, the client code is expected to check that both amounts have the same currency before calling add on them.

The main problem in this design is that the type system does not offer any help. It does not enforce this constraint and failing to check the precondition is not caught at compile time. The probability of error is high and the client is left without support.

Let us see how Idris and its dependent typing feature might help us provide a type safe version of the Martin Fowler’s Money class, such that failing to check the precondition will not compile.

### The hole in our type systems

The type system of most mainstream languages is **unable to express relations and constraints between several arguments** of a function. Even in Haskell, it is hard to express a precondition that involves several arguments with a type, such that the compiler might verify it for us.

Idris is not bound to the same limitation. Its type system allows to build types that express relations between several values, or several arguments of the same function. For instance, we can write a type that represents the fact that two money instances have the same currency.

Put differently, we can **express preconditions on our functions as types**, allowing the compiler to verify for us that these preconditions are satisfied before calling the function.

### Translating Haskell to Idris

Let us go back to our initial implementation of `add` in Haskell and translate it into Idris. Idris being pretty close to Haskell syntax, this is a rather simple task:

```haskell
record Money where      -- Definition of Money
  constructor MkMoney   -- * With a constructor named MkMoney
  currency : Currency   -- * Containing a currency
  amount : Amount       -- * And an amount of money

-- Definition of `add` which takes 2 Money instances and returns a new one
add : (m1, m2 : Money) -> Money
add m1 m2 = MkMoney (currency m1) (amount m1 + amount m2)
```

For now, this implementation is incorrect if the two money instances we try to sum together, `m1` and `m2`, have different currencies. Indeed, the new Money instance is constructed from the currency of `m1` and we have no assert to ensure both money arguments have the same currency.

### Preconditions as types

To fix this, we can enrich the prototype of the function `add` to ask for a proof (the argument `prf` below) that the money arguments have the same currency:

```haskell
add :
  (m1, m2 : Money)                    -- Arguments of the function: 2 Money instances
  -> {auto prf : SameCurrency m1 m2}  -- Proof that m1 and m2 have the same currency
  -> Money                            -- Return type: a new Money instance
```

In this function prototype, `SameCurrency m1 m2` is a type which represents a constraint between the arguments `m1` and `m2` of the function. And in Idris, we can effectively build this type such that being able to instantiate this type is equivalent to proving that the constraint is satisfied (*).

The `auto` keyword tells Idris to look for implicit proofs in the neighborhood of the function. Thanks to this, the following code will compile just fine, and does not require us to provide a proof of its correctness:

```haskell
-- Compile just fine: same currencies
add (MkMoney "EUR" 1) (MkMoney "EUR" 2)
```

The following code will correctly be rejected at compile time, Idris being unable to find a proof that the two currencies “EUR” and “USD” are equal:

```haskell
-- Does not compile: different currencies
add (MkMoney "EUR" 1) (MkMoney "USD" 2)
```

It is important to observe the distinguishing characteristic of Idris there: a type can refer to the other arguments that appear in the prototype of a function.

_(*) Note: you can refer to my [previous article]({% link _posts/2017-07-01-idris-bowling-kata.md %}) on building a type safe Bowling Kata for more detailed information on how to write proofs in Idris._

### Runtime checks

In our previous examples, Idris is able to find an implicit proof that two currencies are either equal or not equal by looking at their values at compile time. For values that are only known at runtime though, Idris cannot do miracles.

We have to provide in our API a function to construct a proof that two money instances known at runtime satisfy the precondition on their currency:

```haskell
sameCurrency :                  -- Predicate function
  (m1, m2 : Money)              -- * Taking two instances of Money
  -> Dec (SameCurrency m1 m2)   -- Returning a proof or a contradiction
```

We will not go in details into what `Dec` means here, but you can think of it as something that either returns *"Yes and here is the proof"* or *"No, and here is a contradiction"*. The implementation of this function is outside the scope of this post, but is provided as reference in this [GitHub Gist](https://gist.github.com/deque-blog/3ab28028c16895eba4536a188d70b1e6).

What matters is that the client code is now forced to call `sameCurrency` before being able to call `add` on runtime values. In fact, the it will only be able to call `add` if `sameCurrency` returns `YES`. Indeed, add requires a `SameCurrency m1 m2` instance to be available in its environment of evaluation. This is only possible if either:

* Idris can find an implicit proof that it holds (for values known at compile time)
* The proof is constructed successfully by `sameCurrency` (upon returning YES)

This sounds like magic, but this is no fairy tale. Type systems such as the one of Idris allow to **ensure that our if statements are correct at compile time**.

### Critics

This last design shares some important characteristics with the one proposed by Martin Fowler in [PEAA](https://www.amazon.com/Patterns-Enterprise-Application-Architecture-Martin/dp/0321127420/ref=sr_1_1?ie=UTF8&qid=1502550798&sr=8-1&keywords=Patterns+of+Enterprise+Application+Architecture). It is lightweight, and does not support adding amounts with different currencies. It basically shares the same pros and cons as the one from Martin Fowler.

The main difference is type-safety: the client code is now ensured to do the appropriate precondition checks before adding amounts. Failing to do the appropriate checks will result in a compilation error.

## Conclusion

The design space of a common class such as Money is surprisingly large. There does not seem to be a single best answer for all situations.

Depending on the required capabilities, such as being able to sum amounts of different currencies, an the performance constraints, it might be more advantageous to chose either the Money or the MoneyBag implementation.

As an opening, we also saw how new type systems with a support for dependent typing, such as the one of Idris, can offer yet more design choices and more compile time guarantees.

_EDIT: I published a new blog post to explain why lifting currencies as type parameters of the Money class would not bring the same benefits as the Idris solution [here]()._