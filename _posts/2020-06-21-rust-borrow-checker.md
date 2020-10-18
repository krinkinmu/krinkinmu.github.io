---
layout: post
title: A few notes on Rust borrow checker
excerpt_separator: <!--more-->
tags: rust borrow-checker
---
[Rust]: https://www.rust-lang.org/ "Rust"
[Kahan]: https://en.wikipedia.org/wiki/Kahan_summation_algorithm "Kahan summation algorithm"
[Quickselect]: https://en.wikipedia.org/wiki/Quickselect "Quickselect"
[Quicksort]: https://en.wikipedia.org/wiki/Quicksort "Quicksort"
[RustBook]: https://doc.rust-lang.org/book/ "The Rust Programming Language"

A while ago I tried to embrace [Rust], but my experience was mostly negative due
to various reasons. However the language and toolchain has machured since then
and I'm giving it another shot.

One of the features of [Rust] that makes it different from the other languages
is it's borrow checker. What follows are a few notes about the Rust borrow
checker.

<!--more-->

# The problem

Let's take a look at a made up C++ example that calculates the mean and the
median of elements in a vector:

```cpp
double mean(const std::vector<double>& values);

double median(std::vector<double>& values);

std::pair<double, double> mean_and_median(std::vector<double>& values) {
    double x, y;

    std::thread t1([&x, &values]() { x = mean(values); });
    std::thread t2([&y, &values]() { y = meadian(values); });
    t2.join();
    t1.join();

    return std::make_pair(x, y);
}
```

Naturally, as the examples normally go, we do the calculations in a somewhat
convoluted way. Specifically, in the example above we calculate median and mean
in two separate threads, potentially in parallel.

It's important to notice the types in the declarations of `mean` and `median`
functions in the example. You may see that `mean` takes a const reference to the
vector, while median take a non-const reference to the vector.

To add a bit of credibility to the example let's think how can we calculate mean
and median of the elements and why types are the way they are. With mean things
are more or less easy, all we need to do is to sum up the values of the element
and divide the result by the size of the vector.

> **NOTE:** It doesn't mean that there is no complexity in calculating means, as
a matter of fact there might be quite a bit of complexity there depending on
your requirements. For example, take a look at [Kahan].

With median the situation is a bit more complicated if we want a reasonably
efficient algorithm. One way to find a median is to use [Quickselect].

Returning back to `C++` you may think of calculating mean as a variation of
`std::accumulate`:

```cpp
#include <numeric>

double mean(const std::vector<double>& values) {
  return std::accumulate(values.begin(), values.end(), 0) / values.size();
}
```

and, similarly, calculation of the median is just a variation of
`std::nth_element`:
```cpp
#include <algorithm>

double median(std::vector<double>& values) {
  auto middle = values.begin() + values.size() / 2;
  std::nth_element(values.begin(), middle, values.end());
  return *middle;
}
```

Important part here is that [Quickselect] and `std::nth_element` both modify the
original sequence and that's why `median` in our example takes the vector by
non-const reference.

Naturally that's a problem, because there is no guarantees what elements the
`mean` function will find since concurrently another function may shuffle
elements of the vector around.

`C++` compiler will most likely allow such a code. Moreover in simple cases you
may even observe that the program consistently returns correct results, even
though it's not guaranteed. And only if you compile your code with something
like `-fsanitize=thread` and run your program you'll get an error message.

There are various ways to deal with the problem, but let's took at what [Rust]
offers here.

# Rust borrow checker

By default [Rust] doesn't allow to create multiple non-const references (or
mutable references as [RustBook] calls them) with overlapping scopes for the
same object. Moreover you're not allowed to have mutable reference together
with unmutable references if their scopes overlap:

```rust
let mut v = vec![1, 2, 3, 4, 5];

let mut ref1 = &mut v;
let ref2 = &v;

println!("&mut ref: {:?}", ref1);
println!("&ref: {:?}", ref2);
```

The code above does not compile and if you try you should see an error similar
to this one:
```
error[E0502]: cannot borrow `v` as immutable because it is also borrowed as mutable
 --> src/main.rs:5:16
  |
4 |     let mut ref1 = &mut v;
  |                    ------ mutable borrow occurs here
5 |     let ref2 = &v;
  |                ^^ immutable borrow occurs here
6 | 
7 |     println!("&mut ref: {:?}", ref1);
  |                                ---- mutable borrow later used here

```

Would it prevent the problem we had in the `C++` code above. Well, yes it would.
Basically, Rust borrow checker makes sure that you will not create multiple
mutable conflicting references (either multiple mutable references or mutable
reference with immutable reference) for the same object with overlapping scopes.

> **NOTE:** the check [Rust] does is local. To understand what I mean by that
let's imagine a situation where your code takes a mutable reference to some
object. Rust borrow checker will make sure that your code doesn't create any new
mutable or immutable references to the same object. However, to guarantee safety
we need to make sure that no mutable references for the same object exists, not
just that your code doesn't create one. It's not really a downside and it's not
trivial to improve on that, so you just should be aware.

# Reference scope

One interesting thing to mention here is how the scope of the reference is
defined in context of borrow checking.

The scope of a local variable in [Rust] is limited by `{}`:

```rust
{
  let x = 42;
}
println!("{}", x);
```

The code above doesn't compile because the variable `x` is not defined in the
scope where `println` tries to refer to it. That's hardly a new concept and
[Rust] is no different in this regard from countless number of other languages.

However the scope of a reference in context of borrow checker and the scope of
a variable are different. Let's look at the follow example:

```rust
let mut v = vec![1, 2, 3, 4, 5];

let ref1 = &mut v;
println!("Mutable: {:?}", ref1);

let ref2 = &v;
println!("Immutable: {:?}", ref2);
```

The code above compiles and works. You may see that `ref1` and `ref2` are both
defined in the same scope, so you may think that the borrow checker might
complain about that. However the borrow checker is happy with the code.

The scope of a reference in context of the borrow checker is limited by the last
place where the reference is used. With that in mind it's easy to see that the
scopes of `ref1` and `ref2` do not overlap.

If we change the order of instruction a little bit we can cause borrow checker
error easily:

```rust
let mut v = vec![1, 2, 3, 4, 5];

let ref1 = &mut v;
let ref2 = &v;

println!("Mutable: {:?}", ref1);
println!("Immutable: {:?}", ref2);
```

# Whole vs part

Let's say we have a structure. Is it possible to have mutable references to the
different fields of the structure with overlapping scopes?

For simple cases it's possible:
```rust
#[derive(Debug)]
struct S {
  x: i32,
  y: i32,
}

fn main() {
  let mut s = S {
    x: 40,
    y: 2,
  };

  let ref1 = &s.x;
  let ref2 = &s.y;

  println!("x = {}, y = {}", ref1, ref2);
  println!("{:?}", s);
}
```

The example above compiles just fine, even though we have immutable `ref1` and
`ref2` together with mutable `s`. However, if you try to use `s` to modify the
fields referenced by `ref1` or `ref2` like in the code below, you'll get a
borrow checker error:

```rust
#[derive(Debug)]
struct S {
  x: i32,
  y: i32,
}

fn main() {
  let mut s = S {
    x: 40,
    y: 2,
  };

  let ref1 = &s.x;
  let ref2 = &s.y;

  s.x = 0;
  s.y = 0;

  println!("x = {}, y = {}", ref1, ref2);
  println!("{:?}", s);
}
```

So borrow checker does appear to be smart enough to deal with structures.
However there is another way to combine data pieces together and it's arrays (or
vectors, lists, etc).

Collections are different from structures because when working with collections
it's not always possible to tell if there is an overlap or not. Let's take a
look at a simple example. Imagine that you have a vector and you want to create
two slices pointing to the different parts of the vector:

```rust
let mut v = vec![1, 2, 3, 4, 5];
let left = &mut v[..3];
let right = &mut v[3..];
println!("left = {:?}, right = {:?}", left, right);
```

The code above doesn't compile because the borrow checker finds it problematic.
It's farily obvious to us that elements pointed by `left` and `right` do not
overlap, however for the compiler it's not as obvious.

To understand why it might not be obvious let's consider a slightly more
complicated situation where instead of using hardcoded constants we actually
calculate somehow the boundaries of the slices:

```rust
let mut v = vec![1, 2, 3, 4, 5];

let (from, to) = find_left();
let left = &mut v[from..to];

let (from, to) = find_right();
let right = &mut v[from..to];

println!("left = {:?}, right = {:?}", left, right);
```

In general values of `from` and `to` may depend on a multidue of things not all
of which are known at the compilation time. So it's impossible for the borrow
checker to know whether they ever overlap or not.

> **NOTE:** even if the borrow checker knew the input in advance there are still
fundamental limitation of computing that do not allow to analyse arbitrary
properties of an arbirary program.

Again, we should consider the credibility of the example. If the operation is
not useful to begin with then it doesn't matter if the borrow checker doesn't
allow it.

Let's return to the [Quickselect] algorithm that I mentioned at the very
begining. The algorithm allows to find an Nth ordered static or, in other word,
an element that would be at the Nth position in the sequence if we sorted it.

[Quickselect] basically partialy sorts the input sequence following an approach
very similar to [Quicksort]. However since we are only interested in finding
just one element that should be in the Nth position in the sorted sequence
[Quickselect] works faster than [Quicksort] on average.

Anyways, one of the key operations in both [Quickselect] and [Quicksort] is
partitioning. The goal is for the given value `x` shuffle all the elements in
the sequnce in such a way that:

* all elements that are less then `x` are placed in front of the sequence
* all the elements that a greater than `x` are at the back of the sequence
* all the elements that are equal to `x` will be in the middle.

After partitioning both algorithms proceed recursively to work on left, middle
and/or back part of the sequence. Due to recursive nature of the algorithms it
makes sense to define the signature of the partitioning function like this:

```rust
fn partition(values: &mut [i32], x: i32) -> (&mut [i32], &mut [i32], &mut [i32])
```

You can spot a problem here. The results of the function are mutable references
to slices and naturally they will have to point to the elements of the `values`
slice. It's a very much the same thing that, as we saw above, borrow checker
does not allow us to do.

We know that the slices do not overlap by the nature of what the `partition`
function does, assuming that it's correctly implemented. However the borrow
checker doesn't know that and will necessarily complain.

So now when we know that the operation is in general useful. The next question
is how to circumvent the limtiation of the borrow checker?

One way to circumvent the limitation is to use `unsafe` to temporarily disable
the borrow checker. With `unsafe` we can create multiple mutable references.
For example, here is the implementation of `split_at_mut` method for slices in
the [Rust] standard library that basically split slice in two at the given
position:

```rust
pub fn split_at_mut(&mut self, mid: usize) -> (&mut [T], &mut [T]) {
  let len = self.len();
  let ptr = self.as_mut_ptr();

  unsafe {
    assert!(mid <= len);

    (from_raw_parts_mut(ptr, mid), from_raw_parts_mut(ptr.add(mid), len - mid))
  }
}
```

We can use a similar trick in our code. However, if we know that the returned
slices do not overlap it always should be possible to just use `split_at_mut`
from the standard library instead of using `unsafe` directly.

# As a conclusion

Rust borrow checker is a bit of curiousity in the programming languages world
and it takes some getting used to.

It might appear a bit restrictive, but I often find that language/company style
guides, best practices guidelines and other conventions in other languages are a
bit restrictive at first as well. Eventually you just learn to embrace them and
find ways to code so that those restrictions don't get in your way too much.
