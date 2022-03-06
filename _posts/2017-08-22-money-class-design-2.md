---
layout: post
title: "Money class design: why not having currencies as type parameters?"
description: ""
excerpt: "A follow up on the study of the ideal design for the Money class, answering an interesting question on Reddit."
categories: [software-design]
tags: [Functional-Programming, Haskell]
---

In the [previous post]({% link _posts/2017-08-17-money-class-design.md %}), we went through a comparative study of 4 different designs for a Money class that represents an amount of money in a given currency. This article received a quite positive review, and also a lot of comments and remarks.

One of the most recurring remark was: **why not encode the currencies as types parameters of the Money class?** If we do so, the currencies will be visible in the type system and should allow us to implement safe arithmetic.

This is a fair and interesting remark, especially since it appears to be such a good design solution at first sight.

In this post, we will first present the solution in details, before explaining why it is not really a viable option for the Money class. We will conclude this post with some words of advice regarding using strong types, and not using them.

## Summary of the last episode

We first start with a quick recap on the 4 different designs we evaluated so far.

### Two broad category of designs

In the [previous post]({% link _posts/2017-08-17-money-class-design.md %}), we went trough two broad categories of design for the Money class.

* Martin Fowler’s design was about forbidding cross-currency addition with an assertion
* Kent Beck and Ward Cunningham designs implemented a sound cross currency addition

In short, we could either choose to support the addition of money amounts in different currencies, or to forbid it.

### Martin Fowler's approach

The design of [Martin Fowler](https://twitter.com/martinfowler) relies on adding a runtime check (an assertion) to forbid cross-currency money addition. In this design, it is invalid to call add on two amounts with different currencies, and the client of the API is responsible for ensuring these preconditions are fulfilled.

The only problem with this solution is that the client of the API might forget to do those checks. If so, the assertion will catch the violation at runtime and not at compile time, hopefully before production.

### Type safe preconditions with Idris

Using Idris, we then improved the design proposed by Martin Fowler to make it more type-safe. We made sure **at compile time** that the add function could only be called in the conditional branches for which the runtime checks of the precondition were done and returned a positive answer.

```haskell
case sameCurrency m1 m2 of  -- Forced to call the predicate
  Yes _ => add m1 m1        -- Can only call `add` in this branch
  No _  => ...              -- Cannot call `add` in this branch
```

It is important to understand that this last solution is not about **ensuring that the structure of the code is correct**. It is about ensuring that the runtime checks are always called correctly before calling add. It is not about ensuring that the two currencies are the same at compile time (it cannot be done if values are created dynamically).

## Currencies as type parameters

Now, let us have a look at the proposal that consists in lifting the currencies as type parameters of the Money class. We will first implement it in this section. The next section will then explain why it does not make for a great design.

### Motivation

The motivation behind this design is to answer the same concern than the Idris design. The goal is to improve on Martin Fowler’s design to make it more type-safe. The goal is to forbid cross-currency addition and to catch violation of that rule at compile time.

To achieve that, the proposed solution is to parametize the Money class with a currency. The currency becomes a type parameter of the Money class, such that we can play with it in the type system.

This solution was proposed by different persons (and in different languages) such as [@MinskAD](https://twitter.com/MinskAD), [u/bstempi](https://www.reddit.com/user/bstempi) or [u/zokier](https://www.reddit.com/user/zokier). The implementation showed below is greatly inspired from these comments, and especially the one of [u/zokier](https://www.reddit.com/user/zokier) in this [reddit post](https://www.reddit.com/r/programming/comments/6u8lq6/a_study_of_4_money_class_designs_featuring_martin/) (and the [Gist](https://gist.github.com/anonymous/acd21e856d57760b0976357a88a4d4a2) associated to it).

### How it works

The solution consists in encoding the currency as a type parameter of the `Money` class, invisible in the runtime representation of the class, a technique reminiscent of [Phantom types](https://wiki.haskell.org/Phantom_type) in Haskell.

Here is a (simplistic) implementation that illustrates this design, where `Currency` is a type, and `Money` is templated on a value of that type:

```cpp
template<Currency MoneyCurrency>
class Money
{
public:
   Money() : m_amount(Amount{}) {}
   explicit Money(Amount amount) : m_amount(amount) {}
   Amount amount() const { return m_amount; }

private:
   Amount m_amount;
};
```

Now that the currency appears as a type parameter, we can exploit it in the C++ type system and write an add function that it only works for two Moneys templated on the same currency:

```cpp
template<Currency SameCurrency>
Money<SameCurrency> operator+ (Money<SameCurrency> const& lhs, Money<SameCurrency> const& rhs)
{
   return Money<SameCurrency>{lhs.amount() + rhs.amount()};
}
```

This solves the problem for add. Money instances with the same currencies can be added normally, while trying to add Money instance with different currencies will fail to compile:

```cpp
auto eur = Money<Currency::EUR>{3};
auto usd = Money<Currency::USD>{5};
	
// Compiles files
auto sum_eur = eur + eur;

// Does not compile
auto invalid = eur + usd;
```

It looks like the problem is solved. But by doing so, this solution creates tons of other problems, as we will see in the next section.

## Down the rabbit hole

To notice the problems that arise with the above design, we need to look at how this class would integrate with the rest of the software. Looking at it in isolation is not enough: we will need to gradually add some more code around this class.

### Support for equality

Let us first add the support for equality. The original implementation provided by u/zokier in his proposal is limited to equality checks between amounts with the same currency.

We can remove this limitation by playing with pattern matching on the currency type parameter of the Money class. We can make sure that comparing Money instances with different currencies always returns false, and otherwise correctly compares the amounts:

```cpp
template<Currency SameCurrency>
bool operator==(Money<SameCurrency> lhs, Money<SameCurrency> rhs)
{
  return lhs.amount() == rhs.amount();
}

template<Currency LeftCurrency, Currency RightCurrency>
bool operator==(Money<LeftCurrency> lhs, Money<RightCurrency> rhs)
{
  return false;
}
```

It works because the most specialized overload will be preferred to the most general one. Now, our equality operator works across moneys amounts in different currencies:

```cpp
// Returns false: different currencies
Money<Currency::EUR>{3} == Money<Currency::USD>{3};

// Returns true: same currency and same amount
Money<Currency::EUR>{3} == Money<Currency::EUR>{3};

// Returns false: same currency but different amount
Money<Currency::EUR>{3} == Money<Currency::EUR>{5};
```

So with a bit of pattern matching on types, we managed to implement equality on Money amount in any currency. It was not hard, but it shows a first sign of why this design is not that great.

### What just happened

By lifting currencies from values to type parameters, we split the concept of Money into several types, one for each currency. When implementing equality, we discovered that some operations do operate on Money amounts with different currencies. This should be a warning sign.

If it so happens that most operations on Money do not require different types for different currencies (like equality), our new design would substantially increase [accidental complexity](https://www.quora.com/What-are-essential-and-accidental-complexity): the special case of add and made it every other money operation's concern.

For now, this is not that bad. We managed to circumvent this split in different types using a simple form of template meta-programming to implement our business logic. This came at the cost of just a bit of verbosity and additional complexity. But we might not be that lucky for other use cases.

### Storage becomes an issue

Containers such as std::vector cannot store values of different types. This means that an std::vector will not be able to contain Money amounts expressed in different currencies.

```cpp
std::vector<Money<Currency::USD>> usds;

// Compiles fine
usds.push_back(Money<Currency::USD>{5});

// Does not compile
usds.push_back(Money<Currency::EUR>{3});
```

This is a real big damn huge problem. Storing moneys with different currencies in a single container is a real need. We might for instance be interested in representing positions in a portfolio, or just represent a wallet. 

This time, solving this problem is not that easy. Simple meta-programming tricks will not do. We could turn to [boost::hana](http://www.boost.org/doc/libs/1_61_0/libs/hana/doc/html/index.html) and its heterogenous containers. It is a great library, but is arguably a bit too complex for such a simple use case.

The other solution would be to do some type erasure, but this would defeat the purpose of lifting the currencies as type parameters. We would lose both information and type-safety.

### Algorithms are out of reach

Because we cannot easily store Money instances with different currencies together in the same container, we cannot easily run algorithms on them either.

With currencies as type parameters, and as many Money types as there are currencies, answering the following needs is getting much more difficult:

* Counting how many currencies appear in a collection of Money.
* Summing the amounts of a collection of Money, by currencies.

Again, we could turn to [boost::hana](http://www.boost.org/doc/libs/1_61_0/libs/hana/doc/html/index.html) which has a nice collection of algorithms on heterogenous sequences. But again, this looks a bit overkill.

### Types are contagious!

The problem is that this design choice is **contagious**. It affects any part of the program which uses the `Money` class. There is no such thing as encapsulation of types (*). Any client code will see the currency as type parameter. Any **client code is coupled to this design decision**.

For instance, let us say we want a type to represent a [Foreign Exchange Spot](https://en.wikipedia.org/wiki/Foreign_exchange_spot), an exchange between two currencies at a given spot date. We are forced to parameterise our class on the two currencies:

```cpp
template<Currency BuyCurrency, Currency SellCurrency>
class foreign_exchange_spot
{
   Date m_spot;
   Money<BuyCurrency> m_buy;
   Money<SellCurrency> m_sell;
	
   // ... construtor, operations ...
};
```

The design of this new class is terrible on many aspects.

From a technical standpoint, instead of having one currency type parameter, we have two. The number of types the compiler will need to instantiate will grow as the square of the number of currencies. More complex financial products will require even more.

From a business logic standpoint, this is problematic as well. This kind of deal is fungible: we expect to store it in numbers and compress multiple instances of it into positions. Satisfying these needs is only made harder by the strong typing.

_(*) Actually there is. Sub-typing would buy us type erasure, but it would defeat the purpose of this design altogether. If we feel constraint to use type erasure, why not just drop this design entirely instead?_

## Conclusion

Types are great tool at ensuring invariants in a program. But they are not the tools for all kinds of invariants. For instance, lifting the currencies as type parameters of the Money class gives us a design much too rigid and far to cumbersome to deal with.

It might not look as interesting as first sight, but encountering a scenario in which a technique failed can actually teach us more than all the success stories on the same technique. Let us review some of the things we saw.

### Types have drawbacks

We have seen a pretty extreme example of the kind of problem that static typing can cause if used excessively. Although in general it does not lead to that much horror, it is often a good idea to assess to the cost and benefits of introducing a type (as with any other technique, it is almost never free).

Types represents a pretty strong form of coupling, whereby we force a client to comply with strict rules to interact with our code. This might be the desired effect. This might be justified. But it is not always a winning trade-off.

Be sure to verify that the cost of adding type safety does not exceed the benefits.

### Type safety VS flexibility

Other technics exist which offer different kinds of trade-offs, such as reduced type-safety. These technics have their pros and cons too, but should not be sacrificed at the altar of type-safety.

Technical limitations of the language might left us with a choice between practical software versus type-safety. Dropping type-safety might be the reasonable choice in some instance.

In our specific case, the design of Martin Fowler, although it is not as type safe, is a much better alternative compared to lifting currencies as type parameters of the Money class. The class is much less cumbersome to deal with, and the resulting software is much more flexible.

It all boils down to where we put the cursor of type safety. Too much of it, at the wrong spot, and the software turns rigid. Too little of it, and flexibility increases marginally at the cost of safety.

### The best of both worlds with dependent typing?

The alternative to compile time checks are runtime checks. In mainstream type systems, these runtimes checks comes at the cost of type-safety. This is due to value and types living in two separate universes.

Languages with support for dependent typing (like Idris) are able to breach the gap between values and types and provide **increased type-safety while avoiding a lot of the coupling costs associated to strong typing**.

We will explore more this topic in future posts.
