---
layout: post
title: AArch64 Interrupt and Exception handling
excerpt_separator: <!--more-->
tags: aarch64 arm system-programming interrupts exceptions
---

[previous post]: {% post_url 2021-01-04-aarch64-exception-levels %} "the previous post"

In the [previous post] I gave a somewhat badly structured introduction to the 
priviledge levels model in AArch64. That was a preparation to make explanation
of the interrupt handling a little bit easier in this post.

So let's get started and as always you can find all the sources on
[GitHub](https://github.com/krinkinmu/aarch64).

<!--more-->

# Recap

In the [previous post] I briefly covered the 4 possible priviledge levels in
the AArch64 architecture: EL0, EL1, EL2 and EL3.

Each level has their own version of system registers, like `SPSR_EL3`,
`SPSR_EL2`, `SPSR_EL1` and `SPSR_EL0`. Depending on the configuration each EL
can use a dedicated stack pointer register or they can use the EL0 stack
pointer.

Not surprisingly, higher ELs can access registers of the lower ELs, but not
vice versa.

We also looked in a bit of details at the way `eret` instruction works. It
takes the values in the `SPSR` and `ELR` registers and passes execution by
address stored in the `ELR` register and sets the state of the processor
according to the value in the `SPSR` register.

In the [previous post] we used `eret` instruction to jump from higher EL to
the lower EL. However, moving to the topic of the current post, the `eret`
instruction can be used to return from an interrupt handler.

The general idea is that when an event that requires immediate attention
happens processor interrupts the currently executing code and calls the
interrupt handler - this process is known as taking an interrupt.

Taking an interrupt processor saves the current state of the processor and
the address, where we should return the execution once the interrupt was
handled, in `SPSR` and `ELR` registers automatically. So once the interrupt
has been handled we can just call `eret` to resume the interrupted code.

# Interrupts and Exceptions

However, I'm jumping a little bit ahead of myself. Let's take a step back and
cover what interrupts and exceptions are and what they are for. As I alluded
above the interrupts and exceptions are a way to notify about events that
require some immediate attention.

That being said, the phrase "immediate attention" has been used rather
loosely, so don't take it literally. What events require attention and what do
not to a significant extent depends on your use case. Let's cover a few
examples to give you an idea.

For an OS that hosts multiple userspace application at a time, one faulty
application should not cause the whole system to crash - that would be way to
fragile and unreliable.

Now imagine if a userspace application has executed an illegal instruction. It
may happen for various reasons:

 * stack corruption caused return from the function to pass control to a wrong
   address;
 * the code has been compiled incorrectly and uses an intstruction that is not
   supported by the CPU;
 * the code was intentionally written to do some harm to the system.

Whatever the reason, we don't want one bad application to crash the whole
system. On the other hand, the processor cannot execute the instruction it
cannot understand. So what should we do?

Instead of hard crashing the processor may notify the OS kernel about the
situation. Naturally, there are some restrictions that this notification
mechanism must satisfy to be of help for the described situation.

Specifically, we cannot depend on the state of the procesor, userspace
application stack and registers making any sense. So when this happens, we
want to pass execution to a known trusted address and swtich the processor to
a known good state.

That's exactly what should happen if we correctly configure exception handling
for the processor. The processor will save the current state, even though it
might not make any sense, and call an interrupt handler that the OS kernel
installed to handle the situation. For the described situation there might not
be much the OS kernel can do, but at least it can shut down the application
that caused it without affecting all other applications in the system.

Such error like conditions are often referred to as exceptions. What situation
are considered errors and how to handle them changes from architecutre to
architecture, but you should get a general idea.

> *NOTE:* to avoid being completely hand-wavy here, here is an example of
  a condition that may or may not cause an exception: integer division by
  zero. In x86 it would generate an exception, but in AArch64 they just
  defined what result the division by zero should be, and this way on AArch64
  there is no need to generate an exception, even though it doesn't
  necessarily aggree with math.

Not all conditions that we may want to handle are as problematic as illegal
instructions. Another common kind of the situation we may want to react is an
external device that needs some attention. For example, a network hardware
may want to notify that it received a new network packed or that it completed
sending a network packed and is ready to send another one.

In this case, the notifications exists as an alternative to polling. As an
alternative to asynchronous notification interface, we can communicate with
devices synchronously. For example, after telling the network hardware to
send a network packet we can wait for the completion of the operation by
checking with the device in a loop: "have you finished yet?", "have you
finished yet?"... This way of interructing with devices is called polling.
And depending on the implemention it might waste quite a bit of the processor
computing power by waiting in a tight loop.

In case of the hardware notifications, interrupts allow the processor to do
something else, until the hardware actually requires attention. So it's
essentially doing work in parallel.

I'm not sure if there is settled terminology in the field, but I will refer
to this kind of hardware notifications as interrupts to separate them from
the error conditions, I covered above, that I will refer to as excetions.

Naturally, interrupts may not really require immediate attention as the case
of illegal instructions. With illegal instruction the processor cannot really
continue the execution until we handle it, but with interrupts we can wait
and may decide to delay the handling to do something else. That's what I meant
when I said that I used the phrase "immediate attention" quite loosely.

# Interrupt and Exception types in AArch64

The exceptions and interrupts in AArch64 come in a few different flavours.
Let's start with interrupts as it's easier. Hardware interrupts come in two
kinds: IRQ and FIQ.

Whether a particular hardware event is IRQ or FIQ appear to be configurable.
So it can be IRQ or it can be FIQ. So then what's the difference between the
two you ask? FIQ are higher priority than IRQ. That basically means if you
have two conditions happening at the same time, the FIQ will be taken first.
As far as this post is concerned, there isn't any difference between the IRQs
and FIQs that we care about, so I'm going to treat them the same.

Regarding exceptions, there are two classes here: SError and everything else.
I don't have complete understanding of SError exceptions at the moment. In the
ARM documentation I found one example of a situation that may result in SError
exception: write-back of dirty data from cache to external memory.

From that example, I imagine, there is little that can be done in such a
situation, so it's sort of a critical hardware error. For now I will consider
it to be an error that we cannot really recover from.

The rest of the exceptions are known as synchronous exceptions. And they
basically represent errors (like illegal instruction example above) and
system calls. System call exceptions are exceptions caused by execution of
special instructions: `svc`, `hvc` and `smc`.

`svc` instruction allows to call from EL0 to EL1. `hvc` instruction allows to
call from EL1 to EL2. And `smc` instruction allows to call from EL1 and EL2 to
EL3.

# About interrupting

One thing that requires a special note is that interrupt and exception
handlers interrupt currently executing code. For the example of illegal
instruction that I presented before it might not really seem like a problem
since the application code could not execute anyways. However, for the case
of hardware interrupts it's not the case.

The problem with interrupts and exceptions is that they are asynchronous. By
asynchronous here I mean, that the code they interrupt doesn't necessarily
expect them to happen and, therefore, in general, it cannot be prepared to
them.

That creates a bit of a problem. The executing code necessarily has some state
in the processor registers, on stack and whatnot. To be able to successfully
return the execution back to the code this state has to be preserved. Since
the interrupted code doesn't exepct the interrupt to happen and cannot prepare
to it, it's completely on the interrupt handler to make sure that this state
is preserved.

# Interrupt Routing

And now when we have a general idea of interrupts and exception, it's time to
stop pouring water and move to some practice.

As was discussed above we have up to 4 priviledge levels in AArch64. We can
handle interrupts and exceptions on each one of them. More specifically, at
each EL we need to decide if we want to handle interrupts and exceptions at
that EL or we should delegate it to the lower EL to decide.

Exceptions like illegal instruction that we discussed above only makes sense
to handle at the same or higher EL. So, for example, the option of routing
exceptions caused by the code running in EL2 to EL1 doesn't make much sense.

With interrupts however we may decide that we don't want to handle it at the
EL2 and, for example, delay the handling until we return to the EL1.

So the first practical question we need to answer is which EL will handle the
interrupts and exceptions. With so many ELs routing of interrupts may get
messy, so let's set a goal.

I want to play with AArch64 to check its virtualization and all my code is
currently executing at the EL2. That implies that the code the exception and
interrupt handlers will interrupt will be running in EL2. So I'm only going
to consider the case of handling interrupts and exceptions on EL2, when the
interrupted code is also on EL2.

> *NOTE:* I didn't really settled on any specific architecture of the
  hypervisor, so I don't really know at what EL the interrupts should be
  handled. However all the current code I have runs in EL2, so if I want to
  actually test how my interrupt handling works I have to route them to EL2.

If we want to handle inetrrupts at EL2 we need to make sure that EL3, the only
higher priviledge level that can exist, didn't configure routing of interrupts
and exceptions to the EL3.

We can only do that if our code starts at EL3 and then switches to the EL2. If
the code starts at the EL2 to begin with, then there is nothing we can do with
the EL3 configuration. In this case, the only option for us is to trust the
firmware running at the higher priority (that is if EL3 is even supported).

To configure routing of exceptions and interrupts in EL3 we need to use the
[`SCR_EL3`](https://developer.arm.com/docs/ddi0595/b/aarch64-system-registers/scr_el3)
register. Specifically, we care about the following bits:

* `EA` or bit 3 - when it's set to 1, SError is taken to the EL3 and there is
  nothing that can be done on EL0, EL1 and EL2 to change that;
* `FIQ` or bit 2 - when it's set to 1, all the FIQs are taken to EL3;
* `IRQ` or bit 1 - when it's set to 1, all the IRQs are taken to EL3.

So if we want to handle FIQs, IRQs and SError at the EL2, then on EL3 we need
to set all those bits of `SCR_EL3` to 0. That's how we can do that on the EL3
before jumping to the EL2:

```asm
    #define SCR_EL3_EA  (1 << 3)
    #define SCR_EL3_FIQ (1 << 2)
    #define SCR_EL3_IRQ (1 << 1)
    mrs x0, SCR_EL3
    and x0, x0, #(~SCR_EL3_EA)
    and x0, x0, #(~SCR_EL3_FIQ)
    and x0, x0, #(~SCR_EL3_IRQ)
    msr SCR_EL3, x0 
```

> *NOTE:* you can read the [previous post] for the explanation of the `mrs`,
  `msr` and `and` instructions, but the code should be more or less easy to
  understand.

Now, let's say we are in the EL2. In EL2 we have similar controls that allow
us to decide if we want to handle FIQs, IRQs and SErrors on EL2 or allow
handling on the EL1.

However, somewhat uninutitively the register we need in this case is
[`HCR_EL2`](https://developer.arm.com/docs/ddi0595/b/aarch64-system-registers/hcr_el2)
and not `SCR_EL2` as one could have thought. As a matter of fact there is no
`SCR_EL2` register at all as there is no `HCR_EL3` register - those registers
are specific to EL3 and EL2.

As with EL3 there are three bits responsible for FIQs, IRQs and SErrors:

* `AMO` or bit 5 - when it's set to 1, SError is taken to EL2;
* `IMO` or bit 4 - when it's set to 1, IRQ is taken to EL2;
* `FMO` or bit 3 - when it's set to 1, FIQ is taken to EL2.

So as you could guess in this case we want to set those bits to 1, to route
FIQs, IRQs and SErrors to the EL2. The story doesn't end there however.

Remember that we have a class of synchronous exceptions. As was mentioned
above it only makes sense to handle the synchronous exceptions at the same
level or a higher level. Typically, exceptions happening on the EL0 are
handled on the EL1 (see above the illegal instruction example as to why it
makes sense).

On EL2 we get to decide if we want to let EL1 handle the exceptions or want
to intervine into that process somehow on EL2. Both options actually make
sense under different circumstances. Fortunately for us, we don't have any
code running on the EL1 and EL0 yet. When we do have code on EL1 and/or EL0
the relevant bits are:

* `TGE` or bit 27
* `E2H` or bit 34.

Explainging the effects of those two bits is somewhat involved and requires
digging into details of the virtualization on the AArch64 architecture, so
I'm going to skip them for now and just claim that I'm going to set them both
to 0.

Here is how my code to configure `HCR_EL2` looks like:

```asm
    #define HCR_EL2_E2H (1 << 34)
    #define HCR_EL2_TGE (1 << 27)
    #define HCR_EL2_AMO (1 << 5)
    #define HCR_EL2_IMO (1 << 4)
    #define HCR_EL2_FMO (1 << 3)
    mrs x0, HCR_EL2
    orr x0, x0, #(HCR_EL2_AMO)
    orr x0, x0, #(HCR_EL2_IMO)
    orr x0, x0, #(HCR_EL2_FMO)
    and x0, x0, #(~HCR_EL2_E2H)
    and x0, x0, #(~HCR_EL2_TGE)
    msr HCR_EL2, x0
```

And for now we are done with interrupt and exception routing - not that bad,
right?

# Exception Vector Table

We figured out how to tell at what level the interrupts and exceptions should
be handled, but we also need to instruct the processor where to find the
interrupt handlers. That's is what exception vector table is for.

Exception vector table in AArch64, unsurprisngly, is an array of entries of
fixed size. There are 16 entries in total and each entry is 128 bytes long. So
in total we have a 2KiB table.

This beginning of the table must be aligned on the 2KiB boundary.

We can separate the table in 4 consequtive logical blocks. Let's take a look
at the content of one block. Each block contains exactly 4 entries: one entry
for synchronous exceptions, one entry for IRQs, one entry for FIQs and one
entry for SErrors.

So we have 4 kinds of exceptions and interrupts to deal with and we have one
entry per kind inside the block. That makes a lot of sense.

Each entry contains the code of interrupt/exception handler. In AArch64
instructions are 4 bytes long, so 128 bytes gives us room for 32 instructions.
That might or might not be enough depending on what we want to do.

If it's not enough for the complete interrupt handler, you can just put in
there a jump instruction that passes execution to another place and circumvent
this limitation. I'll show below an example of how it will work.

Now we will move to a slightly more complicated part. I mentioned that the
exception vector table contains 4 blocks (or 16 entries in total). Each block
looks the same and contains 4 entries with exactly the same meaning, but
depending on the configuration and the current state of the processor a
differnt block is used.

There are couple of relevant factors there. First is whethere the exception
or interrupt interrupts the code at the same EL or lower EL.

We will call the EL we are interrupting is the EL the exception or interrupt
is taken *from*. The EL that will handle the exception or interrupt we will
call the EL the exception or interrupt is taken *to*.

If the exception or interrupt is taken from the same EL that is going to
handle it we will use the first half of the table, otherwise the second half
of the table is used.

If the exception or interrupt is taken from the same EL that handles it, there
are two options to consider: whether we will use `SP_EL0` or `SP_ELx`, where
`x` is the EL the exception or interrupt is handled on.

If you read the [previous post] it covered that the processor can be
configured to either use a dedicated stack pointer to EL or use `SP_EL0`.
Depending on that configuration different block is used.

The other case is when the exception or interrupt is taken from a lower EL to
a higher EL. In this case AArch64 takes into account two different options:
the lower EL may be using AArch64 or it may run in a compatibility AArch32
mode. Depending on that a different block of the exceptions vector table is
used.

Fortunately enough, all my code currently runs at EL2, so I don't need to
consider the case when the exception or interrupt handler interrupts code
running in a lower EL.

The complete exception vector table looks like this:

| offset in the table | event type            | description                |
|---------------------|-----------------------|----------------------------|
| 0x000               | Synchronous Exception | EL is using `SP_EL0` stack |
| 0x080               | IRQ                   | EL is using `SP_EL0` stack |
| 0x100               | FIQ                   | EL is using `SP_EL0` stack |
| 0x180               | SError                | EL is using `SP_EL0` stack |
|---------------------|-----------------------|----------------------------|
| 0x200               | Synchronous Exception | EL is using `SP_ELx` stack |
| 0x280               | IRQ                   | EL is using `SP_ELx` stack |
| 0x300               | FIQ                   | EL is using `SP_ELx` stack |
| 0x380               | SError                | EL is using `SP_ELx` stack |
|---------------------|-----------------------|----------------------------|
| 0x400               | Synchronous Exception | From lower EL in AArch64   |
| 0x480               | IRQ                   | From lower EL in AArch64   |
| 0x500               | FIQ                   | From lower EL in AArch64   |
| 0x580               | SError                | From lower EL in AArch64   |
|---------------------|-----------------------|----------------------------|
| 0x600               | Synchronous Exception | From lower EL in AArch32   |
| 0x680               | IRQ                   | From lower EL in AArch32   |
| 0x700               | FIQ                   | From lower EL in AArch32   |
| 0x780               | SError                | From lower EL in AArch32   |
|---------------------|-----------------------|----------------------------|

The first column contains the offset of the table entry from the beginning of
the exception vector table in bytes. The second column contains the type of
the event and the last column describes the condition under which the block
is used.

All that remains to do is to tell the processor where the table begins. The
address of the beginning of the table for the EL2 is stored in `VBAR_EL2`
register.

In the repository I put the exception vector table to the `bootstrap/ints.S`
file and it starts like this:

```asm
.text
.global vector_table
.balign 2048
vector_table:
    b .
.balign 0x80
    b .
.balign 0x80
    b .
.balign 0x80
    b .

...
```

It's worth clarifing a little bit what different parts there mean. The
`vector_table` label marks the beginning of the exception vector table. The
beginning of the exception vector table must be aligned on 2048 bytes
boundary and that's what the `.balign 2048` assembler directive says.

Each entry is 128 bytes long and that's what we guarantee using `.balign 0x80`
directive. Each entry of the first block contains just one instruction `b .`.
This instruction is just an infinite loop - it transfer control to itself.

I'm using an infinite loop for the first block because it should never be
called. The reason for that is because in the [previous post] I configured the
system to use a dedicated stack pointer register, so we never actually use
`SP_EL0`.

The third and forth block look the same as the first one. As for the the
second block, the only block we care about for now, I will cover it below.

The code writing the `VBAR_EL2` register is located in `bootstrap/start.S`
file and looks like this:

```asm
adr x0, vector_table
msr VBAR_EL2, x0
```

The `adr` instruction takes a relative address of a symbol (`vector_table` in
this case) and calculates the absolute address using the instruction pointer.
TL; DR it writes the absolute address of the `vector_table` into the x0
register.

Then we use `msr` instruction to write the value in `x0` to the `VBAR_EL2`
system register. That completes our configuration.

# Interrupt Handlers

Now let's take at the actual interrupt handling. In this post I will not
cover what should we actually do to handle the exceptions and interrupts.
Instead I will just assume that we have a couple of functions: `exception`
and `interrupt`.

Those functions will magically do the right thing as long as we pass them
enough information. For now for testing we can just print something inside
those functions and be done with it. I will cover that in the testing section
below.

Let's return back to the exception vector table. We only care about the
second block of the exception vector table and here is how it looks in my
case:

```asm
.balign 0x80
    b exception_entry
.balign 0x80
    b interrupt_entry
.balign 0x80
    b interrupt_entry
.balign 0x80
    b exception_entry
```

For now I don't diffirentitate between synchronous exceptions and SError,
which is likely not a right thing to do. I do differentiate between the
exceptions and interrupts however. At this point there will be very little
difference between thouse though.

The exception vector table entries above just pass the control to the
`exception_entry` and `interrupt_entry` labels that we will see below. This
way we are not affected by the limit on the size of the entry in the
exception vector table as the code that will do the actual handling is
outside of the exception vector table.

Let's start from the `exception_entry` and we will build it step by step.
Doing the same for the `interrupt_entry` will be simple after that since both
will have a lot of similarities.

We know that the interrupt handler should use `eret` instruction to finish
the interrupt handler and return the control back to the interrupted code:

```asm
exception_entry:
    eret
```

As was explained above it's on the interrupt handler to make sure that all
the state of the interrupted code has been preserved, so we need to preserve
the state of the general purpose registers that the interrupt handler will
corrupt at least.

We will use a magic `exception` function that will do the actual handling and
we don't know what registers it will use. So we need to preserve all of them.
I will save the registers on stack and before finishing the interrupt handler
I will pop everything from the stack. This way stack state will return to its
original state.

Let's first reserve place on stack by substracting a large enough value from
the stack pointer and then return it back:

```asm
exception_entry:
    sub sp, sp, #192
    add sp, sp, #192
    eret
```

You will see below why I'm reserving 192 bytes for storing the information we
need to store.

To actually store information on stack I will use the `stp` instruction
(store pair). This instruction takes two registers and stores their value at
the address pointer by the third argument.

To restore the values from stack I will use the `ldp` instruction (load
store). That instruction is basically the opposite of the `stp` instruction.

Let's take a look:

```asm
exception_entry:
    sub sp, sp, #192
    stp x0, x1, [sp, #0]
    stp x2, x3, [sp, #16]
    stp x4, x5, [sp, #32]
    stp x6, x7, [sp, #48]
    stp x8, x9, [sp, #64]
    stp x10, x11, [sp, #80]
    stp x12, x13, [sp, #96]
    stp x14, x15, [sp, #112]
    stp x16, x17, [sp, #128]
    stp x18, x29, [sp, #144]
    stp x30, xzr, [sp, #160]

    ldp x0, x1, [sp, #0]
    ldp x2, x3, [sp, #16]
    ldp x4, x5, [sp, #32]
    ldp x6, x7, [sp, #48]
    ldp x8, x9, [sp, #64]
    ldp x10, x11, [sp, #80]
    ldp x12, x13, [sp, #96]
    ldp x14, x15, [sp, #112]
    ldp x16, x17, [sp, #128]
    ldp x18, x29, [sp, #144]
    ldp x30, xzr, [sp, #160]
    add sp, sp, #192
    eret
```

I'm saving registers `x0` to `x18`, plus `x29` and `x30`. The `xzr` is a fake
register containing 0 value, I'm using it just because without that I'd have
an odd number of registers, which doesn't play well with the fact that `stp`
instruction needs a pair of registers and the fact that the stack have to be
aligned on the 16 byte boundary.

The `x30` register is so called link register. In ARM when calling a function
the address to return is stored in the `x30` link register. It's very similar
in function to the `ELR` register, but for regular functions instead of
interrupt handlers.

The `x29` register is commonly used as a frame pointer, but from architecture
point of view it doesn't have any special meaning.

At this point you may wonder about all the registers from `x19` to `x28` that
we didn't store?

To answer that question we need to talk a little bit about calling
conventions. It's common for calling conventions to separate registers to
callee-saved and caller-saved registers.

To understand the difference between them and how they are relevant lets
consider an example. Let's say we have functions `foo` and `bar`. And let's
also say that the function `foo` calls the function `bar`.

If the function `bar` uses any of the callee-saved registers it's the function
`bar` responsibility to make sure that it restores them to their original
value. It's sort of a contract between the caller and the callee.

On the other hand, if the function `foo` cares about any values in the
caller-saved registers, then before calling the function `bar` it's the
function `foo` responsibility to save them somehow before calling `bar`.

Returning to the interrupt handling, if the function `exception` follows the
calling convention, then we only need to preserve the caller-saved registers
and the `exception` function will save the callee-saved registers, so we
don't have to save them.

In AArch64 calling convention registers from `x0` to `x18` are caller-saved.
Due to the nature of the `x30` link register, calling a function will
necessarily corrupt the value there, so if we care about its value it's on us
to preserve it as well.

Strictly speaking I didn't have to preserve `x29` register as it's a
callee-saved register, but I though that it would be an interesting value to
look at inside the exception handler, so I saved it on the stack. Following
that logic you may want to go and save all the registers if want.

The next step is to save the information required to understand what caused
the exception. I will not dig into a lot of details, I will just say that
there are coupld of registers of interest here: `ESR_EL2` and `FAR_EL2`:

```asm
exception_entry:
    sub sp, sp, #192
    stp x0, x1, [sp, #0]
    stp x2, x3, [sp, #16]
    stp x4, x5, [sp, #32]
    stp x6, x7, [sp, #48]
    stp x8, x9, [sp, #64]
    stp x10, x11, [sp, #80]
    stp x12, x13, [sp, #96]
    stp x14, x15, [sp, #112]
    stp x16, x17, [sp, #128]
    stp x18, x29, [sp, #144]
    stp x30, xzr, [sp, #160]

    mrs x0, ESR_EL2
    mrs x1, FAR_EL2
    stp x0, x1, [sp, #176]

    ldp x0, x1, [sp, #0]
    ldp x2, x3, [sp, #16]
    ldp x4, x5, [sp, #32]
    ldp x6, x7, [sp, #48]
    ldp x8, x9, [sp, #64]
    ldp x10, x11, [sp, #80]
    ldp x12, x13, [sp, #96]
    ldp x14, x15, [sp, #112]
    ldp x16, x17, [sp, #128]
    ldp x18, x29, [sp, #144]
    ldp x30, xzr, [sp, #160]
    add sp, sp, #192
    eret
```

Slowly we came to the final step. Now it's time to call the `exception`
function and pass it the parameters it needs. I'm just going to pass it a
pointer to the data we saved on stack (that's why I saved the values of
`ESR_EL2` and `FAR_EL2` on stack).

Simple data types like pointers in AArch64 calling convention are passed to
functions in registers. The first parameter is passed in the `x0` register.
Putting all things together here is what we get:

```asm
exception_entry:
    sub sp, sp, #192
    stp x0, x1, [sp, #0]
    stp x2, x3, [sp, #16]
    stp x4, x5, [sp, #32]
    stp x6, x7, [sp, #48]
    stp x8, x9, [sp, #64]
    stp x10, x11, [sp, #80]
    stp x12, x13, [sp, #96]
    stp x14, x15, [sp, #112]
    stp x16, x17, [sp, #128]
    stp x18, x29, [sp, #144]
    stp x30, xzr, [sp, #160]

    mrs x0, ESR_EL2
    mrs x1, FAR_EL2
    stp x0, x1, [sp, #176]

    mov x0, sp
    bl exception

    ldp x0, x1, [sp, #0]
    ldp x2, x3, [sp, #16]
    ldp x4, x5, [sp, #32]
    ldp x6, x7, [sp, #48]
    ldp x8, x9, [sp, #64]
    ldp x10, x11, [sp, #80]
    ldp x12, x13, [sp, #96]
    ldp x14, x15, [sp, #112]
    ldp x16, x17, [sp, #128]
    ldp x18, x29, [sp, #144]
    ldp x30, xzr, [sp, #160]
    add sp, sp, #192
    eret
```

Now let's take how the `exception` function signature may look. I will use
Rust, but in C you will get something very similar:

```rust
#[derive(Copy, Clone, Debug)]
#[repr(C)]
pub struct InterruptFrame {
    x0: u64,
    x1: u64,
    x2: u64,
    x3: u64,
    x4: u64,
    x5: u64,
    x6: u64,
    x7: u64,
    x8: u64,
    x9: u64,
    x10: u64,
    x11: u64,
    x12: u64,
    x13: u64,
    x14: u64,
    x15: u64,
    x16: u64,
    x17: u64,
    x18: u64,
    fp: u64,
    lr: u64,
    xzr: u64,
    esr: u64,
    far: u64,
}

#[no_magle]
pub extern "C" fn exception(frame: *mut InterruptFrame) {
    unsafe { safe_exception(&mut *frame) }
}

fn safe_exception(_frame: &mut InterruptFrame) {
    loop {}
}
```

> *NOTE:* I used mutable pointers and references above because we actually
  may want to change the values of some registers. Imagine the case of system
  calls, while they are handled in the general exception and interrupt
  handling framework, it's not quite true that the interrupted code isn't
  preprapared. On the countrary, the system calls are caused intentionally and
  we may want to be able to return some results from a system call. In order
  to do that we may want to modify a few registers.

And we are almost done. The difference between the `exception_entry` and
`interrupt_entry` is that `ESR_EL2` and `FAR_EL2` don't make any sense for
interrupts. So we can save `xzr` instead.

The code you can find in the repository is slightly different, but it has the
same idea. The difference is due to a case of unaligned stack pointer that
I want to handle there. You can check the comments to understand the situation
there.

# Masking and Unmasking Interrupts

One final point related to the interrupt handling that is worth covering is
masking and unmasking interrupts. Sometimes we may want to delay handling of
interrupts, because we want to do something else, or because we just don't
want a particular code to be interrupted, because it's unsafe to interrupt.

In this case we have an option of masking interrupt. That is instruct the
processor to not take any interrupts. When we do that, the processor just
takes a note that there is a signal waiting processing, but doesn't actually
call the interrupt handler. The processor will call the interrupt handler
once interrupts are unmasked again.

Naturally, it doesn't make sense to mask all the exceptions. Again consider
the case of illegal instruction situation covered above. However we can mask
some of them, specifically, SError and Debug exceptions. I didn't talk about
the debug exception, but we can largely ignore them for now.

To mask interrupts we can use [`DAIF`](https://developer.arm.com/docs/ddi0595/b/aarch64-system-registers/daif)
system register (you will see where the name comes from below). It has 4 bits
dedicated to interrupt masks:

* `D` or bit 9 - when it's set to 1, debug exceptions are masked;
* `A` or bit 8 - when it's set to 1, SError is masked;
* `I` or bit 7 - when it's set to 1, IRQs are masked;
* `F` or bit 6 - when it's set to 1, FIQs are masked.

To mask all the interrupts all it takes is to wtite 1s in those bits:

```asm
    msr DAIFSet, #0b1111
```

and to unmask you can use this snippet:

```asm
    msr DAIFClr, #0b1111
```

You can also read and write the `DAIF` register directly as we did with other
system registers using `msr` and `mrs` instruction and the register name
`DAIF` instead of `DAIFSet` and `DAIFClr`. However for now we just want to
be able to mask all the interrupts until we are ready to handle them.

# Testing

Now it's time to play with it. I have all the interrupts masked in the code,
but as I explained before, we cannot mask all the exceptions. So what I'm
going to do is to cause an exception to see how our logic works.

Before that however, let's tweak our exception handler to actually see
something when an exception happens:

```rust
#![no_std]
#[macro_use]
extern crate alloc;

use pl011::PL011;

fn safe_exception(frame: &mut InterruptFrame) {
    let serial = PL011:new(
        /* base_address = */0x9000000,
        /* base_clock = */24000000);
    serial.send(format!("{:?}", frame).as_str());
    loop {}
}
```

So every time when the exception happens, we will output the content of the
`InterruptFrame` structure via a serial port and hang in an infinite loop.

> *NOTE:* the serial port code has been covered in posts
  [1]{% post_url 2020-11-29-PL011 %}, [2]{% post_url 2020-12-05-HiKey960 %}
  and [3]{% post_url 2020-12-13-adding-rust-to-aarch64 %}.

> *NOTE:* this way of communicating with hardware is somewhat unsafe in
  general. Consider the case where we have concurrently executing code that
  uses the PL011 as well. The moral here is that it doesn't matter what safety
  features the language provides, the architecture and the implementation of
  the system still have to be correct. That applies to Rust to the same extent
  as it applies to C and C++ - there is no such thing as safe language by
  deafult.

Now it's time to generate some exception to see if our code works. There are
plenty of ways to generate an exception, here is the way I used:

```c
...

static volatile int cont = 1;

...

void main(struct data *data, size_t size)
{
    ...

    // This tries to read an address that just cannot exist - a problem that
    // processor cannot ignore, so it must result in an exception.
    cont = *((volatile int *)0xffffffffffffffffull);

    ...
}
```

When I start the QEMU as a result of this I see this in the output:

```
Setting up the loader...
Loading the config...
Parsing the config...
Loading the kernel...
Loading the data...
Starting the kernel...
Shutting down UEFI boot services
Bootstraping...
InterruptFrame { x0: 0, x1: 1077649408, x2: 3, x3: 1200091392, x4: 1200091464, x5: 4, x6: 1199981932, x7: 12, x8: 18446744073709551615, x9: 1075548168, x10: 1075552272, x11: 1, x12: 4, x13: 8, x14: 0, x15: 0, x16: 1200020096, x17: 4294944427, x18: 0, fp: 1199981984, lr: 1075278384, sp: 1199981936, esr: 2542272528, far: 18446744073709551612 }
```

You can see the content of the `InterruptFrame` structure that was saved by
the `exception_entry` and passed to the `exception` function as an argument.

Another way we can look is by using the QEMU monitor. You can read
[this post]{% post_url 2020-12-26-position-independent-executable %} that
covers some debugging tools includeing QEMU monitor.

Assuming that you know how to connect to the QEMU monitor, you can use the
following command to enable logging of exceptions and iterrupts:

```
(qemu) log int
```

When you do that QEMU will start logging to stdout all the exceptions and
interrupts happening in the system (it might generate a lot of output). It
looks like this in my case:

```
Taking exception 5 [IRQ]
...from EL2 to EL2
...with ESR 0x0/0x0
...with ELR 0x43b9f01c
...to EL2 PC 0x43b34a80 PSTATE 0x3c9
Exception return from AArch64 EL2 to AArch64 EL2 PC 0x43b9f01c
```

When I run my code and it eventually generates an exception this is what I see
there:

```
Taking exception 4 [Data Abort]
...from EL2 to EL2
...with ESR 0x25/0x97880010
...with FAR 0xfffffffffffffffc
...with ELR 0x40177234
...to EL2 PC 0x40178200 PSTATE 0x3c9
```

The system is now hanging in an infinite loop, so we can connect to it with
GDB and see where it spins, to confirm that it indeed happens inside the code
we wrote.

# Instead of conclusion

Hopefully this post is a hands on example of interrupt and exception handling
in AArch64. And hopefully it's detailed enough to understand how various
pieces fit together.

I'm starting to accumulate a noticable amount of code in the repository, so
I had to resort to posting snippets in the post instead of the complete code.
If you find some parts confusing or incomplete, feel free to suggest
improvements.
