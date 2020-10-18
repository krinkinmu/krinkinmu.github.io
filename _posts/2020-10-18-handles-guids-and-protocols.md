---
layout: post
title: UEFI handles, GUIDs and protocols
excerpt_separator: <!--more-->
tags: efi clang microsoft
---

[UEFI Specification]: https://uefi.org/sites/default/files/resources/UEFI%20Spec%202.8B%20May%202020.pdf "UEFI Specification"
[previous post]: {% post_url 2020-10-11-efi-getting-started %} "the previous post"


Continuing exploring UEFI bit by bit. This time I'll cover a small part about
UEFI handles, GUIDs and protocols. All the sources are available on
[GitHub.](https://github.com/krinkinmu/efi)

<!--more-->

# Handles and Protocols

The [UEFI specification] in section 2.3.1 Data Type gives a rather vauge
description of handles. What we know from the specification is that essentially
handles are a *void* pointer that represent a collection of related interfaces.

In the [previous post] I mentioned already that the entry point to the EFI
applications gets two arguments: handle and system table pointer. This handle
represents the image of the application loaded in memory.

What interfaces can an image of the application implement? According to the
[UEFI Specificaion] (section 4.1 UEFI Image Entry Point) all images support at
least `EFI_LOADED_IMAGE_PROTOCOL` and `EFI_LOADED_IMAGE_DEVICE_PATH_PROTOCOL`.
Those protocols are basically what the specification calls interfaces.

Each protocol has an associated data structure. For example, the structure for
the `EFI_LOADED_IMAGE_PROTOCOL` is defined in the section 9.1 EFI Loaded Image
Protocol of the [UEFI Specification] as follows:

```c
typedef struct {
	UINT32 Revision;
	EFI_HANDLE ParentHandle;
	EFI_SYSTEM_TABLE *SystemTable;

	// Source location of the image
	EFI_HANDLE DeviceHandle;
	EFI_DEVICE_PATH_PROTOCOL *FilePath;
	VOID *Reserved;

	// Image’s load options
	UINT32 LoadOptionsSize;
	VOID *LoadOptions;

	// Location where image was loaded
	VOID *ImageBase;
	UINT64 ImageSize;
	EFI_MEMORY_TYPE ImageCodeType;
	EFI_MEMORY_TYPE ImageDataType;
	EFI_IMAGE_UNLOAD Unload;
} EFI_LOADED_IMAGE_PROTOCOL;
```

In the structure you can find all kinds of related information, like the pointer
to the memory where the image has been loaded, how big it is and even where the
image was loaded from.

The `EFI_LOADED_IMAGE_PROTOCOL` is not supever representative because it doesn't
contain many function pointers in the structure compared to the amount of data -
something that you'd might not expect from an interface. That's not always the
case however. Similarly `EFI_LOADED_IMAGE_DEVICE_PATH_PROTOCOL` is not
particularly representative as it's mostly about the data than behavior.

In the [previous post] we actually saw a more representative interface
`EFI_SIMPLE_TEXT_OUTPUT_PROTOCOL`. The structure contained a lot of function
pointers. We got the pointer to the protocol from the system table pointer that
we get as the second argument of the entry point.

However the system table contains just a few pointers to the most basic
protocols, while UEFI supports much more. So how can we get access to them?

# Discovering protocols

If you already have a handle you can easily enumerate what protocols it supports
and get access to one. Let's look at a practical example and see what protocols
the image handle supports.

Before we start we should take a look at the EFI boot services. Access to the
EFI boot services is provided via the EFI Boot services table. In the
[UEFI Specification] the boot services table is defined in the section 4.4 EFI
Boot Services Table.

We don't need all the boot services at the moment, so for the purporse of our
example, here is how the table looks:

```c
struct efi_boot_services {
	struct efi_table_header header;

	// Task Priority Services
	void (*unused1)();
	...

	// Memory Services
	void (*unused3)();
	...
	void (*unused6)();
	efi_status_t (*free_pool)(void *);

	// Event & Timer Services
	void (*unused7)();
	...

	// Protocol Handler Services
	void (*unused13)();
	...
	void (*unused16)();
	void *reserved;
	void (*unused17)();
	...

	// Image Services
	void (*unused21)();
	...

	// Miscellaneius Services
	void (*unused26)();
	...

	// DriverSupport Services
	void (*unused29)();
	...

	// Open and Close Protocol Services
	void (*unused31)();
	...

	// Library Services
	efi_status_t (*protocols_per_handle)(
		efi_handle_t, struct efi_guid ***, efi_uint_t *);
	...

	// 32-bit CRC Services
	void (*unused39)();

	// Miscellaneius Services (cont)
	void (*unused40)();
	void (*unused41)();
	void (*unused42)();
};
```

There are plenty of functions there and I marked most of them except just two as
unused. The core of this example is the `protocols_per_handle` function. What it
does is takes a handle as input and tells us what protocols this handle
supports.

# GUIDs

As you can see from the interface of the `protocols_per_handle` function it
returns a table of `efi_guid` pointers.

> *NOTE:* one `*` is because it's an output parameter, another `*` is because
  the result is an array, and the final `*` is because the result is an array
  of pointers.

The `efi_guid` structure is defined as follows:

```c
struct efi_guid {
	uint32_t data1;
	uint16_t data2;
	uint16_t data3;
	uint8_t data4[8];
};
```

This structure is basically a 128 bit opaque unique identifier, so internal
structure doesn't really have much of a meaning AFACT. That's why the fields
are named so weirdly.

Each protocol has a unique `efi_guid` associated with it. So by looking at the
GUIDs that the `protocols_per_handle` returned we can tell which protocols the
particular handle supports.

> *NOTE:* It's also possible to do a reverse look up. So if you know what
  protocol you need, then using the GUID of the protocol you can find which
  handles support it. It's important to keep in mind however, that there might
  be multiple handles supporting the same protocol. For example, you system
  might have multiple storage devices that all support file system access
  protocols. So the fact that a handle supports the protocol you need doesn't
  mean that it's the right handle to use.

The table returned by the `protocols_per_handle` is allocated dynamically. It
means that once we are done with the table we will be responsible for telling
the firmware that this memory can be reclaimed.

There are a few ways in which you can dynamically allocate memory in EFI
application, and, therefore, there are differnt ways in which you can free the
allocated memory. The way we have to use for the results returned by the
`protocols_per_handle` function is the `free_pool` function. That's the second
function I kept in the boot services structure.

Finally, it's easy to guess what is the third paramater of the
`protocols_per_handle` function. It's the output parameter that will tell us how
many entires in the array that the `protocols_per_handle` function returned.

I will skip showing the actual code here that calls the `protocol_per_handle`
function and outputs the result because most of the code needed for that has to
do with formatting the output in the human readable way. The full code still can
be found on [GitHub](https://github.com/krinkinmu/efi) though.

> *NOTE:* There is no functions like `printf` available, there is only a
  function to output a string, so formatting the output is on us at this point.

# Testing

The [previous post] covers how to start the EFI application in QEMU with OVFM
firmware, so I'm not going to repeat it here again. Let's just look at the
outout I've got:

```
{ 0x752f3136, 0x4e16, 0x4fdc, { 0xa2, 0x2a, 0xe5, 0xf4, 0x68, 0x12, 0xf4, 0xca } }
{ 0xbc62157e, 0x3e33, 0x4fec, { 0x99, 0x20, 0x2d, 0x3b, 0x36, 0xd7, 0x50, 0xdf } }
{ 0x5b1b31a1, 0x9562, 0x11d2, { 0x8e, 0x3f, 0x0, 0xa0, 0xc9, 0x69, 0x72, 0x3b } }
```

In my case I got three different protocols. Two of them, and you can verify it
with the [UEFI Specification], are expected `EFI_LOADED_IMAGE_PROTOCOL` and
`EFI_LOADED_IMAGE_DEVICE_PATH_PROTOCOL`. However the remaining last one I could
not find in the [UEFI specification]. That suggests that firmware creators can
introduce their own GUIDs and their custom protocols as long as they do not
collide with the GUIDs defined in the specification.

# Instead of conclusion

This post gave a bried introduction into the UEFI handles, GUIDs and protocols.
Hopefully it was short and simple enought to understand how those three things
relate to each other. I did not provide an example of obtaining a protocol table
from a handle by a known GUID, but we will have opportunities to demonstrate
that in the future posts.
