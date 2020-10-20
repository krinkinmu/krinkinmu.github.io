---
layout: post
title: Recurrence relations and linear algebra
excerpt_separator: <!--more-->
tags: linear-algebra math recurrence-relations
---
[Exponentiation by squaring]: https://en.wikipedia.org/wiki/Exponentiation_by_squaring "Exponentiation by squaring"

I recently learned that GitHub markdown doesn't have native support for
rendenring math equations. That seemed quite weird, so I figured that I'd try
to look at existing alternatives and flex my tex muscle.

In this post I will try to show an end-to-end example of how to solve linear
ordinary recurrence relations and give an intro to all the linear algebra
required to do that. Some basic understanding of vector spaces and matrices is
still required though.

<!--more-->

# Fibonacci sequence

And of course I will start with a well known example of Fibonacci sequence
defined as follows:

$$
  F_n =
  \begin{cases}
    F_{n - 1} + F_{n - 2}, & \text{if $n \ge 2$} \\
    1, & \text{if $n = 1$} \\
    0, & \text{if $n = 0$}
  \end{cases}
$$

Here are the first few elements of the sequence:

$$
  0, 1, 1, 2, 3, 5, 8, \ldots
$$

The Fibonacci sequence is well known and well studies, so it's hardly worth
spending much time on introducing it. Moreover in this post I will not find
anything new about this sequence, but I will show how we could derive a
non-recurrent formula for the *nth* member of the sequences or, in other words,
how to solve this kind of recurrence.

# Matrix form

The first step would be to rewrite the same recurence relation in a matrix
form. The goal is to find a matrix *A* such that the following equation holds:

$$
  \begin{bmatrix}
    F_{n} \\
    F_{n-1}
  \end{bmatrix} =
  A
  \begin{bmatrix}
    F_{n - 1} \\
    F_{n - 2}
  \end{bmatrix}
$$

It's not hard to see that the following matrix satisfies the conditions:

$$
  \begin{bmatrix}
    F_{n} \\
    F_{n-1}
  \end{bmatrix} =
  \begin{bmatrix}
    1 & 1 \\
    1 & 0 \\
  \end{bmatrix}
  \begin{bmatrix}
    F_{n - 1} \\
    F_{n - 2}
  \end{bmatrix}
$$

We can easily do that because the Fibonacci recurrence relation is linear. In
general for linear recurrence relations the size of the matrix and vectors
involved in the matrix form will be identied by the order of the relation.

In case of the Fibonacci sequence, with exception of the first two elements,
all other elements of the sequence depend on two previous elements.

What about the first two elements of the sequence though? They serve as initial
conditions for the recurrence relation, so let's try to add them into our
matrix form. In order to do that let's expand the vector in the right side of
the equation:

$$
  \begin{bmatrix}
    F_{n} \\
    F_{n-1}
  \end{bmatrix} =
  A
  \begin{bmatrix}
    F_{n - 1} \\
    F_{n - 2}
  \end{bmatrix} =
  A
  A
  \begin{bmatrix}
    F_{n - 2} \\
    F_{n - 3}
  \end{bmatrix} =
  A^2
  \begin{bmatrix}
    F_{n - 2} \\
    F_{n - 3}
  \end{bmatrix}
$$

We can continue this expnasion until we get \\(F_1\\) and \\(F_0\\), our first
two members, in the right side of the equation:

$$
  \begin{bmatrix}
    F_{n} \\
    F_{n-1}
  \end{bmatrix} =
  A^n
  \begin{bmatrix}
    F_1 \\
    F_0
  \end{bmatrix} =
  A^n
  \begin{bmatrix}
    1 \\
    0
  \end{bmatrix}
$$

So now we have Fibonacci sequence defined in a matrix form, does it give us
anything? Conincedentally it gives us a rather cool way to calculate a
Fibonacci number by given index using [Exponentiation by squaring].

The algorithm takes \\(O(\log n)\\) matrix multiplications. Each matrix
multiplication takes a fixed number of arithmetic operations on numbers,
specifically 4 number multiplications and 2 number additions.

That, compared to the naive algorithm that takes \\(O(n)\\) additions, might
seem like an improvement, especially if you assume that both multiplications
and additions of integer numbers are equal constant time operations.

> *NOTE:* Assumption that integer multiplications and additions are equal is
  in fact quite limiting for fast growing sequences like Fibonacci. That
  basically limits you to just a few dozens first members of the Fibonacci
  sequence that fit into machine word. If that's a reasonable limitation for
  your use case, you can as well precalculate all the Fibonacci numbers you
  need in advance and access the in constant time when you need them. That
  being said with the right integer multiplication algorithm you can get an
  asymptotic improvement using the matrix multiplication method over the naive
  method.

# Basises and coordinates

We now have Fibonacci recurrence relation written in the matrix form. That
means that we can treat the matrix *A* as a linear operator in vector space
\\(\mathbb{R}^2\\). How does it help us?

In order to see how does it help us we should talk a little bit about basises
of vector spaces. More specifically, we should note that the same vector space
may have multiple different basises.

For example, let's say we have a vector *t* on a plane (\\(\mathbb{R}^2\\))
and we use *x* and *y* coordinates to describe the vector on that plane.

TODO insert a picture here

Basically *x* and *y* coordinates tell us how many basis vectors we need to
combine to get the original vector *t*. 

The pair of basis vectors chosen above is convinient to use, but that's by no
means the only pair of vectors that could be used for this purpose. For
example, these two vector will work as well:

TODO insert a picture here

We can use those vectors as a basis as well. However the coordinates of the
vector *t* will be different if we use them as a vector.

In this particular case all we need to change the basis is to change the
signes of the *x* and *y* coordinates to the opposite, but in general the
transformation might be more complicated.

So we can have different basises for the same vector space, so what? Well, the
matrix of the same linear operator might look differently in different basises.
What if we can find such a basis that makes the matrix of the linear operator
more convenient to work with?

# Basis changes

Before we go into details regarding what matrix would be convenient to work
with let's close one purely technical question. Say we have a matrix of a
linear operator in one basis, how can we convert it to represent the same
linear operator but in different basis?

For a set of vectors to be a basis it's necessary that every element of the
vector space can be expressed as a linear combination of the basis vectors.
That's, of course, also true for vectors of the different basis.

In practice that means that we can find a matrix *S*, that given coordinates of
a vector in one basis transforms them into coordinates of the same vector in
another basis.

Let's say that the original basis consits of vector *x* and *y* and the target
basis consists of vector *u* and *v*. We know that vectors *x* and *y* can
be expressed as a linear combination of vectors *u* and *v*.

Without loss of generality lets say that  \\(x = s_{0,0} u + s_{1,0} v\\) and
\\(y = s_{0,1} u + s_{1,1} v\\). In other words, in the basis cosisting of
vectors *u* and *v* the vector *x* has coordinates \\(s_{0,0}\\) and
\\(s_{1,0}\\), and the vector *y* has coordinates \\(s_{0,1}\\) and
\\(s_{1,1}\\).

It's easy to see that the matrix that converts coordinates in basis *x* and
*y* to the coordinates in basis *u* and *v* will look like this:

$$
  S =
  \begin{bmatrix}
    s_{0,0} & s_{0,1} \\
    s_{1,0} & s_{1,1}
  \end{bmatrix}
$$

So now we can easily convert coordinates of a vector in one basis to
coordinates of the same vector in another basis. How can we extend it to linear
operators?

Let's say we have a matrix *A* of a linear operator that works with coordinates
in basis *x* and *y*. We also have a matrix *S* that given coordinates in basis
*x* and *y* converts them to coordinates of the same vector in basis *u* and
*v*.

Let's first note that if we can convert *x* and *y* coordinates to *u* and *v*
coordinates using matrix *S*, then we can use the inverse matrix to do the
opposite transformation from *u* and *v* coordinates to *x* and *y*
coordinates.

Now, if we have a vector *t* in *u* and *v* coordinates and we don't know how
to apply the linear transformation to it, we can first convert *u* and *v*
coordinates to *x* and *y* coordinates and then apply the linear operator by
multipling by the matrix *A*.

However that will give us a vector in *x* and *y* coordinates, but we need a
vector in *u* and *v* coordinates. Well, no problem, we can just multiply it
by the inverse of *S* to covert the vector back into the *u* and *v*
coordinates.

So in order to calculate the result of linear operator in *u* and *v*
coordinates we can:

1. convert the *u* and *v* coordinates to *x* and *y* coordinates using *S*
2. multiple by matrix *A*
3. convert the *x* and *y* coordinates to *u* and *v* coordinates using the
   inverse of *S*

Alternatively putting all of that in a matrix form: \\(S A S^{-1} t\\). So
the matrix we need is \\(S A S^{-1}\\).

# Diagonalization

So now we know how to change the matrix of a linear operator when we change
from one basis to another. Now we can try to find a basis in which the matrix
of our operator is simple.

What matrix is easy to work with? Let's return back to the original problem.
We had a matrix that we need to take to the power *n*, maybe we should look for
a matrix that is easy to multiply. Diagonal matrices seem to be rather easy to
multiply.

To formalize the problem we want to find a matrix *B* such that
\\(A = S B S^{-1}\\) and *B* is diagonal. Here *S* is matrix that will convert
from the whatever basis we started with to the basis where the same operator
can be expressed by a diagonal matrix.

If we manage to do that, then it should be easy to get *A* to the nth power:

$$
A^n = (S B S^{-1})^n = S B S^{-1} S B S^{-1} \ldots S B S^{-1} = S B^n S^{-1}
$$

> *NOTE:* it should be noted that a basis that makes *B* diagonal may not
  even exist. So we will try to find it, but no guarantees in general.

To understand how to find a basis that makes *B* diagonal let's consider what
the multiplication by diagonal *B* actually means. This multiplication will
just scale each coordinate of the vector by a constant.

Remember what coordinates of the vector in a basis mean? They mean how many
basis vectors we need to take to construct the original vector from them. So
if a linear operator matrix is diagonal in a basis, it means that the linear
operator when applied to the vectors of this basis just multiplies them by a
constant. In other words it makes them longer or shorter (it can also rotate
them in the opposite direction, that what multiplication by a negative value
means).

So what we are looking for is a bunch of linearly independent vectors that
when trasnformed by *A* just scale by a constant. We only care about
non-trivial vectors that can constitute a basis, so the vector of length 0
is not good.

Putting it in a form of equation:

$$
  A x = \lambda x, \text{where} \lambda \ne 0
$$

Vectors that satisfy this equation are called eigenvectors and the values of
\\(\lambda\\) are called eigenvalues.

So if we find enough linearly independent eigenvectors, we can construct a
basis in which matrix *B* representing the linear operator we care about is
diagonal.

# Finding eigenvectors
