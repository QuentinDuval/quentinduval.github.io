---
layout: post
title: "Paramorph your DSL (C++)"
description: ""
excerpt: "Let's dive in the implementation of paramorphisms in C++, a strictly more powerful recursion scheme than catamorphism, all motivated by an example."
categories: [functional-programming]
tags: [Functional-Programming, C++]
---

In previous posts, we went over the process of building a DSL for arithmetic operations and introduced the concept of Catamorphism as a way to **decouple the traversal of an AST** from the operations we want to perform on it.

We saw how it could help us compose operations before traversing the AST of our DSL, leading to **more efficient composition and better testability**.

We first did this exercise in Haskell, introducing the concept of Catamorphism through its wonderful type system before exploring the limits of its applicability in C++. The full series of post is available below for reference:

* [Catamorph your DSL: Introduction]({% link _posts/2017-01-17-catamorph-dsl-intro.md %})
* [Catamorph your DSL: Deep Dive]({% link _posts/2017-01-20-catamorph-dsl-deep-dive.md %})
* [Catamorph your DSL: Trade-offs]({% link _posts/2017-01-23-catamorph-dsl-tradeoffs.md %})
* [Catamorph your DSL: Clojure]({% link _posts/2017-01-26-catamorph-dsl-clojure.md %})
* [Catamorph your DSL: C++ Port]({% link _posts/2017-01-30-catamorph-dsl-cpp.md %})

This post will build on top of the [Catamorph your DSL: C++ Port]({% link _posts/2017-01-30-catamorph-dsl-cpp.md %}). We will introduce, through a motivated use case, another very close and useful recursion scheme, Paramorphisms, and try to implement them in C++.

## Reminder: our arithmetic DSL

Our Arithmetic DSL allows to build expression composed of operations like addition and multiplication of integer constants and integers variables. An arithmetic expression is one of the following:

* A constant value (and integer for simplicity)
* A variable (identified by a string for simplicity)
* An addition of a vector of sub-expressions
* A multiplication of a vector of sub-expressions

Because an AST alone is not very useful, we added several interpreters on our DSL to play with it:

* `prn` to pretty print our Arithmetic expressions
* `eval` to compute the value of an expression, given the value of all variables
* `optimize` to simplify and optimize our expression
* `partial` to partially evaluate our expression, given the value of some variables
* `dependencies` to retrieve not yet resolved variables from our expression

## Reminder: recursion schemes

Looking at our interpreters, we noticed in our previous post that they all had in common the need to traverse the full AST. We used Catamorphisms to factorize this concern out of our interpreters.

This section is a short overview of what Catamorphism are for and how we implemented them in C++. You can skip it if you feel comfortable with the notion already.

### Open recursion on types

To use Catamorphisms, we first need to generalize our Arithmetic `expression` and make it an open recursive data type: `expression_r` templated on a parameter type `R`, which represents the type of the sub-expressions.

```cpp
using nb = int;
using id = std::string;

struct add_tag {};
struct mul_tag {};

template<typename Tag, typename R>
struct op
{
   op() = default;

   template<typename Range>
   explicit op (Range const& rng) : m_rands(rng.begin(), rng.end()) {}
   
   std::vector<R> const& rands() const { return m_rands; }
   
private:
   std::vector<R> m_rands;
};

template<typename R> using add_op = op<add_tag, R>;
template<typename R> using mul_op = op<mul_tag, R>;

template<typename R>
using expression_r = boost::variant<int, id, add_op<R>, mul_op<R>>;
```

An expression of our DSL is then defined such as the template parameter `R` is itself an expression. In more formal terms, `expression` is the fixed point of the `expression_r` type. We can compute it using CRTP and `boost::recursive_wrapper`.

```cpp
struct expression
   : boost::recursive_wrapper<expression_r<expression>>
{
   using boost::recursive_wrapper<expression_r<expression>>::recursive_wrapper;
};
```

### Functor instance

We made our `expression_r` type an instance of a Functor by implementing a `fmap` function for our type. Note that by Functor here, we mean the [mathematical construct](https://en.wikipedia.org/wiki/Functor), and not function objects.

The transformation map given to fmap applies on the template parameter of the `expression_r` and may modify its type. In other words, `fmap` allows to transform the sub-expressions.

```cpp
template<typename A, typename M>
auto fmap(M map, expression_r<A> const& e)
{
   using B = decltype(map(std::declval<A>()));
   using Out = expression_r<B>;
   
   if (auto* o = get_as_add(e))
      return Out(add_op<B>(o->rands() | transformed(map)));
      
   if (auto* o = get_as_mul(e))
      return Out(mul_op<B>(o->rands() | transformed(map)));
   
   if (auto* i = get_as_cst(e)) return Out(*i);
   if (auto* v = get_as_var(e)) return Out(*v);
   throw_missing_pattern_matching_clause();
}
```

_Note: because the function provided to fmap can have different types as input and output, the type of the template parameter of expression_r can change under fmap, and with it, the nature of the recursion._

### Generic tree traversal

We then defined the `cata` function, which implements the Catamorphism recursion scheme. It takes what we will call an `algebra` as parameter, and returns a transformation on a whole AST. The `algebra` is a function that takes an `expression_r` templated on `Out`, and outputs an Out value.

```cpp
// Catamorphism algebra
Out catamorphism_algebra(expression_r<Out> const& e);
```

The result of `cata` on the algebra (curried) is a function that applies on a full expression and returns an `Out` value:

```cpp
template<typename Out, typename Algebra>
Out cata(Algebra f, expression const& ast)
{
   return f(
      fmap(
         [f](expression const& e) -> Out { return cata<Out>(f, e); },
         ast.get()));
}
```

For instance, the following `print_alg` algebra transforms an `expression_r` templated on string into a string. Its goal is to implement a “pretty print” of the expression at one stage of the expression alone.

```cpp
template<typename Tag>
std::string print_op(op<Tag, std::string> const& e, std::string const& op_repr)
{
   return std::string("(") + op_repr + " " + boost::algorithm::join(e.rands(), " ") + ")";
}

std::string print_alg(expression_r<std::string> const& e)
{
   if (auto* o = get_as_add(e)) return print_op(*o, "+");
   if (auto* o = get_as_mul(e)) return print_op(*o, "*");
   if (auto* i = get_as_cst(e)) return std::to_string(*i);
   if (auto* v = get_as_var(e)) return *v;
   throw_missing_pattern_matching_clause();
}
```

Given to `cata`, it will become an function that transforms a complete expression into a string, basically being promoted to operate on all stages of the expression.

```cpp
expression e = add({
   cst(1),
   cst(2),
   mul({cst(0), var("x"), var("y")}),
   mul({cst(1), var("y"), add({cst(2), var("x")})}),
   add({cst(0), var("x")})
   });

std::cout << para<std::string>(print_infix, e) << std::endl;

// Outputs =>
1 + 2 + 0 * x * y + 1 * y * (2 + x) + 0 + x
```

## The use case that breaks the pattern

Every single technique we use in Software Engineering has its limits. Outside of these limits, the technique will either not apply or might not be the best tool for the job. Catamorphisms are no exception to this rule.

Because Catamorphisms are a way to factorize a particular recursion scheme, it cannot apply for recursions that do not follow the given pattern. We will now explore one use case that demonstrates these limits.

### Infix notation

Let us imagine that the client of our arithmetic DSL does not like the prefix notation of our pretty print function. Instead, he would like the arithmetic expressions to be pretty printed in infix notation. For example, `1 + y + x * 3` instead of `(+ 1 y (* x 3))`.

Now, and contrary to the prefix notation which follows a pretty simple parenthesizing scheme, pretty printing in infix notation requires us to be careful. A wrong use of parentheses could change the meaning of the expression.

### A first attempt

As a first approximation, we could use the following strategy for the placement of the parentheses in our infix notation:

* Systematically surround the arguments of a multiplication with parentheses
* Never surround the arguments of an addition with parentheses

Implementing the pretty printer that follows this strategy can be done using Catamorphisms. We use Boost to make the code simpler:

```cpp
std::string print_infix_op_bad(op<add_tag, std::string> const& e)
{
   return boost::algorithm::join(e.rands(), " + ");
}

std::string with_parens(std::string const& s)
{
   return std::string("(") + s + ")";
}

std::string print_infix_op_bad(op<mul_tag, std::string> const& e)
{
   return boost::algorithm::join(e.rands() | transformed(with_parens), " * ");
}

std::string print_infix_bad(expression_r<std::string> const& e)
{
   if (auto* o = get_as_add(e)) return print_infix_op_bad(*o);
   if (auto* o = get_as_mul(e)) return print_infix_op_bad(*o);
   if (auto* i = get_as_cst(e)) return std::to_string(*i);
   if (auto* v = get_as_var(e)) return *v;
   throw_missing_pattern_matching_clause();
}
```

We can try this first implementation on our test arithmetic expression:

```cpp
expression e = add({
   cst(1),
   cst(2),
   mul({cst(0), var("x"), var("y")}),
   mul({cst(1), var("y"), add({cst(2), var("x")})}),
   add({cst(0), var("x")})
   });

std::cout << cata<std::string>(print_infix_bad, e) << std::endl;

// Outputs =>
1 + 2 + (0) * (x) * (y) + (1) * (y) * (2 + x) + 0 + x
```

What we get is is a correct result: the pretty printed expression has indeed the right meaning. But there are so much unneeded parentheses that the expression is really hard to read. We have to do much better.

### Improving parenthesizing

To get better results, we need our infix notation to avoid adding useless pairs of parentheses. We know how to fix this: a multiplication only needs to add parentheses around sub-expressions that correspond to additions.

Unfortunately, Catamorphisms are not a powerful enough recursion scheme to implement this logic. The algebra given to the Catamorphism has no access to the sub-expressions, only their resulting string.

As a consequence, there is no way to know whether the expression was previously an addition (except by parsing it, which would truly be awful). The Catamorphism has failed us here: we need to turn to a different recursion scheme.

## From catamorphism to paramorphism

The previous use case demonstrated us that we need the information on whether a sub-expression is an addition when dealing with the multiplication.

We will turn to the recursion scheme known as [Paramorphism](https://en.wikipedia.org/wiki/Paramorphism), which is like a Catamorphism but with added contextual information.

### Paramorphisms

Paramorphisms are pretty close to Catamorphisms. The goal is identical: it turns a local transformation on an open recursive data structure into a global transformation on the fixed point of this data structure.

The difference is the prototype of the algebra given as parameter to the recursion scheme. Instead of taking the open recursive type parameterized on the output type of the transformation, the algebra takes a an open recursive type parameterized on a pair:

* The first element of the pair is the output of the sub-expression transformation
* The second element of the pair is the sub-expression before the transformation

In more concrete terms, and applied to our arithmetic expression, here are the prototypes for the algebra of catamorphism and paramorphism:

```cpp
// CATAMORPHISM
Out catamorphism_algebra(
  expression_r<Out> const& e);

// PARAMORPHISM
Out paramorphism_algebra(
  expression_r<std::pair<Out, expression const*>> const& e);
```

The second element of the pair is the contextual information. In our specific case, it will provide us with the information about the sub-expression needed to make the right parenthesizing choice.

### Paramorphism implementation

We will now implement the `para` function, whose goal is to transform a local algebra into a global transformation on an arithmetic expression.

The implementation is very similar to the one of the Catamorphism. The only modification concerns the lambda provided to the `fmap` function, which needs to return a pair instead of a single value:

```cpp
template<typename Out, typename Algebra>
Out para(Algebra f, expression const& ast)
{
   return f(
      fmap(
         [f](expression const& e) -> std::pair<Out, expression const*> {
            return { para<Out>(f, e), &e };
         },
         ast.get()));
}
```

### Example: implementing the infix notation

We now have enough contextual information to implement our infix notation properly. For each sub-expression of addition and multiplication, our algebra has access to:

* The pretty printed sub-expression (first argument of the pair)
* The sub-expression itself (second argument of the pair)

We can therefore implement the pretty printing of the multiplication such that it adds parentheses around addition sub-expressions only.

```cpp
std::string print_op_infix(op<add_tag, std::pair<std::string, expression const*>> const& e)
{
   auto fst = [](auto const& e) { return e.first; }; 
   return boost::algorithm::join(e.rands() | transformed(fst), " + ");
}

std::string print_op_infix(op<mul_tag, std::pair<std::string, expression const*>> const& e)
{
   auto wrap_addition = [](auto const& sub_expr) {
      if (get_as_add(sub_expr.second->get()))
         return with_parens(sub_expr.first);
      return sub_expr.first;
   };
   return boost::algorithm::join(e.rands() | transformed(wrap_addition), " * ");
}

std::string print_infix(expression_r<std::pair<std::string, expression const*>> const& e)
{
   if (auto* o = get_as_add(e)) return print_op_infix(*o);
   if (auto* o = get_as_mul(e)) return print_op_infix(*o);
   if (auto* i = get_as_cst(e)) return std::to_string(*i);
   if (auto* v = get_as_var(e)) return *v;
   throw_missing_pattern_matching_clause();
}
```

We can try this second implementation on our test arithmetic expression. This result is clearly much better. The resulting infix notation has no unnecessary parentheses left:

```cpp
expression e = add({
   cst(1),
   cst(2),
   mul({cst(0), var("x"), var("y")}),
   mul({cst(1), var("y"), add({cst(2), var("x")})}),
   add({cst(0), var("x")})
   });

std::cout << para<std::string>(print_infix, e) << std::endl;

// Outputs =>
1 + 2 + 0 * x * y + 1 * y * (2 + x) + 0 + x
```

## Conclusion

This was the second post dedicated to the implementation of the recursion schemes often used in Haskell into C++.

Through a motivating use case, we discovered the limits of the Catamorphism recursion scheme, and learned about a new strictly more powerful one: Paramorphism.

Although typically unused in C++, we managed to provide a concise implementation of it. The result is clearly not as short and beautiful than in Haskell, but has the merit to show that it is not outside of the expressiveness range of C++.

You can access the full code of the DSL, along with all its associated interpreters, in [this Gist](https://gist.github.com/deque-blog/d683c26256d9724fc9edadab45c2cc08) or alternatively in [Ideone](http://ideone.com/hhCNvK).
