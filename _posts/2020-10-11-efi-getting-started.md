---
layout: post
title: Getting started with EFI
excerpt_separator: <!--more-->
tags: efi clang microsoft
---

[UEFI Specification]: https://uefi.org/sites/default/files/resources/UEFI%20Spec%202.8B%20May%202020.pdf "UEFI Specification"
[Eli Bendersky article]: https://eli.thegreenplace.net/2011/09/06/stack-frame-layout-on-x86-64 "Stack frame layout on x86-64"

I'm trying to explore another relatively new are for me: UEFI. When working on
student and hobbt project many people tend to start from legacy BIOS or
multiboot to boot their hello world kernels.

On the one hand it makes a lot of sense to use the simplest solution possible.
On the other hand EFI complexity serves some purpose and with EFI you get a lot
of useful tools right out of the box.

With all that in mind let's try to cook up something with EFI. Sources for this tutorial 
are available on [GitHub.](https://github.com/krinkinmu/efi/commit/7c837b6)

<!--more-->

# The basics

EFI has a publicly available specification that should serve as a reference:
[UEFI Specification]. At the moment of writing the latest specification version
available is 2.8.

The document is rather long, so reading through all of it is a bit inefficient.
Before diving right into, it might be useful to set straight some basics:

* EFI code is expected to use Microsoft ABI at least for all interaction with
  the firmware
* EFI code is expected to be position independent as there is no guarantees that
  it will be loaded at a fixed address
* The EFI binary format is PE32+ (don't get confused by the 32 in the name, it
  doesn't mean that it's limited to just 32 bit code)

So it looks like what we need is to build a Windows binary, which was part of
the problem with using it for various hobby and student projects. It wasn't
impossible, but building an EFI application used to require all kinds of hacky
solutions, like GNU EFI, or wrapping a normal ELF binary into something that
looks like PE32+ executable, like Linux EFI stub.

Fortunately LLVM and Clang projects got quite mature and LLVM/Clang supports
building EFI binaries, so it's much easier now.

# Dependencies

Now, let's install the tools we'd need to build and test our hello world
example.

```sh
sudo apt-get install qemu-system ovmf clang lld
```

I will be using QEMU for testing and OVMF is the UEFI firmware that QEMU can
use.

We will use Clang for compilation and lld is a linker developed inside the LLVM
ecosystem. We need lld because it supports generating PE32+ binaries.

I will keep my code and data within `ws/efi` directory, so let's prepare that
as well:

```sh
mkdir -p ws/efi
cd ws/efi
cp /usr/share/qemu/OVMF.fd .
mkdir root
```

`OVMF.fd` file contains the image of the UEFI firmware that QEMU can use. I
think it's supposed to be some kind of flash image and flash should contain both
the code and some configuration parameters, so I would expect that QEMU might
need to write something there. That's why I copied the image to the working
directory.

The `root` subdirectory is going to represent the fake filesystem that we will
use inside our virtual machine. Basically we will put our EFI binary there to
make it available inside QEMU.

And we all done.

# EFI programming model

The complete EFI programming model is rather rich and supports a lot of
different usecases, but we just want a hello world, so figuring out all of that
is a bit wasteful. So we concentrate on the simple EFI applications.

We start from looking at the environment in which our application will execute.
As mentioned before we don't know exactly what address the application will be
loaded at for sure. However we do know that it will run in long mode for 64 bit
binaries and in uniprocessor mode, even if there are multiple cores available.

Additionally paging will be enabled, but all relevant addresses are identity
mapped. Whcih means that memory mapped such that virtual address matches with
the physical address.

There are also other details, like how much stack we have and values of various
control bits. You can find more in the section 2.3.2 IA-32 Platforms of the
[UEFI Specification]. I will not go into all the details of the hardware state,
because at this point a high level understanding is enough.

EFI application entry point gets two parameters `EFI_HANDLE` and
`EFI_SYSTEM_TABLE` pointer in registers *RCX* and *RDX* correspondingly. I
myself didn't yet figure out what the `EFI_HANDLE` is needed for, but
`EFI_SYSTEM_TABLE` pointer is quite important.

As I mentioned EFI provides us with a bunch of useful tools out of the box and
this `EFI_SYSTEM_TABLE` pointer is how you can access those tools. Basically
this table (indirectly) contains a bunch of function pointers, that can be used
to call into EFI to request the firmware to do something. For example, you can
ask EFI to print something on the screen/serial port/whatever makes sense for
your platform or you can ask it to load some data from the filesystem in memory.

> *NOTE:* *RCX* and *RDX* is not how you pass arguments to a function in System
  V ABI for x64 architecture, that's how you pass arguments to a function in
  Microsoft ABI. [UEFI Specification] is more or less self contained and it does
  describe the calling convention it uses, so you don't need a separate document
  for that.

# EFI Hello World

So far I've been a bit hand wavy, so it's time to get to the specifics and write
some code. I will start from defining required data types, but I will cheat a
bit and will only define the parts I'm actually going to use.

I'll start with the `EFI_HANDLE` and `EFI_SYSTEM_TABLE` that our entry point
takes as parameters. `EFI_HANDLE` is one of the basic data types in the
[UEFI Specification] and it's basically a generic data pointer, or in `C` terms
it's `void *`. The complete list of basic data types is available in the section
2.3.1 Data Type of the [UEFI Specification].

`EFI_SYSTEM_TABLE` is a structure as you could have guessed already. It's
defined in the section 4.3 EFI System Table of the [UEFI Specifcation]. It
starts with a header. The content of the header is not really important for us
because I'm not going to use it, but we need to get the size right, so I will
provide the complete definition:

```c
struct efi_table_header {
	uint64_t signature;
	uint32_t revision;
	uint32_t header_size;
	uint32_t crc32;
	uint32_t reserved;
};
```

With this header in place this is how I'll define the system table for now:

```c
struct efi_system_table {
	struct efi_table_header header;
	uint16_t *unused1;
	uint32_t unused2;
	void *unused3;
	void *unused4;
	void *unused5;
	struct efi_simple_text_output_protocol *out;
	void *unused6;
	void *unused7;
	void *unused8;
	void *unused9;
	uint64_t unused10;
	void *unused11;
};
```

Huh, not a very useful structure at this point. Well, it's because I'm
implementing a simple hello world, so we don't really need much from the EFI.
As a result most of the fields there are unused at the moment, so I've just put
placeholders there.

There is one field that I populated however and it's `out`. I will describe the
`efi_simple_text_output_protocol` structure shortly, but as a name suggest it
has something to do with putting some text out, which is useful if we want to
create a simple hello world.

```c
typedef uint64_t efi_status_t;
typedef uint64_t efi_uint_t;

struct efi_simple_text_output_protocol {
	efi_status_t (*unused1)(
			struct efi_simple_text_output_protocol *,
			bool);

	efi_status_t (*output_string)(
		struct efi_simple_text_output_protocol *self,
		uint16_t *string);

	efi_status_t (*unused2)(
		struct efi_simple_text_output_protocol *,
		uint16_t *);
	efi_status_t (*unused3)(
		struct efi_simple_text_output_protocol *,
		efi_uint_t, efi_uint_t *, efi_uint_t *);
	efi_status_t (*unused4)(
		struct efi_simple_text_output_protocol *,
		efi_uint_t);
	efi_status_t (*unused5)(
		struct efi_simple_text_output_protocol *,
		efi_uint_t);

	efi_status_t (*clear_screen)(
		struct efi_simple_text_output_protocol *self);

	efi_status_t (*unused6)(
		struct efi_simple_text_output_protocol *,
		efi_uint_t, efi_uint_t);
	efi_status_t (*unused7)(
		struct efi_simple_text_output_protocol *,
		bool);

	void *unused8;
};
```

As you can see the structure consists mostly of pointers to functions and that's
how you interact with EFI. The complete definition can be found in the section
12.4 Simple Text Output Protocol of the [UEFI Specification].

As before most of the functions are of no interest to us at the moment, so I've
only named two functions: `output_string` and `clear_screen`.

The names of the functions should be self explanatory, however we should stop
a little bit on why the `output_string` function takes a pointer to `uint16_t *`
instead of something like `const char *`.

Let's start from the `const` part missing. It appears to me that for whatever
reason the creators of the [UEFI Specification] just didn't bother to put any
const qualifiers into the specification at all, so lack of const doesn't
actually mean anything.

The `uint16_t` part is more interesting. In the specification they use `CHAR16`
type instead of `uint16_t`, which is defined as 2 byte character type. It seems
that accross the [UEFI Specification] they mostly use UCS-2 encoded strings. Why
they don't use something like UTF-8 or UTF-16 is not clear to me. I guess we
will have to live without Egyptian hieroglyphs and emojis for now, all the basic
latin, cyrillic and CJK characters are supported though.

With that out of the way, we can now write our hello world example:

```c
typedef void *efi_handle_t;

efi_status_t efi_main(
	efi_handle_t handle, struct efi_system_table *system_table)
{
	uint16_t msg[] = u"Hello World!";
	efi_status_t status;

	status = system_table->out->clear_screen(system_table->out);
	if (status != 0)
		return status;

	status = system_table->out->output_string(system_table->out, msg);
	if (status != 0)
		return status;

	return 0;
}
```

# Bulding

Now to the compilation and linking of the code. In LLVM ecosystem code is
initially translated into an intermediate byte code that is later translated
into machine code. That's not a novel idea even GCC is doing something very
similar.

We are interested in configuring the second step, the one that is responsible
for translating the intermediate byte code into the actual machine code, that
in the end we want to get a position independent code that follows microsoft
ABI.

It appears that the only thing we need to do is to specify LLVM the right target
platform via `-target=x86_64-unknown-windows` parameter. In addition to that a
couple of additional parameters might be needed: `-ffreestanding` and
`-mno-red-zone`.

> *NOTE:* it appears that we don't even need to tell LLVM separately that the
  code should be position independent, moreover parameters like `-fpic` aren't
  even recognized with the windows backend.

`ffreestanding` flag tells the compiler to compile the code for the freestanding
environment, as opposed to the hosted environment. What freestanding environment
means exactly according to language standards is to a significant extent
implementation defined. That being said, in practice it tells the compiler that
it cannot depend on the availability of many common library functions, like, for
example, memory allocation functions. If the code will not have access to common
OS services, like in our use case, you should specify this flag.

`mno-red-zone` is a x64 specific flag. It disables so called *red zone*, which
is in essense a form of optimization. You can read in more details about it in
the [Eli Bendersky article]. This optimization doesn't quite work well with
interrupt handling in freestanding environment, so for most of the
kernel/firmware code it should be disabled.

So compilation is not really that tricky, so let's move to linking. First, let
me remind you that earlier we installed the `lld` linker. That's because we need
a linker capable of generating a PE32+ binary. `lld` with the flag
`-flavor link` can do that.

Additionally we will have to tell the linker that we want to create an EFI
application. Format PE32+ supports a few different types of
executables/libraries, so we need to tell the linker that what we need in the
end is not a regular Windows executable or DLL, but an EFI application. That can
be done with `-subsystem:efi_application` flag.

Every executable has to have an etnry point. Normally, the entry point is known
to be a function with a specific name (like, main), so you don't need to
explicitly specify it. In our case however, this convention doesn't work and we
need to explicitly tell the linker what is our entry point using
`-entry:efi_main` flag.

> *NOTE:* you may find other examples on the Internet that also pass `-dll`
  flag. This flag tells the linker that they need to generate a DLL, which makes
  sense if the binary can be located anywhere in memory, but I found that this
  flag makes little difference and the code still works without it.

Here is how my Makefile ended up looking:

```makefile
CC := clang
LD := lld

CFLAGS := -ffreestanding -MMD -mno-red-zone -std=c11 \
	-target x86_64-unknown-windows
LDFLAGS := -flavor link -subsystem:efi_application -entry:efi_main

SRCS := main.c

default: all

%.o: %.c
	$(CC) $(CFLAGS) -c $< -o $@

bootx64.efi: main.o
	$(LD) $(LDFLAGS) $< -out:$@

-include $(SRCS:.c=.d)

.PHONY: clean all default

all: bootx64.efi
```

When the building finishes it should produce a file name `bootx64.efi`. We can
look at the file headers using `objdump` to check it's format:

```sh
objdump -x bootx64.efi

bootx64.efi:     file format pei-x86-64
bootx64.efi
architecture: i386:x86-64, flags 0x0000012f:
HAS_RELOC, EXEC_P, HAS_LINENO, HAS_DEBUG, HAS_LOCALS, D_PAGED
start address 0x0000000140001000

Characteristics 0x22
	executable
	large address aware

Time/Date		Sun Oct 11 14:13:26 2020
Magic			020b	(PE32+)
MajorLinkerVersion	14
MinorLinkerVersion	0
SizeOfCode		0000000000000200
SizeOfInitializedData	0000000000000200
SizeOfUninitializedData	0000000000000000
AddressOfEntryPoint	0000000000001000
BaseOfCode		0000000000001000
ImageBase		0000000140000000
SectionAlignment	00001000
FileAlignment		00000200
MajorOSystemVersion	6
MinorOSystemVersion	0
MajorImageVersion	0
MinorImageVersion	0
MajorSubsystemVersion	6
MinorSubsystemVersion	0
Win32Version		00000000
SizeOfImage		00003000
SizeOfHeaders		00000400
CheckSum		00000000
Subsystem		0000000a	(EFI application)
DllCharacteristics	00008160
SizeOfStackReserve	0000000000100000
SizeOfStackCommit	0000000000001000
SizeOfHeapReserve	0000000000100000
SizeOfHeapCommit	0000000000001000
LoaderFlags		00000000
NumberOfRvaAndSizes	00000010

The Data Directory
Entry 0 0000000000000000 00000000 Export Directory [.edata (or where ever we found it)]
Entry 1 0000000000000000 00000000 Import Directory [parts of .idata]
Entry 2 0000000000000000 00000000 Resource Directory [.rsrc]
Entry 3 0000000000000000 00000000 Exception Directory [.pdata]
Entry 4 0000000000000000 00000000 Security Directory
Entry 5 0000000000000000 00000000 Base Relocation Directory [.reloc]
Entry 6 0000000000000000 00000000 Debug Directory
Entry 7 0000000000000000 00000000 Description Directory
Entry 8 0000000000000000 00000000 Special Directory
Entry 9 0000000000000000 00000000 Thread Storage Directory [.tls]
Entry a 0000000000000000 00000000 Load Configuration Directory
Entry b 0000000000000000 00000000 Bound Import Directory
Entry c 0000000000000000 00000000 Import Address Table Directory
Entry d 0000000000000000 00000000 Delay Import Directory
Entry e 0000000000000000 00000000 CLR Runtime Header
Entry f 0000000000000000 00000000 Reserved

Sections:
Idx Name          Size      VMA               LMA               File off  Algn
  0 .text         000000c9  0000000140001000  0000000140001000  00000400  2**4
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
  1 .rdata        0000001a  0000000140002000  0000000140002000  00000600  2**4
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
SYMBOL TABLE:
no symbols
```

> *NOTE:* We can additionally disassemble the code in the file to make sure that
  it's position independent. However there isn't much code in the file to begin
  with and none of the code there will benefit in any way from being position
  dependent anyway. In other words, even if my building procedure isn't quite
  correct, we'd hardly will be able to see it on this simple example.

# Testing

Now it's time to try and run the binary inside the QEMU. We will start from
copying the binary to the root directory of the virtual machine:

```sh
mkdir -p root/efi/boot
cp bootx64.efi root/efi/boot
```

> *NOTE:* at this point we can actually use any path inside the root directory,
  so there is nothing magical about `efi/boot` specifically at this point.

To start the QEMU we need a command similar to this:

```sh
qemu-system-x86_64 \
  -drive if=pflash,format=raw,file=/home/kmu/ws/efi/OVMF.fd \
  -drive format=raw,file=fat:rw:root \
  -net none \
  -nographic
```

Most important options are the two `-drive` options. In general `drive` option
provide QEMU with information about various storage devices it need to emulate,
where storage is a rather broad term in this case.

`-drive if=pflash,format=raw,file=/home/kmu/ws/efi/OVMF.fd` option tells QEMU to
emulate a some kind of flash memory with content from the file
`/home/kmu/ws/efi/OVMF.fd`. The `OVMF.fd` file contains the image of the UEFI
firmaware that we installed earlier.

`-drive format=raw,file=fat:rw:root` option tells QEMU to simulate a disk-like
device that contains a readable and writable FAT filesystem. The interesting
thing is that it will use the `root` directory on the host computer as
filesystem storage, which makes it a rather convenient way to share state
between the host and the virtual machine.

The rest of the options are less relevant. I use the `-net none` to tell QEMU
that there is no network and that firmware should not try to boot the system
over network, but things should work even without this option. `-nographic`
option just disables the graphical screen and all the input/output will be done
via terminal. I do that because I often work without a mouse and it's more
convenient to interract via terminal then use touchpad.

The command above should start the virtual machine and eventually drop you into
the EFI shell. It will not start our EFI application automatically, so we need
to do that ourselves.

In the shell type this:

```sh
fs0:
cd efi/boot
bootx64.efi
```

The first command selects `fs0` volume. You need to do that before you can
perform any filesystem operations. `cd efi/boot` command is more or less self
explanatory. And the last command just starts the binary.

Our application clears the screen, outputs a string and exits. That should
happen pretty quickly, so you may not notice anything, except that the screen
has been cleared. You may want to change the code a little bit to put an
infinite loop before the last return in the `efi_main` to see the results of
your work.

> *NOTE:* a tip, backspace button doesn't work with `-nographic`, instead you
  shdoul use CTRL + H combination.

# Instead of conclusion

A hello world application is hardly anything to be proud of, but there is a lot
happening under the hood. It might look easy now, when Clang/LLVM supports
generating EFI binaries out of the box, but it wasn't always the case. So
appreciate and enjoy the good tooling available nowdays.
