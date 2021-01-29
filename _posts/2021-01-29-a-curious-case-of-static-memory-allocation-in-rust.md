---
layout: post
title: A curious case of static memory allocation in Rust
excerpt_separator: <!--more-->
tags: rust
---

[previous post]: {% post_url 2021-01-17-devicetree %} "the previous post"

In the [previous post] I covered the binary representation of the Flattened
DeviceTree or DeviceTree Blob and was already starting to work on memory
management for my hobby project, but I got stuck for quite some time trying
to come up with a reasonable way to work with statically allocated memory
in Rust.

I don't think that I found an obviously convincing approach here, but what
can you do...

As always, I have some sources related to the post on
[GitHub](https://github.com/krinkinmu/aarch64), though in this particular
post I will be construction a purely hypothetical example, so you will not
be able to find the snippets from the post in the repository.

<!--more-->

# Static Memory Allocation Strawman

I'd like to start by separating globals and statics. Not all statically
allocated objects have to be global. You can look at it this way: we have
two questions to anwer here:

* where to store data?
* how to access data?

When we talk about statics we are talking about how the data for the object
is allocated. The object may or may not be globally accessible. On the other
hand when we talk about globals it's mostly about who can access the object.
We don't have to allocate memory for global objects statically.

## Where to store data?

The programming model of typical imperative programming language that offloads
memory management responsibilities to the program itself often provides three
high-level options here:

1. we can dynamically reserve memory for the data when we need it from a
   global memory pool - dynamic memory allocation;
2. we can reserve part of the stack of a thread to store some data - this is
   a less generic, but still quite useful form of dynamic memory allocation;
3. we can reserve memory in the program binary - similarly to how memory is
   reserved for the code of the program itself.

All three options provide *an* answer to the question of where to store data,
but all three answers have somewhat different properties. And as you may have
guessed statics belong to the option 3.

Option 1 is the most generic, but also is the most demanding. That's why there
are problems where dynamic memory allocation is just not available. So we
cannot reduce everything to dynamically allocated memory.

We can however store all the things that we cannot use dynamic memory
allocation for on stack. However, while theoretically possible, storing
everything on stack create a few practical concerns:

* we need to make sure that we have enough space on stack - with globals
  compilers just magically handle it for you (well, not magically, but you
  don't really need to care about it);

* we need to make sure that the thread and therefore the stack of the tread
  surives long enough - otherwise we may end up with a dangling pointer on
  our hands.

Those two practical concerns have to be addressed and it's by no means always
easy to do correctly.

## How to access data?

Accessing globals is easy. You don't need to structure your code to make sure
that all the right data is available where you need it - global data is
available everywhere.

So naturally, with such a power comes responsibility and this ease of access
can be employed in a good way and in a bad way. And, again unsurprisingly, if
something can be used in a bad way you can be sure that it will absolutely be
used in a bad way at some point.

So I have an impression that the "globalness" property that is why people
bash global variables. For the same reason people often bash singleton
pattern.

Unfortunately though, globals and statics often come together, but what
follows will be specifically about static memory allocation and not about
creating global objects.

# The Problem

Back to buisness. My problem is that I'm setting up a memory management system
in my hobby project. As such dynamic memory allocation is not available and
that sucks. I don't want to allocate all the things I need on stack, since
it comes with caveats. So static allocation it's!

Without loss of generality we will be allocating an array of objects. That's
what I needed to do for my project, but it's also turned out to be a more
interesting problem.

The code is in Rust. You know one of those "safe" languages that shouldn't
allow you to shut yourself in the foot.

The safety of Rust comes with a few caveats:

* Mutable static (*regardless of whether they are global or not*) objects are
  unsafe;
* Everything have to be initialized.

One of the fundamental properties of safe Rust typesystem and safe Rust
libraries is that there cannot be multiple mutable references to the same
object at the same time.

It's easy to see how this property can be easily violated with global mutable
objects. And Rust doesn't provide means to allocate memory statically in a way
that makes safe subset of Rust happy.

> *NOTE:* that Rust does actually provide means to circumvent the restriction
  on only one mutable reference at a time using so called interrior mutability
  pattern, however interrior mutability implementation ultimately boil down to
  using `UnsafeCell` which implies dropping into unsafe code at some point.
  The unsafe code might be hidden inside the Rust library, but you can be sure
  that it's there. The actual safety guarantees in such cases come from
  carefully designed interface and library implementation correctness, that
  should not allow leaking mutable references outside of the library
  abstractions.

The second caveat is really easy to understand - working with uninitialized
state is basically asking for trouble. What makes it a bit difficult in Rust
specifically is somewhat lacking support for default initialization. You'll
see what I'm talking about in a moment.

# Definitions

Let's start from some basic definitions. Let's introduce a type of objects
I'd want to allocate:

```rust
pub struct AnObject {
  /* there might be some fields here */
}
```

The `AnObject` type is not copyable. In my case it was because it contained
a mutable reference to a slice and copying those is not allowed in the safe
subset of Rust - so there are practical reasons for that.

The importance of the `Copy` trait implementation (or lack of thereof) will
become important later.

What I want to implement is a function like this:

```rust
pub fn allocate_slice(size: usize) -> Option<&'static mut [AnObject]> {
   /* some magic happening here */
}
```

I don't use statics for the ease of access, I only care about allocating
enough memory. So the interface the function provides is what you can expect
from an allocator:

* it returns a reference or a pointer insted of `AnObject` directly;
* it might fail, so the returned value is wrapped in `Option`.

Since we want to allocate memory statically the lifetime of the returned
value is `'static` as well.

Of course, we have to guarantee that we will not return references to the
same memory twice from this function. Otherwise we'd be violating only one
mutable reference at a time rule.

# Basic Idea

The basic idea is that we create a statically allocated arrays of `AnObject`
objects inside the `allocate_slice` function. When the function is called we
will need to check if the array has enough elements and if it does we mark
it as reserved and return the slice.

On the subsequent allocations I will return `None` for simplicity, but in
general we don't have to mark the whole array as reserved and can satisfy
more then one allocation from the same array if we wanted to.

Let's take a look at the straighforward attempt to translate the basic idea
into code:

```rust
pub fn allocate_slice(size: usize) -> Option<&'static mut [AnObject]> {
    const MAX_SIZE: usize = 32;
    static mut BUSY: bool = false;
    static mut OBJECTS: [AnObject; MAX_SIZE];

    if BUSY {
        return None;
    }

    if size <= MAX_SIZE {
        BUSY = true;
        return Some(&mut OBJECTS[0..size]);
    }

    None
}
```

It's a rough attempt, so unsurprisingly Rust compiler didn't like it:

```
error: free static item without body
 --> test.rs:8:5
  |
8 |     static mut OBJECTS: [AnObject; MAX_SIZE];
  |     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^-
  |                                             |
  |                                             help: provide a definition for the static: `= <expr>;`
```

# Initialization

So what's the problem? As I mentioned before in Rust everything should
be initialized, we didn't initialize `OBJECTS` array with anything.

In C and C++ it might not be a problem, but it is not because in C and C++
it's ok to work with uninitialized data. On the contrary in C and C++
statically allocated objects are initialized.

In C there is concept of zero initialization and statically allocated objects
are guaranteed to be initialized with zeros. It actually makes a lot of sense
too, but of course there are caveats.

What if for your particular type zero initialization doesn't make sense and
doesn't produce an object with a valid state. Well, tough luck then.

Situation in C++ is better. It does inherit the zero initialization from C,
but when zero initialization is not good enough you can provide a constructor
that will initialize the object in whatever state you need. So it's up to you
to tell the compiler if zero initialization is what you need or not.

That's quite reasonable, but it also comes at a cost - at some point somebody
has to call the constructors for the objects. So in C++ some initialization
will have to happen during runtime as opposed to compile time.

So what the story in Rust? Well, in Rust when it comes to initialization of
arrays the story is quite... well, bad. There are roughly four options:

1. You have to explicitly enumerate each element of the array - quite horrible
   for any array with more than just a handful elements;
2. If the type implements `Copy` trait you can provide just one element
   - that's quite convenient, but `Copy` trait implementation is a non-trivial
   requirement;
3. If the type implements `Default` trait, you can just use
   `Default::default()` to initialize the array - again, that's quite
   convenient, but doesn't work for statics initialization;
4. Finally, you can write some `unsafe` code to initialize the array element
   by element - which is neither convenient nor does it work for statics.

The option 2 doesn't work for me because my type doesn't implement `Copy`
trait.

The options 3 and 4 don't work for statics because they do initialization
during runtime and Rust doesn't allow runtime initialization for statics,
like, for example, C++ does with its constructors.

Which lives us with option 1, which is so terrible that I'd rather cut my
hands off.

So what should we do? Well, there is little choice here, but to delay the
initialization until the runtime. So essentially we want to initialize array
on the first access to the `allocate_slice` function.

There are multiple options to do that. Rust library provides a dedicated type
to deal with potentially uninitialized state: `MaybeUninit`. However to
extract a value from `MaybeUninit` you have to resort to unsafe code. While
for this particular problem unsafe code seem to be unavoidable, we can do
better at least for the array initialization with `Option`.

```rust
static mut OBJECTS: Option<AnObject; MAX_SIZE> = None;
```

We don't have a good way to initialize an array with some default values
during compile time, however we do have an easy way to initialize `Option`
during compile time by storing `None` there.

As to the runtime initialization, for the example sake I will resort to
implementing `Default` trait. This gives us the following code:

```rust
pub struct AnObject {
    /* there might be some fields here */
}

impl Default for AnObject {
    fn default() -> AnObject {
        AnObject {}
    }
}

pub fn allocate_slice(size: usize) -> Option<&'static mut [AnObject]> {
    const MAX_SIZE: usize = 32;
    static mut OBJECTS: Option<[AnObject; MAX_SIZE]> = None;

    if OBJECTS.is_some() {
        return None;
    }

    if size <= MAX_SIZE {
        OBJECTS = Some(Default::default());
        return Some(&mut OBJECTS.as_mut().unwrap()[..size]);
    }

    None
}
```

Note how using `Option` allowed us to get rid of `BUSY` flag, since `Option`
provides us with means to figure out if the `allocate_slice` function has
been called before or not.

Naturally, if you want to be able to satisfy more than one allocation from
`allocate_slice` function you'd have to keep track what parts of the array
has been allocated already and just one `Option` wouldn't be enough.

The `OBJECTS.as_mut()` just converts the `Option<[AnObject; MAX_SIZE]>` to
an `Option` with a mutable refernce to the data inside. The rest should be
quite straighforward.

# Mutability

Unfortunately the new code doesn't compile either. If you try to feed it to
the compiler you'll get a bunch of errors like this:

```rust
error[E0133]: use of mutable static is unsafe and requires unsafe function or block
  --> test.rs:15:8
   |
15 |     if OBJECTS.is_some() {
   |        ^^^^^^^ use of mutable static
   |
   = note: mutable statics can be mutated by multiple threads: aliasing violations or data races will cause undefined behavior
```

As the compiler error suggests, imagine the case of multithreaded code. If
this function were to be called from multiple threads concurrently, since it
doesn't provide any kind of syncrhonization we will have a race condition on
our hands.

Note though, that the problem here is more general than just a case of bad
concurrency. As you can see we basically return from the function a mutable
reference to the statically allocated data.

By just looking at types for Rust compiler it's impossible to proove that
we never return a mutable reference to the same object twice and therefore
do not create multiple mutable references.

Coincidentally (or maybe not) when you wrap an object that you want to mutate
in `Mutex` or some similar syncrhonization primitive in Rust you don't have to
make the `Mutex` itself mutable.

Why? Well, as far as Rust type system is concerned "immutable" `Mutex` can
return a mutable reference to its internals (though it does it indirectly).
So `Mutex` "cheats" Rust type system, but it does provide runtime guarantees
that make up for that.

Not suprisingly `Mutex` does it via the interior mutability pattern that I
mentioned above, so you can bet that the implimintation of `Mutex` uses unsafe
Rust internally, but provides a safe interface to the users.

Can we use `Mutex` in this example? Unfortunately not. As of now you cannot
create static `Mutex` in Rust because there is no way to initialize it in
compile time.

We can't even resort to the runtime initialization trick we used earlier
because there is no guarantees that initialization of `Option` will be
atomic. Not to mention, that such a static `Option` will have to be mutable
and that completely defeats the point.

So short of implementing your own syncrhonization primitive that allows
compile time initialization you will have to stick to the unafe code. So here
is the version of the code I settled on:

```rust
pub unsafe fn allocate_slice(size: usize) -> Option<&'static mut [AnObject]> {
    const MAX_SIZE: usize = 32;
    static mut OBJECTS: Option<[AnObject; MAX_SIZE]> = None;

    if OBJECTS.is_some() {
        return None;
    }

    if size <= MAX_SIZE {
        OBJECTS = Some(Default::default());
        return Some(&mut OBJECTS.as_mut().unwrap()[..size]);
    }

    None
}
```

# Generalization (Failure!)

Ok, we have a more or less sensible approach to static allocation. Can we
generalize it?

Rust provides a few options to write generic, or, I should say, polimorphic
code. For this particular problem we care about compile time polimorphism
only. Which leaves us with a few options:

* Rust generics
* Rust macroses
* code generation.

I have a personal despise for code generation. My personal opinion here is
driven by my negative experince trying to read code of the projects that
involve code generation step as part of their build process. So I didn't
even consider generating code.

Having an experience with C++ templates I decided to go with Rust generics as
the problem looked like an easy one. I figured that Rust has both type
generics and const generics, just like C++, so the code like this should
work just fine:

```rust
pub unsafe fn allocate_slice<T: Default, const N: usize>(size: usize) -> Option<&'static mut [T]> {
    static mut OBJECTS: Option<[T; N]> = None;

    if OBJECTS.is_some() {
        return None;
    }

    if size <= N {
        OBJECTS = Some(Default::default());
        return Some(&mut OBJECTS.as_mut().unwrap()[..size]);
    }

    None
}
```

So what is this monstrosity is supposed to mean? Well, our function has two
generic parameters: the type of the allocated objects and the maximum size of
the storage. There is nothing surprising there - we just replace `AnObject`
with a generic `T` and hardcoded `32` with `N`.

I additionally require the type `T` to implement the `Default` trait to be
able to initialize an array of objects of type `T` - that's exactly how I did
it with `AnObject` type before as well.

All-in-all, it's not that bad and by C++ standards that is pretty ordinary
use of generics. There is one downside with this implementation though - it
doesn't compile:

```
error[E0401]: can't use generic parameters from outer function
  --> test.rs:12:33
   |
11 | ...e_slice_gen<T: Default, const N: usize>(size: usize) -> Option<&'stat...
   |                - type parameter from outer function
12 | ...S: Option<[T; N]> = None;
   |               ^ use of generic parameter from outer function

error[E0401]: can't use generic parameters from outer function
  --> test.rs:12:36
   |
11 | ...e_gen<T: Default, const N: usize>(size: usize) -> Option<&'static mut...
   |                            - const parameter from outer function
12 | ...ion<[T; N]> = None;
   |            ^ use of generic parameter from outer function

error: aborting due to 2 previous errors
```

Turns out in Rust you cannot use generic parameters as part of static object
type or initialization. The error from the Rust compiler is not particularly
descriptive, but if you run `rustc --explain E0401` you'll get a few examples
and should be able to understand why the error message is the way it is.

The same documentation provides a few ways to circumvent the problem for some
cases, but unfortunately none of them works for me, at least not without
moving problem to runtime.

The reason for this is still a mistery for me, but from googling around I got
that it has more to do with the linking rather than with some fundamental
property of the Rust language. You can read more by looking at [this comment]
(https://internals.rust-lang.org/t/how-about-generic-global-variables/8351/8).

Unfortunately I don't know enough about Rust dylibs, so it's hard to
understand why instantiation of such generics is a problem for dylibs.

BTW, if you got scared by "monomorphisation" don't be. Behind the scary word
there is an easy to understand concept. Just take a look at [static dispatch]
(https://en.wikipedia.org/wiki/Static_dispatch).

This leaves us with the Rust macroses as the last resort. However I didn't
go that way and left the code as it's without trying to further generalize
it. You can take that as an exercise and implement a Rust macro for static
memory allocation.

# Instead of conclusion

Well, I didn't expect that working with statics in Rust would be that much
of a pain. I'd say Rust would benefit a little bit from introducing a concept
of zero initialized types.

And judjung by existance of `intrinsics::assert_zero_valid` Rust creators
do not object to the concept on the fundamental level. With some efforst it
could turned into some kind of special trait `Zero`, similarly to `Copy`,
`Sync` or `Send`.

Anyways, all well that ends well. I learned something new (for example, that
Rust generics aren't quite at the same level as C++ templates), so that's
something.
