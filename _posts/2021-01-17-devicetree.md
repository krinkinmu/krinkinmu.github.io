---
layout: post
title: An Introduction to Devicetree specification
excerpt_separator: <!--more-->
tags: devicetree system-programming
---

[Devicetree Specification]: https://github.com/devicetree-org/devicetree-specification/releases/download/v0.3/devicetree-specification-v0.3.pdf "Devicetree Specification"

Devicetree is a configuration commonly used to describe hardware present in
various platforms. In Linux Devicetree is used for ARMs, MIPSes, RISC-V,
XTensa and PowerPC (and probably others).

In this post I'm going to cover the problem that Devicetree is trying to
solve, briefly touch on the available alternatives and finally show some code
for parsing the binary representation of the Devicetree (a. k. a. Flattened
Device Tree or DTB).

All the sources are available on
[GitHub](https://github.com/krinkinmu/aarch64).

<!--more-->

# Introduction

When you boot your PC it somehow figures out how many CPUs the system has,
how much memory is installed in the system, it figures out what HDD or SSD
connected to the system, that there is a keyboard and a mouse, etc.

Likely you don't need to install a new OS when you add more memory to the
system or when you change a mouse. So the same OS can serve multiple different
variants of hardware. To put it another way OS doesn't always know in advance
what hardware it runs on and has in some way discover that information.

There are multiple ways OS can discover that information. For example, most of
external devices directly or indirectly connect to the system via PCI and USB.
Those two support means to automatically enumerate connected devices and find
out some basic information about them. Using this basic information OS can
figure out if it has a driver that supports the connected devices and loads it.

However, CPUs and memory are not some kind of PCI or USB devices. PCI and USB
host controllers cannot depend on PCI and USB enumeration mechanisms either.
So for those a different mechanism have to be used.

In simplest cases OS and drivers can just probe for the connected devices.
That basically means that a driver can try to interact with the device in
hope that it's there. If the device responds, then it must be there, and it
doesn't respond, then it's not there. It doesn't work though with things like
memory.

When the simplest method doesn't work, in the PC world the problem can be
addressed via BIOS, UEFI, ACPI or a combination of those. They provide various
interfaces to explore the available memory and basic hardware.

However hardware platforms in the PC world are more or less standartized. In
the embedded systems worlds there is a bit more variety and there an
alternative configuration interface became popular: [Devicetree Specification].

Historically [Devicetree Specification] originates from
[Open Firmware](https://en.wikipedia.org/wiki/Open_Firmware) which for most
intents and purposes is long gone now, but the [Devicetree Specification] is
alive and well.

# Devicetree Overview

Essentially, the [Devicetree Specification] describes a data description
format, like XML or json, but designed for the purporse of describing
hardware. As you could have figured out from the name, the description
forms a sort of tree structure.

You can think of a root node as a node that describes the hardware platform
as a whole. It must have at least one `memory` children node that describe
the available memory in the system and one `cpus` node that enumerates the
CPUs available in the system as the children.

For other types of hardware the tree structure like this also generally works
well. For example, for buses that do not support discovery, like I2C, you can
have a node for an I2C controller and describe the devices connected to the
I2C controller as children nodes of the controller node and so on.

# DTS

The [Devicetree Specification] provides two formats for the description:
a human readable/writable text format (DTS) and binary format (DTB or FDT).
The idea is that when you create a tree description you use the DTS, then
you compile DTS to DTB and pass the DTB to the OS.

Here is how you can install the DeviceTree Compiler (`dtc`):

```sh
sudo apt-get install device-tree-compiler
```

After successful installation you should have `dtc` binary available. This
binary can be used to convert between DTS and DTB formats.

Let's take a look at an example. We can ask QEMU to dump the tree for the
hardware it simulates in DTB format like this:

```sh
qemu-system-aarch64 -machine virt,dumpdtb=virt.dtb -cpu max
```

In the command above `-machine virt` part specifies a particular virtual
hardware configuration that QEMU supports. And the `dumpdtb=virt.dtb` asks
QEMU to store the description of that hardware configuration to the `virt.dtb`
file.

We can then convert this file from DTB to DTS format using the compiler:

```sh
dtc -I dtb -O dts virt.dtb
```

This command will parse `virt.dtb` as DTB and output the same data in DTS
format to the stdout. The flag `-I` is used to specify the input format and
`-O` is used to specify the output format.

The QEMU generates a reasonably useful hardware platform, so the resulting
Devicetree is quite large, so let's create our own Devicetree for testing and
to get familiar with the DTS format a little bit.

Before we start, DTS format appear to be versioned and the current version is
1. There was only one version of DTS ever, but we still have to start DTS file
with a version clause:

```
/dts-v1/;
```

The version clause can follow with a few memory reservation clauses. I will
cover memory reservations a bit further down the post, but for now it's
sufficient to note that memory reservations just describe a range of memory
addresses. I will introduce three memory reservation just for testing:

```
/dts-v1/;
/memreserve/ 0x40000000 0x1000;
/memreserve/ 0x40002000 0x1000;
/memreserve/ 0x40004000 0x1000;
```

Then goes the root node of the tree. The root node uses a format slightly
different compared to other nodes because it doesn't have a name:

```
/dts-v1/;
/memreserve/ 0x40000000 0x1000;
/memreserve/ 0x40002000 0x1000;
/memreserve/ 0x40004000 0x1000;
/ {
}
```

The rest of the description should go inside the root node. The [Devicetree
Specification] mandate a few required properties for the root node:

* `#address-cells`
* `#size-cells`
* `model`
* `compatible`

I'm only going to cover `#address-cells` and `#size-cells` for now, because
`dtc` cannot work without them and at the same time is fine when the other
two are missing.

Essentially, in the [Devicetree Specficiation] property values can have a few
possible types. For example, they can be arrays of 32-bit values (cells) or
zero-terminiated strings (though you cannot see zero-terminal in DTS format).
For addresses and sizes cells are commonly used, and for text properties
zero-terminated strings are used.

For example, if you have some memory mapped device, then registers of that
device are mapped to some place in memory. To communicate with the device,
drivers and OS kernels want to know where exactly the device registers are
mapped. So the node describing the device inside the tree would need to have
a property that contains the address of the memory where the device registers
are mapped.

Here you might ask, but what if my device has more than 4GiB of memory? 32-bit
value is not enough to describe that, right?

That's where the `#size-cells` and `#address-cells` properties come into play.
They basically describe how many 32-bit cells do we need to describe size and
addresses. For 64-bit systems you'd need two cells to describe an address in
memory.

To take it a bit further, addresses and sizes don't necessarily have to refer to
addresses in memory. For example, I2C devices have addresses in context of I2C
bus. Those are not memory addresses, but addresses nontheless. So what address
means in a particular case depends on the node and its place in the tree.
Consequently, you can have `#address-cells` and `#size-cells` in multiple
nodes. Those parameters apply to all the children nodes of the node that
contains them, unless overridden somewhere down the tree.

Let's return to the example. For now we are working with memory addresses, so
I will use two cells to specify addresses and sizes to support 64-bit memory:

```
/dts-v1/;
/memreserve/ 0x40000000 0x1000;
/memreserve/ 0x40002000 0x1000;
/memreserve/ 0x40004000 0x1000;
/ {
	#address-cells = <0x02>;
	#size-cells = <0x02>;
};
```

Now, let's move to the first child node: `memory`. According to the
[Devicetree Specification] we must have at least one `memory` node in the
tree:

```
/dts-v1/;
/memreserve/ 0x40000000 0x1000;
/memreserve/ 0x40002000 0x1000;
/memreserve/ 0x40004000 0x1000;
/ {
	#address-cells = <0x02>;
	#size-cells = <0x02>;

	memory@40000000 {
		reg = <0x00 0x40000000 0x00 0x8000000>;
		device_type = "memory";
	};
};
```

The node names generally contain two parts: name and the unit address. Those
two parts are separated by `@` character. The full name is known as unit name.
In the example above we have `memory` node with unit name `memory@40000000`.
As far as I can tell, the unit address isn't actually required and normally
isn't used for anything. In other words, unit address seem to be there solely
for the purpopse of disambiguating node names when you have more than one node
of the same type.

`memory` nodes have two required parameters:

* `device_type` - must be zero-terminated string `memory`;
* `reg` - an array of cells containing the address and size of the memory
  region.

As you can see in the example, the value of `reg` property contains four cells.
It's because the `reg` property value should contain a pair: address and size.
And according to the values of the `#address-cells` and `#size-cells`
properties we use two cells to describe both. All-in-all, we have four cells.

When putting everything together, the `memory` node above describes an address
range that starts at the address `0x40000000` that is `0x8000000` bytes long.

Another mandatory node is `cpus`. I will describe a system with just one ARM
cpu, so let's take a look at the example:

```
/dts-v1/;
/memreserve/ 0x40000000 0x1000;
/memreserve/ 0x40002000 0x1000;
/memreserve/ 0x40004000 0x1000;
/ {
	#address-cells = <0x02>;
	#size-cells = <0x02>;

	memory@40000000 {
		reg = <0x00 0x40000000 0x00 0x8000000>;
		device_type = "memory";
	};

	cpus {
		#address-cells = <0x01>;
		#size-cells = <0x00>;

		cpu@0 {
			reg = <0x00>;
			compatible = "arm,cortex-a57";
			device_type = "cpu";
		};
	};
};
```

`#address-cells` and `#size-cells` are mandatory properties for the `cpus`
node. To understand what values we assign to them we need to understand what
address space we are working with.

In this address space address is just an identifier of a CPU. And each CPU
occupies just one unit of the address space, so the size is always 1 and we
don't need to specify it explicitly.

That's why the [Devicetree Specification] tells that the `#size-cells` shall
contain 0, as in the example above. And one 32-bit cell is plenty enough to
contain the CPU identifier, thus `#address-cells` contains 1.

Each CPU is described by a child `cpu` node. As with `memory` node it has a
few required properties:

* `device_type` - must contain `cpu`;
* `reg` - unique CPU identifier (address);
* `clock-frequency`;
* `timebase-frequency`.

Even though, the [Devicetree Specification] says that `clock-frequency` and
`timebase-frequency` are required properties, the compiler eats the DTS even
without them, so I've skipped them for now.

# DTB/FDT

The name DTB stands for DeviceTree Blob. FDT stands for Flattened DeviceTree
and it's just another name for the same binary format.

The binary format is what I care about the most. Normally the OS kernels work
with DTB and not DTS. I want to use DTB to pass information to my toy
hypervisor.

In the previous section we created a simple and not particularly useful DTS
that has one memory node, one CPU and three reserved memory ranges (even
though they are not part of the tree itself).

I assume that the DTS is saved in file `test.dts`. Let's first compile this
DTS into a DTB and store the result to `test.dtb`:

```sh
dtc -I dts -O dtb test.dts > test.dtb
```

Each DTB basically contains four parts in the following order:

* header
* memory reservation block
* structure block - contains the description of the tree
* strings block - contains text data referenced from the structure block

## Header

The DTB must start with a header:

```rust
struct FDTHeader {
    magic: u32,
    totalsize: u32,
    off_dt_struct: u32,
    off_dt_strings: u32,
    off_mem_rsvmap: u32,
    version: u32,
    last_comp_version: u32,
    boot_cpuid_phys: u32,
    size_dt_strings: u32,
    size_dt_struct: u32,
}
```

> *NOTE:* I will use Rust in the examples and Rust doesn't specify the
  internal layout of the structures, but as will be shown below, I don't
  really depend on any particular internal layout of Rust structures.

The header describes the overall layout of DTB. Fields `off_dt_struct`,
`off_dt_strings` and `off_mem_rsvmap` contains offsets from the beginning of
DTB to the memory reservation, structure and strings blocks. `size_dt_strings`
and `size_dt_struct` contain sizes of structure and strings blocks in bytes.

There might be gaps between the blocks and some systems use those gaps to
dynamically manipulate the structure. For example, to add more reserved memory
ranges.

`magic` field must contain `0xd00dfeed` (big-endian) to indicate that the data
is indeed in DTB format. As for the rest of the fields, I will refer you to
the [Devicetree Specification].

## Memory Reservations

We saw above how we can describe memory available on the platform in DTS format.
However for various purposes we may want to reserve parts of that memory and
tell the OS that to not use those.

Imagine a situation when firmware needs some memory to store firmware related
data. If OS need to call the firmware we need to keep the firmware data in the
working state. In such case firmware can mark the memory it needs inside the
devicetree as reserved to tell the OS to stay away from it.

The block describing the reserved memory ranges contains an array of
structures like this:

```rust
struct ReservedMemory {
    addr: u64,
    size: u64,
}
```

The array begins at the offset `off_mem_rsvmap` from the beginning of DTB.
The array must have a zero-filled entry as a terminating entry - that's how
you can find the end of the array.

> *NOTE:* the header doesn't contain enough information to find out where the
  array ends, so we have to use other means for that.

## Structure Block

The structure block contains a flattened representation of the tree of nodes
starting with the root node. Logically, it can be viewed as a sequency of
pieces. Each piece starts with a 32-bit token value that tells us what kind
of piece it is. After the token there might be some additional data, depending
on the token.

In total there are five different tokens:

* `FDT_BEGIN_NODE` with value `0x01` - marks the beginning of a node;
* `FDT_END_NODE` with value `0x02` - marks the end of a node;
* `FDT_PROP` with value `0x03` - indicates a node propery;
* `FDT_NOP` with value `0x04` - means nothing and should be ignored;
* `FDT_END` with value `0x09` - marks the end of the structure block.

Each token must be aligned on the 4-byte aligned boundary and, when
necessary data is padded with `\0` symbols to enforce the alignment.

The last two tokens are more or less self explanatory, so I'm not going to
spend time on them (see the implementation below for details of the [Devicetree
Specification]).

`FDT_BEGIN_NODE` marks the beginning of a node description. Each node except
the root has a name (unit name). The name of the node is stored as a
null-terminated string right after the `FDT_BEGIN_NODE` token. Since the root
node doesn't have a name, the token is followed by just `\0` symbol. After
the node name there might be zero or more `\0` symbols to make sure that the
next token is aligned on the 4-bytes boundary.

As you saw above inside the node we can have properties or other nodes.
Description of node children and properties goes after the `FDT_BEGIN_NODE`
token and its data.

Children nodes are described in a recursive way, each of the children starts
with the `FDT_BEGIN_NODE` token and ends with the `FDT_END_NODE`, potentially
containing properties and other nodes between them.

Unlike `FDT_BEGIN_NODE`, `FDT_END_NODE` doesn't have any data.

Since `FDT_BEGIN_NODE` and `FDT_END_NODE` mark the boundaries of a node,
`FDT_BEGIN_NODE` and `FDT_END_NODE` must always go in pairs, like parenthesis
in a correct arithmetic expression.

Each property starts with `FDT_PROP` token. The `FDT_PROP` token is then
followed by two 32-bit values: name offset and length of the property value.

The name offset is the offset of the null-terminated string that contains the
property name inside the strings block. The value of the property follows right
after and, again, padded with `\0` symbols until the next 4-bytes aligned
boundary.

The description is somewhat wordy and it might be hard to figure out what's
going on from just reading it. So let's move to implement the DTB parser to
see how the pieces fit together.

# DTB Scanner

Before we start with the actual DTB parsing let's create a helper utility to
extract values from bytes. We need to be able to extract 32-bit values,
64-bit values, null-terminated strings, data blobs of fixed size. And
additionaly we need to align the position inside the data stream on the
4-bytes boundary. So that's exactly what I'm going to implement here.

I'll call the tool that does it `Scanner`, let's take a look:

```rust
struct Scanner<'a> {
    data: &'a [u8],
    offset: usize,
}

impl<'a> Scanner<'a> {
    pub fn new(data: &'a [u8]) -> Scanner<'a> {
        Scanner{ data, offset: 0 }
    }
}
```

In the snippet above `data` is our data stream and it's represented as a
slice of `u8` values. The `offset` is the position in the stream - that's the
current state of the `Scanner`.

Before we move to the actual implementation one thing to keep in mind is that
all integer values in DTB use Big-endian according to the [Devicetree
Specification].

With that in mind, let's move to the implementation starting with parsing
32-bit and 64-bit integer values from the stream:

```rust
use core::convert::TryFrom;
use core::result::Result;

impl<'a> Scanner<'a> {
    ...
    pub fn consume_be32(&mut self) -> Result<u32, &'static str> {
        if self.offset + 4 > self.data.len() {
            return Err("Not enough data in the stream for be32.");
        }

        let value = &self.data[self.offset..self.offset + 4];
        match <[u8; 4]>::try_from(value) {
            Ok(v) => {
                self.offset += value.len();
                Ok(u32::from_be_bytes(v))
            },
            Err(_) => Err("Error while parsing be32."),
        }
    }

    pub fn consume_be64(&mut self) -> Result<u64, &'static str> {
        if self.offset + 8 > self.data.len() {
            return Err("Not enough data in the stream for be64.");
        }

        let value = &self.data[self.offset..self.offset + 8];
        match <[u8; 8]>::try_from(value) {
            Ok(v) => {
                self.offset += value.len();
                Ok(u64::from_be_bytes(v))
            },
            Err(_) => Err("Error while parsing be64."),
        }
    }
    ...
}
```

The core of the snippet above are `u64::from_be_bytes` and
`u32::from_be_bytes` functions that take as input arrays of `u8` values and
parse them as 64-bit or 32-bit Big-endian numbers.

The `try_from` function basically serves as type cast to convert from slices
to arrays.

The rest of the code is error checking and updating the `offset` field to
move further in the byte stream.

The next on the line is parsing null-terminated string:

```rust
use core::str;

impl<'a> Scanner<'a> {
    ...
    pub fn consume_cstr(&mut self) -> Result<&'a str, &'static str> {
        for i in self.offset.. {
            if i >= self.data.len() {
                return Err("Failed to find terminating '\0' in the stream.");
            }

            if self.data[i] != 0 {
                continue;
            }

            match str::from_utf8(&self.data[self.offset..i]) {
                Ok(s) => {
                    self.offset = i + 1;
                    return Ok(s);
                },
                Err(_) => return Err("Not a valid UTF8 string."),
            }
        }
	Err("Unreachable")
    }
    ...
}
```

In the snippet above we just go through the data until we find a zero. Then
we take all the data up to the zero character we found and try to interpret
it as a `utf8` string and convert to `str` using `std::from_utf8`. The
[Devicetree Specification] is rather restrictive on the characters possible
in the property names, so all of them should be possible to interpret as
`utf8` strings.

Now let's take a look at consuming fixed-size data. It's much simpler than
`consume_cstr` because we don't need to find the null-terminator or convert
data into a string:

```rust
impl<'a> Scanner<'a> {
    ...
    pub fn consume_data(&mut self, size: usize) -> Result<&'a [u8], &'static str> {
        if self.offset + size > self.data.len() {
            return Err("Not enough data in the stream.");
        }

        let begin = self.offset;
        let end = begin + size;
        self.offset += size;

        Ok(&self.data[begin..end])
    }
    ...
}
```

And finally we need to be able to align the position in the stream on a
4-bytes boundary. I will pass the alignment as a parameter instead of
hardcoding the 4 bytes value:

```rust
impl<'a> Scanner<'a> {
    ...
    pub fn align_forward(&mut self, alignment: usize) -> Result<(), &'static str> {
        if alignment == 0 || self.offset % alignment == 0 {
            return Ok(());
        }

        let shift = alignment - self.offset % alignment;

        if self.offset + shift >= self.data.len() {
            return Err("Not enough data in the stream.");
        }

        self.offset += shift;
        Ok(())
    }
    ...
}
```

That's quite a bit of code, but all of that more or less straightforward.

# Device Tree Representation

Before moving forward with the parsing of DTB I need to describe what result
the parser will return. What I want to do is to create a tree-like structure
that represents the tree encoded in DTB and make parser return it.

Let's start with the tree node. Each node of the tree contains properties and
other nodes:

```rust
use alloc::collections::btree_map::BTreeMap;
use alloc::string::String;
use alloc::vec::Vec;

struct DeviceTreeNode {
    children: BTreeMap<String, DeviceTreeNode>,
    properties: BTreeMap<String, Vec<u8>>,
}
```

The data structure by itself is not super useful. We also need to have some
interface to modify and explore the nodes:

```rust
use core::options::Options;

impl DeviceTreeNode {
    pub fn child(&self, name: &str) -> Option<&DeviceTreeNode> {
        self.children.get(name)
    }

    pub fn children(&self) -> Children {
        Children{ inner: self.children.iter() }
    }

    pub fn property(&self, name: &str) -> Option<&[u8]> {
        self.properties.get(name).map(|v| v.as_slice())
    }

    pub fn properties(&self) -> Properties {
        Properties{ inner: self.properties.iter() }
    }

    fn new() -> DeviceTreeNode {
        DeviceTreeNode{
            children: BTreeMap::new(),
            properties: BTreeMap::new(),
        }
    }

    fn add_child(&mut self, name: String, child: DeviceTreeNode) {
        self.children.insert(name, child);
    }

    fn remove_child(&mut self, name: &str) -> Option<DeviceTreeNode> {
        self.children.remove(name)
    }

    fn add_property(&mut self, name: String, value: Vec<u8>) {
        self.properties.insert(name, value);
    }
}
```

> *NOTE:* I made non-modifing functions public as they will serve as the
  interface for the users of the library. All the modifing function will be
  used during parsing only and not intended for use by anybody else, so they
  are not public.

The snippet above omits the definition of `Children` and `Properties`. Those
are just simple iterator wrapped around the `BTreeMap` iterator. I'm not
totally sure that wrapping will serve any purpose yet, so I'm not going to
cover that part here. You can find the complete code on
[GitHub](https://github.com/krinkinmu/aarch64).

Another piece that we need to represent parsed DTB is reserved memory ranges.
I already provided the structure for the reserved memory above, so let's not
spend any more time on that.

Finally, I want to introduce another piece that combines all things together
and provides functions to lookup nodes in the tree. Let's take a look:

```rust
pub struct DeviceTree {
    reserved: Vec<ReservedMemory>,
    root: DeviceTreeNode,
    last_comp_version: u32,
    boot_cpuid_phys: u32,
}

impl DeviceTree {
    pub fn new(
            reserved: Vec<ReservedMemory>,
            root: DeviceTreeNode,
            last_comp_version: u32,
            boot_cpuid_phys: u32) -> DeviceTree {
        DeviceTree{ reserved, root, last_comp_version, boot_cpuid_phys }
    }
}
```

It's simple so far. I've just put a few piecies that we can find in the DTB
header, the vector with reserved memory ranges and the root node. At the
moment we probably only care about the root node and reserved memory ranges,
but when I wrote the code I thought that the other two might become useful in
the future.

Let's take a look at the helper functions that would make this structure
useful. Those functions will provide access to various pieces and I'll start
with the simplest one - reserved memory ranges:

```rust
impl DeviceTree {
    ...
    pub fn reserved_memory(&self) -> &[ReservedMemory] {
        self.reserved.as_slice()
    }
    ...
}
```

There isn't much to explain here - it just returns the reserved ranges in a
slice.

Let's take a look at a function that is more interesting:

```rust
impl DeviceTree
    ...
    pub fn follow(&self, path: &str) -> Option<&DeviceTreeNode> {
        let mut current = &self.root;

        if path == "/" {
            return Some(current);
        }

        for name in path[1..].split("/") {
            if let Some(node) = current.child(name) {
                current = node;
            } else {
                return None;
            }
        }

        Some(current)
    }
    ...
}
```

This function takes a string representing a path in a tree and returns the
node corresponding to this path. For example, for the CPU in our test DTB,
that we created above, the path will be `/cpus/cpu@0` and for the memory node
in the same DTB it will be `/memory@40000000`. To get access to the root node
we should pass just `/`. That's just a convient way to identify a node in the
tree.

# DTB Parser

With all the preparations out of the way we can actually parse the DTB now.
And I'll start from the beginning of the DTB - it's header:

```rust
fn parse_header(fdt: &[u8]) -> Result<FDTHeader, &'static str> {
    let mut scanner = Scanner::new(fdt);
    let magic = scanner.consume_be32()?;
    let totalsize = scanner.consume_be32()?;
    let off_dt_struct = scanner.consume_be32()?;
    let off_dt_strings = scanner.consume_be32()?;
    let off_mem_rsvmap = scanner.consume_be32()?;
    let version = scanner.consume_be32()?;
    let last_comp_version = scanner.consume_be32()?;
    let boot_cpuid_phys = scanner.consume_be32()?;
    let size_dt_strings = scanner.consume_be32()?;
    let size_dt_struct = scanner.consume_be32()?;

    Ok(FDTHeader{
        magic,
        totalsize,
        off_dt_struct,
        off_dt_strings,
        off_mem_rsvmap,
        version,
        last_comp_version,
        boot_cpuid_phys,
        size_dt_strings,
        size_dt_struct,
    })
}
```

> *NOTE:* above I mentioned that I don't need to depend on any particular
  internal layout of Rust structures. `Scanner` is the tool that allows to do
  that, all we need is to call the `Scanner` methods in the right order.

Parsing the list of reserved ranges is also quite straighforward:

```rust
fn parse_reserved(data: &[u8]) -> Result<Vec<ReservedMemory>, &'static str> {
    let mut scanner = Scanner::new(data);
    let mut reserved = Vec::new();

    loop {
        let addr = scanner.consume_be64()?;
        let size = scanner.consume_be64()?;

        if addr == 0 && size == 0 {
            break;
        }
        reserved.push(ReservedMemory{ addr, size });
    }

    Ok(reserved)
}
```

The function above assumes that the list of reserved memory ranges starts
right at the beginning of the `data` parameter, so we cannot just pass it
the whole DTB - we need to find the offset of the reserved memory ranges list
in the DTB and pass it only that part.

Now let's move to the fun part - parsing the actual tree from the DTB. Since
the DTB encodes a tree some kind of recursive algorithm or additional memory
is required.

I will use additional memory instead of recursion here, so let me introduce a
structure that I will use to keep track of the current state:

```rust
struct State<'a> {
    parents: Vec<(&'a str, DeviceTreeNode)>,
    current: DeviceTreeNode,
}
```

The `current` field will store the `DeviceTreeNode` strcture for the latest
node we discovered. The `parents` field is a stack of all the parents of the
`current`. Every time we will discover a new node, we will push the `current`
node on the stack and create a new node. Every time a node ends, we will take
the parent from the stack and make it the `current` node.

Let's take a look:

```rust
impl<'a> State<'a> {
    fn new() -> State<'a> {
        State{
            parents: Vec::new(),
            current: DeviceTreeNode::new(),
        }
    }
    ...
}
```

In the initial state is the `parents` stack is empty. I create a dummy node
and store it in `current` to avoid handling corner case of the first node -
we will have to keep that in mind for later.

When the code discovers a new node it should push the current node on stack
and create a new node and store it as `current`:

```rust
impl<'a> State<'a> {
    ...
    fn begin_node(&mut self, name: &'a str) {
        let node = mem::replace(&mut self.current, DeviceTreeNode::new());
        self.parents.push((name, node));
    }
    ...
}
```

> *NOTE:* `mem::replace` replaces the value of `self.current` with a new
  value and return the old one to the caller.

In the code above I also store the name of the current node on stack together
with the parent. We will use this name later.

When we discover the end of node token we need to do a somewhat opposite
operation:

```rust
impl<'a> State<'a> {
    ...
    fn end_node(&mut self, name: &'a str) -> Result<(), &'static str> {
        if let Some((name, parent)) = self.parents.pop() {
            let node = mem::replace(&mut self.current, parent);
            self.current.add_child(String::from(name), node);
            return Ok(());
        }
        Err("Unmatched end of node token found in FDT.")
    }
    ...
}
```

The `end_node` is slightly more complicated than `begin_node` for two reasons:

* we need to handle errors (when the stack is empty);
* we need to add the node we close as a child to the parent.

We we discover a property in DTB we just need to add it to the current node:

```rust
impl<'a> State<'a> {
    ...
    fn new_property(&mut self, name: &str, value: &[u8]) {
        self.current.add_property(String::from(name), Vec::from(value));
    }
    ...
}
```

Finally, remember that we started with a fake node in `current`? When parsing
is done we need to take that into account and remove that fake node:

```rust
impl<'a> State<'a> {
    ...
    fn finish(&mut self) -> Result<DeviceTreeNode, &'static str> {
        if !self.parents.is_empty() {
            return Err("Parsed FDT contains unclosed nodes.");
        }
        if let Some(root) = self.current.remove_child("") {
            return Ok(root);
        }
        Err("FDT root node name is not empty.")
    }
    ...
}
```

The `finish` function also does some error checking. Specifically, when we
finished parsing the `parents` stack must be empty as each `FDT_BEGIN_NODE`
token must have a matching `FDT_END_NODE` token. So if the `parents` stack
is not empty, then some of the `FDT_BEGIN_NODE` tokens didn't have a matching
`FDT_END_NODE` token.

The code we've covered so far just describes how the state should be updated
when we discover different tokens in the DTB, let's actually take a look at
the code that reads the DTB.

The code will essentially go in a loop, one token at a time, updating the
state. The loop will end when we hit an error or discover the `FDT_END`
token. When we discover `FDT_END` token we just need to extract the result
from the `State` and return it:

```rust
fn parse_node(structs: &[u8], strings: &[u8]) -> Result<DeviceTreeNode, &'static str> {
    const FDT_BEGIN_NODE: u32 = 0x01;
    const FDT_END_NODE: u32 = 0x02;
    const FDT_PROP: u32 = 0x03;
    const FDT_NOP: u32 = 0x04;
    const FDT_END: u32 = 0x09;

    let mut scanner = Scanner::new(structs);
    let mut state = State::new();

    loop {
        match scanner.consume_be32() {
            ...
            Ok(token) if token == FDT_END => return state.finish(),
            Err(msg) => return Err(msg),
            _ => return Err("Unknown FDT token."),
        }
    }
}
```

The `FDT_NOP` token is not very interesting, since we don't need to do
anything:

```rust
fn parse_node(structs: &[u8], strings: &[u8]) -> Result<DeviceTreeNode, &'static str> {
    const FDT_BEGIN_NODE: u32 = 0x01;
    const FDT_END_NODE: u32 = 0x02;
    const FDT_PROP: u32 = 0x03;
    const FDT_NOP: u32 = 0x04;
    const FDT_END: u32 = 0x09;

    let mut scanner = Scanner::new(structs);
    let mut state = State::new();

    loop {
        match scanner.consume_be32() {
            ...
            Ok(token) if token == FDT_NOP => {},
            Ok(token) if token == FDT_END => return state.finish(),
            Err(msg) => return Err(msg),
            _ => return Err("Unknown FDT token."),
        }
    }
}
```

The interesting tokens are `FDT_BEGIN_NODE`, `FDT_END_NODE` and `FDT_PROP`.
Let's start with the first two:

```rust
fn parse_node(structs: &[u8], strings: &[u8]) -> Result<DeviceTreeNode, &'static str> {
    const FDT_BEGIN_NODE: u32 = 0x01;
    const FDT_END_NODE: u32 = 0x02;
    const FDT_PROP: u32 = 0x03;
    const FDT_NOP: u32 = 0x04;
    const FDT_END: u32 = 0x09;

    let mut scanner = Scanner::new(structs);
    let mut state = State::new();

    loop {
        match scanner.consume_be32() {
            Ok(token) if token == FDT_BEGIN_NODE => {
                state.begin_node(scanner.consume_cstr()?);
                scanner.align_forward(4)?;
            },
            Ok(token) if token == FDT_END_NODE => {
                state.end_node()?;
            },
            ...
            Ok(token) if token == FDT_NOP => {},
            Ok(token) if token == FDT_END => return state.finish(),
            Err(msg) => return Err(msg),
            _ => return Err("Unknown FDT token."),
        }
    }
}
```

Since after the `FDT_BEGIN_NODE` goes a null-terminated string that
contains the name of the node we extract it and also make sure that the
current position in the stream is aligned on the 4-bytes boundary to be ready
to read the next token.

The `FDT_END_NODE` doesn't have any data after, so all we have to do is to
update the state.

And now the last token - `FDT_PROP`:

```rust
fn parse_node(structs: &[u8], strings: &[u8]) -> Result<DeviceTreeNode, &'static str> {
    const FDT_BEGIN_NODE: u32 = 0x01;
    const FDT_END_NODE: u32 = 0x02;
    const FDT_PROP: u32 = 0x03;
    const FDT_NOP: u32 = 0x04;
    const FDT_END: u32 = 0x09;

    let mut scanner = Scanner::new(structs);
    let mut state = State::new();

    loop {
        match scanner.consume_be32() {
            Ok(token) if token == FDT_BEGIN_NODE => {
                state.begin_node(scanner.consume_cstr()?);
                scanner.align_forward(4)?;
            },
            Ok(token) if token == FDT_END_NODE => {
                state.end_node()?;
            },
            Ok(token) if token == FDT_PROP => {
                let len = scanner.consume_be32()? as usize;
                let off = scanner.consume_be32()? as usize;
                let value = scanner.consume_data(len)?;
                let name = Scanner::new(&strings[off..]).consume_cstr()?;
                state.new_property(name, value);
                scanner.align_forward(4)?;
            },
            Ok(token) if token == FDT_NOP => {},
            Ok(token) if token == FDT_END => return state.finish(),
            Err(msg) => return Err(msg),
            _ => return Err("Unknown FDT token."),
        }
    }
}
```

When we discover an `FDT_PROP` token we need to extract the size of the
property value and the offset of the property name in the strings block.
Then, knowing the size, we can extract from the same stream the actual value
of the property.

The name of the property is stored inside the strings block. I think the
reason for that is that different nodes actually have the same property names
and storing property names separately allows to avoid storing duplicates and
save some space.

Anywyas, we still can use `Scanner` to extract the property name from the
strings block, but we need to create a new `Scanner` all the time.

Now we have different pieces that parse different parts of the DTB. It's time
for the last effort - we need to call those in the right order:

```rust
pub fn parse(fdt: &[u8]) -> Result<DeviceTree, &'static str> {
    let header = parse_header(fdt)?;

    if header.magic != 0xd00dfeed {
        return Err("Incorrect FDT magic value.");
    }

    if header.totalsize as usize > fdt.len() {
        return Err("The FDT size is too small to fit the content.");
    }

    let reserved = parse_reserved(&fdt[header.off_mem_rsvmap as usize..])?;

    let begin = header.off_dt_struct as usize;
    let end = begin + header.size_dt_struct as usize;
    let structs = &fdt[begin..end];

    let begin = header.off_dt_strings as usize;
    let end = begin + header.size_dt_strings as usize;
    let strings = &fdt[begin..end];

    let root = parse_node(structs, strings)?;

    Ok(DeviceTree::new(
        reserved, root, header.last_comp_version, header.boot_cpuid_phys))
}
```

And that's it - a functional DTB parser is ready to be used. We can test the
code on the test DTB that we created earlier. If you put the `test.dtb` in
the same directory with the code you can access it in tests like this:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_parse() {
        let dtb = include_bytes!("test.dtb");
        let dt = parse(dtb).unwrap();

        assert_eq!(
            dt.reserved_memory(),
            vec![
                ReservedMemory{ addr: 0x40000000, size: 0x1000 },
                ReservedMemory{ addr: 0x40002000, size: 0x1000 },
                ReservedMemory{ addr: 0x40004000, size: 0x1000 }]);
        assert_eq!(
            dt.follow("/").unwrap().property("#size-cells"),
            Some(&[...][..]));
        ...
    }
}
```

# Instead of conclusion

That was a long post, but since it mostly covers the specification and
doesn't introduce any complicated ideas it wasn't too hard to understad.

I used Rust for the examples in the post and ommitted some of the details,
for example, implementation of some interfaces and `#[derive(...)]` as well
as how the code is split between files.

Omitted parts should not be that big of a problem since the complete version
is available on [GitHub](https://github.com/krinkinmu/aarch64). For the code
to this post you need to look inside the `kernel/devicetree` directory.

Any suggestions and questions are welcome, as always.
