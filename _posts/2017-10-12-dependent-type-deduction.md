---
layout: post
title: "Why template parameters of dependent type names cannot be deduced, and what to do about it"
description: ""
excerpt: "C++ template parameters of dependent type names cannot be deduced. Let's see why C++ cannot do much better considering its type system and some basic maths."
categories: [modern-cpp]
tags: [C++]
use_math: true
---

Following the presentation [a type by any other name](https://cppcon2017.sched.com/event/BguD/a-type-by-any-other-name) at the CppCon, a great talk that shows how to incrementally refactor code using tooling and type aliases, I got interested in one of the observation made by Jon Cohen:

*Template parameters of dependent type names cannot be deduced*.

This short post aims at describing this issue and explaining why it is so. We will explain why C++ cannot do much better considering its type system and some basic mathematics. We will conclude by showing different ways to solve this issue, including one from Haskell.

## Introducing the problem

Before we dive into the explanations, we first have to describe the issue raised by Jon Cohen and introduce the notion of dependent names in C++.

### Dependent names

Inside the definition of a template, the meaning of some identifiers might depend on one or several template parameters. For instance in the following code, the meaning of `const_iterator` depends on the type `T`:

```cpp
template<typename T>
void f(Container<T> const& cont)
{
   Container<T>::const_iterator it = ...; //Ambiguous (and will not compile)
}
```

We call such identifiers *dependent names*.

Because C++ syntax uses overloading, such a name could in theory refer to a static member as much as a type. This ambiguity is the reason why the code above does not compile.

### Dependent type names

To get rid of the ambiguity, dependent names are not considered as referring to types _“unless the keyword typename is used or unless it was already established as a type name”_ (from [cppreference.com](http://en.cppreference.com/w/cpp/language/dependent_name)).

This is why we have to add the `typename` keyword to make the code above valid:

```cpp
template<typename T>
void f(Container<T> const& cont)
{
   typename Container<T>::const_iterator it = ...; // OK, const_iterator refers to a type
}
```

*Note: This was a rather quick introduction to dependent names and dependent type names. The reader is encouraged to look at the [cppreference.com](http://en.cppreference.com/w/cpp/language/dependent_name) or this [blog post from Eli Bendersky](https://eli.thegreenplace.net/2012/02/06/dependent-name-lookup-for-c-templates) for more detailed information on the subject.*

From now on, we will refer to these dependent names qualified with `typename` as *dependent type names*.

### Dependent type names as arguments

Dependent type names can appear inside the signature of function templates, as shown below in this over-constrained implementation of `for_each`:

```cpp
template<typename T, typename Consumer>
void for_each(
   typename std::vector<T>::const_iterator first, //dependent type on T
   typename std::vector<T>::const_iterator last,  //dependent type on T
   Consumer consumer)
{
   for (;first != last; ++first)
      consumer(*first);
}
```

While this code may look fine on the surface, there is a problem here. Upon calling `for_each`, the compiler will be able to deduce the type of the `iterator`, but not be able to infer the template parameter `T`.

And so the following code, which looks perfectly valid, will not compile: the compiler will complain *“unable to deduce the template parameter T”*:

```cpp
std::vector<int> ints = {1, 2, 3};
for_each(ints.begin(), ints.end(), [](int i) { // Does not compile
   std::cout << i << '\n';
});
```

The problem is what Jon Cohen referred as **“template parameters of dependent type names cannot be deduced”** in his talk. This kind of error will likely confuse more than one newcomer to C++.

Why can’t the compiler figure out the type of `T`?

### Hidden dependent type names

The previous example featured a case in which the dependent type name was pretty easy to spot. The `typename` and the `::const_iterator` appeared explicitly. But things get more complex when we introduce type aliases.

For instance, we can write an alias template `container_t` to select a different container depending on the type of element we would like to store in it (we do not use `std::conditional_t` to show we are dealing with dependent type names):

* If the type is copyable, we want to use a `std::vector`
* Otherwise we want to use a `std::list`

```cpp
template<class T>
using container_t =
    typename std::conditional<
        std::is_copy_constructible<T>{}(),
        std::vector<T>,
        std::list<T>>::type;
```

We can now define a function `f` which takes as parameter a `container_t`. As we can see, it gets really hard to spot that we are dealing with a dependent type name from the signature alone:

```cpp
template<class T>
void f(container_t<T> const& v) { /* ... */ }
```

But hidden or not, the problem still remains. The compiler will be unable to deduce the type of elements stored in the container:

```cpp
container_t<int> v;
f(v);      // KO: Does not compile, compiler cannot deduce T
f<int>(v); // OK: Compiles file
```

This is precisely the tricky case that Jon Cohen raised in his presentation, and this one will most likely annoy C++ beginners for quite a long time.

Again, why can’t the compiler figure out the type of `T`?

## Why the compiler cannot do much better

### Type level functions

We can conceive a type name depending on a template parameter as the result of a function applied on the template parameter. The syntax is different, but it obey the same principle (*).

For instance, here is the type-level function that computes the common type between two or more types in C++, followed by what would the syntax of a normal function call:

```cpp
using result_t = std::common_type_t<t1, t2>;

// In principle similar to:
auto result_t = std::common_type_t(t1, t2);
```

Similarly, the `const_iterator` dependent type name can be seen as the result of the application of a type-level function on the vector class template:

```cpp
using iterator_t = std::vector<t1>::const_iterator;

// Could be rewritten as:
using iterator_t = std::const_iterator_t<std::vector<t1>>;

// In principle similar to:
auto iterator_t = std::const_iterator_t(std::vector<t1>);
```

_(*): The syntax for calling a type-level function is what [Odin Holmes](https://twitter.com/odinthenerd) refers as the “unicorn call syntax” as a reference to the “uniform call syntax” C++ standard proposal. We just replace `using` by `auto` and angle brackets by parentheses to get the normal call syntax._

### Injective functions

A function f is *injective* is for all input $x$ and $y$, $f(x) = f(y) \implies x = y$.

In plain english, it means that feeding the function with different inputs lead to different outputs. For each outputs of the function, there is only one possible input that could have led to this output.

One interesting consequence is that if a function is not injective, there is no way we can find back the input value that led to a specific output (since there might be several inputs available).

If on the contrary a function is injective, we can build a reverse function on its image (the set of all possible outputs of the function) to find back the input element for a given output.

### Type level functions are not necessarily injective

Upon encountering a dependent type name, which as we mentioned could be viewed as a function operating on types, the compiler has unfortunately no guaranty that the function is injective.

In fact, among the examples of type level functions that we talked about earlier, we already found at least an example of non injective function:

```cpp
// Clearly not injective
using result_t = std::common_type_t<t1, t2>;

// Is it really injective?
using iterator_t = std::const_iterator_t<std::vector<T>>;
```

If we consider that `std::vector` has an additional template parameters for the allocator, the `const_iterator_t` type level function is clearly not injective either. We do not expect the iterator type to be different for different type of allocator.

### The compiler cannot deduce the template parameter of a dependent type

The compiler is unable to ensure that a type-level function is injective (*). And so it cannot reasonably invert the type level function and deduce a parameter type `T` given only a type which depends on that parameter as there might be several valid `T` for this same output!

Going back to our bad `for_each` function template, the compiler will not be able to deduce the template parameter `T` given only an iterator:

```cpp
template<typename T, typename Consumer>
void for_each(
   typename std::vector<T>::const_iterator first, //dependent type on T
   typename std::vector<T>::const_iterator last,  //dependent type on T
   Consumer consumer)
{
   for (;first != last; ++first)
      consumer(*first);
}
```

There might indeed be several `T` matching the `const_iterator` type provided as parameter: the compiler cannot know which overload to select.

_(*): This is not even taking into account the possibility of [explicit template specialization](http://en.cppreference.com/w/cpp/language/template_specialization), which means that even so we could ensure a type level function is injective today, we could not guaranty it would stay that way._

### This is not specific to C++

All languages that offer the possibility to compute types depending on some other value or types will face the same kind of issue. For instance, in Haskell, the same issue appears, although in a slightly different form.

The following code declares a `typeclass` (an interface or type trait if you wish) named `Func`, with a template parameter `a`. This type class exposes a type alias `Res` which depends on the type parameter `a`.

```haskell
class Func a where
  type Res a :: *
```

We provide two instances (specialization, or implementation of the interface, if you wish) of this type class for integer and string, which show how it looks like practically:

```haskell
instance Func Int where
  type Res Int = Int -- Res Int is an alias for Int

instance Func String where
  type Res String = String -- Res String is an alias for String
```

The “equivalent” code (in very loose terms) in C++ would be something like:

```cpp
template<typename T>
struct Func {};

template<>
struct Func<int> {
   using Res = int;
};

template<>
struct Func<std::string> {
   using Res = std::string;
};
```

If we try to define a function taking a `Res a` as input, we get a compilation error which says that we cannot deduce the type parameter `a` from `Res a`, in spirit the same error as we got in C++:

```haskell
-- Equivalent to
-- void f(typename Func<a>::Res)

f :: Res a -> IO ()
f _ = print "Does not compile (type `a` is ambiguous)"
```

The main difference with Haskell is that we get the compilation error at the very definition of the function, while the error occurs at the call site in C++.

## How to circumvent the issue?

There are different ways we can resolve the ambiguity on the compiler side, by giving it some hints on which specialization should be selected. Let us review some of them.

### Relying on additional parameters

While the compiler cannot deduce the template parameter of a dependent type name, it is still perfectly able to deduce a template parameter if it appears somewhere else in the signature of the function, as shown in this implementation of `std::accumulate`:

```cpp
template<typename T, typename BinaryOp>
T accumulate(
   typename std::vector<T>::const_iterator first,
   typename std::vector<T>::const_iterator last,
   T init,
   BinaryOp op)
{
   for (;first != last; ++first)
      init = op(init, *first);
   return init;
}
```

Thanks to the `init` parameter, the compiler is able to deduce the template parameter `T` and the following code compiles:

```cpp
std::vector<int> ints = {1, 2, 3};
accumulate(ints.begin(), ints.end(), 0, [](int res, int i) {
   return res + i;
});
```

But, granted, we do not always have this luxury of having this additional parameter that makes sense.

### Explicit template specialization

If we cannot rely on an additional parameter to help the compiler deduce a template parameter, we can always help the compiler and explicitly provide it the template parameter it cannot infer.

We can for instance correct our previous example by explicitly requiring for the specialization of `for_each` for elements of type integers, and it will compile just fine:

```cpp
std::vector<int> ints = {1, 2, 3};
for_each<int>(ints.begin(), ints.end(), [](int i) {
   std::cout << i << '\n';
});
```

In some instances, this might be very annoying or even impractical: what if the template parameter to explicitly specialize is not the first one?

### Improving the design by removing constraints

In our specific case, we can generalize our `for_each` function to take the iterator type as template parameter, just like as the STL does:

```cpp
template<typename Iterator, typename Consumer>
void for_each(Iterator first, Iterator last, Consumer consumer)
{
   for (;first != last; ++first)
      consumer(*first);
}
```

A good rule of thumb is to first try to fix our design by removing constraints before relying on the other tricks. Making the algorithm more generic (the less constraints, the more use cases an algorithm can satisfy) will usually removing the dependent type names along the way and thus get rid of the problem.

### Proxy parameters

In some languages, such as Haskell, we cannot ask for an explicit template specialization: the template parameters have to be deduced from the arguments. The solution then is to fall back on the idea of providing the compiler an additional parameter, whose only purpose is to guide the compiler.

This parameter does not carry any runtime value. It is parameterized on the type needed to guide the type deduction, and its only purpose is to carry this type. We usually call it a **proxy parameter**:

```cpp
template<typename T>
struct proxy {}
```

Or, equivalently, in Haskell:

```haskell
data Proxy a = Proxy
```

Thanks to the proxy guiding type deduction, the following code now compiles fine:

```haskell
f :: Proxy a -> Res a -> IO ()
f _ _ = print "Useless function, but compiles fine"
```

In some instances, this trick is useful in C++ as well, especially when explicit specialization is awkward (for instance if the template parameter to specialize is not the first one).

### Injective type level functions (Haskell only)

The last solution is to find a way to inform the compiler that the type-level function is injective. As of today (October 12, 2017), this capability is not yet available in C++, but is available in Haskell.

How? By explicitly requiring that the dependent type has to be a brand new type, specific to the instantiation of the type-class, and not an alias to an existing type (by using data instead of type in the declaration below).

```haskell
class Func a where
  data Res a :: *
```

Here are two examples of instances of this type-class for integer and string. In each case, we are required to create a new type (`ResInt` and `ResString` here) as output to the type-level function `Res`:

```haskell
instance Func Int where
  data Res Int = ResInt { wrappedInt :: Int }

instance FuncInj String where
  data Res String = ResString  { wrappedString :: String }
```

Now, because the Haskell compiler knows that the `Res` type-level function is injective, Haskell will be able to automatically deduced `a` from its dependent type and therefore will accept the function as valid:

```haskell
f :: Res a -> ()
f _ = ()
```

This is not the perfect solution: creating a new type is a sufficient condition for injectivity, but not a necessary one. So it is a bit over-constraining, but at least offers a solution to this annoying issue.
