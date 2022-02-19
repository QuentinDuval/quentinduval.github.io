---
layout: post
title: "Pythagorean Triples in Modern C++"
description: ""
excerpt: "Generating Pythagorean triples using linear algrebra in modern C++ as an answer to the heated discussions around std::ranges in the C++ community."
categories: [modern-cpp]
tags: [C++]
use_math: true
---

The [post of Eric Niebler on standard ranges](http://ericniebler.com/2018/12/05/standard-ranges/) and the answer post of Aras [Modern C++ lamentations](https://aras-p.info/blog/2018/12/28/Modern-C-Lamentations/) generated quite a lot of heat in the C++ community lately.

The discussion it triggered was quite interesting, with amazing answers such as the [one of Ben Deane](http://www.elbeno.com/blog), but no answer was as intriguing to me as the answer of Sean Parent, in his blog post [Modern C++ rumination](https://sean-parent.stlab.cc/2018/12/30/cpp-ruminations.html). In particular, this sentence "primitive Pythagorean triples can be generated efficiently using linear algebra".

So let's try and write a somehow efficient and modern C++ Pythagorean Triples generator, based on linear algebra, and using the modern multi-dimensional array C++ library [xtensor](https://github.com/QuantStack/xtensor) (inspired from the great numpy Python library).

## Linear algebra and Pythagorean triples

The connection between linear algebra and Pythagorean Triples is described in this [Wikipedia article](https://en.wikipedia.org/wiki/Formulas_for_generating_Pythagorean_triples), so we do not need to go into the details here.

For the sake of this article, we just need to know that the 3 following linear transformations, represented below as three matrices, generate 3 new Pythagorean triples from a known Pythagorean triple [a, b, c]:

$\\begin{pmatrix} a_1 \\\ b_1 \\\ c_1 \\end{pmatrix} = \\begin{pmatrix} -1 & 2 & 2 \\\ -2 & 1 & 2 \\\ -2 & 2 & 3 \\end{pmatrix} \\begin{pmatrix} a \\\ b \\\ c \\end{pmatrix}$

$\\begin{pmatrix} a_2 \\\ b_2 \\\ c_2 \\end{pmatrix} = \\begin{pmatrix} 1 & 2 & 2 \\\ 2 & 1 & 2 \\\ 2 & 2 & 3 \\end{pmatrix} \\begin{pmatrix} a \\\ b \\\ c \\end{pmatrix}$

$\\begin{pmatrix} a_3 \\\ b_3 \\\ c_3 \\end{pmatrix} = \\begin{pmatrix} 1 & -2 & 2 \\\ 2 & -1 & 2 \\\ 2 & -2 & 3 \\end{pmatrix} \\begin{pmatrix} a \\\ b \\\ c \\end{pmatrix}$

For the rest of this article, we will use the following notation for matrices. A matrix will be represented as a “stack” of row vectors, where each vector is represented in square brackets. For instance, our representation of the 3 matrices above are:

```
    [[-1, 2, 2],        [[1, 2, 2],        [[1, -2, 2]
A =  [-2, 1, 2],    B =  [2, 1, 2],    C =  [2, -1, 2]
     [-2, 2, 3]]         [2, 2, 3]]         [2, -2, 3]]
```

Now, if we take the row vector V = [3, 4, 5], transpose it to make it a column vector Vt, and multiply it with the matrices A, B and C, we obtain the following new column vectors (written as row vectors to make it readable):

```
[15,  8, 17], [21, 20, 29], [5, 12, 13]
```

We can easily check that the triples generated are indeed valid Pythagorean triples.

## Pythagorean Triples in Modern C++

Now that we know about linear algebra and Pythagorean triples, we can think about how to implement it. Instead of just doing it naively, we will add some some key refinements (where would be the fun otherwise?) to make our code both shorter, more elegant and more efficient as well.

### Stacking matrices

Having a row vector V representing a Pythagorean triple, do we really need to perform 3 different matrix products with the 3 separate matrices A, B and C to get the 3 next Pythagorean triples?

We don’t. Our first trick will be to stack the matrices A, B and C on top of each other to form one big matrix D:

```
    [[-1, 2, 2],
     [-2, 1, 2],
     [-2, 2, 3],
     [1, 2, 2],
D =  [2, 1, 2],
     [2, 2, 3],
     [1, -2, 2],
     [2, -1, 2],
     [2, -2, 3]]
```

Now, if multiply our column vector [3, 4, 5] with this matrix D, we get a column vector of size 9, which we can then transpose and split into 3 vectors of size 3.

```
                                                [[15, 8, 17]
[15, 8, 17, 21, 20, 29, 5, 12, 13]  ---------->  [21, 20, 29]
                                      (split)    [ 5, 12, 13]]
```

Now, with one single matrix product, we can get our 3 next Pythagorean triples. While <ins>the complexity of the operation is exactly the same</ins> as doing 3 separate matrix products, the resulting code will be shorter and faster.

### Transforming several vectors at once

In [Data Oriented Design in C++](https://www.youtube.com/watch?v=rX0ItVEVjHc), [Mike Acton](https://twitter.com/mike_acton) said: _“when there is one, there are many”_. In short, when considering a particular operation, this operation rarely occurs only once. It often occurs many times, spread around a short time frame. In such cases, we often benefit from bulking these operations.

For instance, and in the context of linear algebra, we will often be interested in applying the same transformation (like a rotation) to a bunch of points (like the points of a triangle) at once.

In our case, we will be interested in generated the next Pythagorean triples for a bunch of previously computed Pythagorean triple (and not a single one in isolation):

* We will multiply our matrix D with a single vector [3, 4, 5] and get three vectors back
* We will then multiply our matrix D with these 3 previously computed vectors and get 9 vectors back

And so on until we decide to stop generating triangles…

### Batching linear transformations

So how do we bulk these linear transformations? We just concatenate our column vectors $V_1$, $V_2$ up to $V_n$ side by side as a matrix U and multiply our stacked matrix D by U.

For instance, if we want to generate the next Pythagorean triples for the row vectors `[5, 12, 13]`, `[15, 8, 17]` and `[21, 20, 29]`, here is the matrix U we have to multiply the matrix D with:

```
    [[-1, 2, 2],
     [-2, 1, 2],
     [-2, 2, 3],
     [1, 2, 2],        [[ 5, 15, 21]
D =  [2, 1, 2],    U =  [12,  8, 20]
     [2, 2, 3],         [13, 17, 29]]
     [1, -2, 2],
     [2, -1, 2],
     [2, -2, 3]]
```

The result of this multiplication, correctly reshaped and transposed (cf the “stacking matrices” paragraph) will give us the expected result:

```
[[ 35  12  37]
 [ 65  72  97]
 [ 33  56  65]
 [ 77  36  85]
 [119 120 169]
 [ 39  80  89]
 [ 45  28  53]
 [ 55  48  73]
 [  7  24  25]]
```

We can then feed this matrix to the next iteration, multiplying them with our stacked matrix D to get the 27 next Pythagorean triples, and so on…

### Implementation with xtensor

The following function implements one iteration of the process of generating Pythagorean triples.

It takes as input the previously generated Pythagorean triples (for instance [[3, 4, 5]]) and generates the next Pythagorean triples (for instance [[5, 12, 13], [15, 8, 17], [21, 20, 29]]). This next result can in turn be fed to the function again for the Pythagorean cycle of life to continue.

```cpp
#include <xtensor/xtensor.hpp>
#include <xtensor-blas/xlinalg.hpp>

xt::xarray<int> next_pytharogian_triples(xt::xarray<int> const& previous_stage)
{
   static const xt::xarray<int> stacked_matrices = {
      { –1, 2, 2 },
      { –2, 1, 2 },
      { –2, 2, 3 },
      { 1, 2, 2 },
      { 2, 1, 2 },
      { 2, 2, 3 },
      { 1, –2, 2 },
      { 2, –1, 2 },
      { 2, –2, 3 }
   };

   auto shape = previous_stage.shape();
   xt::xarray<int> next_three = xt::transpose(xt::linalg::dot(stacked_matrices, xt::transpose(previous_stage)));
   next_three.reshape({ 3 * shape[0], shape[1] });
   return next_three;
}
```

It makes use of the following xtensor elements:

* `stacked_matrices` is our matrix D, concatenation of matrices A, B and C
* `xt::linalg::dot` is the matrix product
* `xt::transpose` is the matrix transpose
* `reshape` decomposes the result of the multiplication by D in the results of what would have been the separate multiplication by the matrices A, B and C

The code is pretty short, and quite comparable the equivalent Python numpy implementation:

```python
def next_pythagorean_triples(previous):
    matrices = np.array(
        [[–1, 2, 2],
         [–2, 1, 2],
         [–2, 2, 3],
         [1, 2, 2],
         [2, 1, 2],
         [2, 2, 3],
         [1, –2, 2],
         [2, –1, 2],
         [2, –2, 3]])

    next_triples = np.transpose(matrices @ np.transpose(previous))
    next_triples = next_triples.reshape((3 * previous.shape[0], previous.shape[1]))
    return next_triples
```

(The `@` operator above is used for the matrix multiplication in numpy)

### Sticking it in a loop... or elsewhere

We can now integrate `next_pytharogian_triples` into an iterator, or a simple loop to generate as many triples as we wish. The one thing we cannot do quite yet is integrate it inside a coroutine (*) to generate an infinite stream of triples, as demonstrated below in Python:

```python
def pythagorean_triples():
    current = np.array([[3, 4, 5]])                     # Initial seed
    yield from current                                  # Yield first triple
    while True:
        current = next_pythagorean_triples(current)     # Next iteration
        yield from current                              # Yield each triple
```

This creates a generator that lazily produces an infinite stream of Pytharogean triples like so:

```
[3 4 5]
[15  8 17]
[21 20 29]
[ 5 12 13]
[35 12 37]
[65 72 97]
[33 56 65]
[77 36 85]
...
```

We could also add some filtering (for example to only keep triples with values below 1000 if we are only interested in small triangles) and run the next iterate with the reduced set of triples:

```python
def pythagorean_triples(filter):
    current = np.array([[3, 4, 5]])                     # Initial seed
    yield from current                                  # Yield first triple
    while current.shape[0]:                             # While work to do
        current = next_pythagorean_triples(current)     # Next iteration
        current = filter(current)                       # Filter desired triples
        yield from current                              # Yield each triple
```

(*) As mentioned in [Ranges, Code Quality, and the Future of C++](https://medium.com/@jasonmeisel/ranges-code-quality-and-the-future-of-c-99adc6199608), coroutines might be more appropriate than ranges for lazy consumption of a stream of data. In Python, a language in which we have both, I find coroutines much easier to write and read than their range algorithm counterpart for use cases where both are usable. I suspect it will apply the same in C++.

## What about performance?

How does the linear algebra based implementation compares to a raw loop?

### Runtime performance

In Visual Studio 2017 Debug build, we can generate around 30,000 Pythagorean triples in less than 33 milliseconds. In Release build, this time goes down to 1,5 milliseconds (*). Said differently, it takes around 50 nanoseconds to generate a Pythagorean triple, which is amazingly fast!

What happens if we do not batch our linear transformations (one separate multiplication for each input triple)? The performance drops to 638 milliseconds in Debug build (20 times slower) and 29 milliseconds in Release build (20 times slower). Batching linear operations does matter, and quite a lot!

There are other linear algebra tricks we could have used as well, such as fast matrix exponentiation or eigenvector decomposition, which are useful if we want to get just a bunch of big Pythagorean triples really fast (and not enumerate all of them).

_(*) This is the time it would take for the naive raw loop algorithm to find around tens of Pythagorean triples._

### Build times

In terms of build times, again in Visual Studio 2017, the Debug build takes around 2,4 seconds. The Release build takes also around 2,4 seconds.

This can be seen as the curse of using header only libraries. But it also makes xtensor (as well as xtl and xtensor-blas used here as well) really good in term of performance (*).

_(*) As reference, the equivalent Python code, based on the pretty optimized numpy, takes around 2,6 milliseconds to run, almost twice the time needed for xtensor to complete the task._