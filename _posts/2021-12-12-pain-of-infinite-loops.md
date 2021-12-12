---
layout: post
title: The pain of infinite loops
excerpt_separator: <!--more-->
tags: c++ undefined-behavior compiler-optimizations clang llvm aarch64
---

I've been struggling dealing with various Rust quirks in my hobby projects and
some day I had enough, purged all the Rust code and moved to C++, just to hit
an expected, but still interesting quirk of C++.

<!--more-->

# Background

For a bit of background, for my hobby projects that are on a "low-level" side of
the software stack I mostly used C. C tooling is predictable and you don't need
to think about runtime much.

C++ and Rust are in the similar category from that point of view, as they both
require some non trivial runtime support, and it's not immediately obvious what
runtime do they actually need and what properties it should satisfy.

> *NOTE:* I've heard claims that Rust has zero-runtime, but under closer
> inspection those claims turned out to be not true.

> *NOTE:* for C++ at least there are various ABIs that document the runtime,
> plus there are compiler docs that shed some light, but I've always been
> struggling to build a complete picture because C++ runtime is actually quite
> large.

# Introduction

With that background out of the way, we have LLVM-based C++ toolchain in our
disposal and we can build freestanging programs.

> *NOTE:* compiler support for C++ freestsanding environment is quite
> attrocious. Don't mean to offent anyone, just my personal opinion on the
> current state of the available tools.

I'm working with `aarch64` architecture, but I don't think it matters that much
for the thing I'm going to cover in this post.

Finally, to make our life miserable, I'll be building everything using `-Ofast`.
And that is a bit of a give away. There are stories of C and C++ compilers
performing pretty un-intutive "optimizations" making a lot of assumptions about
what software engineers should and should not write in their code. This is
another such story.

Now to the problem, in a freestanding environment what should we do when the
program discovers something it cannot handle? I want to create a function that
I'll call in this situations, and so, I'm going to call this function `Panic`
going forward.

One options is to reset/restart the machine. This option is undesirable for the
hobby project, because I'd like to be able to debug what lead to the `Panic`
call and reseting the machine will lose all the valuable debugging information.

> *NOTE:* reset doesn't have to lose all the valuable info, we can for example
> collect all that information in the `Panic` call and save it somewhere, but
> it's too hard, so for now I'll assume that it's not an option.

Another option is to just hang in the `Panic` call. While the machine is hanging
in the `Panic` call I cann attach to it with debugger and callect everything I
can for debugging. And, compared to the first option, it's just an infinite
loop - it couldn't be any simpler, right?

# Implementation

Enough with the suspence, you all know where it's going. Let's start with the
naive implementation:

```c++
namespace {

void Panic() {
    while (1);
}

}  // namespace

extern "C" void kernel() {
    Panic();
}
```

> *NOTE:* the `kernel` function is called from the assembler, so it's marked as
> `extern "C"` to avoid C++ name mangling.

In this example, the `kernel` function does nothing, it's just expected to hang
right away in the `Panic` call. To surprise however, it actually didn't do what
i wanted it to do.

When I looked at the generated assembler code with `llvm-objdump -D` at the code
of the `kernel` function it looked like this:

```
00000000000020c8 <kernel>:
    20c8: e0 03 1f 2a   mov     w0, wzr
    20cc: c0 03 5f d6   ret
```

It should be more or less easy to see that there is no function call or infinite
loop here, but just for the clarity, `mov w0, wzr` just writes 0 to the register
`w0` and `ret` is just an instruction that returns from a function call.

> *NOTE:* `wzr` is a "register" that contains 0 and `zr` in the name is a
> referece to that.

If I remove the `-Ofast` flag from the compiler parameters, I see a very
different picture:

```
0000000000002148 <kernel>:
    2148: fd 7b bf a9   stp     x29, x30, [sp, #-16]!
    214c: fd 03 00 91   mov     x29, sp
    2150: 03 00 00 94   bl      0x215c <_ZN12_GLOBAL__N_15PanicEv>
    2154: fd 7b c1 a8   ldp     x29, x30, [sp], #16
    2158: c0 03 5f d6   ret
```

I will ignore irrelevant parts and will just point out that `bl` instruction
is a branch instruction commonly used in ARM to call functions and
`_ZN12_GLOBAL__N_15PanicEv` is the mangled name of our `Panic` function.

So without optimizations the `kernel` function does some stack manipuations (
`sp` is a stack pointer register) and then calls the `Panic` function.

For completeness less look at the `Panic` function itself:

```
000000000000215c <_ZN12_GLOBAL__N_15PanicEv>:
    215c: 00 00 00 14   b       0x215c <_ZN12_GLOBAL__N_15PanicEv>
```

The function contains just one instruction - `b`. `b` is an unconditional jump
in ARM and in this case it jumps to the instruction itself, so it's an infinite
loop - pretty much as you expect.

> *NOTE:* without explicitly enabled optimizations I would expect that compiler
> would put a `ret` instruction at the end, but it did realize that there is no
> escape from the inifinite loop and removed all the dead code after the loop.

When I saw that my code behaves differently with and without optimizations I
knew a screwed up somewhere, but where?

In the example I've shown the code pretty much does nothing, so where could the
problem be?

# Infinite loops are the problem

The problem is the inifnite loop itself. Turnes out many-many years ago in a
glaxy far-far away, standardization committee decided that loops have to
terminate: [Optimizing away infinite loops](http://www.open-std.org/jtc1/sc22/wg14/www/docs/n1509.pdf).

> *NOTE:* N1509 is a document from the WG14 - a working group responsible for
> the C standard, not C++, but it's still interesting and I found that one
> before I found other documents.

*TL;DR:* if the body of the loop and it's condition/control parts don't contain
any operations that can be considered an observable behavior, compiler is
allowed to assume that this loop will terminate.

What does observable behavior mean? It's technical, but it's easy to understand:

* input/output operations (like operations with `std::fstream`) are considered
  observable behavior
* operations over `volatile` objects are also considered observable
* synchronization and atomic operations are observable as well.

As you can see our initial implementation had no operations that can be
considered observable. So in our case the compiler was allowed to assume that
the loop will terminate.

It's not a bit leap from that to understand why compiler dropped the loop all
together. If the loop doesn't have any visible effects and we know that it will
terminate - it's useless and can be dropped.

# The good

We now know what is happening, but given it's somewhat unintuitive consequences
of this optimization, it's worth understanding why it was introduced, surely
the smart folks in the standardization committee didn't do it just for the fun
of seeing C and C++ developers struggle.

We can find a somewhat shallow explanation of the intent in the N1509 itself:

```
This is intended to allow compiler transformations such as removal of empty
loops even when termination cannot be proven.
```

Initially it looks like a cop-out. Basically they are saying that in some
limited set of situations when compiler really-really wants to optimize
something away, but can't demonstrate that it's correct, we just let the
complier to assume that everything is fine.

Well, that does not sound great, does it? And if you read it through, the author
of the N1509 calls out the fact that it's a bit of a breaking change despite
some claims that it just preserves the status quo of how compilers already work.

So did they actually introduce it to annoy people after all? I'm starting to
lean toward this conspiracy theory, but let's give them a bit of benefit of a
doubt and keep digging.

It was the time when C++ standard committee made some pretty significant changes
to the C++ standard to support concurrency. In a concurrent environment with
shared memory they had to formally specify how this shared memory works. They
introduced a memory model in a set of documents culminating in:
[N2334](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2007/n2429.htm).

In N2334 they indeed argued that the specification of the C++ at the time
already didn't impose much of restrictions on infinite loops without side
effects and that compiler already eliminate such loops.

I haven't looked at the compiler implementations of that time, so I'm not going
to comment on that. However the argument that C++ standard at the time allowed
it anyways (or didn't specify it well enough to disallow such "optimization") is
just bad. Hanging in an infinite loop is a pretty noticeable side affect that
does in fact affects observable behavior of the prgram, doesn't matter how you
look at it.

I'm genuenly having a very hard time beliving that
[Hans Boehm](https://hboehm.info/) didn't realize that this part of the argument
was not particularly sound. And, I think, I'm not the only one and that's how
N1509 came to be in the first place. So why?

Hans actually wrote a response to N1509: [N1528](http://www.open-std.org/jtc1/sc22/wg14/www/docs/n1528.htm).
The response argues three points:

* consistency between C and C++
* status quo
* optimizations that would not be possible without this assumption

Let's get the point about consistency between C and C++ out of the way, since
it's mostly irrelevant. As I mentioned above, N1509 is a document from the C
standardization working group, not C++. They tried to reconcile C and C++, by
adopting some of the things introduced in the new C++ standard to C. So this
argument can be rephrased as "C++ has it and there are reasons why C should not
diverge from C++". While it's an argument, it doesn't explain why it was
introduced to C++ standard in the first place.

Status quo argument is again about compilers already eliminating such loops.
As I said before, I cannot comment on what compilers did at the time. That being
said, it seems like they are trying to save some work for compiler developers
at the cost of the experience of those who will use those compilers.

> *NOTE:* I don't want to argue that this kind of reasoning is always bad, but
> I feel that in this particular case it was a bad decision, especially in the
> light that the last argument somewhat lacking supporting evidence.

Now to the interesting argument, what optimizations this special assumptions
allows compilers to do that they would not be able to do otherwise?

There are no general algorithms that could determine if an arbitrary loop
terminates or not - that follows from the fact that the
[Halting problem](https://en.wikipedia.org/wiki/Halting_problem) has no
solutions. Just think of a loop that exectues steps of a turing machine based
on it's description.

So it's clear, that the compiler cannot always prove if a loop terminates. This
special dispansation for loops in the standard allows compilers to assume that
a loop terminates even if it can't prove it, sometimes incorrectly.

N1528 even provides a toy example of a situation where it could happen:

```
for (p = q; p != 0; p = p -> next) {
    ++count;
}
for (p = q; p != 0; p = p -> next) {
    ++count2;
}
```

The two loops traverse the same linked list (if you replace 0 with `NULL` it
might be a bit easier to see). If we can assume that all loops without side
effects terminate, then the first loop terminates as well (assuming that there
are no volatile objects involved).

With that we can rewrite the two loops the following way without losing
correctness:

```
for (p = q; p != 0; p = p -> next) {
    ++count;
    ++count2;
}
```

This is somewhat better as we only need to traverse the list once. We might
even benefit from some vector instructions for `++count` and `++count2` (purely
hypothetically - I don't think it will make much of a difference in practice).
So we do allow some optimizations by introducing this special assumption about
terminating loops.

How ubiqoutous such cases? I don't know and the N1528 doesn't provide any data.
So I can't really say anything on whether this kind of optimizations even worth
the trouble.

> *NOTE:* I don't make a claim either way. These optimization may or may not be
> significant. The only claim I can confidently make that N1528 didn't provide
> any evidence that would demonstrate the significance.

All-in-all, assuming that loops without side effects always terminate allows for
some optimizations that otherwise would be impossibe and that's all I can say.

# The bad

I started by presenting the problem of creating a `Panic` function and then
immediately showed you a problem that a naive implementation like that creates.
However, that's not at all how I hit this problem.

I hit this problem only when my codebase grew to 1000s of lines of code. When
my code started to misbehave the initial set of possibilities that I explored
were running out of stack, corrupt memory and so on.

> *NOTE:* some might say that rewriting code in Rust would remove all the
> memory related bugs out of the picture and would have made my life easier.
> That's actually not the case due to the nature of the problem I was solving.
> You see, when the problem you're trying to solve is allocation of memory
> resources, you have no choice but to use unsafe from the Rust point of view
> manipulations, so all the same set of problems would still be in scope even
> if the code was written in Rust.

I did a bunch of tests to create a small reproducer, but I couldn't really come
up with anything. It's only much later, when I started going through individual
assembly commands in GDB, I noticed that the compiled code looked weird and not
at all what I expected it to be.

Some might say here that there are plenty of questions and posts already written
about non-terminating loops in C and C++. And indeed this topic is well covered
all over the Internet. However, when my code started to misbehave I didn't
apriori know what caused the problem and it's quite a debugging jump from a
generally misbehaving program to non-terminating loops.

# The ugly

With my whining and hurt pride out of the way, what can we do?

Well, now when we know what compiler will look at when it decides whether the
loop can be dropped or not, rewriting our `Panic` function should be
straightforward.

I, however, to redeem myself wanted to try and find how we can make the compiler
warn me when it eliminates loops. My best attempt was to write something like
this:

```c++
[[ noreturn ]] void Panic() {
    while (1);
}
```

As the name suggests, `[[ noreturn ]]` tells compiler that the function is not
expected to return to the caller. The logic behind introducing the
`[[ noreturn ]]` was that at least the compiler I use warns if I write code
like this:

```c++
[[ noreturn ]] void Panic() {}
```

The error I get from the compiler looks something like this:

```
main.cc:9:1: error: function declared 'noreturn' should not return [-Werror,-Winvalid-noreturn]
```

So, I was thinking that if compiler can assume that `while (1);` always
terminates then, just like in the example above, it can warn me that a function
marked as `[[ noreturn ]]` terminates and it's not right.

Unfortunately, Clang is "smart" enough to eliminate the loop, but is not smart
enough to realize that it would mean reaching the end of function that should
never terminiate.

> *NOTE:* some of you may have heard or seen `__builtin_unreachable()` function
> and thought about using it here. `__builtin_unreachable()` function however
> does not solve the problem. The point of `__builtin_unreachable()` is to help
> compiler when it cannot figure out on its own if some code is unreachable.
>
> For example, the following compiles without a warning for me in Clang:
> 
> ```c++
> [[ noreturn ]] void Panic() {
>     __builtin_unreachable();
> }
> ```

> As I shown above, without the `__builtin_unreachable()` the compiler sees a
> problem in the code and rightfully so. With the `__builtin_unreachable()` it
> doesn't see the problem anymore, because we "helped" it.

Needless to say, I couldn't find in the Clang docs any magical `-W` command line
argument I could use to ask compiler to warn me about dropped loops.

Here I will go into a bit of a speculation and suggest that, at least in Clang,
all the compilation warnings are generated long before LLVM applies any of its
optimizations.

The mental model I have is as follows: we have a compiler frontend that parses
the source code of the program and produces the internal representation of the
code. The frotend can analyze the structure of the program and emit some
warnings.

The generated internal representation is then processed by the compiler backend.
The backend generates the machine code and applies optimization, but, it looks
like, the backend assumes that the internal representation it got from the
frontend describes a well formed program already and doesn't generate any
warnings.

In the end I settled on the following ugliness in my code to avoid spending
even more time dealing with this silly issue:

```c++
[[ noreturn ]] void Panic() {
    while (1) {
        asm volatile("":::"memory");
    }
}
```

# Instead of conclusion

* Don't have hobby projects - they are bad for your mental health;
* Don't make mistakes in your code - they are bad for your mental health;
* If you made a mistake in your code, write a inconsequentially long post about
  it, and don't forget to mention how it's everybody's else fault - it's good
  for your mental health.

