---
layout: post
title: Debugging AArch64 using QEMU and GDB
excerpt_separator: <!--more-->
tags: clang aarch64 qemu rust gdb ld assembly format!
---

[previous post]: {% post_url 2020-12-13-adding-rust-to-aarch64.md %} "the previous post"

In the [previous post] I added Rust to the project and since then I was
experimenting with parsing DeviceTree, however while doing that I stumbled
on a mistery problem.

In this post I will cover the background that lead to the problem,
investigations and finally the solution. There isn't terribly a lot of code
related to this post, but nevertheless all the sources are available on
[GitHub](https://github.com/krinkinmu/aarch64).

<!--more-->

# Background

After setting up basic Rust infrstructure for the project I wanted to start
working with DeviceTree. DeviceTree describes the system hardware and
configuration to some extent. Firmware is expected to pass the DeviceTree to
the OS kernel and the OS kernel can use it to understand what kind of hardware
the OS is running on:

* how much memory does it have
* how many compute units it has (CPU or other kinds)
* what devices are connected to the system (timers, serial ports, host
  controllers for things like PCI and USB)
* etc.

So basically unless you want to hardcode the hardware configuration in the
kernel code, it's quite useful piece of information.

While working on the DeviceTree parsing I came to the conclusion that to
provide a convenient interface from that library I might need a dynamic
memory allocator.

So I implemented a simplistic memory allocator, specficailly to be used only
during early stages of the setup process. However, while playing with it in
Rust I found out that things don't quite work reliably and that's where
the story begins...

# The Problem

After plugging in the dynamic memory allocator to the Rust code I played a
little bit with Rust `alloc::vec::Vec` and it worked just fine. So I was
ready to celebrate the vicotry and write another post on dynamic memory
allocation.

However out of curiousity I went to look at what's else available in the
`alloc` that might come in handy. One thing that I found is the `format!`
macro. `format!` macro is a `printf`-likeish Rust macro to for string
interpolation. I though that it might be quite useful for logging purporses
and decided to try it.

When I loaded a newly built binary with some `format!` invocations to see how
it works, things broke and QEMU reported that the emulated system caught an
exception.

Adding and removing `alloc::vec::Vec` and `format!` confirmed that the
problem somehow manifests only when I use `format!`, but using
`alloc::vec::Vec` doesn't result in any problems I could see.

Interesting... It's time to debug...

# Hypothesis 1: bad memory allocator

It's weird that the problem only manifested when I use `format!`, but not
when I use `alloc::vec::Vec`, but memory related bugs are often claimed to
be a sort of a mistery: hard to test to prevent them, hard to understand
what's going once you hit them, and hard to debug to figure out where the
problem is. So I didn't really give much thought to the difference in
behavior between `format!` and `alloc::vec::Vec` and went after the memory
allocator.

To quickly scope the problem with the memory allocator I figured there are
roughly two options:

* the allocator code itself is somehow broken, so we hit a problem somehwere
  inside the allocator code
* the allocator returns somehow incorrect memory pointer to the caller.

To narrow it down, I added logging to the allocator code. Logging was
supposed to help with the first possibility when the allocator code itself
is buggy. Logging would help to understand in what part of the code do we
detect the problem.

To help with the second possibility I added a `memset` call to the allocation
functions to zero-out the returned memory range before returning the pointer
to the user. If the returned memory range is somehow bad, writing something
there might reveal it.

After this exercise I was almost certain that:

* the problem is triggered somewhere in the Rust code and not in the
  allocator itself
* the memory the allocator returns is correct.

So all in all, the allocator and dynamic memory allocation didn't seem to
cause this particular problem and I need to look somewhere else.

# Hypothesis 2: `format!` is bad

After ruling out memory allocator as a culprit I started to look for
something problematic about the `format!` macro. After all without using
`format!` the problem doesn't manifest.

Turned out that the `format!` macro itself is super uninteresting and all the
magic happens inside the `format_args!` macro. However trying to look at the
implementation of `format_args!` I found just this:

```rust
#[stable(feature = "rust1", since = "1.0.0")]
#[allow_internal_unstable(fmt_internals)]
#[rustc_builtin_macro]
#[macro_export]
macro_rules! format_args {
    ($fmt:expr) => {{ /* compiler built-in */ }};
    ($fmt:expr, $($args:tt)*) => {{ /* compiler built-in */ }};
}
```

The way I read it, is that `format_args!` is sort of a compiler magic or an
intrinsic function. In other words it's handled by the compiler in a special
way and if I wanted to understand what's happening there I'd have to look at
the Rust compiler code. I wasn't ready to do that, so I had to look somehwere
else.

> *NOTE:* That's interesting how Rust creators decided to go with a compiler
  magic to solve a problem of string interpolation. In C++ for example it's
  rather simple to achive type-safety and flexibility of macroses using
  variadic templates for this particular problem. However, even without fancy
  variadic templates, in C++ you can use `std::stringstream` and co to solve
  the same problem in a straighforward way, without sacrifacing type-safety
  (as was the case in C with their `printf`-like functions) and at the same
  time providing a convenient interface. Personally, I don't think that there
  was any need for compiler magic here.

I didn't want to investigate the `format_args!` further and it wasn't even
clear if the `format_args!` is the problem here. After all, `format_args!`
just generates some code and `format_args!` implementation is only
interesting as a way to find out what code it is. So we can probably find
some other means to fit the pieces together.

> *NOTE:* in retrospective though and spoiling the mistery a little bit, I
  should probably have looked a little bit bitter into what `format_args!`
  does.

According to the `format_args!` description it just generates a value of type
`fmt::Arguments`. The `format!` macro uses it as an argument for the
`fmt::format` function. So we can take a look at what the `fmt::format`
function does and how it uses `fmt::Arguments` value.

```rust
#[stable(feature = "rust1", since = "1.0.0")]
pub fn format(args: Arguments<'_>) -> string::String {
    let capacity = args.estimated_capacity();
    let mut output = string::String::with_capacity(capacity);
    output.write_fmt(args).expect("a formatting trait implementation returned an error");
    output
}
```

So what's going on here? Well, actually nothing interesting, the function
creates an object of type `String`, trying to preallocate enough memory to
fit in the results of the string interpolation, but then the actual string
interpolation happens in the function `write_fmt` that is part of the `Write`
trait implemented for `String`.

Moreover, `String` uses a generic implementation of `write_fmt` that
ultimately just calls `fmt::write` function. The `fmt::write` function takes
an output buffer (something that implemenets `Write` trait) and the
arguments (the thing that `format_args!` macro creates) and performs the
actual string interpolation (delegating a bunch of stuff to other functions).

From here we have a few options:

* maybe allocating memory for the String is what causing the problem
* maybe the problem happens during the string interpolation.

Another way to look at the current state is that, if memory allocation for
`String` is not a problem, then the problem is definitely happening somehwere
in the deeps of the Rust standard library.

I did check before that memory allocation is not the problematic part here.
However, just to be sure before moving forward, I double checked that memory
allocation for the `String` is not really a problem here.

So what's now? At this point I was somewhat at loss and needed some time to
think of a way to move forward. In the meantime while considering throwing
havier tools at the problem I went to try some random guesses.

# Hypothesis 3: running out of stack

One randome hypothesis that I actually verified was running out of stack. The
way the binary has been loaded it was using the stack that UEFI firmware gave
to the UEFI application that loaded the binary. I didn't really know neither
where the stack was located, how much stack we had or how much stack we
already used. Additional fuel to the fire of this hypothesis was this
question:
[Excessive stack usage for formatting](https://users.rust-lang.org/t/excessive-stack-usage-for-formatting/8023).

First step on exploring this hypothesis was to check the UEFI specification.
And indeed in Chapter 2 Overview, Section 2.3 Calling Conventions and
Subsection 2.3.6 AArch64 Platforms of the UEFI specification it's explicitly
said that UEFI application should have at least 128KiB of stack available.

At this state I was already confident that stack is not a problem and there
is no way my code could have used 128KiB. That being said, I still went to
look at the generated assembler code to do a rough estimate on how much stack
we might use.

For example, here are a first few instructions of the `main` function:

```sh
$ llvm-objdump -d kernel.elf | grep -A10 'main:'
0000000000001028 main:
    1028: ff c3 01 d1                  	sub	sp, sp, #112
    102c: f4 4f 06 a9                  	stp	x20, x19, [sp, #96]
    1030: f3 03 01 aa                  	mov	x19, x1
    1034: f4 03 00 aa                  	mov	x20, x0
    1038: 08 02 00 b0                  	adrp	x8, #266240
    103c: fd 7b 03 a9                  	stp	x29, x30, [sp, #48]
    1040: f8 5f 04 a9                  	stp	x24, x23, [sp, #64]
    1044: f6 57 05 a9                  	stp	x22, x21, [sp, #80]
    1048: fd c3 00 91                  	add	x29, sp, #48
    104c: 09 01 40 b9                  	ldr	w9, [x8]
```

The first instruction reduces the stack pointer by 112 bytes, so that's what
I figured is the stack frame size for the `main` function. Similarly we can
check the Rust code entry point:

```sh
$ llvm-objdump -d kernel.elf | grep -A10 'start_kernel:'
0000000000001860 start_kernel:
    1860: ff 83 03 d1                  	sub	sp, sp, #224
    1864: 01 c0 86 52                  	mov	w1, #13824
    1868: e8 23 00 91                  	add	x8, sp, #8
    186c: 00 20 a1 52                  	mov	w0, #150994944
    1870: c1 2d a0 72                  	movk	w1, #366, lsl #16
    1874: fd 7b 08 a9                  	stp	x29, x30, [sp, #128]
    1878: fc 6f 09 a9                  	stp	x28, x27, [sp, #144]
    187c: fa 67 0a a9                  	stp	x26, x25, [sp, #160]
    1880: f8 5f 0b a9                  	stp	x24, x23, [sp, #176]
    1884: f6 57 0c a9                  	stp	x22, x21, [sp, #192]
```

With other Rust functions it might be more complciated because of the name
mangling, but poking around into a few of them, I figured that there is no
way we could have used all the 128KiB this way.

If you see at the functions above they use just a few hundreds bytes for
stack. If we round it up to 1KiB of stack on average per function, then we'd
need a call chain of around 128 functions to exhaust the stack - that's
quite a lot for a program without recursion.

# Deploying Heavy Tools

As a last ditch effort in the pursue of the stack hypothesis and also as an
attempt to start debugging the problem more systematically I decided to
deploy QEMU monitor and GDB.

I was going to start with checking where the stack of the program is located
and whether we indeed had 128KiB of stack available. In order to do that I
made this modification to the C code:

```c
static volatile int cont = 0;

void main(...)
{
    /* Here go variable declarations that are not important */

    while (!cont);

    /* The rest of the main implemnetation goes after */
}
```

This is essentially an infinite loop. The idea is for the program to hang
once it reaches `main`, so while it's inside the infinite loop I can
investigate the state of registers and memory either using QEMU monitor
interface or via GDB.

The `volatile int cont` serves a few purporses. First of all, the compiler
looking at this code cannot conclude that it's an infinite loop and consider
the rest of the code unreachable, since it has to read the value of `cont`
periodically as it's part of the observable behavior of the program thanks to
the `volatile` part. So the program execution will hang in that infinite
loop, but the compiler cannot optimize the rest of the code away.

The second reason to have this variable is that using GDB we can actually
overwrite its value. So the program will hang in an infinite loop right after
start, but then we can connect to it with GDB, setup a few breakpoints and
break the infinite loop by directly modifing the value of `cont` from GDB.
So in other words, we have a way to exit the loop and continue the execution
when we want to by changing the value of the variable during runtime.

> *NOTE:* I suspect that from GDB we could just directly write program
  counter and jump to whatever instruction we want and break the infinite
  loop this way, but I didn't try.

Finally, when I'm not actively debugging anything I can just set `cont = 1`
in the code to disable this mechanism by default. So it's a relatively easy
hack for debugging that I can leave in the code permanently (or until I have
a better tool instead).

So how can we use QEMU monitor and GDB for debugging? Let's take a look at
the script I use to start QEMU:

```sh
#!/bin/bash

QEMU="qemu-system-aarch64 -machine virt -cpu max"
OVMF="/home/kmu/ws/aarch64/OVMF.fd"

cp /home/kmu/ws/aarch64/src/kernel.elf /home/kmu/ws/aarch64/root/efi/boot/kernel
cp /home/kmu/ws/aarch64/virt.dtb /home/kmu/ws/aarch64/root/efi/boot/virt.dtb
cp /home/kmu/ws/aarch64/config.txt /home/kmu/ws/aarch64/root/efi/boot/config.txt

${QEMU} \
  -drive if=pflash,format=raw,file=${OVMF} \
  -drive format=raw,file=fat:rw:/home/kmu/ws/aarch64/root \
  -net none \
  -serial telnet:localhost:1234,server \
  -monitor telnet:localhost:1235,server,nowait \
  -nographic
```

Pay attention to the `-serial` and `-monitor` flags. `-serial` is where all
the output of the program goes, since I (and UEFI firmware implementation
that QEMU uses) use serial port for output. I can connect using telnet to the
`localhost:1234` to see the output.

Similarly, I can use telnet and connect to `localhost:1235` to get access to
the QEMU monitor. QEMU monitor is a shell that allows to explore the current
state of the virtual machine including the state of registers, so it should
be enough for us to get an idea of where the stack lives.

We can take it one step further and also tell QEMU to setup a GDB server for
the virtual machine by adding a `gdb` flag to the command invocation:

```sh
#!/bin/bash

QEMU="qemu-system-aarch64 -machine virt -cpu max"
OVMF="/home/kmu/ws/aarch64/OVMF.fd"

cp /home/kmu/ws/aarch64/src/kernel.elf /home/kmu/ws/aarch64/root/efi/boot/kernel
cp /home/kmu/ws/aarch64/virt.dtb /home/kmu/ws/aarch64/root/efi/boot/virt.dtb
cp /home/kmu/ws/aarch64/config.txt /home/kmu/ws/aarch64/root/efi/boot/config.txt

${QEMU} \
  -drive if=pflash,format=raw,file=${OVMF} \
  -drive format=raw,file=fat:rw:/home/kmu/ws/aarch64/root \
  -net none \
  -serial telnet:localhost:1234,server \
  -monitor telnet:localhost:1235,server,nowait \
  -gdb tcp::1236 \
  -nographic
```

However keep in mind, that virtual machine emulates aarch64 architecture, so
we need GDB that understands it:

```sh
$ sudo apt-get gdb-multiarch
```

Now, we can try and use those tools. After going through the normal UEFI
loading process and finally starting my binary it hanged at the beginning of
main function - working as expected.

Now we can connect to the QEMU monitor like this:

```sh
$ telnet localhost 1235
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
QEMU 4.2.1 monitor - type 'help' for more information
(qemu) 
```

To get the state of registers inside the virtual machine we can use the
`info` command:

```sh
(qemu) info registers
 PC=000000004017704c X00=000000004050f098 X01=0000000000000004
X02=00000000401b4da8 X03=000000004787f100 X04=000000004787f148
X05=0000000000000004 X06=000000004786456c X07=000000000000000c
X08=00000000401b8000 X09=0000000000000000 X10=0000000040530000
X11=0000000000000003 X12=0000000042aab2f0 X13=0000000000000008
X14=0000000000000000 X15=0000000000000000 X16=000000004786da80
X17=00000000ffffa6ab X18=0000000000000000 X19=0000000000000004
X20=000000004050f098 X21=000000004048b000 X22=000000004042fe84
X23=00000000478649c8 X24=000000004042c000 X25=0000000000000000
X26=000000004149b918 X27=000000004048b000 X28=000000004148f818
X29=00000000478645b0 X30=000000004017701c  SP=0000000047864580
PSTATE=600003c5 -ZC- EL1h  BTYPE=0     FPCR=00000000 FPSR=00000000
Q00=0000000000000000:0000000000000000 Q01=0000000000000000:0000000000000000
Q02=0000000000000000:0000000000000000 Q03=0000000000000000:0000000000000000
Q04=0000000000000000:0000000000000000 Q05=0000000000000000:0000000000000000
Q06=0000000000000000:0000000000000000 Q07=0000000000000000:0000000000000000
Q08=0000000000000000:0000000000000000 Q09=0000000000000000:0000000000000000
Q10=0000000000000000:0000000000000000 Q11=0000000000000000:0000000000000000
Q12=0000000000000000:0000000000000000 Q13=0000000000000000:0000000000000000
Q14=0000000000000000:0000000000000000 Q15=0000000000000000:0000000000000000
Q16=0000000000000000:0000000000000000 Q17=0000000000000000:0000000000000000
Q18=0000000000000000:0000000000000000 Q19=0000000000000000:0000000000000000
Q20=0000000000000000:0000000000000000 Q21=0000000000000000:0000000000000000
Q22=0000000000000000:0000000000000000 Q23=0000000000000000:0000000000000000
Q24=0000000000000000:0000000000000000 Q25=0000000000000000:0000000000000000
Q26=0000000000000000:0000000000000000 Q27=0000000000000000:0000000000000000
Q28=0000000000000000:0000000000000000 Q29=0000000000000000:0000000000000000
Q30=0000000000000000:0000000000000000 Q31=0000000000000000:0000000000000000
```

As you can see the program counter contains address `0x4017704c` and the
current value of the stack pointer is `0x47864580`. Note that by convention
the stack on AArch64 grows toward the lower addresses. So there is over
100MiB between the stack pointer the the current program counter.

With the size of the program in memory at around 2MiB, there is no way that
stack might collide with the rest of the program data and code.

Can there be something else? Pretty unlikely. If we look at the DeviceTree
for the virtual machine hardware we can find there this node describing
memory of the system:

```
memory@40000000 {
    reg = <0x00 0x40000000 0x00 0x8000000>;
    device_type = "memory";
};
```

So the physical memory available to the system spans from the address
`0x40000000` (which is pretty close to where our program resides in memory)
until `0x48000000` (which is pretty close to where the stack of the program
lives). There is no holes in between them and there shouldn't be anything
else of importance running in the system for us to corrupt it.

# GDB

Now let's cover how to connect to the virtual machine using GDB for
completness even though we didn't really need to do it to understand where
the stack is. It's quite easy to do:

```sh
$ gdb-multiarch 
GNU gdb (Ubuntu 9.2-0ubuntu1~20.04) 9.2
Copyright (C) 2020 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.
Type "show copying" and "show warranty" for details.
This GDB was configured as "x86_64-linux-gnu".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
    <http://www.gnu.org/software/gdb/documentation/>.

For help, type "help".
Type "apropos word" to search for commands related to "word".
(gdb) target remote localhost 1236
localhost 1236: No such file or directory.
(gdb) target remote localhost:1236
Remote debugging using localhost:1236
warning: No executable has been specified and target does not support
determining executable automatically.  Try using the "file" command.
0x000000004017704c in ?? ()
(gdb) 
```

As soon as we connect with GDB to the virtual machine, the virtual machine
will stop its execution. Not surprising the program counter where the virtual
machines was when I connected to it with GDB turned out to be `0x4017704c` -
exactly the same address we saw before in QEMU monitor. So this address is
likely somehwere within the infinite loop I created inside the `main`
function.

We can actually see the instruction at the current address like this:

```
(gdb) display/i $pc
1: x/i $pc
=> 0x4017704c:  ldr     w9, [x8]
```

Similarly we can look at, for example, next and previous instructions using
`$pc + 4` and `$pc - 4`. That's because on aarch64 the instructions are 4
bytes in size.

> *NOTE:* `display` command doesn't just print what you requested once, it
  will automatically print it every time when state changes, for example,
  when you go to the next instruction. To disable it you can use `undisplay`
  command. And if you want to just print something once there is `print`
  command, but you'd have to look at the documentation to understand how to
  use it.

Instead of asking GDB to print individual instructions you can use
`layout asm` to see a bit more context. I'm not going to cover GDB in more
details here and for further details will refer you to the documentation and
countless tutotrials on GDB available on the Internet.

So our program currently spins in the infinite loop that consists just from
two instructions:

```
0x4017704c  ldr w9, [x8]
0x40177050  cbz w9, 0x4017704c
```

Instruction `ldr` loads a value from memory to a register. The memory address
in our case is stored in register `x8` and the read value is stored in the
`x9` register (`w9` is name for lower byte of the `x9` register).

Instruction `cbz` is a conditional jump instruction. Basically, if the value
we read from memory is 0 it jumps to `0x4017704c` and otherwise moves to the
next instruction after `cbz`.

You can probably figure out from this that register `x8` contains the address
of the `cont` variable. In my case I found out that the register `x8`
contained value `0x401b8000`, hence that's where our `cont` variable is
stored.

Now if I wanted to break the infinite loop I could set a breakpoint somewhere
after the infinite loop and then overwrite the value at the address
`0x401b8000`:

```
(gdb) break *0x40177054
Breakpoint 1 at 0x40177054
(gdb) set *0x401b8000 = 1
```

If we tell GDB to continue the execution (using `c` command) it will read the
new value of `cont` and will exit the loop and stop at the breakpoint. In the
snippet above I set breakpoint at the next instruction after the loop, but it
could be any other address.

One thing to keep in mind is that *function addreses in the binary do not
match runtime address*. In other words, addresses that we can see in the
output of `objdump` aren't expected to match what we see in GDB. So when you
want to set a breakpoint on a function, you cannot just use the address you
found in the binary unchanged. Instead you need to calculate the difference
between the runtime addresses (addresses where the program was loaded or
runtime addresses) and the addresses recorded in the binary (linktime
addresses) and adjust the addresses accordingly.

For example, according to GDB the `ldr` instruction of the infinite loop was
at address `0x4017704c`, but according to the `objdump` the same instruction
has address `0x104c`. So the difference is `0x40176000`. When using `objdump`
we can correct for this difference using `--adjust-vma` flag:

```sh
$ llvm-objdump --adjust-vma=0x40176000 -d kernel.elf | grep -A10 'main:'
0000000040177028 main:
40177028: ff c3 01 d1                  	sub	sp, sp, #112
4017702c: f4 4f 06 a9                  	stp	x20, x19, [sp, #96]
40177030: f3 03 01 aa                  	mov	x19, x1
40177034: f4 03 00 aa                  	mov	x20, x0
40177038: 08 02 00 b0                  	adrp	x8, #266240
4017703c: fd 7b 03 a9                  	stp	x29, x30, [sp, #48]
40177040: f8 5f 04 a9                  	stp	x24, x23, [sp, #64]
40177044: f6 57 05 a9                  	stp	x22, x21, [sp, #80]
40177048: fd c3 00 91                  	add	x29, sp, #48
4017704c: 09 01 40 b9                  	ldr	w9, [x8]
```

As you can see once we told `objdump` about the difference between the
linktime and runtime addresses with the flag, it started displaying address
information correctly. So we know can use `objdump` to find the functions we
are interested in and their runtime addresses to set breakpoints in GDB.

> *NOTE:* runtime address, as you could have guessed, can change every time
  you load the program again, since it can be loaded at any available
  address.

# Finding where things break

Now when we have QEMU monitor and GDB at our disposal we can change our
strategy a little bit. Before I basically tried to guess where the problem
might be and explore the hypothesis. Some of the guesses had more merit and
some had less, but at the end of the day it's guessing. With GDB and QEMU
monitor at our disposal we can find out what broke exactly where.

One way to do it is to tell QEMU to show the exceptions in the QEMU monitor
and then let the program run normally until it breaks:

```
(qemu) log int
```

After this command the `qemu-system-aarch64` will start producing some output
messages that look like this:

```
Taking exception 5 [IRQ]
...from EL1 to EL1
...with ESR 0x0/0x0
...with ELR 0x4075fcac
...to EL1 PC 0x43b34a80 PSTATE 0x3c5
Exception return from AArch64 EL1 to AArch64 EL1 PC 0x4075fcac
```

It's a brief description of exceptions and interrupts happening in the
system. If we let the system run eventually it will hit that one exception
that is causing problems and show us some basic information about it:

```
Taking exception 3 [Prefetch Abort]
...from EL1 to EL1
...with ESR 0x21/0x86000007
...with FAR 0x0
...with ELR 0x0
...to EL1 PC 0x43b34a00 PSTATE 0x3c5
```

So what can we take from this meesage?

First of all the problematic execption is instruction prefetch (that's what
Prefetch Abort exceptions are about according to the ARM documentation). So
basically we tried to jump to an invalid address.

Both `FAR` and `ELR` contain zeros. `ELR` is normally the address where the
execution should return to after handling the exception and `FAR` contains
the address of the instruction that processor failed to fetch.

I'm not sure why we need both, since it appears that both of them should
point to the same place, but whatever the reason they both agree that we
tried to execute instruction with address 0.

So we know that we tried to jump to an instruction at the address 0, but we
still don't know where we jumped from. Unfortunately exploring register state
in QEMU monitor at this point is not very reliable. I didn't bother to setup
my own exception handlers, so existing exception handlers may have clobbered
the state of registers (specifcally `LR` aka `R30` might be helpful).

> *NOTE:* I was quite surprised to find out that there are some interrupts
  enabled in the system, those might cause problems down the road, so I was
  lucky to discover that early and will do something about that later.

To find from where the bad jump happened we can use GDB by single stepping
until the instruction that breaks. That's would very likely take quite a bit
of time, so I changed the code a little bit to remove all the irrelevant
parts of the Rust entry point down to this simple code:

```rust
#[no_mangle]
pub extern "C" fn start_kernel() {
    let msg = format!("haba-haba\n");
    let serial = PL011::new(
        /* base_address = */0x9000000,
        /* base_clock = */24000000);
    serial.send(msg.as_str());
}
```

So far we know that the problem is somehow related to the use of `format!`
macro, so it is the very first thing inside the `start_kernel` function. The
serial port related calls are there so that the result of the `format!` call
is actually used somehow, so compiler will not optimize it away.

To help with single stepping, we'd need to find the runtime address of the
`start_kernel` function and set a break point there. And from there it's all
hard manual work single stepping instruction after instruction paying
attention to the various jump instructions (including function calls and
function returns).

I will skip the boring description of how I was heroically instructing GDB
to execute instructions one at a time. Eventually I stumbled on this peculiar
snippet:

```
0x4017e5c4  ldp x0, x8, [sp, #32]
0x4017e5c8  add x9, x20, x22, lsl #4
0x4017e5cc  ldp x1, x2, [x9]
0x4017e5d0  ldr x8, [x8, #24]
0x4017e5d4  blr x8
```

Pay attention to the `blr x8` part. This instruction basically takes an
address from the register `x8` and calls a function at that address. And
checking the value in the `x8` register I found out that it's actually 0:

```
(gdb) print $x8
$2 = 0
```

Bingo!

This instruction happened to be inside a function with a peculiar name
`_ZN4core3fmt5write17h4f29494a403b7853E`. Which is, I suspect, just a mangled
name of the function we've encountered before in this post: `fmt::write`.
It's time to take a look at the function again:

```rust
#[stable(feature = "rust1", since = "1.0.0")]
pub fn write(output: &mut dyn Write, args: Arguments<'_>) -> Result {
    let mut formatter = Formatter {
        flags: 0,
        width: None,
        precision: None,
        buf: output,
        align: rt::v1::Alignment::Unknown,
        fill: ' ',
    };

    let mut idx = 0;

    match args.fmt {
        None => {
            // We can use default formatting parameters for all arguments.
            for (arg, piece) in args.args.iter().zip(args.pieces.iter()) {
                formatter.buf.write_str(*piece)?;
                (arg.formatter)(arg.value, &mut formatter)?;
                idx += 1;
            }
        }
        Some(fmt) => {
            // Every spec has a corresponding argument that is preceded by
            // a string piece.
            for (arg, piece) in fmt.iter().zip(args.pieces.iter()) {
                formatter.buf.write_str(*piece)?;
                run(&mut formatter, arg, &args.args)?;
                idx += 1;
            }
        }
    }

    // There can be only one trailing string piece left.
    if let Some(piece) = args.pieces.get(idx) {
        formatter.buf.write_str(*piece)?;
    }

    Ok(())
}
```

What's interesting in this function is that it calls a function by a pointer.
Take a look at this part specifically:

```rust
(arg.formatter)(arg.value, &mut formatter)?;
```

The function pointer ultimately comes from the `args` argument of the
`fmt::write` function. This argument is, as was discussed before, somehow
created by the `format_args!` macro.

So the picture so far is that `format_args!` creates a structure with a
function pointer inside and this function pointer somehow ends up being zero.
Let's take a look at how this `fmt::Arguments` structure is actually created.
And again, I'm not particularly keen on looking up the `format_args!`
implementation in the Rust compiler source code, so let's take the assembly
code of the `start_kernel` function:

```
0000000040177854 start_kernel:
40177854: ff 83 01 d1                   sub     sp, sp, #96
40177858: e8 01 00 b0                   adrp    x8, #249856
4017785c: 08 41 3a 91                   add     x8, x8, #3728
40177860: 29 00 80 52                   mov     w9, #1
40177864: 4a 01 00 b0                   adrp    x10, #167936
40177868: 4a 81 03 91                   add     x10, x10, #224
4017786c: e8 27 02 a9                   stp     x8, x9, [sp, #32]
40177870: e8 23 00 91                   add     x8, sp, #8
40177874: e0 83 00 91                   add     x0, sp, #32
40177878: fe 4f 05 a9                   stp     x30, x19, [sp, #80]
4017787c: ff 7f 03 a9                   stp     xzr, xzr, [sp, #48]
40177880: ea 7f 04 a9                   stp     x10, xzr, [sp, #64]
40177884: 2d 00 00 94                   bl      #180 <_ZN5alloc3fmt6format17h8bfac749fb37b0f7E>
```

That's the prefix of the `start_kernel` function up to the call to
`fmt::format` function. The `fmt::format` takes just one argument and it's
`fmt::Arguments` structure.

It's hard to tell for sure what calling convention Rust compiler uses
internally, but if I read it correctly this code prepares a bunch of data on
stack and potentially passes a pointer to that data in the `x0` register.

If we look at the structure that it prepares we can guess what code prepares
what parts of this structure:

```rust
#[stable(feature = "rust1", since = "1.0.0")]
#[derive(Copy, Clone)]
pub struct Arguments<'a> {
    pieces: &'a [&'static str],
    fmt: Option<&'a [rt::v1::Argument]>,
    args: &'a [ArgumentV1<'a>],
}
```

So the structure basically contains a few slices. Slices in Rust are
represented by two values: pointer to the data and the size. I don't know
how `Option` affects data representation, but assuming that it's sensibly
optimized `Option` can use the pointer inside the slice to represent
everything it needs. So all in all this structure should boild down to three
pairs of 8 byte integers.

On stack the structure is stored starting from the offset 32 from the value
of stack pointer `SP`. And we interested in the `args` field of the structure
that actually contains the problematic function pointer. So we need to look
at what this code generates at the offset 64 starting from the value in `SP`.

> *NOTE:* I definitely missing something here. If my interpretation is
  correct then starting from the offset 64 from `SP` the code should
  generate slice: pointer to the data and the number of elements. However if
  I'm reading it correctly the pointer is generated in the register `x10`,
  but surprisingly for the size of the slice the code uses `xzr` which is
  basically a zero register - register that always contains zero. That
  doesn't quite make sense.

The value in the register `x10` is generated using these two instructions:

```
40177864: 4a 01 00 b0                   adrp    x10, #167936
40177868: 4a 81 03 91                   add     x10, x10, #224
```

The `adrp` instruction takes the current value of `pc` register, adds the
given argument to that (which is in our case the value 167936) and then
aligns it to the 4KiB page boundary. The ARM documentation describes this
instruction as address of 4KiB page at a `PC`-relative offset.

`add` isntruction is easy to understand - it just adds 224 to the `x10`.

So if put all the pieces together these two instructions store `0x401a00e0`
in the register `x10`. The runtime address `0x401a00e0` corresponds to the
linktime address `0x2a0e0`. If we check the structure of our binary with
`readelf` we can find that this address belongs to the `.rodata` section:

```sh
$ readelf -S kernel.elf 
There are 18 section headers, starting at offset 0x6c320:

Section Headers:
  [Nr] Name              Type             Address           Offset
       Size              EntSize          Flags  Link  Info  Align
  [ 0]                   NULL             0000000000000000  00000000
       0000000000000000  0000000000000000           0     0     0
  [ 1] .text             PROGBITS         0000000000001000  00001000
       0000000000028084  0000000000000000  AX       0     0     4
  [ 2] .rodata           PROGBITS         000000000002a000  0002a000
       0000000000011b09  0000000000000000 AMS       0     0     4096
  [ 3] .eh_frame         PROGBITS         000000000003bb10  0003bb10
       0000000000000094  0000000000000000   A       0     0     8
  [ 4] .rela.dyn         RELA             000000000003bba8  0003bba8
       00000000000031e0  0000000000000018   A       5     0     8
  [ 5] .dynsym           DYNSYM           000000000003ed88  0003ed88
       0000000000000018  0000000000000018   A       8     1     8
  [ 6] .gnu.hash         GNU_HASH         000000000003eda0  0003eda0
       000000000000001c  0000000000000000   A       5     0     8
  [ 7] .hash             HASH             000000000003edbc  0003edbc
       0000000000000010  0000000000000004   A       5     0     4
  [ 8] .dynstr           STRTAB           000000000003edcc  0003edcc
       0000000000000001  0000000000000000   A       0     0     1
  [ 9] .dynamic          DYNAMIC          000000000003edd0  0003edd0
       00000000000000c0  0000000000000010  WA       8     0     8
  [10] .data.rel.ro      PROGBITS         000000000003ee90  0003ee90
       0000000000002548  0000000000000000  WA       0     0     8
  [11] .got              PROGBITS         00000000000413d8  000413d8
       0000000000000048  0000000000000000  WA       0     0     8
  [12] .data             PROGBITS         0000000000041420  00041420
       0000000000000000  0000000000000000  WA       0     0     1
  [13] .bss              NOBITS           0000000000042000  00041420
       0000000000201c18  0000000000000000  WA       0     0     4096
  [14] .comment          PROGBITS         0000000000000000  00041420
       0000000000000033  0000000000000001  MS       0     0     1
  [15] .symtab           SYMTAB           0000000000000000  00041458
       0000000000012570  0000000000000018          17   2442     8
  [16] .shstrtab         STRTAB           0000000000000000  000539c8
       000000000000008c  0000000000000000           0     0     1
  [17] .strtab           STRTAB           0000000000000000  00053a54
       00000000000188cb  0000000000000000           0     0     1
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings), I (info),
  L (link order), O (extra OS processing required), G (group), T (TLS),
  C (compressed), x (unknown), o (OS specific), E (exclude),
  p (processor specific)
```

And that's where I came up with a new hypothesis. Normally, in AArch64
architecutre it's quite easy to write position independent code - the code
that can be loaded at any address and work just fine. That's because most
of the time function calls and data accesses can be relative to the value in
the program counter (`pc` register).

However things break if we decide to store a function pointer in some global
variable and then use that variable to call the function. Function pointer
address in such a case would have to be determined when compiling/linking the
binary and therefore we cannot use `pc` relative addressing to call the
function via the pointer.

That's what appear to be happening here. `format_args!` macro actually tells
Rust compiler to generate a global structure. This global structure have to
contain a function pointer in some form. When the `fmt::format` function is
called the argument of the function references that global structure.

We can easy verify this hypothesis by checking if the resulting binary
contains any dynamic relocations. And indeed turned out that there are quite
a few of them:

```sh
$ readelf -D --relocs kernel.elf 

'RELA' relocation section at offset 0x3bba8 contains 12768 bytes:
  Offset          Info           Type           Sym. Value    Sym. Name + Addend
00000003ee90  000000000403 R_AARCH64_RELATIV                    2a0d0
00000003eea0  000000000403 R_AARCH64_RELATIV                    2a0e0
00000003eeb8  000000000403 R_AARCH64_RELATIV                    18d0
00000003eed0  000000000403 R_AARCH64_RELATIV                    9f5c
00000003eed8  000000000403 R_AARCH64_RELATIV                    2a14e
...
```

# Self-relocation

Normally dynamic relocations resolved using dynamic linker. When your program
depends on a shared library, during loading of the program dynamic linker
identifies this dependency, finds the required shared library, loads it in
memory and then resolves the references.

For example, if you buld a binary that calls a function `foo` that is defined
in a shared library `libfoo.so`, compiler at build time cannot know the
actual address of the function `foo`. So instead of the actual address of the
function the compiler leaves a placeholder for the function address and
creates a note in the binary describing how this placeholder have to be
filled later.

When loading the program, dynamic linker finds `libfoo.so` and loads it in
memory. After loading the library in memory dynamic linker can figure out the
actual address of the function `foo` and fill in the placeholders.

However as you saw above, dynamic address resolution might be needed even if
the shared libraries are not involved at all. The case when shared libraries
are not involved is somewhat simpler than the general problem that dynamic
linker solves. It's because to resolve local references normally all you need
to know is the difference between linktime and runtime addresses.

It's so simple in fact that the binaries can actually do this kind of dynamic
address resolution themselves. A relatively recently toolchains and OS
started to support position independent executables. Those exectuables should
be possible to load at any address.

> *NOTE:* Using position independent executables allows to randomize loading
  address of the binary in memory. Such a randomization is viewed as a form
  of hardening that makes it harder to explot some kinds of vulnerabilities.
  That being said I'm using position independence for simplicity and not for
  security here.

Normally such executables are linked against special kind of C runtime
library. Part of the initialization performed by this C runtime library is
resolving local addresses. My binary doesn't use the standard runtime
libraries and therefore doesn't do this dynamic address resolution.

Fortunately enough we can implement it ourselves, let's get started.

# Implementation

I will start from providing a linker script for the linker. Linker script is
basically a piece of configuration that allows to instruct the linker how
various sections (code, data, etc) will be layed out in the final binary.

I will use custom linker script because it allows to introduce some symbols
(think of them as global variables) that will mark beginning and end of
various sections in the binary file. For example, we can create symbols that
will tell us where relocation information is located.

Relocations information is the information that we need to dynamically
resolve addresess. Basically it tells us what parts of the binary image in
memory have to be changed and how exactly.

Besides that custom linker script allows us to control the linktime
addresses. Controlling linktime addresses is quite handy to figure out the
difference between the linktime and runtime addresses.

The idea is rather simple, since we control linktime addresses we can
rememeber linktime address of some function/variable/etc, in other words,
some known entity. During the runtime we can find out the runtime address of
the same entity using `pc`-relative addressing. Knowing the two we can
calculate the difference.

Let's take a look at the linker script I came up with:

```
OUTPUT_FORMAT(elf64-aarch64)
ENTRY(start)

PHDRS
{
    headers PT_PHDR PHDRS;
    text PT_LOAD FILEHDR PHDRS;
    rodata PT_LOAD;
    data PT_LOAD;
    dynamic PT_DYNAMIC;
}

SECTIONS
{
    . = SIZEOF_HEADERS;
    . = 0x1000;
    _IMAGE_START = .;

    .text : {
        _TEXT_BEGIN = .;
        *(.text)
        _TEXT_END = .;
    } :text

    .rodata : ALIGN(0x1000) {
        _RODATA_BEGIN = .;
        *(.rodata)
        _RODATA_END = .;
    } :rodata

    .rela.dyn : {
        _RELA_BEGIN = .;
        *(.rela.dyn)
        _RELA_END = .;
    } :rodata

    .dynamic : {
        _DYNAMIC = .;
        *(.dynamic)
    } :rodata :dynamic

    .data : {
        _DATA_BEGIN = .;
        *(.data)
        _DATA_END = .;
    } :data

    .bss : {
        _BSS_BEGIN = .;
        *(.bss)
        _BSS_END = .;
    } :data
}
```

The script is quite long and might be quite confusing if you're seeing it
for the first time. However only a few parts are relevant for this
discussion:

```
. = 0x1000;
_IMAGE_START = .;
```

`_IMAGE_START` - is our dedicated known entity with a fixed known linktime
address. This snippet basically instructs the linker to create a sort of
global variable with linktime address `0x1000`. During runtime we can find
out the runtime address of `_IMAGE_START` and compare it to `0x1000` to
figure out the difference between the linktime and runtime addresses.

The other relevant part is:

```
.rela.dyn : {
    _RELA_BEGIN = .;
    *(.rela.dyn)
    _RELA_END = .;
} :rodata
```

This snippet instructs the linker to collect the relocation information
together inside the `.rela.dyn` section. The important part however is two
symboles: `_RELA_BEGIN` and `_RELA_END`. We will use those symbols to find
where the relocation information is stored once the program has been loaded
in memory.

Now I'm going to move further by introducing a startup code that will be
executed before our `main` function. I will write it in assembly because it
provides slightly more control when calculating addresses this way. No
worries it's quite simple:

```
.text
.global start
.extern main, relocate_kernel

start:
    // Preserve parameters from the UEFI bootloader so we can pass it later
    // to the main function.
    mov x19, x0
    mov x20, x1

    // Calculate the difference between the linktime and runtime addresses
    adr x0, _IMAGE_START
    sub x0, x0, 0x1000
    // _RELA_BEGIN and _RELA_END mark the begining and the end of the
    // relocation information
    adr x1, _RELA_BEGIN
    adr x2, _RELA_END

    // relocate_kernel performs the actual relocation/address resolution
    bl relocate_kernel

    // finally when things are done we can call main
    mov x0, x19
    mov x1, x20
    b main
```

Let's cover the important pieces, starting from calculating the difference
between the linktime and runtime addresses:

```
adr x0, _IMAGE_START
sub x0, x0, 0x1000
```

Instruction `adr` is similar to the instruction `adrp`, but it calculates the
exact address and not the 4KiB page address using offset from `pc`. So this
instruction will store the runtime address of `_IMAGE_START` in the register
`x0`.

The second part is easy to understand - it just calculates the difference
between the calculated runtime address of `_IMAGE_START` and the known
linktime address (which as was covered above is `0x1000`).

Now we need to find our relocation information, that's also quite easy:

```
adr x1, _RELA_BEGIN
adr x2, _RELA_END
```

I'm again using the `adr` instruction, but this time to calculate the runtime
addresses of `_RELA_BEGIN` and `_RELA_END` in registers `x1` and `x2`.

It also so happens that according to the calling convention registers `x0`,
`x1` and `x2` are used to pass 3 first arguments of the function (when they
fit in 64-bit registers).

Finally, we call the function that actually performs the resolution with
arguments in registers `x0`, `x1` and `x2`:

```
bl relocate_kernel
```

Before we take a look at the `relocate_kernel` (no worries, the
implementation will be in C, so we are done with the assembly) let's take a
look at how the relocation information is structured.

Actually there are a few different types of relocations and information for
different types of relocation might be different. In our case all the
relocations happen to be of type `R_AARCH64_RELATIV` as we saw above. This
relocation type has a so called addend part (thus the 'a' part in the
'rela').

All the relocations with an addend are described by this structure (for
64-bit ELFs that is):

```c
#include <stdint.h>

struct elf64_rela {
    uint64_t r_offset;
    uint64_t r_info;
    int64_t r_addend;
};
```

How the fields are interpreted might depend on the specific relocation type.
They however should follow a general idea:

* `r_offset` should contain linktime address of the place that needs to be
  adjusted (that's where we have to write the resolved address)
* `r_addend` is a value that we should add to the difference between the
  linktime and runtime addresses.

So basically we need to take the difference between linktime and runtime
addresses, add the value of the `r_addend` to it and write it to the place
pointed by the `r_offset` field. Let's take a look at the whole thing now:

```c
static const uint32_t R_AARCH64_RELATIVE = 1027;

static inline uint32_t rela_type(const struct elf64_rela *rela)
{
    return rela->r_info & 0xffffffff;
}

void relocate_kernel(
    int64_t diff, struct elf64_rela *begin, struct elf64_rela *end)
{
    for (struct elf64_rela *ptr = begin; ptr != end; ++ptr) {
        while (rela_type(ptr) != R_AARCH64_RELATIVE);

        *(uint64_t *)(ptr->r_offset + diff) = ptr->r_addend + diff;
    }
}
```

> *NOTE:* that we also have to apply modification to the `r_offset` value
  because it contains the linktime address and not the actual runtime
  address.

In the code above, if we discover a relocation that is not
`R_AARCH64_RELATIVE` the code will just hang there in the infinite loop. This
way we'd be able to detect early if we stumble on some unexpected relocation
type.

And that's it. With this little piece of code the problem was resolved and
`format!` macro stopped causing problems.

# Instead of conclusion

That's turned out to be quite a lengthy post, but the investigation itself
took quite some time.

I was probably a bit handwavy in some parts of the investigation, there are
a few reasons to that:

* I myself had to learn a bunch of stuff while investigating this problem,
  so, unsurprisingly, I have some knowledge gaps;
* it's a bit difficult to write lengthy posts, so I may have cut corners here
  and there.

If you think that I could cover some aspects in more details or have a better
explanation yourself, suggestions are welcome.
