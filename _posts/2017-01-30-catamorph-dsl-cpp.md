---
layout: post
title: "Catamorph your DSL (C++ Port)"
description: ""
categories: [functional-programming]
tags: [Functional-Programming, C++]
---

In the previous posts, we went over the process of building a DSL for arithmetic operations and we introduced the concept of Catamorphism as a way to decouple the traversal of the AST from the operations we want to perform on it.

We saw how it could help us compose operations before traversing the AST of our DSL, leading to more efficient composition and better testability.

Unless you are quite familiar with catamorphisms, you might want first to read these previous posts before this one:

* [Catamorph your DSL: Introduction]({% link _posts/2017-01-17-catamorph-dsl-intro.md %})
* [Catamorph your DSL: Deep Dive]({% link _posts/2017-01-20-catamorph-dsl-deep-dive.md %})
* [Catamorph your DSL: Trade-offs]({% link _posts/2017-01-23-catamorph-dsl-tradeoffs.md %})

All these previous posts were based on Haskell. Because Haskell is such a special (wonderful) language, the last post started to explore whether this concept would port as nicely in other languages.

We translated our Haskell code into Clojure, a functional language that put much less emphasis on types as Haskell. We saw that this concept translated to a post-order depth first search. The resulting Clojure code was both short and elegant.

This post will be focused on exploring the limits of the applicability of this concept some more, by trying to bring it into C++.

## Boost to the rescue

We could implement our DSL in C++ with only the STL available, using Design Patterns such as the Visitors. To keep the code short and concise, we will however rely boost as well. All the code that follows will assume the following Boost header inclusions:

```
#include <boost/algorithm/string/join.hpp>
#include <boost/range/algorithm.hpp>
#include <boost/range/adaptors.hpp>
#include <boost/range/numeric.hpp>
#include <boost/variant.hpp>
```

In particular, _boost::variant_ will be one of the cornerstone of the design, as it allows to avoid the Visitor design pattern. You might want to have a look at the documentation of _boost::variant_ before proceeding.

The rest of the header are mostly for convenience. We could do without it, but at the expense of some more code to write.

## Arithmetic expressions in C++ Boost

There are many ways we could represent our expression DSL in C++. We could choose to use the Visitor design pattern for example. The solution below is based instead of boost::variant for convenience.

The recursive expression_r implements our open recursive data type. This type is implemented as a boost::variant on:

* _nb_ which represents an integer constant
* _id_ which represents a variable identifier
* *add_op* which represents an addition operation
* *mul_op* which represents an multiplication operation

The *add_op* and *mul_op* operations are themselves factorized as a single tagged op data type. Two tags represent the two operations, *add_tag* and *mul_tag*.

```
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

To get our recursive expression, we will use the same trick as we used in Haskell: we will compute the fixed point of the type recursion.

As in Haskell, C++ is not able to create infinite types. The Fix wrapper type of Haskell needed to circumvent this limitation is replaced by the boost::recursive_wrapper. The overall design is the same:

```
struct expression
   : boost::recursive_wrapper<expression_r<expression>>
{
   using boost::recursive_wrapper<expression_r<expression>>::recursive_wrapper;
};
```

*Note: this naive implementation triggers deep copies at each level of our AST. This might lead to a quadratic behaviour in the depth of our tree when playing with transformations on our AST. A production ready implementation would have to handle these aspects we voluntarily left out for simplicity. We will come back to this issue in the conclusion.*

## Factory functions and accessors

The trouble with constructors of class templates in C++ is that they require their explicit templates to be listed in the code. The usual trick is to define factory functions outside of the class to automatically deduce the template parameters.

We will define some factory functions to help us write code more easily. These functions are somehow similar to the “smart constructors” of Haskell:

```
expression cst(int i) { return expression(i); };

expression var(id id) { return expression(id); };

expression add(std::vector<expression> const& rands)
{
   return expression(add_op<expression>{ rands });
}

expression mul(std::vector<expression> const& rands)
{
   return expression(mul_op<expression>{ rands });
}
```

Having introduced this layer of “named constructors” allows us create expression more succinctly and to better declare our intent:

```
expression e = add({
    cst(1),
    cst(2),
    mul({cst(0), var("x"), var("y")}),
    mul({cst(1), var("y"), cst(2)}),
    add({cst(0), var("x")})
    });
```

Similarly, to “pattern match” on our boost::variant, we will introduce some helper accessors functions. Indeed, the trouble with boost::get is that it requires explicit templates to be listed in the code. Dedicated and correctly named functions will simplify the pattern matching greatly by removing noisy templates from our code:

```
template <typename T>
int const* get_as_cst(expression_r<T> const& e)
{
   return boost::get<int>(&e);
}

template <typename T>
id const* get_as_var(expression_r<T> const& e)
{
   return boost::get<id>(&e);
}

template <typename T>
add_op<T> const* get_as_add(expression_r<T> const& e)
{
   return boost::get<add_op<T>>(&e);
}

template <typename T>
mul_op<T> const* get_as_mul(expression_r<T> const& e)
{
   return boost::get<mul_op<T>>(&e);
}
```

*Note: boost::variant also offers visitors to handle pattern matching as an alternative to boost::get. But as our interpreters will not need to exhaustively match all cases, sticking to boost::get ends up being less code. This is ultimately a matter of taste: we could have done the other choice instead.*

## Our C++ catamorphism

Now that we have our expression_r type, we will need to write the equivalent of the catamorphism function cata of Haskell. This function will be responsible for lifting local AST transformations into operations on the whole AST.

The details on how to build a catamorphism from an open recursive type were already covered in the previous post on catamorphism in Haskell. This section will only aim at translating this code into C++.

### Functor Instance

The first step is to make our expression_r type an instance of a Functor. In C++, one way to do it is to implement fmap for our type:

The map argument represents the transformation to apply
Constants and variable matches are left unchanged by this transformation
For operations, we recursively apply the transformation on the sub-expressions
We use boost::ranges to make the code more succinct:

```
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

### Catamorphism

Next, we need to write the catamorphism function, whose goal is to apply fmap at each level of our expression, starting from the lower levels.

The details behind the inner workings of this function are provided in a previous post on Haskell. We only translate here this definition to C++:

* We replace function composition in Haskell by nested function call in C++
* We use get to extract the expression from the recursive_wrapper

Some of the types listed below are deduced automatically. We left them in the code to show what is going on under the cover:

```
template<typename Out, typename Algebra>
Out cata(Algebra f, expression const& ast)
{
   return f(
      fmap(
         [f](expression const& e) -> Out { return cata<Out>(f, e); },
         ast.get()));
}
```

One noticeable drawback of this implementation is the need to provide the output type of the _cata_ function explicitly. This could probably be dealt with.

## Pretty printing our expressions in C++

We now have everything to start implementing our first interpreter, our expression pretty printer. It is noticeably more verbose than in both Haskell and Clojure, but not that much. I was quite surprised by the overall concision:

* *print_alg* handles pattern matching and the two trivial cases
* *print_op* handles the operations by joining the sub-expression strings, prepends the operation and wraps the whole with parentheses

```
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

At call site, the code is not that verbose either. The main drawback is the need to specify the output type of the catamorphism (std::string in this case):

```
expression e = add({
  cst(1),
  cst(2),
  mul({cst(0), var("x"), var("y")}),
  mul({cst(1), var("y"), cst(2)}),
  add({cst(0), var("x")})
  });

std::cout << cata<std::string>(print_alg, e) << '\n';

//Will output:
"(+ 1 2 (* 0 x y) (* 1 y 2) (+ 0 x))"
```

Now, this is not the most efficient code that we could come up with. It does create quite a lot of intermediary strings. This is the subject for another time.

## Eval and dependencies

We can as translate to C++ our next two most straightforward interpreters, *eval* and *dependencies*. The *eval* function needs an environment holding the value of the variables in the arithmetic expression. To make it simple and keep the code short, we will:

* Rely on a std::map from id to integer to model this environment
* Ignore errors such as an unbound variable in the environment

In addition, and to compensate for the absence of currying in Haskell, our eval function returns a lambda:

```
using env = std::map<id, nb>;

auto eval_alg(env const& env)
{
   return [&env] (expression_r<int> const& e)
   {
      if (auto* o = get_as_add(e))
         return boost::accumulate(o->rands(), 0, std::plus<int>());
         
      if (auto* o = get_as_mul(e))
         return boost::accumulate(o->rands(), 1, std::multiplies<int>());
      
      if (auto* v = get_as_var(e)) return env.find(*v)->second;
      if (auto* i = get_as_cst(e)) return *i;
      throw_missing_pattern_matching_clause();
   };
}

int eval(env const& env, expression const& expr)
{
   return cata<int>(eval_alg(env), expr);
}
```

The *dependencies* functions lists all the variables of the expression. It sub-expressions are transformed to a set of variable identifiers, which are combined (set union) while climbing up the AST.

```
template<typename Tag>
std::set<id> join_sets(op<Tag, std::set<id>> const& op)
{
   std::set<id> out;
   for (auto r: op.rands())
      out.insert(r.begin(), r.end());
   return out;
}

std::set<id> dependencies_alg(expression_r<std::set<id>> const& e)
{
   if (auto* o = get_as_add(e)) return join_sets(*o);
   if (auto* o = get_as_mul(e)) return join_sets(*o);
   if (auto* v = get_as_var(e)) return {*v};
   return {};
}

std::set<id> dependencies(expression const& e)
{
   return cata<std::set<id>>(dependencies_alg, e);
}
```

## Composable C++ optimisations

Until now, the main advantage of introducing Catamorphism was to specify local transformations, which the cata function could lift to operations on the whole AST of our arithmetic DSL.

We will now leverage on the other advantage of Catamorphism, the ability to compose them efficiently, to implement our arithmetic AST optimization functions.

We left out the detailed implementation of the has_zero and optimize_op functions in the snippet below. The detailed implementation of these functions is available on GitHub.

This allows us to concentrate on the real value added by Catamorphisms, composability of transformations:

* *opt_add_alg* is only concerned in optimising additions
* *opt_add_mul* is only concerned in optimising additions
* *optimize_alg* is only concerned in combining these algebra into one

This leads to the following code:

```
expression opt_add_alg(expression_r<expression> const& e)
{
   if (auto* op = get_as_add(e))
      return optimize_op(*op, 0, std::plus<int>());
   return e;
}

expression opt_mul_alg(expression_r<expression> const& e)
{
   if (auto* op = get_as_mul(e))
   {
      if (has_zero(op->rands()))
         return cst(0);
      return optimize_op(*op, 1, std::multiplies<int>());
   }
   return e;
}

expression optimize_alg(expression_r<expression> const& e)
{
   return opt_mul_alg(opt_add_alg(e).get());
}
```

Note that how the *optimize_alg* function is able to combine the two algebra by composing them. The call to get allows to unwrap the expression of the first algebra, before calling the second algebra. We can give it a try:

```
expression e = add({
  cst(1),
  cst(2),
  mul({cst(0), var("x"), var("y")}),
  mul({cst(1), var("y"), cst(2)}),
  add({cst(0), var("x")})
});

//Initial expression
std::cout << cata<std::string>(print_alg, e) << '\n';

//Will output
"(+ 1 2 (* 0 x y) (* 1 y 2) (+ 0 x))"

//Optimized expression
auto o = cata<expression>(optimize_alg, e);
std::cout << cata<std::string>(print_alg, o) << '\n';
  
//Will output
"(+ (* y 2) x 3)"
```

## Partial evaluation

Our last remaining challenge is to implement the partial evaluation of an arithmetic expression.

The partial_eval_alg simply replaces variables bound in the evaluation environment with their respective value. As for our evaluation function, our partial evaluation returns a lambda to compensate for the lack of currying in C++.

```
auto partial_eval_alg(env const& env)
{
   return [&env] (expression_r<expression> const& e) -> expression
   {
      if (auto* v = get_as_var(e))
      {
         auto it = env.find(*v);
         if (it != env.end()) return cst(it->second);
         return var(*v);
      }
      return e;
   };
}
```

We can combine this function with our optimization functions to obtain a fully fledged partial application that simplifies expression in one single tree traversal.

```
expression partial_eval(env const& env, expression const& e)
{
   return cata<expression>(
      [&env](expression_r<expression> const& e) -> expression {
         return optimize_alg(partial_eval_alg(env)(e).get());
      },
      e);
}
```

We can also re-implement our evaluation function in terms of our partial evaluation:

* If the resulting expression, after optimizations, is a constant, we output it.
* Otherwise, we can use our dependency function to output the missing variables

This leads to the following evaluation function, which handles errors much nicer than our previous one:

```
int eval_2(env const& env, expression const& e)
{
  auto reduced = partial_eval(env, e);
  if (auto* i = get_as_cst(reduced.get()))return *i;
  throw_missing_variables(dependencies(reduced));
}
```

We can give it a try to convince ourselves that it works:

```
expression e = add({
  cst(1),
  cst(2),
  mul({cst(0), var("x"), var("y")}),
  mul({cst(1), var("y"), cst(2)}),
  add({cst(0), var("x")})
});

//Environment of evaluation
env full_env = {
    {"x", 1},
    {"y", 2}
};

//Evaluations
std::cout << eval(full_env, e3) << '\n';
std::cout << eval_2(full_env, e3) << '\n';

//Will output
8
8
```

## Conclusion

In this post, we managed to port the Catamorphism notion in C++ to build a series of arithmetic DSL interpreters made of:

* Composable local operations (our different algebra)
* Decoupled from the traversal of the AST (handled by cata)
* Avoiding any kind of mutations on our AST

The complete resulting code is about three times the length of the equivalent Haskell or Clojure code (320 lines give or take), which is quite good overall. But there is a catch: this implementation is unrealistically simplistic. This is the topic of the next section.

### What's the catch?

Our simplistic memory management did exhibit some problematic quadratic behaviour in terms of copy. Addressing this concern would require some manual work: C++ offers no standard support for immutable data structure with structural sharing in its standard library, so we would probably have to rely on move semantics.

We also had some remaining issues regarding type deduction, which would require some more tricks to improve. The biggest issue is however dealing with the long error messages GCC outputs upon the simplest type error. Try removing the .get() in the catamorphism function, or messing up with the types of an algebra function, and you shall see. Concepts could probably help there.

So I do not think that the 320 lines of C++ code do quite compare in maturity with the 120 corresponding lines in Haskell.

### On catamorphims in C++

This concludes our study of the applicability of Catamorphism in C++. Overall, I felt like this design quite not as natural as it was in Haskell or Clojure. It sometimes felt like going against the idiomatic use of the language, which is not a good sign.

I recently read some good articles on the functional paradigm bursting in into C++ (like this series of posts on http://www.modernescpp.com).

Although this is true that C++ made great leaps in this direction by encouraging higher order functions usage with the addition of lambdas, it felt awkward using them in this context.

But the main difficulty in porting this concept to C++ is that it is implicitly based on efficient persistent data structure to exist in the language. And the support for immutable data and persistent data structures in C++ is simply not there.

To summarize, implementing catamorphisms in C++ was quite a fun exercise (except for the compilation error messages spanning over 2 or more screens), but it did not feel like it fits C++ as it does for Haskell or Clojure.
