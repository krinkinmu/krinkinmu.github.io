---
layout: post
title: How U-boot loads Linux kernel
excerpt_separator: <!--more-->
tags: u-boot quemu gdb aarch64 linux-kernel
---

I'm continuing my exploration of how to use U-boot. Last time I covered some
basics, this time I will build on that and will dive into a bit more realistic
example - how U-boot loads Linux kernel.

<!--more-->

# Linux kernel

Last time I used a plain binary containing binary code without any structure
and that works fine for simple scenarios.

However Linux kernel is a bit more complicated in multiple ways. For example,
Linux kernel images can be compressed and can support EFI - that means that
they should look like PE executables. As a result, U-boot also has to be able
to handle different binary formats.

On top of that, Linux kernel for Aarch64 required a description of perefery
attached to the board in form of a flat device tree blob (fdt). So not only
U-boot has to support different binary formats, it also need to be able to pass
some arguments to the kernel, e.g. address of the fdt in memory.

It's also quite common for Linux to have an image of init RAM filesystem, so
that should be provided somehow as well, though it's optional.

The format of the Linux kernel image is documented in
https://www.kernel.org/doc/Documentation/arm64/booting.txt. The documentation
is farily straighforward, but let's play with it a bit to understand it better.

I will skip compressed kernel images and right away start with decompressed
image format. Linux kernel, at the beginning of the decompressed binary puts a
header in the following format:

```c
struct Image_header {
	uint32_t	code0;		/* Executable code */
	uint32_t	code1;		/* Executable code */
	uint64_t	text_offset;	/* Image load offset, LE */
	uint64_t	image_size;	/* Effective Image size, LE */
	uint64_t	flags;		/* Kernel flags, LE */
	uint64_t	res2;		/* reserved */
	uint64_t	res3;		/* reserved */
	uint64_t	res4;		/* reserved */
	uint32_t	magic;		/* Magic number */
	uint32_t	res5;
};
```

> NOTE: You can find this structure in the U-boot code base in the
> [arch/arm/lib/image.c](https://github.com/u-boot/u-boot/blob/master/arch/arm/lib/image.c).
> The same structure is documented in the Linux kernel docs.

There are multiple fields here, but when it comes to U-boot the only relevant
fields are:

* `code0`/`code1`
* `text_offset`
* `image_size`
* `flags`
* `magic`

## `code0`/`code1`

> TL;DR: the only functionally relevant content of the `code0`/`code1` is the
> unconditional jump instruction that jumps over the rest of the header fields and
> transfers control to the actual kernel code.

`code0` and `code1`, as the commend in the U-boot code indicates are expected
to contain executable code. That basically means that at the beginning of the
Linux kernel binary we are expected to have some actually executable code.

Two fields are just 8 bytes long together, so there is not a lot of code that
could be put there, but we can fit there an unconditional jump instruction
(e.g. `b` instruction in ARM).

Linux kernel, depending on the configuration, puts in `code0` and `code1`
either:

```gas
ccmp x18, #0, #0xd, pl
b primary_entry
```

or

```gas
nop
b primary_entry
```

> NOTE: The header in Linux kernel code base is populated inside
> [arch/arm64/kernel/head.S](https://elixir.bootlin.com/linux/latest/source/arch/arm64/kernel/head.S).

Instruction `nop`, as the name suggest, does nothing - it's just used to fill
the space. `ccmp x18, #0, #0xd, pl`, while looks complicated, doesn't actually
do anything useful and is only there because binary encoding of the
instruction, when interpreted as ASCII string, spells "MZ".

As I mentioned before, Linux kernel may support EFI and when it is configured,
the Linux kernel binary is expected to look like
[PE executable](https://en.wikipedia.org/wiki/Portable_Executable). One of the
valid PE executable binaries may start from a so called DOS Header which
contains string "MZ" as its first two bytes. So this way Linux kernel
maskarades itself as a PE executable.

## `text_offset`

> TL;DR: This field does affect how and more imporatntly where U-boot relocates
> the binary in memory, but in practice it is always set to 0.

As I showed last, U-boot has commands to load binaries from different places to
memory (e.g. from a disk with FAT filesystem). With Linux kernel the story is
similar - we still need to put the binary in memory. However the story does not
end there and U-boot may relocate the binary in memory to a different location
to satisfy certian constraints Linux kernel puts forward.

One such constraint is that Linux kernel image has to be relocated to a
`text_offset` bytes from a 2MiB aligned memory address.

I'm not familiar with the history of the constraint, so cannot really say why
exactly Linux kernel has it. However, in the recent kernels at least,
`text_offset` is always set to 0. So in practice all U-boot needs to do is to
make sure that the kernel image is aligned on 2MiB boundary and if it's not, it
will move it to satisfy the constraints.

> NOTE: You can find the U-boot logic responsible for relocating Linux kernel
> in [rch/arm/lib/image.c](https://github.com/u-boot/u-boot/blob/master/arch/arm/lib/image.c).

## `image_size`

The name of this field is fairly self descriptive. It just indicates the size
of the "useful" part of the kernel image. That's how many bytes U-boot would
need to relocate if the kernel is not aligned in memory on 2MiB boundary.

## `flags`

There are 3 properties that can be recorded in the flags:

1. Kernel endianness (Little Endian or Big Endian) - as far as I can tell
   U-boot does not really care about the value of this flag;
2. Kernel page size (4KiB, 16KiB, 64KiB or unspecified) - similarly to the flag
   above it does not seem like U-boot cares about the value of this flag;
3. Kernel placement - it's a hint flag, when it's not set, U-boot should
   relocate the kernel as close as possible to the begining of the usable RAM.

Linux kernel endiannes is configurable, so it seems like in principle you can
have both Little Endian and Big Endinan kernel. However, the kernel page size
for Aarch64 seem to always be 4KiB and and nowdays Linux kernels for Aarch64
can be placed anywhere without restrictions.

So for example, if we assume that we are building a Linux kernel for Little
Endian and given the values of all other flags appear to be fixed, then the
value of the flags we are looking for should be `0x000000000000000a`.

## `magic`

This field just contains a fixed value: `0x644d5241`. U-boot will check the
value in this field to confirm that the image it tries to load is actually a
Linux kernel binary in the proper format.

If the value will not match the expected magic value, U-boot will refuse to
boot the image.

# Practice

I read the documentation and got some level of understanding of how a Linux
kernel binary should look like. Now it's time to test that knowledge - the task
for today is to create a binary that for U-boot will look like a Linux kernel
binary and U-boot should load it.

## Output

Last time I used gdb to confirm that my binary was loaded and is running. It's
a good tool, but it's a bit tiresome to always use gdb to figure out if things
worked.

So to simplify my life a little bit I will change the binary to output some
text via serial port. Qemu emulate `PL011` compatible UART device, so we can
create a simple driver that outputs some text to serial port.

I will not cover in details how to interract with `PL011` here - I have another
article covering this very topic specifically. Just for completeness I will say
that I assume that we have the following two functions implemented:

```c
// Initializes pl011 structure given the provided paramters.
//
// addr - physical address where the pl011 registers are mapped to
// clock - frequency of the clock that pl011 is using
void pl011_setup(struct pl011 *serial, uintptr_t addr, uint64_t clock);

// Sends given data to the serial port.
void pl011_send(const struct pl011 *serial, const uint8_t *data, size_t size);
```

I also happen to know (and it's also covered in another article on `PL011` that
I have on the site) that the frequency of the clock that `PL011` uses is 24GHz
and the PL011 registers are mapped to address `0x9000000`.

Putting it all together we have the following code:

```c
#include "pl011.h"

void kernel_start(void *fdt) {
    struct pl011 serial;
    char greeting[] = "Hello, World\n";

    pl011_setup(&serial, 0x9000000, 24000000);
    pl011_send(&serial, (uint8_t *)greeting, 13);
}
```

I will show how the `kernel_start` function will be called later in this
article.

## Header

Now I have a code that sends some data to a serial port that I will be able
to see when (and if) U-boot successfuly loads and boots the kernel image. And
I have to make sure that the binary will have a proper header now.

Fortunately generating the header in assmebler is mostly straighforward:

```gas
.section ".head.text","ax"
.global _start
.extern _end, kernel_start

_start:
    // code0/code1
    nop
    b entry

    // text_offset
    .quad 0

    // image_size
    .quad _end - _start

    // flags
    .quad 0b1010

    // Reserved fields
    .quad 0
    .quad 0
    .quad 0

    // magic - yes 0x644d5241 is the same as ASCII string "ARM\x64"
    .ascii "ARM\x64"

    // Another reserved field at the end of the header
    .byte 0, 0, 0, 0

entry:
    bl kernel_start 

loop:
    b loop
```

I covered most of the header fields above already, so I will not go over them
again. I will just mention, that in `code0`/`code1` I just put `nop`
instruction and a jump to the actual entry point `entry`.

The only thing that the actual entry code does is calls `kernel_start` that I
showed above already. Once the `kernel_start` function returns the code enters
infinite loop.

Two important things to note in this snippet are these:

1. `image_size` is calculated as a difference between `_start`, the lable that
   marks the beginning of the header, and `_end` that is not in this code, but
   I will create this symbol in the linker script.
2. Instead of using `.text` GNU Assember directive to create a section, I use
   `.section` directive - that's because I want to use a custom name for the
   section - it will be useful to make sure that the code in this file will
   be at the beginning of the final binary.

## Linker script

I use the linker script from the previous article with some minor
modifications. I will not copy the whole linker script and will just show the
relevant parts that I changed:

```
OUTPUT_FORMAT(elf64-aarch64)
// NOTE: The name of the entry point changed from start to _start
ENTRY(_start)

...

SECTIONS
{
    ...

    .text : {
        _TEXT_BEGIN = .;
        // NOTE: Here I use the name of the section from the assmbler code above.
        //       That's why I used .section directive, so that I can give it a
        //       different name and refer to that name here and make sure that
        //       the header will actually be at the beginning of the file and not
        //       in some random location.
        *(.head.text)
        *(.text)
        _TEXT_END = .;
    } :text

    ...

    // NOTE: Here I create the _end symbol with an appropriate value, so that
    //       linker can automatically calculate the size of the image and put
    //       it in the header.
    _end = .;
}
```

There are two really relevant changes here:

1. I had to make changes to make sure that header will be at the beggining of
   the binary where the header should be;
2. I had to create `_end` symbol to be able to calculate the size of the image.

## Build and compare

I assume that the result of the build is stored in `kernel.bin` file, just like
in the previous article. Let's take a look at what is at the beginning of the
binary:

```sh
hexdump -C -n 64 kernel.bin
00000000  1f 20 03 d5 0f 00 00 14  00 00 00 00 00 00 00 00  |. ..............|
00000010  08 11 00 00 00 00 00 00  0a 00 00 00 00 00 00 00  |................|
00000020  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
00000030  00 00 00 00 00 00 00 00  41 52 4d 64 00 00 00 00  |........ARMd....|
00000040
```

The fist 8 bytes are just encoding instructions `nop` and `b entry`. Then goes
the `text_offset` value which happens to be 0.

The image size that was recorded in the image is `0x1108` (4360 bytes) recorded
in the Little Endian format (e.g. starting from the least significant byte).
That's a bit more than I expected, but it matches the size of the file in my
case, so nothing outrageously wrong here either:

```sh
ls -al kernel.bin 
-rwxrwxr-x 1 kmu kmu 4360 Aug 21 17:38 kernel.bin
```

Then go flags, and the value, as expected, is `0xa`. Finally, after some
reserved fields, we have a magic value `0x644d5241`, again in Little Endian
format.

So the header looks close to what I expected, but I can do another check here.
Given that it's the header used by Linux kernel, I can just build a Linux
kernel and compare its header with mine:

```sh
sudo apt-get install gcc-aarch64-linux-gnu
git clone git://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git linux
cd linux
ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- make allnoconfig
ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- make -j8
ls -al arch/arm64/boot/Image
-rw-rw-r-- 1 kmu kmu 4040712 Aug 21 17:10 arch/arm64/boot/Image
hexdump -C -n 64 arch/arm64/boot/Image
00000000  1f 20 03 d5 0d f0 0a 14  00 00 00 00 00 00 00 00  |. ..............|
00000010  00 00 43 00 00 00 00 00  0a 00 00 00 00 00 00 00  |..C.............|
00000020  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
00000030  00 00 00 00 00 00 00 00  41 52 4d 64 00 00 00 00  |........ARMd....|
00000040
```

The headers don't match perfectly, but they are pretty close. Let's take a look
at the details starting with `code0`/`code1`. The 4 bytes of the `nop`
instruction do match between the two headers, but encoding of the `b`
instructions are different.

The difference here makes perfect sense, because Linux kernel jumps to a
slightly different place from my code and the instruction also encodes the
offset of the jump, so it stands to reason that instruction encodings may be
somewhat different.

The size of the Linux kernel image is much larger (0x430000 or 4390912 bytes).
Interestingly enough the size of the image is actually greater then the size of
the binary file.

> NOTE: I suspect that it might have something to do with BSS sections that are
> not stored in the binary file itself, but ultimately given that I use Linux
> kernel in this task as a perfect example, I will not dig deeper and assume
> that Linux does it right.

The flags and magic values that Linux kernel has in the header match the values
in my binary, so nothing to look at here - the headers appear to losely match.

## Device Tree

One last thing we need is a flat device tree blob that contains description of
the board. I've already covered in one of the previous articles how to get
device tree from Qemu and how to compile it into a flat device tree.

I will just mention that for the example here I will assume that the flat
device tree lives in file `virt.dtb` in the `rootfs` directory, so U-boot has
access to it.

## Loading and running

So I have `kernel.bin` and `virt.dtb` in my `rootfs` and they seem, at least
after a brief inspection, to be correct. Let's test it with U-boot. I described
how to launch U-boot in Qemu before, so will not cover it here, so I start from
U-boot shell right away.

Unlike the last time, we now have to load 2 binary files in memory and they
should not overlap, otherwise one will overwrite parts of the other. In order
to do that all we need is to just space them in memory far enough apart:

```sh
U-Boot 2023.10-rc1-00323-gb1a8ef746f (Aug 07 2023 - 13:43:27 +0100)

DRAM:  128 MiB
Core:  51 devices, 14 uclasses, devicetree: board
Flash: 64 MiB
Loading Environment from Flash... *** Warning - bad CRC, using default environment

In:    pl011@9000000
Out:   pl011@9000000
Err:   pl011@9000000
Net:   eth0: virtio-net#32
Hit any key to stop autoboot:  0 
=> fatload virtio 0:1 0x40000000 kernel.bin
4360 bytes read in 2 ms (2.1 MiB/s)
=> fatload virtio 0:1 0x41000000 virt.dtb
1048576 bytes read in 2 ms (500 MiB/s)
```

We now have kernel image loaded in memory starting with address `0x40000000`
and flat device tree starting with address `0x41000000`. We can check that the
device tree is valid and U-boot can parse it by running:

```sh
fdt addr 0x41000000
fdt print /
```

The first command will set the address of the "working" device tree and the
second command prints the working device tree starting from the root of the
tree.

To actually load the binary the same way as Linux kernel binary U-boot provides
`booti` command. The command needs 3 parameters:

1. The address where the kernel binary was loaded;
2. The address where init RAM filesystem image was loaded;
3. The address where flat device tree was loaded.

> NOTE: I don't use init RAM filesystem, so I will specify `-` instead of the
> address.

So let's try:

```sh
booti 0x40000000 - 0x41000000
## Flattened Device Tree blob at 41000000
   Booting using the fdt blob at 0x41000000
Working FDT set to 41000000
   Loading Device Tree to 0000000045c8c000, end 0000000045d8efff ... OK
Working FDT set to 45c8c000

Starting kernel ...

Hello, World
```

The output suggest that U-boot successfully loaded the image and it sent
"Hello, World" to the output. It also suggests that U-boot relocated the device
tree from `0x41000000` to `0x45c8c000`.

I'm not completely certain why U-boot decided to relocate the device tree to a
different place, but I suspect that U-boot may sometimes make some changes to
the provided device tree (e.g. mark some memory addresses as reserved) and in
such cases it would make sense to construct a new device tree in a different
location.

# Instead of conclusion

It was a nice and simple exercise to try to create a Linux kernel-like binary
image that U-boot can load and boot. I was also able to get a bit more familiar
with bits an pieces of U-boot and Linux kernel code bases, which is a bonus as
well.
