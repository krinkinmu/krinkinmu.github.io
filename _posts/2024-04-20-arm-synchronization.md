---
layout: post
title: AArch64 shared memory synchronization
excerpt_separator: <!--more-->
tags: aarch64 synchronization concurrency
---

I'm continuing playing with 64 bit ARM architecture and the next thing I want
to try is spinning up multiple CPUs. However before doing that I need to get
out of the way the question of synchronizing multiple concurrently running
CPUs and that's what I touch on in this post.

<!--more-->

# The plan

Basically for synchronization of multiple CPUs it should be enough to implement
a simple mutual exclusion mechanism sometimes referred to as spinlock. And the
simplest implementation of the spinlock if pretty straighforward, as you will
see.

Once we have a simple implementation, I'm going to use it as an opportunity to
look at somewhat more subtle details of shared memory synchronization like
memory model and memory barriers. Hopefully that would show why correct
synchronization is difficult and why using simple synchronization primities
like mutual exclusion is so convenient.

# Simple spinlock

In the simple implementation I will rely on an atomic read-modify-write
operation that exists in many programming languages. To be specific I will be
using C, but C++ and Rust would have a very similar mechanisms available.

In the standard C there is a header `stdatomic.h` that defines a type
`struct atomic_flag`. You can think of this type as a `bool` variable, with a
caveat that you can't (or shouldn't) work with it as a simple `bool` variable
and instead should rely on special functions.

The basic idea of the implementation is that we have a bool variable that
initially is set to `false`. We want to implement two functions: `lock` and
`unlock`.

The point of those two function is that a thread of execution that successfully
finished execution of function `lock` and before it executed `unlock` becomes
an "owner" of the lock on which the function was called.

And while the lock has an owner like that, all other threads of execution that
call the `lock` on the same object will get stuck in the `lock` function and
will not complete until the current owner of the lock calles `unlock` function.

Let's take a look at an example and for that we will start with defining an
interface for our implementation:

```c
struct spinlock {
    ...
};

void spinlock_lock(struct spinlock *l);
void spinlock_unlock(struct spinlock *l);
```

Now with that interface in mind, let's consider an example when we have a
structure with a complex state that is being accessed from multiple threads.
Let's say for example, we have a binary search tree or something like that:

```c
struct node {
    struct node *parent;
    struct node *left;
    strict node *right;
};

struct search_tree {
    struct node *root;
};
```

If we start inserting and deleting elements from the tree from multiple threads
without any synchronization, because making changes to the search tree requires
multiple operations done in the correct order, those threads will start steping
on each other feet.

To avoid that we can use the spinlock as follows:

```c
struct spinlock lock;
struct search_tree tree;

...

void search_tree_insert(struct search_tree *tree, struct node *new_node)
{
    spinlock_lock(&lock);
    tree_insert(tree, new_node);
    spinlock_unlock(&lock);
}
```

Because the lock can have no more than one owner, at most one thread can
execute `tree_insert` at the same time, thus we avoid situation when multiple
threads step on each other feet.

> NOTE: We only get this guarantee of mutual exclusion if all the threads of
> execution follow the same protocol and take ownership of the same lock before
> making any updates to the search tree in the example above.

Now, let's take a look at how we can implement this using only C functions
 defined in the standard (and are not really architecture specific):

```c
struct spinlock {
    struct atomic_flag flag;
};

void spinlock_setup(struct spinlock *l)
{
    atomic_flag_clear(&l->flag);
}

void spinlock_lock(struct spinlock *l)
{
    while (atomic_flag_test_and_set(&l->flag));
}

void spinlock_unlock(struct spinlock *l)
{
    atomic_flag_clear(&l->flag);
}
```

There are basically just a couple of functions:

1. `atomic_flag_test_and_set`
2. `atomic_flag_clear`

`atomic_flag_clear` writes to the `atomic_flag` structure a value of `false`
atomically - rather simple, if we leave aside the details of what atomic
actually means.

`atomic_flag_test_and_set` is somewhat more interesting it atomically as one
indivisable operation performs the following actions:

1. read the current value in the flag
2. write `true` to the flag

So it reads the old value and writes a new one as one operation. The old value
is returned as a result of the function.

Taking that semantic let's look at the `spinlock_lock` function in a bit more
details. The function will spin in the loop as long as the value in the flag is
`true`.

If the function at some point observes that the value is not `true` it will
write `true` there again and returns.

Intuitively it's rather easy to understand - when `atomic_flag` structure
contains `true` it means that the lock has an owner and when it contains
`false` the lock is free and does not have an owner.

We can aquire the lock by writing `true` value to the flag, but only as long
as it does not have an owner already. And we can release the lock by writing
`false` there, but only the current owner of the lock should do that obviously.

This implementation of mutual exclusion is actually correct, but as we will see
below its simplicity might be a bit misleading and there is much more going on
here than meets the eye. Moreover, as we will see there are some things that we
can actually change in this implementation to gain some additional properties.

# Memory ordering constraints

Quoting Leslie Lamport, "accessing a single memory location in a multiprocessor
is traditionally assumed to be atomic. Such atomicity is a fiction; a memory
access consists of a number of hardware actions, and different accesses may be
executed concurrently".

What it means is basically, just because the instructions in the program go in
a certain order, does not mean that the processors executing them has to do
them in the same order.

It might sound weird, but it actually makes sense if you consider that processor
may try to optimize what it is doing to achieve the same result faster or more
efficiently.

The problem is that when you have multiple cores working independently it's
difficult to tell what the expected end result should be. Even without potential
re-orderings that processor might do, when you run a program on multiple cores,
without proper synchronization, the result might essentially be unpredicatble.

And that's what it comes to in the end - proper synchronization must tell the
processor what operations can and cannot be re-ordered. Now, with that in mind
let's take a look at the spinlock implementation and figure out what are the
ordering constraints we have.

Looking again at the hypothetical example of updating a search tree:

```c
// Code before the critical section
spinlock_lock(&lock);

// Critical section that should only be executed by one thread at a time
tree_insert(tree, new_node);

spinlock_unlock(&lock);
// Code after critical section
```

One thing that we want to enforce here is that `tree_insert` function is not
executed concurrently by multiple threads and that's why we use the lock. So
at the very least we want to prevent re-orering of `tree_insert` or any of its
parts before `spinlock_lock` or after `spinlock_unlock`, otherwise it defeats
the purpose of having the lock in the first place.

On the other hand, for correctness at least, we don't need to prevent any code
before or after the critical section being moved into the critical section.
That's because that code could be run by multiple threads at the same time, so
it could be run by a single thread as well, without violating correctness.

As I mentioned earlier, the implementation of the spinlock was correct, so what
exactly in the implementation prevents re-ordering of `tree_insert` logic
outside of the critical section guarded by spinlocks?

What controls memory ordering constraints around atomic operations like
`atomic_flag_clear` or `atomic_flag_test_and_set` is memory order parameter.
The atomic operations I used above don't take such a paramter, but there are
versions of these functions that do: `atomic_flag_clear_explicit` and
`atomic_flag_test_and_set_explicit`. The versions I used above are equivalent
to the explicit versions with the parameter `memory_order_seq_cst`.

`memory_order_seq_cst` means sequentially consistent memory operation ordering.
To understand what kind of constraints does the `memory_order_seq_cst` implies
let's look at the C standard. The relevant parts we need to pay attention to
are:

1. "sequenced before" relation
2. "synchronizes with" relation
3. "inter-thread happens before" relation
4. "happens before" relation

Quite a few things to unpack here, but once you're go through the formalisms,
it's not that hard to develop basic intution about those things.

## Sequenced-before relation

Let's start with the "sequenced before" relation. It's an asymetric and
transitive relation between evaluations executed in a single thread.

This relation induces a partial order between evaluations in a single thread.
Which in this case means that we can tell for some of the evaluations whether
one of them is computed before another.

Let's take a look at a toy example:

```c
#include <stdio.h>
#include <stdlib.h>
...
char *buf = malloc(large_enough_buffer_size);
char *p = buf;
...
while ((*p++ = getchar()) != EOF);
```

What can we say about evaluations in this example and how they are ordered?
I can say that `malloc` is definitely "sequenced before" `p = buf`. That's
because those two statements are separated by ';' which creates a so called
sequence point between the two.

Similarly, I can tell that `p = buf` is "sequenced before" `p++` and `getchar`
because they are separated by a sequence point. Because "sequenced before" is
transitive relations, I can also say that `malloc` is "sequenced before" `p++`
and `getchar`, because it's "sequenced before" `p = buf`, and, in turn,
`p = buf` is "sequenced before" `p++` and `getchar`.

In the same example however, I cannot tell whether increment of `p` "sequenced
before" `getchar` or not. So sequencing of those two evaluations is not
determined, so it demonstraints that the relation is not total, but in fact
partial - not all evaluations can be ordered with this relation.

To summarize, "sequence before" relation defines how evaluations happenning in
a single thread can or cannot be ordered.

One point that I want to stress is that "sequenced before" relation is strictly
single threaded. It does not order, at least directly, evaluations that happen
in different threads. For example, just because A is sequenced before B in
thread T1, as unnatural as it sounds, it does not actually imply, that another
thread T2 will see the effects of A before it will see the effects of B.

## Synchronizes-with relation

This relation is really the core and probably the most important thing we need
to understand.

Unlike the previous relation, this one is about evaluations or operations
happening in different threads. And language standard has to define what
operations synchronizes with each other.

The version of the C standard covers several cases, but one particularly
important is atomic release and atomic acquire operations on the same object
synchronize with each other under some conditions.

What is an atomic release operation? Atomic release operation is an atomic
operation that performs a write or stores a value in the atomic variable using
one of the following memory ordering parameters:

1. `memory_order_release`
2. `memory_order_ack_rel` - this is just short for `acquire and release`
3. `memory_order_seq_cst` - this is the one which atomic operations use by
   default.

An acquire operations is an atomic operation that performse a read or loads a
value from an atomic variable using one of the following memory ordering
parameters:

1. `memory_order_acquire`
2. `memory_order_ack_rel`
3. `memory_order_seq_cst`.

> NOTE: some atomic operations can perform more than just load or store, like,
> for example, `atomic_flag_test_and_set` performs both read and write, in this
> case the operation still can be a release or an acquire operation even though
> it does more than just store or just a load.

Now, I mentioned that two atomic operations synchronize with each other if they
act on the same atomic variable, use right memory ordering paramter and satisfy
certain conditions, what are those conditions?

Let's say we consider two operations A and B, both acting on the same atomic
variable M. Let also A be a release operation, it means that among other things
it writes a value to M. And let B be an acquire operation, so among other
things it reads from M.

A synchronizes with B if B reads from M a value written by A or any atomic
operation that wrote a value to M after A. In other words, A synchronizes with
B, if B reads a value written by A or a value written after A wrote something
to the atomic variable.

So to put it even simpler, two operations on the same atomic variable
synchronize with each other if one operation sees the result of another
operation plus all the addition conidtions on the memory order parameters.

## Inter-thread happens before and happens before relations

So now we have some idea about "sequenced before" and "synchronizes with"
relations, what remains is to understand some rules of how those relations can
be combined with each other and what properties those combinations have.

In order to do that C standard introduces additional relation called
"inter-thread happens before". We don't need to know all the rules of this
relation for this post, so I will simplify and only cover those that I'm going
to use.

Again, let's say we have two evaluations A and B, unlike in case of "sequenced
before relation" these evaluations may potentially be in different threads. We
say that A "inter-thread happens before" B if one of those is true:

* A synchronizes with B
* There is some evaluation X and A is sequenced before X and X inter-thread
  happens before B
* There is some evaluation X and A inter-thread happens before X and X sequenced
  before B.

So it's a recursively defined relation, but it's a rather intutive definition
that allows to combine sequences of "sequenced before" and "synchronized with"
relations together.

Happens before relation is even simpler, evaluation A happens before B if one
of the following is true:

* A is sequenced before B, which implies that A and B are executed in the same
  thread
* A is inter-thread happens before B, and in this case A and B can be executed
  in differnt threads.

Hopefully, the name of the relation "happens before" gives you a clue as to what
this relation implies in practice, but when it comes to the language standard it
additionally formalizes it by introducing a concept of visible side effect.

Basically a side effect of an operation A on some object M is visible to the
operation B on the same object M, if A happens before B and there is no other
operation X that affects M such that A happens before X and X happens before B.

> NOTE: that in this case M does not have to even be an atomic object - as long
> as we can establish that A happens before B, side effects of A should be
> visible as long as they were not overriden by some other operation.

## Putting things together

Now let's try to put things together and return to our hypothetical example:

```c
// Code before the critical section
spinlock_lock(&lock);

// Critical section that should only be executed by one thread at a time
tree_insert(tree, new_node);

spinlock_unlock(&lock);
// Code after critical section
```

Let's consider an example with just two threads, T1 and T2, to be specific. This
example, will extend the same way for a larger number of threads.

Let's say that thread T1 owns the lock `lock` at the moment and thread T2 tries
to take ownership by calling `spinlock_lock`.

Assuming that `tree_insert` call does not get stuck indefinitely, thread T1 will
eventually call `spinlock_unlock`. Remember that `spinlock_unlock` inside uses
a function `atomic_flag_clear` which writes to an atomic variable and uses
`memory_order_seq_cst`. *So it's a release operation.*

Concurrently, thread T2 calls `spinlock_lock` and in its implementation we used
`atomic_flag_test_and_set` function, which, among other things, reads from an
atomic variable using `memory_order_seq_cst`. *So it's an aquire operation.*

If inside `spinlock_lock` T2 reads from the atomic variable value `true` it
means that either T1 didn't write `false` into the atomic variable yet or the
write is not visible to T2 yet. Either way, T2 will re-try the operation in the
loop.

Once T1 actually writes `false` in the atomic variable and T2 reads the written
value from the same variable, we can tell that `atomic_flag_clear` inside
`spinlock_unlock` syncrhonizes with `atomic_flag_test_and_set` inside
`spinlock_lock`.

And at this point, we know that everything that in thread T1 was sequenced
before `atomic_flag_clear` or `spinlock_unlock` is guaranteed to have happened
before `atomic_flag_test_and_set` and everything that is sequenced after it
and `spinlock_lock` function in the thread T2.

Thus we demonstrated that T2 cannot take ownership of the lock before thread T1
completed all its operations in the critical section and released the lock.

## Improved implementation

The name of this section is somewhat misleading - I don't actually know how much
of an improvement this is in practice. That being said, careful reader may have
noticed, that earlier I mentioned some other memory ordering options besides
`memory_order_seq_cst`.

Specifically, there were as well the following memory ordering parmaters that
would have provided us with the same properties as `memory_order_seq_cst`:

* `memory_order_acquire`
* `memory_order_release`.

`memory_order_seq_cst` provides the same guarantess as these two memory
orderings plus some more. It stands to reason, that if we don't need those
additional guaratees and can use `memory_order_acquire` and
`memory_order_release` it will give complier and processor more freedom and
might yield better relults.

I additionally will introduce here another type of operation in C called a
fence. Fences can be used to estabslish a synchronizes with relations in the
execution of the program, just like atomic variables, except that they are not
tied to any particular location in memory, like atomic variables do.

Finally, I will introduce one last additional type of memory ordering that C
standard allows for: `memory_order_relaxed`. Atomic operations that use
`memory_order_relaxed` do not establish synchronizes with relations in the
program execution and, in a way, `memory_order_relaxed` provides the weekest
guarantees across all the memory orderings and allows the most freedom to
re-order operations.

> NOTE: atomic operations using `memory_order_relaxed` are still atomic though,
> meaning that observers will see the operation either complete or not, but will
> never see intermediate results of an atomic operation using
> `memory_order_relaxed`.

With those additional tools and considerations we can rewrite our locks as
follows:

```c
void spinlock_lock(struct spinlock *l)
{
    while (atomic_flag_test_and_set_explicit(&l->flag, memory_order_relaxed));
    atomic_thread_fence(memory_order_acquire);
}

void spinlock_unlock(struct spinlock *l)
{
    atomic_thread_fence(memory_order_release);
    atomic_flag_clear_explicit(&l->flag, memory_order_relaxed);
}
```

Similarly to the analysis above, when `atomic_flag_test_and_set_explicit`
operation in the `spinlock_lock` sees a value written by
`atomic_flag_clear_explicit` inside `spinlock_unlock`, the `atomic_thread_fence`
calls in the two functions helps us to establish "synchronizes with" relation.

This implementation might appear to be more intuitive if you think of
`atomic_thread_fence` as a barrier that prevents re-ordering operations.
Specifically, `atomic_thread_fence(memory_order_acquire)` prevents re-ordering
of reads and writes that happen after the fence before the fence and
`atomic_thread_fence(memory_order_release)` prevents re-ordering reads and
writes before the fence after the fence.

This way we get exactly the property that I started with - operations that
happen in the critical section cannot be moved outside of the critical section
due to the fences we put around them.

> NOTE: the new implementation has two advantages over the previous one - it
> replaces stronger `memory_order_seq_cst` with weaker `memory_order_acquire`
> and `memory_order_release`; however, besides that it makes
> `atomic_flag_test_and_set_explicit` which is called in the loop inside
> `spinlock_lock` use the weakest `memory_order_relaxed`, so we don't have to
> call "a heavy" `memory_order_seq_cst` atomic operation in a loop.

# What about ARM?

So far, all we've looked at was provided by the standard of C in an architecture
agnostic way. The same spinlock should work just as correctly on ARM machines as
it would on Intel machines.

FWIW, C is still called a high-level programming language, so it should as much
as practical abstract away the details of the specific hardware and, at least
when it comes to memory ordering, it does it well enough for us not to worry
about how it would work on ARM, Intel or any other hardware for that matter.

However, it would be interesting to actually see what this implementation
actually translates to, so let's take a look...

```sh
llvm-objdump -d spinlock.o 

spinlock.o:	file format elf64-littleaarch64

Disassembly of section .text:

0000000000000000 <spinlock_setup>:
       0: 1f 00 00 39  	strb	wzr, [x0]
       4: c0 03 5f d6  	ret

0000000000000008 <spinlock_lock>:
       8: 28 00 80 52  	mov	w8, #1
       c: 09 7c 5f 08  	ldxrb	w9, [x0]
      10: 08 7c 0a 08  	stxrb	w10, w8, [x0]
      14: 29 01 00 12  	and	w9, w9, #0x1
      18: 49 01 09 2a  	orr	w9, w10, w9
      1c: 89 ff ff 35  	cbnz	w9, 0xc <spinlock_lock+0x4>
      20: bf 39 03 d5  	dmb	ishld
      24: c0 03 5f d6  	ret

0000000000000028 <spinlock_unlock>:
      28: bf 3b 03 d5  	dmb	ish
      2c: 1f 00 00 39  	strb	wzr, [x0]
      30: c0 03 5f d6  	ret
```

I will start with the `spinlock_unlock` function given that it's simpler. `strb`
instruction is just a varian of a regular store instruction. The `b` suffix just
means that this store stores a single byte.

It takes a value from `wzr` register which is a special register which always
reads as 0, so it's just a fancy way to get 0 value.

And `x0`, according to the calling convention, contains the first parmater of
the function, which in our case would the the `struct spinlock` pointer.

Putting things together `strb wzr, [x0]` writes 0 to the first byte of
`struct spinlock` passed as a paramter. Given that `struct spinlock` consists
of just `atomic_flag`, not hard to see that this writes 0 to the flag.

> NOTE: there is nothing specific about the `strb` instruction that makes the
> store atomic, that's because there isn't any need for that and some stores
> in ARM are atomic, i.e. indivisable, by default without any additional
> actions.

The only interesting part of this function is `dmb ish` instruction. This is
a memory barrier instruction and that's basically what
`atomic_thread_fence(memory_order_release)` translated to in the end - a single
instruction.

`dmb` is a data memory barrier instruction. As per ARM refernce manual,
`dmb ish` prevents re-ordering of read and write operations before the barier
with read and write operations after the barrier.

> NOTE: one, rather simplistic and inaccurate, way to think about the `dmb`
> instruction is that it waits until the relevant operations complete before
> allowing any other loads or stores, i.e. `dmb ish` might wait for all loads
> and stores before the barrier to complete before any new loads and stores
> would be allowed to start.

Now, let's take a look at the implementation of the `spinlock_lock`. We know
that it must contain a loop of some form, so let's figure out where in the code
the loop is hidden first.

The key to finding a loop is finding a conditional jump instruction. In this
case `cbnz` is such instruction. As you can see it has two parmaters: register
`w9` and some address.

This instruction compares the value in the register with 0 and if it's not the
case it will transfer control to the address given as the second paramter.
Otherwise the control will go to the next instruction after it.

Looking at the address where `cbnz` would transfer control in our case we can
derive the general structure of the loop:

```asm
    mov w8, #1

# before the loop

body:
    # The loop body
    ldxrb w9, [x0]
    stxrb w10, w8, [x0]
    and w9, w9, #0x1
    orr w9, w10, w9
    cbnz w9, body

# after the loop

    dmb ishld
    ret
```

Let's tryt to parse the body of the loop next. There are two important
instructions to pay attention to here: `ldxrb` and `stxrb`.

`ldxrb` is Load Exclusive Register Byte. This is a special instruction that
reads a byte from address in memory, indicated by by address `[x0]`, to register
`w9`.

`stxrb` is a Store Exclusive Register Byte. This is another special instruction
that writes a byte from a register `w8` to memory, indicated by address `[x0]`.
However, `stxrb` is only successfull if the memory indicated by address `[x0]`
has not been modified since the `ldxrb` instruction.

Whether `stxrb` was able to successfully write the data or not is recorded in
register `w10`. It would contain 0 if the store operation was successful and 1
otherwise.

`ldxrb` and `stxrb` instructions work together with each other and form a pair
of operation often refered to as "load linked/store conditional". These
operations basically implement `atomic_flag_test_and_set_explicit` in our case.
And we have to do them in the loop, until the store succeeds, just like it's
written in the C code.

One last finishing touch is `dmb ishld` instruction. Just like `dmb ish`
instruction we looked at above, it's a memory barrier instruction. The only
difference is that this particular memory barrier prevent loads that happened
before the fence to be re-ordered with loads and stores after the fence.

Why the difference? `dmb ish` that was used earlier would work here as well,
but `dmb ishld` is a weaker barrier that `dmb ish`. And while `dmb ishld` is
weaker than `dmb ish`, it's still enough for correctness, so it was preferred.

Remember that we want loads and stores that happened within the critical section
to stay within the critical section and not escape it. In other words, by the
time we finish the memory barrier instruction, we want all the effects of the
loads and stores to be visible to everyone.

We don't need to prevent any loads and stores that happen outside of critical
section getting inside the critical section, as it's not needed for correctness.

The only reason why `dmb ish` was used in the `spinlock_unlock` is because there
is no other weaker barier that would give the right properties in ARM
instruction set.

> NOTE: let's think a bit about what properties we actually need here in terms
> of barriers? In the `spinlock_lock` function we need to make sure that all
> loads and stores within the critical section happen strictly after the load
> from the `atomic_flag` that `atomic_flag_test_and_set` does, so we need to
> prevent re-ordering of all the loads and stores after the barrier with the
> load before the barrier.
> Thinking similarly about `spinlock_unlock`, we want to prevent all loads and
> stores before the barrier to be re-ordered with the store after the barrier.
> For the first case we have `dmb ishld` that provides exactly the properties
> we need, but ARM architecture does not have the instruction with exactly the
> properties we need for the latter case, so we have to resort to the next best
> thing and it's `dmb ish`.

## Back to sequential consistency

Out of curiousity, does returning back to `memory_order_seq_cst` produces a
different code? Based on what we've learned above, it should, but let's take
a look at what this code translates to:

```c
void spinlock_lock(struct spinlock *l)
{
    while (atomic_flag_test_and_set_explicit(&l->flag, memory_order_relaxed));
    atomic_thread_fence(memory_order_seq_cst);
}

void spinlock_unlock(struct spinlock *l)
{
    atomic_thread_fence(memory_order_seq_cst);
    atomic_flag_clear_explicit(&l->flag, memory_order_relaxed);
}
```

And indeed it does:

```sh
llvm-objdump -d spinlock.o 

spinlock.o:	file format elf64-littleaarch64

Disassembly of section .text:

0000000000000000 <spinlock_setup>:
       0: 1f 00 00 39  	strb	wzr, [x0]
       4: c0 03 5f d6  	ret

0000000000000008 <spinlock_lock>:
       8: 28 00 80 52  	mov	w8, #1
       c: 09 7c 5f 08  	ldxrb	w9, [x0]
      10: 08 7c 0a 08  	stxrb	w10, w8, [x0]
      14: 29 01 00 12  	and	w9, w9, #0x1
      18: 49 01 09 2a  	orr	w9, w10, w9
      1c: 89 ff ff 35  	cbnz	w9, 0xc <spinlock_lock+0x4>
      20: bf 3b 03 d5  	dmb	ish
      24: c0 03 5f d6  	ret

0000000000000028 <spinlock_unlock>:
      28: bf 3b 03 d5  	dmb	ish
      2c: 1f 00 00 39  	strb	wzr, [x0]
      30: c0 03 5f d6  	ret
```

As you can see, with `memory_order_seq_cst` we now use `dmb ish`, the strongest
barrier of those we've seen, both in `spinlock_lock` and `spinlock_unlock`. So
we do in fact see a difference in the generated code, though it's a difference
in just a single instruction.

# How does linux do it?

We looked at how to implement spinlocks in standard C and then looked at what
ARM instructions it translates to. I hope I didn't make any terrible mistakes
there that would lead you towards incorrect assumptions about how
synchronization works in ARM.

That being said, let's take a look at how Linux Kernel does it and compare what
I created above. Linux Kernel spinlock implementation comes with a complexity of
large code base that needs to support multiple different architectures and a lot
of tooling (including tooling used to enforce correctness).

Once you get through the complexities, you will find that the actual
implementation of ARM spinlocks lives in `arch/arm/include/asm/spinlock.h` and
it's not that large and complicated.

And even though it's not that large, it has a few fundamentral differences from
the implementation above:

1. It uses a different algorithm
2. It employs an ARM specific relaxation strategy
3. It uses `dmb ish` barriers in both lock and unlock.

Let's start with briefly touching at the algorithm it uses. It appears that
ARM spinlock implemnetation in Linux Kernel relies on so-called ticket lock
algorithm.

Like our algorithm it relies some kind of read-modify-write operation and has
a loop in the lock function. However, unlike our algorithm, ticket lock provides
additional guarantees of fairness that our algorithm does not have. 

The next difference is that Linux Kernel implementation does not just spin in
a loop like I did above. Instead it executes in `wfe` instruction in the loop
inside lock implementation while it waits to acquire ownership of the lock.

ARM documnetation describes `wfe` or Wait For Event instruction as a hint to
the CPU that it could enter a low-power state and remain in that state until a
wake up event occurs.

It seem to suggest, that ARM CPU can potentially get to a mode of operation
that consumes less energy and stay in such a state until something, i.e. another
CPU presumably holding the lock, calls `sev` instruction that will wake it up.

Finally, as you can see, `arch_spin_lock` function ends with `smp_mb` and
`arch_spin_unlock` function starts with the same `smp_mb` call. `smp_mb` is
a memory barrier function, an equivalant of C `atomic_thread_fence` function
if you will.

So Linux Kernel uses the same type of barrier in both lock and unlock functions.
And digging through the code code you can indeed find that in ARM, `smp_mb`
function translates into `dmb ish` instruction, which is a stronger barrier
than needed, at least to the best of my understanding.

# Instead of conclusion

Hopefully I explored a rather well explored subject of creating a spinlock with
details and follwed up with specifics of the memory model for this post not to
be way too boring.

I was a bit suprised to find lock in linux kernel forces a full memory barrier.
Naturally, I suspected that I was wrong in my understanding of how things work
and that's still certainly a plausible explanation.

That being said, reading doc/Documentation/memory-barriers.txt suggests that
lock operation in Linux Kernel indeed should only behave as an acquire
operation.

Whether linux kernel implemenetation is too conservative or not in the use of
memory barriers, at least from correctness point of view, using stronger and
simpler to reason about barriers seems wise given the overall complexity. And
I'm not even sure if using a weaker barrier will have a measurable effect.

