---
layout: math
title: Recurrence relations and linear algebra
excerpt_separator: <!--more-->
tags: linear-algebra math recurrence-relations
---

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
    1, & \text{if $n \eq 1$} \\
    0, & \text{if $n \eq 0$}
  \end{cases}
$$

It defines a sequence starting like this:

$$
  0, 1, 1, 2, 3, 5, 8, \ldots
$$

The rest of the article is TBD, after I test the math rendering.
