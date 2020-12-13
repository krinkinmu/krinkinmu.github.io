---
layout: post
title: Adding a little bit of Rust to AARCH64
excerpt_separator: <!--more-->
tags: clang aarch64 qemu uart pl011 hikey960 rust
---

[previous post]: {% post_url 2020-12-05-PL011-HiKey960 %} "the previous post"
[PL011 specification]: https://developer.arm.com/documentation/ddi0183/g/ "PL011 TRM"
[HiKey960]: https://www.96boards.org/product/hikey960/ "HiKey960"
[HiKey960 Hardware User Manual]: https://www.96boards.org/documentation/consumer/hikey/hikey960/hardware-docs/hardware-user-manual.md.html "HiKey960 Hardware User Manual"

In the [previous post] we managed to make PL011 UART work on my [HiKey960].
We did that using C, but all the cool guys these days use Rust, so let's see
how we can make it work.

The sources for this post are available on
[GitHub](https://github.com/krinkinmu/aarch64).

<!--more-->

# Intro

I want to put a disclaimer here that I'm by no means a fan of Rust language
and tooling. My displeasure with Rust is mostly caused by two reasons:

* Rust tooling is quite less mature than the existing tooling for C and C++
* I've had experience with Rust "cultist" who believe in some magic properties
  of Rust that are capable of addressing all the fundamental problems of
  software engineering no matter the circumstances.

In other words, take everything I write here with a grain of salt.

# Preparations

First things first, we need the toolchain to cross compile for the target
platform. Rust toolchain is based on LLVM, so I didn't expect any problems
with that.

One difference between default distribution of Clang/LLVM I have in Ubuntu
and Rust/Cargo is that Clang somehow comes with the built-in support for all
the architectures supported by LLVM, but in case of Rust I had to install it:

```sh
$ rustup target add aarch64-unknown-none
```

It's rather hard to find what is the exact configuration of the
`aarch64-unknown-none` target, but we can see some basic configuration
parameters of the target using nightly version of the toolchain:

```sh
$ rustc +nightly -Z unstable-options --print target-spec-json --target aarch64-unknown-none
{
  "arch": "aarch64",
  "data-layout": "e-m:e-i8:8:32-i16:16:32-i64:64-i128:128-n32:64-S128",
  "disable-redzone": true,
  "executables": true,
  "features": "+strict-align,+neon,+fp-armv8",
  "is-builtin": true,
  "linker": "rust-lld",
  "linker-flavor": "ld.lld",
  "linker-is-gnu": true,
  "llvm-target": "aarch64-unknown-none",
  "max-atomic-width": 128,
  "panic-strategy": "abort",
  "relocation-model": "static",
  "target-pointer-width": "64",
  "unsupported-abis": [
    "stdcall",
    "fastcall",
    "vectorcall",
    "thiscall",
    "win64",
    "sysv64"
  ]
}
```

A few things to notice is that redzone is disabled, which is good, however
there are some instruction set extensions enabled (`neon` and `fp`), so we
need to make sure that those will not slip in into the resulting code.

# Project Organization

Since I'm not a big Rust fan I will leave a room for myself to use C and
Assembly language in the project.

Regarding the Assembly laguage specifically I personally prefer using
separately compiled Assmebly code to the inline Assembly whenever possible.
So being able to compile Assembly code and add the results to the project is
useful this way as well.

With that in mind I will compile Rust code into a statically linked library.
I will then link this library to the resulting binary.

Even though the Rust code will be compiled in one static library, internally
it will be organized in multiple crates as needed. So I will use a workspace.

I will have a high level `Makefile` that will call into Cargo and compile the
C/Assembly code and then link them together.

So let's create the setup:

```sh
$ cd ~/ws/aarch64/src
$ mkdir kernel
$ touch kernel/Cargo.toml
```

`~/ws/aarch64/src` is where I store all the code for the project on my machine.
`~/ws/aarch64/kernel` is the top level directory for the Rust workspace.

In the `Cargo.toml` file we will put workspace configuration:

```
[workspace]
members = [
    "runtime",
    "start",
    "pl011",
]

[profile.dev]
panic = "abort"

[profile.release]
panic = "abort"
```

`members` list includes the list of modules that constitues the workspace. We
will create those crates a bit later.

`panic = "abort"` instructs the toolchain what behavior to choose when an
unrecoverable error is detected in the program. By default Rust will built the
code to do fancy stack unwinding and so on. That requires some runtime support
and to simplify the things I disable it.

*NOTE:* none of this is new, you can find plenty of explanations for the
`panic = "abort"` on the Internet.

Let's move on to create the crates mentioned in the `Cargo.toml`:

```sh
$ cd kernel
$ cargo new --lib --vcs=none --edition=2018 runtime
$ cargo new --lib --vcs=none --edition=2018 start
$ cargo new --lib --vcs=none --edition=2018 pl011
```

I also had to modify a little bit `Cargo.toml` files for each module to specify
dependencies and adjust a little bit crate types:

* `runtime/Cargo.toml`:

```
[package]
name = "runtime"
version = "0.1.0"
authors = ["Mike Krinkin <krinkin.m.u@gmail.com>"]
edition = "2018"

[lib]
name = "runtime"
crate-type = ["rlib"]
```

* `pl011/Cargo.toml`:

```
[package]
name = "pl011"
version = "0.1.0"
authors = ["Mike Krinkin <krinkin.m.u@gmail.com>"]
edition = "2018"

[lib]
name = "pl011"
crate-type = ["rlib"]
```

* `start/Cargo.toml`:

```
[package]
name = "start"
version = "0.1.0"
authors = ["Mike Krinkin <krinkin.m.u@gmail.com>"]
edition = "2018"

[lib]
name = "start"
crate-type = ["staticlib", "rlib"]

[dependencies]
runtime = { path = "../runtime" }
pl011 = { path = "../pl011" }
```

Everything except the `start` crate will be build as `rlib`, which I presume
a Rust specific library format. Crates `runtime` and `pl011` are dependencies
for the `start` crate, so you could probably guess that `start` is the top
level crate for our static library. That's why crate type contains `staticlib`.

I didn't explain before what are those crates are for and the names might not
be that great either, so let's tackle those one by one next.

## Runtime

Depending on the language features it might require some help to do its
functions. For example, when in C or C++ you're allocating memory there might
be some piece of code that requests memory from the OS when needed and then
allocates it to the program by request.

Hypothetically toolchain might generate and build the code that does it into
the binary, but it's rather inconvenient. For example, different OS will have
different interfaces to support dynamic memory allocation, not to mention there
are situation where there is no OS at all.

So instead this logic can be put into a library created sopecifically for the
target platform and this library is just linked into the resulting binary.
This kind of language features support library is what I call runtime here.
In other words, runtime is the code that the toolchain expects to be available
for the target platform.

*NOTE:* here are some other interesting examples of runtime things:
* constructors of global objects in C++ (did you think who calls those before
  the `main` is called)
* similarly destructors of the global objects are called somehow
* normally, even before `main` is called there is some code that should prepare
  the environment and call `main` with the right arguments
* more marginal example is when compilers use `memcpy` functions to copy large
  objects, if that's the case `memcpy` should be provided to the compiler as
  part of the runtime as well.

In normal circumstances the runtime is automatically provided, but for the
target `aarch64-unknown-none` it's not the case and it makes perfect sense.
So now it's on us to provide the toolchain with all the runtime it needs.

Fortunately for Rust with the `panic = "abort"` option all we need is to
provide the implementation of the `panic` function. So here is the content of
`runtime/src/lib.rs`:

```rust
#![no_std]

use code::panic::PanicInfo;

#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
  loop {}
}
```

In my siple case I don't really know what I want to do when the Rust code
panics, so I've put an infinite loop here and that's it. After all for the
compilation to work the implementation doesn't have to be good or even correct.

One important thing to notive is the `#![no_std]` declaration that instructs
the toolchain that this module does not and cannot depend on the standard
library.

In case of `aarch64-unknown-none` there is no default standard library
provided. So if we don't provide this directy the build will fail:

```
error[E0463]: can't find crate for `std`
  |
  = note: the `aarch64-unknown-none` target may not be installed
```

Building it for a target that actually has a standard library provided also
will not work without `#![no_std]` directive but for a different reason:

```
error[E0152]: found duplicate lang item `panic_impl`
 --> runtime/src/lib.rs:4:1
  |
4 | fn panic(_info: &PanicInfo) -> ! {
  | ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  |
  = note: the lang item is first defined in crate `std` (which `runtime` depends on)
  = note: first definition in `std` loaded from /home/kmu/.rustup/toolchains/stable-x86_64-unknown-linux-gnu/lib/rustlib/x86_64-unknown-linux-gnu/lib/libstd-93cbfed54dd1bac8.rlib
  = note: second definition in the local crate (`runtime`)
```

The error tells us that we re-implemented a thing that already exists, which
again makes perfect sense.

## PL011

`PL011` create is just re-implemented PL011 code in Rust. I will not cover the
actual implementation in any details and only cover some aspects that I found
interesting.

First of all, the C implementation made use of `volatile` to mark accesses to
the memory mapped PL011 registers as part of the visible behavior of the
program. How can we achieve a similar behavior in Rust?

In Rust there are `core::ptr::read_volatile` and `core::ptr::write_volatile`.
Those functions are part of unsafe subset of Rust and those are the only unsafe
Rust functions we need. I wrapped them into a couple of functions:

```rust
use core::ptr;

#[derive(Copy, Clone)]
pub struct PL011 {
    base_address: u64,
    base_clock: u32,
    baudrate: u32,
    data_bits: u32,
    stop_bits: u32,
}

#[derive(Copy, Clone)]
enum Register {
    UARTDR = 0x000,
    UARTFR = 0x018,
    UARTIBRD = 0x024,
    UARTFBRD = 0x028,
    UARTLCR = 0x02c,
    UARTCR = 0x030,
    UARTIMSC = 0x038,
    UARTICR = 0x044,
    UARTDMACR = 0x048,
}

impl PL011 {
    ...
    fn load(&self, r: Register) -> u32 {
        let addr = self.base_address + (r as u64);

        unsafe { ptr::read_volatile(addr as *const u32) }
    }

    fn store(&self, r: Register, value: u32) {
        let addr = self.base_address + (r as u64);

        unsafe { ptr::write_volatile(addr as *mut u32, value); }
    }
    ...
}
```

With those two functions available, the rest of the PL011 C implementation
translate into Rust pretty straighforwardly.

Is the Rust code safer compared to C? Well, not. At the end of the day the
addreses of memory mapped registers are coming of the outside as a parameter
and there is no way to statically verify that those addresses are correct for
a particular platform.

However, we can benefit from Rust here is a slightly different way. Rust has
builtin support for tests, something that C as a language doesn't have.

Unfortunately Rust test framework depends on `std` which comes with caveats:

* we need to do some additional manipulations to enable tests for our `no_std`
  crate
* `aarch64-unknown-none` doesn't have `std`, so no tests for that platform.

The last one is not a big problem for me however. It's my opinion that at
least unit tests must be able to run on the developer platform first of all.
My developer platform is `x86-64` Linux and Rust does have `std` implementation
for that.

So I only need to deal with the first caveat. Rust does provide some means
of conditiation compilation, so all it takes is to wrap `#![no_std]` to avoid
using it when compiling tests:

```rust
#![cfg_attr(not(test), no_test)]
```

## Start

Finally, our `start` crate which is going to be an entry point into the Rust
part of our program:

```rust
#![no_std]
extern crate runtime;
use pl011::PL011;

#[no_mangle]
pub extern "C" fn start_kernel() {
    // For HiKey960 board that I have the following parameters were found to
    // work fine:
    // 
    // let serial = PL011::new(
    //     /* base_address = */0xfff32000,
    //     /* base_clock = */19200000);
    let serial = PL011::new(
        /* base_address = */0x9000000,
        /* base_clock = */24000000);
    serial.send("Hello, from Rust\n");
    loop {}
}
```

We don't explicitly use any functions from the `runtime`, so I had to
explicitly import it with the `extern` directive.

Additionally, I had to mark the `start_kernel` function with `extern "C"` and
`#[no_mangle]`. As far as I understand `extern "C"` part is to tell the
toolchain to use some well defined calling convention for the function.

*NOTE:* there is a problem with this `extern "C"`: "C" is not actually a
calling convention name. For example, `"C"` on `x86-64` is very different from
`"C"` on `aarch64`. Even on one platform there might be multiple different
things refered as `"C"` (like, Windows and Linux use different calling
conventions on `x86-64` for some reason). So what does `extern "C"` mean is not
clear, but it "should work"!

`#[no_mangle]` part is to forbid the toolchain from transforming the function
name. It's important for external code that will refer to this function by
name.

After that we should be able to call the `start_kernel` function from the C
code like this:

```c
void start_kernel(void);

void main(void)
{
    ...
    start_kernel();
    ...
}
```

# Building and Testing

Now it's time to build the thing:

```sh
$ cd ~/ws/aarch64/src/kernel
$ cargo build --release --target=aarch64-unknown-none
```

The resulting static library should be available in
`kernel/target/aarch64-unknown-none/release/libstart.a`. It should be possible
to directly link this library into our binary.

I correct the `Makefile` in the repository to call Cargo and link the resulting
library into the `kernel.elf` binary. There is nothing particularly interesting
in the `Makefile`, so for detail I will send you to check the code.

When it comes to testing, I actually wasn't able to find a working way to run
all the tests on the Rust workspace level, so I had to run tests for individual
crates that support tests (which in my case just `pl011` at this point):

```sh
$ cd ~/ws/aarch64/src/kernel/pl011
$ cargo test
```

*NOTE:* we don't need to explicitly specify the target as by default the host
platform target will be used and that's exactly what I want from unit tests.

# Instead of conclusion

Yeah, Rust, Wow!

Not a lot of benefits from Rust at this point, but we will see if it's going
to change in the future.
