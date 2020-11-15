---
layout: post
title: Loading an ELF image from EFI
excerpt_separator: <!--more-->
tags: efi clang microsoft elf
---

[UEFI Specification]: https://uefi.org/sites/default/files/resources/UEFI%20Spec%202.8B%20May%202020.pdf "UEFI Specification"
[previous post]: {% post_url 2020-10-31-efi-file-access %} "the previous post"
[EFI getting started]: {% post_url 2020-10-11-efi-getting-started %} "EFI getting started"
[ELF Specification]: https://refspecs.linuxbase.org/elf/elf.pdf "ELF Specification"
[System V ABI]: http://www.sco.com/developers/gabi/latest/contents.html "System V ABI"


Continuing exploring UEFI after some break. Last time I looked at file access,
now I'm going to read an ELF file from the file system, load it in memory and
transfer control to the ELF image.

As usual all the sources are available on
[GitHub](https://github.com/krinkinmu/efi).

<!--more-->

# Mithical ELF creature

What is ELF? ELF is a rather widely used executable (and not only) file format.
You might now already that EFI binaries use PE32+ file format, ELF is just
another file format that tries to solve similar problem.

Unlike PE32+ ELF is, at this point, basically the default file format for
executables and shared libraries. It's also often supported by other Unix-like
operating systems.

To understand how we can load correctly an ELF image in memory we'd have to
refer to the [ELF Specification].

If you bothered to follow the link and actually read through the specification
and specifically section Program Loading in the chapter 2. Program Loading and
Dynamic Linking, you might be very disappointed since the specification doesn't
really cover anything useful.

Instead of the [ELF Specification] you might refer to the base [System V ABI].
It covers the subject of loading ELF files marginally better.

ELF is a truly mythical creature - various accounts of it are confusing and/or
incomplete.

# Structure of ELF file

ELF files starts with a header. The header serves multiple purposes:

1. it helps to quickly identify a particular file as ELF file
2. it contains some basic information about this particular ELF file
3. pointers to other relevant parts of the file.

What kind of basic information should a header contain? Well, ELF files are
used for multiple purposes: executables, shared libraries, compiler object
files.

Besides that ELF format supports multiple different hardware/software platforms,
like 32-bit and 64-bit x86, 32-bit and 64-bit ARM and multiple others. So ELF
file also contains information about what kind of platform this file is for.

> *NOTE:* in this article I will only work with 64-bit version of ELF files.
  Covering both 32-bit and 64-bit will essentially double the amount of the code
  for very little benefit to the understanding.

Let's take a look at the header format:

```c
struct elf64_ehdr {
	unsigned char e_ident[16];
	uint16_t e_type;
	uint16_t e_machine;
	uint32_t e_version;
	uint64_t e_entry;
	uint64_t e_phoff;
	uint64_t e_shoff;
	uint32_t e_flags;
	uint16_t e_ehsize;
	uint16_t e_phentsize;
	uint16_t e_phnum;
	uint16_t e_shentsize;
	uint16_t e_shnum;
	uint16_t e_shstrndx;
};
```

The first four entries of the `e_ident` field must contain magic values: `0x7f`,
`'E'`, `'L'` and `'F'`. So checking the first four bytes allows to quickly
identify an ELF file. Other elements in `e_ident` allow to check if this
ELF file is 64-bit or 32-bit, if it's big-endian or little-endian, but those
don't seem to be that important.

The `e_type` field contains, as the name suggests, the type of the ELF file:
executable, shared library, object file, etc.

The `e_machine` field encodes the hardware architecture this file targets: x86, x86-64, arm, aarch64, etc.

I don't really know how the fields `e_version` or `e_flags` can be useful and
the code in the examples never refers to them, so I will not cover them in this
post.

One field that is actually quite important is `e_entry`. When you write program
code in a language like `C` you need to create function `main` that serves as
an entry point into your program - execution of your program starts from calling
function `main`.

> *NOTE:* especially nitpicky and knowledgable readers may say that constructors
  of global objects in `C++` are executed before `main`, some might say that
  even in `C` you can create a code that will be executed before `main`. For
  those readers, keep your pants on and skip the part where I explain what entry
  point is.

Execution of your program has to starts somewhere. In programming languages we
have various conventions that are used to denote the entry point of the program.
Once you build your program however there are no function names anymore, so you
need some other way to tell where the execution of the code should start.

The `e_entry` field contains the address of the first instruction of the
program. Once the ELF image loaded in memory, you should transfer execution by
this address to start executing the program.

Hopefully, you now understand the meaning of the `e_entry` field. The other
fields more or less point or describe other part of the ELF file. Most of them
are of no interest for our example, but fields `e_phoff` and `e_phnum` are of
immediate importance for our example.

Those two fields describe where in the ELF file the array of program headers is
stored and how big it is. ELF program header, according to the specification,
describes a segment of memory or some other piece of information the system
loading the ELF file need to prepare for the program to execute.

It's a rather vague description, so I'm going to make it a little bit more
specific by simplifing the problem a little bit. Program to run needs the code
and the data this code works with. When you build an executable the code and the
data is stored into the executable file. To load the program you need to read
them from the file into the right places in memory.

Essentially ELF program headers describe where in the ELF file code and data are
stored, and where we should put them in memory.

> *NOTE:* I mentioned that I simplify the problem a little bit here. Code and
  data aren't the only things that program headers can describe. Moreover not
  everything that program headers describe has to be loaded in memory. So there
  are quite a few different kinds of program headers and in the simplified
  version of the problem we don't need to care about all of them.

The structure of 64-bit ELF program header is as follows:

```c
struct elf64_phdr {
	uint32_t p_type;
	uint32_t p_flags;
	uint64_t p_offset;
	uint64_t p_vaddr;
	uint64_t p_paddr;
	uint64_t p_filesz;
	uint64_t p_memsz;
	uint64_t p_align;
};
```

There are multiple types of program headers, but in this simplified example we
only will care about `p_type == PT_LOAD` - just one type of program header. The
purpose of `PT_LOAD` program header is to exactly describe code and data the
program needs to run.

The `p_flags` field describes permissions of a particular segment in memory. For
example, some data might be read-only, code in memory have to be executable,
etc. In this example I will not care about setting up permissions, so `p_flags`
is of little relevance at this point.

`p_offset` and `p_filesz` describe where the code and data are stored inside
the ELF file. Similarly `p_vaddr` and `p_memsz` describe where the code and
data should be in memory.

It's important to note here that `p_filesz` does not have to be equal to
`p_memsz`. When `p_filesz` is smaller than `p_memsz` the rest of data in memory
should be filled with zeros. The idea here is that zero-initialized data is
quite common and there is no need to store zero inside the ELF file for no
reason if we can just fill the memory with zeros when we load the ELF file.

In this example I will assume that the code we will be loading is position
independent. Essentially, it would allow us to ignore the exact values of
`p_vaddr` field in the program header. It's however important for us to maintain
relative offsets between data and code in memory.

For example, if in the ELF file we have two `PT_LOAD` program headers that
describe two memory segments that need to be loaded at addresses `X` and `Y`
correspondingly. We can instead load them at addresses `X - D` and `Y - D`, so
addresses will not match the content of the program headers, but the distance
between the two segments in memory will stay the same. On the other hand loading
the two segments at addresses `X` and `Y - D` will break that requirement.

Important to note that don't load the segments in memory at addresses written
in program headers, when we transfer control to the program we will have to
adjust the value in `e_entry` accordingly.

# Loading ELF file

Here is a rather high level description of the loading process:

1. Read the ELF header and do some basic verification
2. Calculate how much memory we need for the program and allocate it
3. Copy data from the ELF file in memory
4. Call the entry point of the program.

In the [previous post] we covered basics of the file operations in EFI
environment, so I assume that the reader already familiar with it. What I didn't
cover there is how to actually read from a file in EFI. The signature of the
read function of the File Protocol is as follows:

```c
struct efi_file_protocol {
	...
	efi_status_t (*read)(struct efi_file_protocol *, efi_uint_t *, void *);
	...
};
```

We would additionally need to be able to read at a specific offset in the file.
The signature of the function to set position in the file is as follows:

```c
struct efi_file_protocol {
	...
	efi_status_t (*set_position)(struct efi_file_protocol *, uint64_t);
	...
};
```

Here is a helper function that reads the specified amount of data at the
specified offset in the file:

```c
static efi_status_t efi_read_fixed(
	struct efi_file_protocol *file,
	uint64_t offset,
	size_t size,
	void *dst)
{
	efi_status_t status;
	unsigned char *buf = dst;
	size_t read = 0;

	status = file->set_position(file, offset);
	if (status != EFI_SUCCESS)
		return status;

	while (read < size) {
		efi_uint_t remains = size - read;

		status = file->read(file, &remains, (void *)(buf + read));
		if (status != EFI_SUCCESS)
			return status;

		read += remains;
	}

	return EFI_SUCCESS;
}
```

> *NOTE:* the function above can handle short reads, but I don't really know if
  it's an issue in EFI environment.

With that out of the picture we can read the ELF header and verify it's content.
I will not provide the verification function here, mostly because I do not hope
that this verification function is complete. You can always find it in the
repository if you want though (it's called `verify_elf64_header` there).

In the code in the repository I read all the program headers in memory, it's
just easier to work with them when they are available in memory. To calculate
the required amount of memory we need to find the minimum value of `p_vaddr`
and the maximum value of `p_vaddr + p_memsz` among all `PT_LOAD` program
headers. The difference between those is how much memory we need.

> *NOTE:* there might be gaps in memory between segments, so it might make sense
  to avoid allocating a contigous segment of memory for the program, but it's
  much simpler to allocate one contigous segment of memory for the program, so
  that's what I'm going to do.

I'm also going to use page allocator to allocate memory for the program, so when
calculating minimum and maximum address I'm also aligning them to the page size
boundary, which is 4KiB.

Once the required amount of memory was calculated the actual allocation of
memory and loading is rather simple:

```c
static efi_status_t load_elf_image(struct elf_app *elf)
{
	uint64_t size = elf->page_size + (elf->image_end - elf->image_begin);
	uint64_t addr;
	uint16_t start_msg[] = u"Loading ELF image...\r\n";
	uint16_t finish_msg[] = u"Loaded ELF image\r\n";
	efi_status_t status;

	status = elf->system->out->output_string(elf->system->out, start_msg);
	if (status != EFI_SUCCESS)
		return status;

	// Allocate the required number of pages
	status = elf->system->boot->allocate_pages(
		EFI_ALLOCATE_ANY_PAGES, EFI_LOADER_DATA, size / elf->page_size, &addr);
	if (status != EFI_SUCCESS)
		return status;

	// Save some bookkeeping information for cleanup in case of errors
	elf->image_pages = size / elf->page_size;
	elf->image_addr = addr;

	// Entry point has to be adjusted, given that the ELF image might not
	// be loaded at the addresses stored in program headers
	elf->image_entry = elf->image_addr + elf->page_size
		+ elf->header.e_entry - elf->image_begin;

	// Fill in everything with zeros, whatever data is not read from the
	// ELF file itself has to be zero-initialized.
	memset((void *)elf->image_addr, 0, size);

	// Go over all program headers and load their content in memory
	for (size_t i = 0; i < elf->header.e_phnum; ++i) {
		const struct elf64_phdr *phdr = &elf->program_headers[i];
		uint64_t phdr_addr;

		if (phdr->p_type != PT_LOAD)
			continue;

		phdr_addr = elf->image_addr + elf->page_size
			+ phdr->p_vaddr - elf->image_begin;
		status = efi_read_fixed(
			elf->kernel, phdr->p_offset, phdr->p_filesz, (void *)phdr_addr);
		if (status != EFI_SUCCESS)
			return status;
	}

	return elf->system->out->output_string(elf->system->out, finish_msg);
}
```

In the code above `struct elf_app` is a bookkeeping structure that I created. It
mostly exists to group together various ELF file information and simplify
cleanup procedures:

```c
struct elf_app {
	struct efi_system_table *system;
	struct efi_file_protocol *kernel;
	struct elf64_ehdr header;
	struct elf64_phdr *program_headers;
	uint64_t image_begin;
	uint64_t image_end;
	uint64_t page_size;

	// Only populated when image is loaded into mamory
	uint64_t image_pages;
	uint64_t image_addr;
	uint64_t image_entry;
};
```

So far so good. There is quite a bit of code that needs to be created to read,
parse and load an ELF file in memory, but most of that code was rather
straightforward (assuming some knowledge of ELF). Now comes the tricky part.

The program is now in memory, it's time to transfer the control to the program.
I will assume that entry point of the program will point to a function that
takes no arguments and returns an `int`.

Additionally, I will assume that the calling convention used by the function is
`System V ABI` (this is commonly used calling convention in the Unix world).

Finally, for the sake of testing, I will assume that the program will return
back the number 42. That's basically just to verify that our code works: we will
pass control to the loaded image and then check the returned value. That's why
the entry point in this example returns `int`.

With those assumptions out the way, here is how the code look:

```c
static efi_status_t start_elf_image(struct elf_app *elf)
{
	uint16_t msg[] = u"Starting ELF image...\r\n";
	uint16_t fail[] = u"Starting ELF image failed\r\n";
	uint16_t success[] = u"ELF image successfully returned\r\n";
	int (__attribute__((sysv_abi)) *entry)(void);
	efi_status_t status;
	int ret = 0;

	status = elf->system->out->output_string(elf->system->out, msg);
	if (status != EFI_SUCCESS)
		return status;

	entry = (int (__attribute__((sysv_abi)) *)(void))elf->image_entry;
	ret = (*entry)();
	if (ret != 42) {
		elf->system->out->output_string(elf->system->out, fail);
		return EFI_LOAD_ERROR;
	}

	elf->system->out->output_string(elf->system->out, success);
	return EFI_SUCCESS;
}
```

Ignoring various output printing and attribute manipulations, the code is rather
simple essentially. It just converts the entry point address to a function
pointer and then calls the function using this pointer. The only magical part is
the attribute that tells the compiler that we use `System V ABI` calling
convention.

# Testing

Now we need an ELF binary to test. All we need from this binary is to return
42 and that's it. As a result the code is rather simple:

```c
int main(void)
{
	return 42;
}
```

Compiling this code is somewhat tricky however. While this code looks like a
regular `C` program we cannot compile it as such. We often this of `main` as an
entry point of our program and in many ways it's the case. However, compilers
tend to put some additional code in the binary that executes before `main` and
just calls `main` when it's done. In other words, `e_entry` in the ELF header
doesn't necessarily contain the address of the `main`. To make sure that it's
the case we have to compile and link the program in a somewhat special way.

Here are the commands I used to build the code above:

```sh
clang \
	-ffreestanding -MMD -mno-red-zone -std=c11 -Wall -Werror -pedantic \
	-c kernel.c \
	-o kernel.o
lld \
	-flavor link -subsystem:efi_application -entry:efi_main \
	kernel.o \
	-o kernel
```

> *NOTE:* In the repository those commands are in the `Makefile`, so you can
  just use `make` to build everything.

Before trying our EFI code we can explore the binary a little bit using
available commandline tools. For example, we can check the ELF header of the
file:

```sh
$ hexdump -n 4 -C kernel
00000000  7f 45 4c 46  |.ELF|
00000004
```

As you can see the first four bytes are `0x7f`, `'E'`, `'L'` and `'F'`. Which is
the expected magic at the beginning of the ELF file.

There are also dedicated tools that parse ELF files. For example, here is how
you can take a look at the ELF header in a more human friendly way:

```sh
$ readelf -h kernel
ELF Header:
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00 
  Class:                             ELF64
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              EXEC (Executable file)
  Machine:                           Advanced Micro Devices X86-64
  Version:                           0x1
  Entry point address:               0x201120
  Start of program headers:          64 (bytes into file)
  Start of section headers:          488 (bytes into file)
  Flags:                             0x0
  Size of this header:               64 (bytes)
  Size of program headers:           56 (bytes)
  Number of program headers:         4
  Size of section headers:           64 (bytes)
  Number of section headers:         6
  Section header string table index: 4
```

From the header you can find that there are example four program header in the
resulting binary and they are located starting from the byte 64 from the
beginning of the file.

We can also explore the program headers themselves:

```sh
$ readelf -l kernel

Elf file type is EXEC (Executable file)
Entry point 0x201120
There are 4 program headers, starting at offset 64

Program Headers:
  Type           Offset             VirtAddr           PhysAddr
                 FileSiz            MemSiz              Flags  Align
  PHDR           0x0000000000000040 0x0000000000200040 0x0000000000200040
                 0x00000000000000e0 0x00000000000000e0  R      0x8
  LOAD           0x0000000000000000 0x0000000000200000 0x0000000000200000
                 0x0000000000000120 0x0000000000000120  R      0x1000
  LOAD           0x0000000000000120 0x0000000000201120 0x0000000000201120
                 0x000000000000000b 0x000000000000000b  R E    0x1000
  GNU_STACK      0x0000000000000000 0x0000000000000000 0x0000000000000000
                 0x0000000000000000 0x0000000000000000  RW     0x0

 Section to Segment mapping:
  Segment Sections...
   00     
   01     
   02     .text 
   03
```

You can see that while there are four program headers in the file only two of
them are `PT_LOAD`. Also pay attention to flags for the two `PT_LOAD` program
headers: only one of them is executable. So you can probably figure out which
one of them is for data and which one of them is for code.

To actually test our code with this binary you need to put the `kernel` binary
together with the `bootx64.efi` file in the root file system. You can refer for
details to the [EFI getting started] post.

If everything goes as expected you should be able to see the
`ELF image successfully returned` message on the screen. That's what the
`start_elf_image` function prints when the loaded image returns control back.

# Instead of conclusion

We now can load ELF binaries from UEFI application and give them control. This
post is probably far from complete coverage of ELF file format, but it's
functional and can be converted into a normal OS loaded, which is kind of cool.

