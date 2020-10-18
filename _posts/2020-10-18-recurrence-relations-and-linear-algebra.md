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
to members, in the right side of the equation:

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
  \end{bmatrix} =
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

