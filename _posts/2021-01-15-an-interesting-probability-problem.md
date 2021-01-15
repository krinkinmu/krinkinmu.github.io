---
layout: post
title: An interesting conditional probability problem
excerpt_separator: <!--more-->
tags: math probability conditional-probability bayes-theorem the-law-of-total-probability
---

Occasionally I read university books on math and comuter science to refresh my
memory. At work I mostly use some linear and mixed integer programming solvers
and ready function fitting implementation, but I don't get to actually solve
math problems that often. So I find it entertaining to go through the study
book problems from time to time.

This post is about one simple problem that I for some reason find quite cool.

<!--more-->

# The Problem

The problem I'm about to present is not hard and doesn't require clever
observations or any kind of enlightenment moment. The solution to this problem
just follows from basic definitions in a straighforward way.

The reason I find this problem cool is because despite being simple, it
somehow seems very practical and may serve as a good introduction as to why
condition probability is useful.

Let's get to the problem. Imagine we have two coins. One coin is fair and
tossing it will give us a head with probability \\(0.5\\). However another
coin is not fair and lands heads up with probability \\(0.75\\).

Imagine that we randomly, with a probability of \\(0.5\\), picked one of the
coins and tossed it three times and all three times the coin landed heads up.
Given that information, what is the probability that we picked the fair coin?

# Conditional Probability

First of all, we need to start from introducing a tools for calculating
probabilities given additional information. The tool we need here is called
conditional probability.

Let's denote the probability of an event \\(A\\) given that event \\(B\\)
happend as \\(P\left(A \mid B\right)\\) - this what we call conditional
probability.

Before we go anywhere further it's important to note, that when we talk about
condition probability, the order of events is irrelevant. The event \\(A\\)
does not have to happen after the event \\(B\\) or vice versa. Time is not of
any relevance here, only the available information is important.

Additional information may or may not exclude certain outcomes of the
experiment out of the picture. For example, of the problem above, any outcome
where we got a tail after tossing a coin is out of the picture. 

We can think about the conditional probability in the following way.
Additional information changes the sample space of the experiment and as a
result affects the probabilities of the event. In other words conditional
probability is just a probability, but with a different sample space.

The formula for calculating the conditional probability value follows quite
naturally:

$$
  P\left(A \mid B\right) = {P\left(A \cap B\right) \over P\left(B\right)}
$$

The formula restricts the sample space to the outcomes that are included in
\\(B\\) and calculates how many of them are also in \\(A\\). Putting it
slighly fancier it calculates the probability of \\(A \cap B\\) and normilizes
it on the probability of \\(B\\).

# Bayes Theorem

From the formula for the conditional probability we can get another useful
equation by considering both \\(P\left(A \mid B\right)\\) and
\\(P\left(B \mid A\right)\\) together.

On the one hand:

$$
  P\left(A \mid B\right) = {P\left(A \cap B\right) \over P\left(B\right)}
    \implies P\left(A \cap B\right) = P\left(A \mid B\right) P\left(B\right)
$$

On the other hand, if we consider \\(P\left(B \mid A\right)\\), we can get
the following:

$$
  P\left(B \mid A\right) = {P\left(A \cap B\right) \over P\left(A\right)}
    \implies P\left(A \cap B\right) = P\left(B \mid A\right) P\left(A\right)
$$

Since the left sides of both equations are the same, then the right sides have
to be the same as well, so we can combine them together:

$$
  P\left(A \mid B\right) P\left(B\right) =
    P\left(B \mid A\right) P\left(A\right)
      \implies P\left(A \mid B\right) =
        {P\left(B \mid A\right) P\left(A\right) \over P\left(B\right)}
$$

This result is known as Bayes Theorem and it's quite useful. The result is
quite useful for our problem because the efforts required to calculate
\\(P\left(A \mid B\right)\\) and \\(P\left(B \mid A\right)\\) may not be the
same.

For example, in the case of our problem it's quite easy to calculate the
probability of getting heads three times for the fair and unfair coins is
trivial, while the opposite is not that trivial.

# The Law of Total Probability

Let's again return back to the formula for the conditional probability:

$$
  P\left(A \mid B\right) = {P\left(A \cap B\right) \over P\left(B\right)}
    \implies P\left(A \cap B\right) = P\left(A \mid B\right) P\left(B\right)
$$

With that in mind, let's say that we partition the sample space \\(S\\) to a
set of events \\(\\{B_i : i = 1, 2, 3, \dots\\}\\) such that
\\(B_i \cap B_j = \emptyset\\) when \\(i \neq j\\) and \\(\cup_{i} B_i = S\\).
In words, that basically means that we want to split all outcomes to separate
non-overlapping groups, so that every outcome belongs to one and exactly one
group.

Using the equation above and the partition we can express the probability of
an event \\(A\\) like this:

$$
  P\left(A\right) = \sum_i P\left(A \cap B_i\right) =
    \sum_i P\left(A \mid B_i\right) P\left(B_i\right)
$$

Since all the \\(B_i\\) united together give us \\(S\\) then \\(A \cap B_i\\)
united together give us \\(A\\). And because the \\(B_i\\) do not overlap then
\\(A \cap B_i\\) do not overlap either. So we can just sum up the
probabilities of \\(A \cap B_i\\) to get the (total) probability of \\(A\\) (
thus the name).

However, the important question here is why would we want to do it this way?
The benefit is very similar to the one I alluded to above for the Bayes
Theorem - given the right partition of the sample space, it might be easier to
calculate individual \\(P\left(A \mid B_i\right) P\left(B_i\right)\\) than
directly calculate \\(P\left(A\right)\\).

Specifically, applied to the problem we want to solve, it's easy to calculate
the probability of tossing three heads in a row for the fair and unfair coins,
and, as a result, it's easy to calculate the total probability of three heads
in a row.

# Putting everything together

I will start from introducing the notiation for events. First I will call the
event of tossing three heads in a row \\(H\\). The event of picking the fair
coin out of two I will denote as \\(F\\). The event of picking the unfair coin
is a complement to the event \\(F\\), so I will use \\(F^c\\) to denote it.

As you could figure out, we have only two options: we can pick either the fair
coin of the unfair coin. So naturally, \\(F\\) and \\(F^c\\) form a partition
of the sample space.

Using this partition let's first calculate the probability of tossing three
heads in a row:

$$
  P\left(H\right) =
    P\left(H \mid F\right) P\left(F\right) +
    P\left(H \mid F^c\right) P\left(F^c\right) =
      P\left(H \mid F\right) P\left(F\right) +
      P\left(H \mid F^c\right) \left(1 - P\left(F\right)\right)
$$

From the statement of the problem we know that \\(P\left(F\right) = 0.5\\)
and, therefore, \\(P\left(F^c\right) = 1 - P\left(F\right) = 0.5\\). So we
only need to calculate \\(P\left(H \mid F\right)\\) and
\\(P\left(H \mid F^c\right)\\). Calculating those is quite straighforward.

Given a fair coin, tossing three heads in a row is just
\\(P\left(H \mid F\right) = 0.5^3\\). Given a coin that lands heads up with
the probability of \\(0.75\\), the probability of three heads in a row is
\\(P\left(H \mid F^c\right) = 0.75^3\\).

In the end we get the following:

$$
  P\left(H\right) = 0.5^3 \times 0.5 + 0.75^3 \times 0.5 = 0.2734375
$$

So the probability isn't actually that high, even with the unfair coin in the
picture. It's still more than two times higher than the probability of tossing
three heads in a row with just a fair coin.

The probability of \\(H\\) is not what we actually want, what we actually want
to find out is \\(P\left(F \mid H\right)\\). And here is when the Bayes
Theorem will be handy:

$$
  P\left(F \mid H\right) =
    {P\left(H \mid F\right) P\left(F\right) \over P\left(H\right)}
$$

We already know \\(P\left(H\right)\\). Moreover, we calculated
\\(P\left(H \mid F\right)\\) as part of the \\(P\left(H\right)\\) above as
well. And, of course, the probability \\(P\left(F\right)\\) is known from the
problem statement to be \\(0.5\\).

Putting everything together:

$$
  P\left(F \mid H\right) = {0.5^3 \times 0.5 \over 0.2734375} = 0.228571429
$$

# Verifying the result

We can get some confidence in the result by just simulating the experiment.
Writing a quick python script to simulate the experiment multiple times as
described in the problem probably will take less time than solving the problem
precisely:

```python
import random


def simulate(coins, n):

    def toss(head_probability):
        '''Returns True if the coin landed heads up.'''
        return random.random() < head_probability

    count = [0] * len(coins)

    for _ in range(n):
        coin = random.choice(list(range(len(coins))))
        # We only care about the outcomes when all the tosses result in head
        if all(toss(coins[coin]) for _ in range(3)):
            count[coin] += 1

    return count


if __name__ == '__main__':
    random.seed()
    count = simulate(coins=[0.5, 0.75], n=10000000)
    print('Observed frequency of fair coin:', count[0] / sum(count))
```

With the 10000000 repetitions of the experiment I get the first three digits
after the point right more or less consistently. The experiment however takes
a few seconds to complete.

As far as simulations go this one is not particularly efficient or precise.
You see, out of all the experiment results we discard those that didn't result
in three heads in a row. So, all in all, we discard quite a few outcomes.

We calculated above the probability of tossing three heads in a row to be
\\(0.2734375\\). So out of 10000000 experiments only around 2.7 millions will
be interesting.

We can add to that the number of the useful outcomes in a simulation is also
a random value and changes from simulation to simulation. So all in all, we
calculate the frequency using different sample sizes in each simulation.

# Instead of conclusion

I wrote this post because I really liked the problem. The problem itself is
easy and doesn't require any clever thinking, but allows to introduce
conditional probability on a practial-looking problem.

I found the problem in the second edition of "Introduction to Probability" by
Joseph K. Blitzstein and Jessica Hwang. The book is freely available at
[probabilitybook.net](http://probabilitybook.net). It appear to contain quite
a few wordy and hand-wavy explanations, but it tries to build some intutive
understanding of the subject this way. It also has plenty of example problems
with solutions.
