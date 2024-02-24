---
layout: post
title: AArch64 memory and paging
excerpt_separator: <!--more-->
tags: aarch64 virtual-memory
---

[PL011]: {% post_url 2020-11-29-PL011 %} "PL011"

In this post I will return to my exploration of 64 bit ARM architecture and will
touch on the exciting topic of virtual memory and AArch64 memory model.

Hopefully, by the end of this post I will have an example of how to configure
paging in AArch64 and will gather some basic understanding of the relevant
concepts and related topics along the way.

<!--more-->

# The End Goal

Setting up simple address translation in AArch64 is not that hard, however it
might not be easy to see because of sheer number of different translation modes
and various options supported by the architecture.

So, somewhat at odds with how I usually structure my posts, I will start with
the end result I want to get and will use this to limit the complicated
configuration surface of ARM memory mangement unit that we need to explore to
achieve the end result.

By the end of the post I want to configure and test a very simple translation
scheme. The translation scheme I chose can be characterized as follows:

1. It will only support a single exception level EL2, so we don't need to worry
   about priviledge levels when accessing memory, e.g. there will be no memory
   that can only be accessed by highly priviledged code or anything of this
   sort;
2. All memory will be readable, writable and executable in my very simplistic
   setup;
3. I will use 64 KiB transaltion granule - I will cover brifly what it means
   later, for now I will only mention that there are 3 supported options (4 KiB,
   16 KiB and 64 KiB) and for what this post tries to do it really does not
   matter much which one we chose, so I picked 64 KiB and will stick with it;
4. I will use translation scheme that have a single input (virtual) address
   range (the other alternative is to have 2 address ranges);
5. I will only need a single translation stage (the other alternative is to
   have 2 stages of address translation);
6. Finally, I will setup identity mapping.

If some or all of those things sound like a bunch of giberish to you, don't
worry. Some of those will be clarified later and as for the rest, hopefully by
the end of this post you will have good enough bird-eye view of the whole thing
to figure out the details on your own.

# Device Memory

I first will start from seemingly unrelated topic of caching memory accesses.
Putting it very simply and generally, cache memory is a relatively fast memory
that can store results of access to a relatively slow memory to speed up the
work of a computing system striking a balance between performance and cost.

For example, RAM is faster than HDD, so we can use RAM as a cache for HDD access
and store there the results we recently read from HDD or even data that we may
want to write to HDD, but didn't yet. So where a system with enough RAM to
replace the whole HDD will be too costly and the system with just HDD will be to
slow, as system with RAM as a cache and HDD is somewhere in between.

With that basic idea of the cache, we now will look at the other side of using
caches - correctnes. You may have heard a phrase "single source of truth", this
phrase refers to a pattern of organizing systems in such a way when there is an
authoritative source of information that contains the latest correct version.

When you introduce caches in the system you create a situation when you don't
have a single source of truth anymore. E.g. when a CPU tries to write something
in memory, the write may be cached. In this case the latest correct version of
the data will be in the cache. When we en entry from the cache, the memory will
contain the latest correct version of the data. So as you can see, we don't have
a single place where we can get the latest correct version of the data - we have
to check both cache and memory.

The presence of caches complicates things significantly and we need to explore
some of those complications to understand the concept of memory type in ARM,
and I will start with the idea of memory mapped device registers.

In modern hardware systems (and actually in the old compute hardware as well),
it's often the case that if you want to interact with a device, the interface
you use look like memory access. For example, if you take a look at [PL011] to
configure the serial interface you need to write a certain magic numbers to
certain magic memory addresses. That's because the registers of the device are
mapped to memory, meaning that they have actually memory addresses and attempts
to read and write at those addresses result in reading and writing from the
device registers.

However, those memory reads and writes have side effects that are not typical
for regular memory. Because of that you often may want to have a very tight
control over how those reads and writes actually happen.

In ARM it is possible to mark ranges of memory where device registers are mapped
as *device memory* and that gives you a level of control over reads and writes
that regular memory accesses don't have (or don't allow to do it as easily at
least).

ARM specification defines 3 properties of device memory that we may control:

* Gathering (G)
* Reordering (R)
* Early Write Acknowledgement (E).

*Gathering* is when multiple memory accesses, typically to the same location,
are merged together into one memory transaction. In a very simple example, if
you program reads from the same memory location twice, without writing anything
in between those reads, CPU can *gather* those reads together and issue just a
single read instead of issuing two separate reads.

That makes perfect sense and something that you can expect to see when you
access regular memory. After all if both reads read the same memory and there
is no reason to believe that this memory would have changed between the reads
you can skip an unnecessary read.

On the other hand, if this is not a usual memory, but a memory mapped register
it might be undesirable to skip a read. One very simplistic example is when a
program wants to wait until the device finishes an action. One way to achieve
that is for the device to expose a read-only status register that will set or
drop a flag when devices completes an action. In this case, multiple reads from
the same address may in fact return different results, even if there are no
writes.

*Reordering*, as the name suggests, is when memory accesses can be re-ordered
and issued in a different order from what is encoded in the executed program.

This is somewhat more complicated to explain than *gathering*, but when you work
with regular memory the order of memory accesses does not always affect the end
result of your program, but it may affect performance of the program. When it
is the case, CPU might potentially re-order those accesses in whatever way it
deems more efficient.

With memory mapped device registers such re-ordering may or may not be
desirable, however unlike with the regular memory, it's definitely harder for
the CPU to tell when such re-ordering affects visible behavior or the end result
of the program, so it may be benefitial to disable re-ordering all together for
memory mapped devices.

*Early write acknowledgement* refers to even more obscure for a regular
programmer idea that when you write to some memory mapped device register you
may want to wait until the write will fully propagate and device will
acknowledge that write.

The *early* part basically means that the write has to be acknowledged by the
device itself and not some intermediary standing between the CPU and the device.
Imagine a complicated bus (e.g. PCIe). The bus itself is a complicated device,
but we can also connect other devices to the bus including various bridge
devices or hubs that just pass messages along.

When we say that we don't want *early* write acknowledgement we mean that none
of those intermediate devices (e.g. bus itself, bridges, hubs, etc) will
acknowledge the write. Instead they just pass the message along to the target
device and once the device responds, they pass back the responce as well.

Typically, when we write something we don't wait for any memory write
acknowledgments (at least not explicitly), but in ARM we could wait for it
using the `DSB` barrier instruction in the program and wait until the write
fully propagates to the target device explicitly.

Depending on whether early write acknowledgement enabled or disabled the
behavior of the `DSB` instruction may yield different results.

Each of the three properties or behaviors can be allowed or disallowed in ARM
memory configuration. AArch64 supports the following combinations of those:

1. `nGnRnE` - gathering, reordering and early write acknowledgement are not
    allowed;
2. `nGnRE` - gathering and reording are prohibitted, but early write
   acknowledgement is allowed;
3. `nGRE` - gathering is prohibited, but reordering and early write
   acknoweldgement are allowed
4. `GRE` - gathering, reording and early write acknowledgment are allowed.

And it's actually up to us to configure which memory type system will use when
accessing memory. Naturally, given that `nGnRnE` is the configuration that
prohibits all the strange memory behaviors, it's the simplest device memory
type to reason about. On the other hand, device memory accesses with `nGnRnE`
memory type may perform somewhat worse given that they reduce the processing
unit flexbility to optimize memory accesses.

> NOTE: You an find more details about device memory and additional pointers in
> the section B2.7.2 "Device Memory" of Arm Architecture Reference Manual for
> A-profile architecture.

# Normal Memory

The normal memory, unlike the device memory that we covered above, does not
produce side effects that a CPU cannot expect when we access it, so in a way
working with normal memory is much simpler than working with device memory.

However, normal memory can still be used for communications. For example, two
CPUs of a multi-processing system may communicate with each other via memory.
We can also use normal memory for communicating between CPU and a device, as
long as they have access to the same memory.

In cases where we use regular memory for communcation, either explicitly or
implicitly, caches can still result in various strange and un-intuitive effects.
To avoid those effects different parts of the compute system (e.g. CPUs,
devices, etc) have to communicate with each other to learn of the state of
various hardware caches and modify those caches in a way that makes sense.

Such communcation protocol that creates a consistent shared view of memory in
presence of various caches is commonly refered to as cache coherency protocol.
There are probably many different cache coherency protocols that exist and
I will not discuss them in this post. However, it's probably not going to be a
surprise for you if I said that depending on what parts of the system have to
be kept in sync by cache coherency protocols, the protocol may be more or less
complicated, more or less performant, more or less power efficient, etc.

To put it another way, layout and number of components may affect the how well
the cache coherency protocol works. Not a far leap from this is to imagine a
system that may support multiple different cache coherency protocols depending
on what parts of the system we want to keep in sync with each other.

That's where the concept of memory sherability comes into play. Normal memory
shareability is about communications via memory between different parts of the
system and required synchronization.

ARM defines three different sharebility types for normal memory:

* Non-sharebable
* Inner shareable
* Outer shareable

*Non-shareable* memory is, as the name suggest, memory not used for
communcations and therefore there is not need to maintain cache coherency for
that memory.

Basically, if we for example had a memory range that is used exclusively by a
single CPU, meaning that nothing else, besides that CPU, reads or writes from
and to that memory, we don't have to maintain cache coherency for that memory
range at all.

*Inner* and *outer* sharebavle normal memory is used for communication between
different parts of the system. The different between the two is rather hard to
explain coherently because ARM specification actually leaves it up to the
hardware to decide.

However, the general idea is that different parts of the system will be kept
in sync for *inner* sharable memory and for *outer* shareable memory. When a CPU
or device access a memory range that is marked as inner shareable, hardware will
use the cache coherency protocol that will keep all the CPUs and devices in
*the same inner shareability domain* in sync. When a CPU or devices access a
memory marked as outer shareable, a cache coherency protocol that keeps all the
CPUs and devices in *the same outer sharebility domain* in sync will be used.

ARM specification says that each device or CPU (each agent) belongs to a single
inner shareablity domain and a single outer sharebility domain. It additionally
says that all devices in the same inner sharebility domain also must belong to
the same outer shareability domain. So agents, inner sharebility domains and
outer sharebility domains form a hierarchy.

I speculate that the idea here is that keepin agents inside a single sharebility
domain in sync may potentially be faster or more power efficient, then keeping
all the agents in the outer sharebility domain in sync. So, if we know that a
particual memory is only used by agents withing a single inner shareability
domain we may mark it as *inner shareable* and make accesses to the memory a bit
more efficient.

On the other hand, if a memory range is used by devices from different inner
shareability domain, but within the same outer shareable domain, we have to
mark it as *outer shareable*.

So far so good, but how do we know what agents belong to what inner/outer
shareable domain? ARM specification does not actually say that explicitly, so
ultimately it's up to the specific hardware implementation.

ARM specification, however, provides a guidance that the inner shareable domain
is expected to include all the CPUs controlled by the same OS/hypervisor. If I
understand it correctly, in practice it means that all the CPUs that an OS has
available are withing the same inner shareability domain. 

> NOTE: You an find more details about sharebaility attributes of normal memory
> in the section B2.7.1 "Normal Memory" of Arm Architecture Reference Manual for
> A-profile architecture.

# Normal Memory Caching Mode

The other attribute of normal memory is cacheability. ARM supports 3 options for
configuring cache behavior:

* No caching
* Write-through caching
* Write-back caching

The first of those options is self explanatory. Write-through and write-back
refer to two well known caching strategies. In case of write-through cache when
we write a value to memory, it's written to both cache and memory right away.

In the case of write-back cache, when we write a value to memory, it's written
in cache only. The write to memory happens when, for whatever reason, we need
to evict the value from the cache.

At least to me, write-back caching strategy appears a bit more natural, as slow
write to memory happens asynchronously, but write-through strategy allows for a
nice propety that the write will be visible in memory without delay.

In addition to caching strategy, ARM also supports so called allocation and
transient hints. What those hints do if anything is not exactly well defined and
is left up to the specific implementation.

That being said, at least the idea of those hints is to indicate to hardware
that a particular memory would benefit from caching. Whether cache is going to
be effective or not is really a property of a workload, so those hints are
intended to convey something about the workload that access the memory.

Caching strategy can be configured on two levels: inner and outer. What is the
exact scope of the inner and outer level is implementation defined, but to give
you an idea think of multiple levels of caches. For example, in a
multiprocessor system each CPU may have their own cache and they may also have
a shared cache. In this case shared cache might belong to the outer level and
the cache on each CPU is inner.

# Configuring Memory Types

So now we have some idea about memory types and what properties we can
configure for memory. Let's try to put it in practice. Before we start though
I'd like to warn you that the practice is going to be a bit underwelming at
this step.

The type of memory that a system uses are "configured" in `MAIR_ELx` register.
This is a 64 bit register, that could be thought of as an array of 8 entries.
Each entry describes a type of memory (device vs normal) and its attributes and
is 8 bits long.

> NOTE: `x` in this case corresponds to the ARM exception level.

An entry describing a device memory type would follow the binary pattern
`0b0000xx00`, where `xx` is one of the following:

1. `0b00` - `nGnRnE`
2. `0b01` - `nGnRE`
3. `0b10` - `nGRE`
4. `0b11` - `GRE`.

Putting it all together, if you want to encode `nGnRnE` device memory you would
use `0b00000000 = 0x00` for this and if you wanted to encode `nGnRE` device
memory you would use `0b00000100 = 0x04`.

Encoding of a normal memory type is somewhat more complicated and follows the
pattern `0bxxxxyyyy`. `xxxx` defines caching strategy on the outer level and
`yyyy` defines the caching strategy on the inner level.

> NOTE: To avoid overlap with device memory, `xxxx` and `yyyy` cannot be zeros.

The encoding of caching configuration has the following options:

1. `0b00RW` - write-through normal memory *with* transient hint bit set
2. `0b0100` - non-cacheable normal memory
3. `0b01RW` - write-back normal memory *with* transient hint bit set
4. `0b10RW` - write-though normal memory *without* transient hint bit set
5. `0b11RW` - write-back normal memory *without* transient hint bit set

The values of `R` and `W` are read and write hint bits. Note that in case of
write-through normal memory *with* transient hint bit, `R` and `W` cannot be `0`
at the same time because it will conflict with the encoding of the device
memory.

For example, normal non-cacheable memory (for both inner and outer levels) would
be encoded as `0b01000100 = 0x44`. Normal write-back cacheable (for both inner
and outer level) memory *without* transient hint bit set and with read/write
hint bits set would be encoded as `0b11111111 = 0xff`.

> NOTE: The encoding format of each entry can be find in the description of
> the `MAIR_ELx` register, e.g. D19.2.101 MAIR\_EL2, memory Attribute
> Indirection Register (EL2).

Now, we can write the memory type configurations in any of the 8 slots in the
MAIR\_ELx refister, we just need to remember what slot do we use for what
memory type as it will become important later.

Given that ultimately MAIR\_ELx is just a 64 bit register, writing to the
register is just writing a 64-bit value to a system register. In Aarch64 there
is a special instruction for writing to a system register - `msr` (Move to
System register). Reading from a system register is done via `mrs` instruction.
Here is how the code in assembly might look like:

```asm
mair_el2_store:
  msr mair_el2, x0
  ret

mair_el2_load:
  mrs x0, mair_el2
  ret
```

> NOTE: In the code above I rely on a calling convetion in which an argument is
> passed to a function in `x0` register and the value is returned from a
> function also in `x0` register, so `mair_el2_store` takes the value we want
> to write to MAIR\_EL2 in `x0` register and `mair_el2_load` returns the
> current value of MAIR\_EL2 in `x0` register.

Now, what would we write into the register? For the sake of being specific
let's say that we will use the following memory types:

1. Normal inner/outer write-back cacheable memory with read/write hint bits and
   without transient hint bit in slot 0
2. Normal non-cacheable memory in slot 2
3. `nGnRnE` device memory in slot 3
4. 'nGnRE' device memory in slot 4

The rest of the slots we will not use at all and they will be filled with zeros
(which also happens to mean that they will describe `nGnRnE` device memory). So
together it gives us MAIR value equal to `0x4004400ff = 17184325887`.

And a complete code might look something like this:

```c
#define MT_NORMAL             0u
#define MT_NORMAL_NO_CACHING  2u
#define MT_DEVICE_NGNRNE      3u
#define MT_DEVICE_NGNRE       4u

static const unsigned NORMAL_MEMORY = 0xff;
static const unsigned NORMAL_MEMORY_NO_CACHING = 0x44;
static const unsigned DEVICE_NGNRNE = 0x00;
static const unsigned DEVICE_NGNRE = 0x04;

static uint64_t mair_attr(uint64_t attr, unsigned idx)
{
    return attr << (8 * idx);
}

void memory_cpu_setup(void)
{
    void mair_el2_store(uint64_t);

    const uint64_t mair =
        mair_attr(NORMAL_MEMORY, MT_NORMAL) |
        mair_attr(NORMAL_MEMORY_NO_CACHING, MT_NORMAL_NO_CACHING) |
        mair_attr(DEVICE_NGNRNE, MT_DEVICE_NGNRNE) |
        mair_attr(DEVICE_NGNRE, MT_DEVICE_NGNRE);

    mair_el2_store(mair);
}
```

You can also refer how Linux Kernel configures the MAIR register (though in
their case it's MAIR\_EL1 register) by checking the
[\_\_cpu_setup](https://elixir.bootlin.com/linux/v6.7.1/source/arch/arm64/mm/proc.S#L393) function.

The value Linux Kernel uses to initialize MAIR\_EL1 can be found in
[proc.S](https://elixir.bootlin.com/linux/v6.7.1/source/arch/arm64/mm/proc.S#L66)
and it's quite similar to the example I showed above with just a few
differences.

Some of you may be confused at this point as to what did we actually configure
here. Well, to tell you the truth we didn't actually configure anything yet. To
give you a hint into what purpose the MAIR register actually serve let's look
at the name of the register. MAIR stands for Memory Attribute Indirection
Register.

Basically, the actual configuration that will determine how we will access
memory (including it's type) will live in a different place. However, instead
of encoding memory type directly there we will just use an index of a slot in
the MAIR register instead - thus the name of the register.

Why is that? To encode memory type and related attributes we need 8 bits, but
to encode an index in the MAIR register we just need 3 bits. 

# Translation Table

The next item on the agenda is translation table. Translation table is
basically a map-like hierarchical data structure that describes:

1. How virtual or logical addresses should be mapped to the physical addresses
2. What kinds of access are allowed and disallowed for different virtual
   addresses (e.g. can we read something at a given virtual address, can we
   write there, can we execute code at that virtual address, etc).

As I mentioned earlier I will setup identity mapping translation table. That
basically means that virtual address `X` will be translated by this translation
table to a physical address `X`. To even further simplify the matters, the
translation table will not even need to be hyerarchical at all - it's going to
consist of just a flat array of entries.

A typical translation table usually looks like a tree with several levels (like
3, 4 or 5 levels, depending on the specific configuration). Each node of the
tree would contain an array of entries, each entry either contains a pointer to
a node at a higher level or directly encodes how to map vritual addresses
corresponding to the entry to the physical addressed. The pointer to the root
of the tree is stored in TTBRx\_ELy register (or VTTBRx\_ELy).

The data structure is rather simple to comprehend once you have an idea of how
it's used by hardware, so let's take a look at an example. Say a program tries
to access memory at virtual address `X`. To find the actual physical address
corresponding to `X` process has to "walk" the page table.

It would first start by going to the TTBRx\_ELy register and find the physical
address of the root of the translation table. Let's for the sake of being
specific say that code executes in EL2, so the register the process will look
at will be TTBR0\_EL2.

> NOTE: Naturally the address has to be physical, otherwise processor will have
> to translate it going into an infinite loop.

From TTBR0\_EL2 process will take the physical address of the *level 1*
translation table. This table basically is just a massive array of 64-bit entries.
The exact size of the translation table, format of each entry and so on depends
on the configuration.

For the sake of being specific and as was mentioned earlier, I will be using
64 KiB translation granule and in this case the root node of the translation
table may contain up to 1024 entries and the minimum of 64 entries depending on
the configuration.

The next step is to find the entry corresponding to the virtual address `X`. In
order to do that processor will look at a few significant high bits in the
value `X` and use them as an index in the table.

To cover the whole 1024 entries you need 10 bits and in our case processor will
look at bits `X[51:42]` to decide which entry from the table it needs. For the
case when the root table only contains 64 entries, we need just 6 bits, so in
this case the process will look at `X[47:42]`.

> NOTE: Even though the virtual address typically would be described as a
> 64-bit value, as you can see, bit 52 and higher are not actually being used;
> that's not a mistake, but even a 52-bit virtual address space is so large,
> that it's not that big of constraint at the moment.

Now processor knows the right entry in the *level 1* table and it can decode it.
There a few options for what the entry could describe:

1. The entry might be invalid - that would mean that virtual address `X` is not
   mapped to any physical address, the translation will stop and processor will
   report an error;
2. The entry might be a so called *block* entry - that means that this entry
   directly describes how virtual address `X` is mapped to a physical address;
   the translation will stop and processor will learn the right physical
   address;
3. The entry might point to the next level page table (e.g. *level 2* page
   table in our example) - that means that processor will continue page table
   walk to resolve the physical address.

Let's say we reached the case 3, so the entry processor just looked at
contained a physical address of a *level 2* page table. *level 2* page table is
very similar to the *level 1* page table in structure - it's an array of 64-bit
entries, up to 8129 entries long.

> NOTE: Tables at *level 2* and *level 3* always contain 8192 entries and
> require 13 bits for an index unlike the table at the *level 1*.

To find the right entry in the *level 2* page table, processor would look at
the bits `X[41:29]` of the virtual address this time to find the right entry in
the table. As for *level 1* there are the same 3 options available for the type
of the entry in the table, so the "walk" may stop there or continue to the
*level 3*.

With 64 KiB translation granule this tree can have up to 3 levels. So a
*level 3* page table cannot point to a *level 4* page table anymore and must
contain either invalid entries that stop the translation process or direct
description of how virtual address corresponding to the entry maps to the
physical address.

Let's assume that "walking" the translation table we finally reached an entry
at *level 3* that describes how to translate virtual address to a physical
address. How would this entry look like and how do we use it to get a physical
address?

The entry at the *level 3* like that should contain an address of a contiguous
64 KiB long range of physical memory - that's what we call a physical page. If
we take unused bits of the `X` and use them as an offset within the page we
will get a physical address.

E.g. bits `X[63:52]` can be ignored, we used bits `X[51:42]` as index in the
*level 1* table, bits `X[41:29]` as index in the *level 2* table and bits
`X[28:16]` as index in *level 3* table, which leaves us with `X[15:0]` unused -
those unusued bits are the offset inside a 64 KiB physical page we need and so
we can form the final address by adding up the physical address of the page
and offset of the page together.

> NOTE: So page table is basically a kind of radix-tree or a prefix tree of
> some sort.

# Translation Table Entry Format

Now, when we have a general idea of how translation tables look like and how
they are used, we will create one. Just like in the example above we will use
64 KiB translation granule - which basically means that we will use 64 KiB
physical pages. However, instead of creating all the 3 levels of page tables we
will cut the "walk" short at the *level 1* by using that describe mapping of
virtual addresses to physical addresses directly instead of point to a higher
level tables.

In general each entry format includes type of the entry, physical address and
some attributes. There are multiple different attributes that we can set in
the translation table entry, but I will cover only a small subset of those and
assume the rest to be set to 0.

Here are the attributes we are going to set:

1. bits [47:16] - will contain a 64 KiB aligned physical address of a
   block; it will not be a physical address of 64 KiB page, but an address of
   4 TiB contiguous physical memory block because that's the size of memory a
   single entry at *level 1* is responsible for;
2. bits [9:8] - sharebility attributes; we will use inner sharebale for
   everything and that is encoded as value `0b11`;
3. bits [7:6] - access permission bits; we will make everything writable
   and that is encoded as value `0b01`;
4. bits [4:2] - index in the MAIR register for the corresponding memory
   type entry; I will use 0 there which would mean that all memory will be
   normal memory as was configured in the example above;
5. bits [1:0] encode the type of the entry and in our case, bit 1 has to
   contain value 0 and bit 0 have to contain value 1.

Addtionally, we will limit ourselves to 48-bit large physical address space for
now, so we will only need 64 entries and not maximum possible 1024 entries in
the root table for now.

In the code the whole setup might look like this:

```c
#define PF_TYPE_BLOCK      ((uint64_t)1 << 0)
#define PF_MEM_TYPE_NORMAL ((uint64_t)MT_NORMAL << 2)
#define PF_READ_WRITE      ((uint64_t)1 << 6)
#define PF_INNER_SHAREABLE ((uint64_t)3 << 8)
#define PF_ACCESS_FLAG     ((uint64_t)1 << 10)

void ttbr0_el2_store(uint64_t ttbr);

void paging_idmap_setup(void)
{
    static uint64_t idmap[64] __attribute__ ((__aligned__(512)));

    const uint64_t block_size = 0x40000000000ull;
    const uint64_t block_attr =
        PF_TYPE_BLOCK |
        PF_MEM_TYPE_NORMAL |
        PF_READ_WRITE |
        PF_INNER_SHAREABLE |
        PF_ACCESS_FLAG;
    uint64_t phys = 0;

    for (size_t i = 0; i < sizeof(idmap)/sizeof(idmap[0]); ++i) {
        idmap[i] = phys | block_attr;
        phys += block_size;
    }

    ttbr0_el2_store((uint64_t)idmap);
}
```

> NOTE: I had to forecfully align `idmap` to 512 bytes boundary, becaus it is
> one of the requirements of the AArch64 architecture that the root table as
> well as all other tables have to be aligned on the size of the table.

The definition of `ttbr0_el2_store` function might look something like this:

```asm
ttbr0_el2_store:
    msr ttbr0_el2, x0
    ret
```

Given that TTBR0\_EL2 is also a system register, just like MAIR\_EL2 that we
saw above, the function looks very similar to `mair_el2_store`.

> NOTE: The details of the format of various translation table entries can be
> found in section D8.3 Translation table descriptor format of Arm Architecture
> Reference Manual for A-profile architecture.

> NOTE: I didn't explain the meanining of the PF\_ACCESS\_FLAG, but you soon
> will see that it is not only present in the code snippet above, but also in
> the Linux Kernel implementation. Without that flag the page table will not
> work unless we tell hardware to manage the access bit automatically - without
> it attempts to translate address will result in a Data Abort.

<details>
<summary>
We can also look at how Linux Kernel sets up the initial page table, though it's
a bit less straighforward.
</summary>
<br>
Linux Kernel also uses identity mapping (and there is a good reason for that),
but unlike the example above it's more complex because:

1. Linux Kernel actually spports multiple different configurations for the
   inital mapping by the same code;
2. Linux Kernel uses multiple levels compared to the example above which stops
   at *level 1*.

With that in mind, let's try to understand first what does Linux Kernel initial
mapping covers. The intial mapping is created in
[create\_idmap](https://elixir.bootlin.com/linux/v6.7.1/source/arch/arm64/kernel/head.S#L334)
which is called relatively early in the Linux Kernel boot sequence.

The goal of the function is to create an identity mapping large enough to cover
the image of the Linux Kernel in memory and FDT. `_text` marks the beginning of
the Linux Kernel image in memory and `_end` marks the end. So unlike the
example above, it covers a much smaller range.

The logic of creating the actual table entries is implemented inside the
[map\_memory](https://elixir.bootlin.com/linux/latest/source/arch/arm64/kernel/head.S#L262)
macro.

The macro is rather hard for me to understand, because I don't actually know
the macro language used there, but, I think, I got the general picture. First,
of all, it appears that this macro relies on the pages of each level to be
consequitve in memory.

What does it mean? Here is an example to understand, let's say we want to
create a 3 level translation table. We would need a root page for the
translation table, so at the *level 1* we just need one page.

Each entry in the root table we use will point to a table at *level 2*, so we
need as many pages for *level 2* table as many entries we use in the root table.

Finally, each entry at *level 2* we use points to a *level 3* table, so we need
as many pages for *level 3* tables as the number of entries we use at the
*level 2*.

Now, when I say that the pages of the same level should be consequtive in
memory, I mean that in memory all the pages of *level 2* are located
consequtevilty - one right after another. And the same applies for the
*level 3* pages.

> NOTE: In practice we might need just a single page at each level if we only
> need to create a translation table for a small virtual address range.

So why am I mentioning this layout of pages in memory? It has a nice property that
all entries of the same level form a consequtive array of entries in memory.
E.g. all entries at *level 1* that we need to fill form an array, all entries
of *level 2* that we need to fill form an array, even if individual entries
belong to different pages they are consequtive in memory. And the same applies
to the *level 3* entries.

That allows to structure creation of the translation table in the following way:

1. In the first loop go over all the entries at *level 1* and initialize them;
2. In the second loop go over all the entries at *level 2* and initialize them;
3. In the third loop go over all the entreis at *level 3* and initialize them.

So all we need is just 3 simple loops to setup the translation table and it's
possible because all entries of the same level are consequitive in memory. So
with that in mind, you can look at the implementation of the map\_memory
function.

It uses compute\_indices macro to calculate the indices in the arrays of
entries on different levels and populate\_entries contains the actual loop
that populates them with data.

Let's take a look at those starting with compute\_indices first as it's simpler:

```
.macro compute_indices, vstart, vend, shift, order, istart, iend, count
	ubfx	\istart, \vstart, \shift, \order
	ubfx	\iend, \vend, \shift, \order
	add	\iend, \iend, \count, lsl \order
	sub	\count, \iend, \istart
.endm
```

The input of this macro includes the beginning (*vstart*) and the end (vend) of
the virtual address range that we want to map - you can think of those as the
first and the last virtual address of the range.

It also takes *shift* and *order* as input. Those parameters tell what bits of
the virtual address are used as index at the given level. For example, if table
at the *level 1* is indexed by bits [47;42], then *shift* parameter should
contain 42 and order should contain 6.

Let's see if Linux Kernel arrives to the same numbers under some assumptions to
better understand what is going on in that code. The order that initially
passed in the map\_memory macro is defined by macro IDMAP\_PGD\_ORDER that is
defined right there in the create\_idmap function. It appears that depending on
the configration of the kernel it is calculated differently.

As before let's assume that our virtual addresses are 48-bit long, then
IDMAP\_PGD\_ORDER is calculated as `PHYS_MASK_SHIFT - PGDIR_SHIFT`.

The value of PHYS\_MASK\_SHIFT is definied by the Linux Kernel configuration
option CONFIG\_ARM64\_PA\_BITS and let's assume that it's 48, meaning that the
maximum size of a physical address is 48 bits.

The value of PGDIR\_SHIFT is calculated in a somewhat more complex way and
depends on the number of levels in the translation table that we want to use.
In Linux Kernel CONFIG\_PGTABLE\_LEVELS configuration option is responsible for
the number of levels in the translation table. As in the example above with 64
KiB physical pages let's assume that the number of level is 3. Then applying
the formulae we get PGDIR\_SHIFT equal to 42 as expected.

Adding it all together we get that IDMAP\_PGD\_ORDER in this case will be 6.

So with this assumptions and values we calculated, the first invocation of
compute\_indices inside map\_memory macro will calculate the range of indexes
used in the *level 1* table.

The `ubfx` is just an instruction that in this case would extract bits [47:42]
from the argument, shift it right by 42 and save the result in the output
register.

The compute\_indices macro stores the results in *istart* and *iend*. Those will
be later passed into populate\_entries that will actually fill in the entries
in that range. Let's take a look at that:

```
.macro populate_entries, tbl, rtbl, index, eindex, flags, inc, tmp1
.Lpe\@:	phys_to_pte \tmp1, \rtbl
	orr	\tmp1, \tmp1, \flags	// tmp1 = table entry
	str	\tmp1, [\tbl, \index, lsl #3]
	add	\rtbl, \rtbl, \inc	// rtbl = pa next level
	add	\index, \index, #1
	cmp	\index, \eindex
	b.ls	.Lpe\@
.endm
```

This macro takes as input the pointer to the page table that we want to fill in
in *tbl* and the physical address of the next level page table or actual
physical address if we are at the last level in *rtbl*.

*index* and *eindex* are the indexes in the table that we calculated earlier in
*istart* and *iend*. *flags* is the memory attributes that we want to put in
the translation table.

Finally, *inc* is by how much do we increase *rtlb* after filling up each entry
and it's basically always equal to the size of the physical page we use.

The macro itself contains a simple loop with a rather straighforward body:

1. combine physical address with the flags using `orr` instruction
2. store the result in the table using `str` instruction
3. increase physical address and the index by the right amount

> NOTE: Ignore phys\_to\_pte - it's a macro, but for many intents and purposes
> you can just think of it as a `mov` instruction that just copies *rtbl* to
> *tmp1*.

So now we hopefully have some understanding of the two basic building blocks of
the map\_memory macro. Let's see how it combines those together. I will start
with ignoring the *extra\_shift* logic - that's inline with the assumptions I
made above.

Further I will assume that *SWAPPER\_PGTABLE\_LEVELS* is 3, which again aggrees
with the assumptions above (Linux Kernel uses the same number of levels for the
initial translation table as for the regular one if the page size is 64 KiB).

With that in mind, compute\_indices and populate\_entries will be called 2 more
times for *level 2* and *level 3*. There is a bit of additional complexity
there on top of what I covered already, but, hopefully, you at least got a gist
of what this code is trying to do and how.

With this macro-mess out of the way, there are only a few more points to make
remaining. SWAPPER\_RX\_MMUFLAGS contains the actual memory attributes used
during mapping. If you look at the flags used, you will find that the initial
translation table maps everything as read-only, which is quite surprising at
first.

However later in create\_idmap function, Linux Kernel changes attributes for
range of addresses to SWAPPER\_RW\_MMUFLAGS which allows write access. This
range for which change of attributes happens corresponds to the memory that
Linux Kernel will later be using to set up another translation table.

Basically, initial translation table that Linux Kernel creates in create\_idmap
function is just used to enable paging and to bootstrap the kernel enough to
create a proper translation table the kernel will actually be using. So Linux
Kernel progresses through a multi-stage initialization process.
</details>

# Enabling MMU

Let's take a look at what has been done so far... We created a very simplistic
translation table and saved a pointer to it to the right register. At this point
the translation table is not actually being used, because we didn't configure
and enable MMU yet. So let's do that.

ARM supports multiple different address translation schemas and for each of
those there are multiple different configuration parameters, so let's try to
recall what parameters do we need to configure:

1. Translation granule - we have a choice between 4 KiB, 16 KiB and 64 KiB,
   remeber that I decided at the beginning that we will use 64 KiB granule;
2. We will use a single contiguous VA range, instead of splitting it in lower
   and higher halfs;
3. We will only support EL2 exception level for now;
4. Finally, we will only have a single translation stage.

Looking at section D8.1.2 Translation regimes from the Arm Architecture
Reference Manual for A-profile architecture, you can see the list of the
supported translation regimes. Among those we need so called "Non-secure EL2
translation regime".

> NOTE: I mostly ignored the topic of security states all together so far and
> I don't intend to cover it, so going forward just assume that everything we
> do happens in non-secure state. I also ignore the topic of realm state, so
> please ignore those as well and assume that we are not using it.

> NOTE: When documentation says something like "EL1&0" in the translation
> scheme description, it means that the translation scheme supports multiple
> exception levels, e.g. "EL1&0" supports EL1 and EL0.

So how can we tell the CPU to use "Non-secure EL2 translation regime"? Once you
know what translation scheme you want to use, documentation provides a pretty
clear answer:

1. CPU must be in non-secure state and in EL2 - I just assume that these
   conditions hold true and we don't need to do anything to make it true;
2. The value of `HCR_EL2.E2H` bit is 0.

> NOTE: In case the CPU is in secure state or not in EL2, we can change it if
> the CPU supports changing to EL2 at all, but to simplify an already lengthy
> post, I omit details of how to do it. 

So we need to make sure to reset bit `E2H` in the HCR\_EL2 register to 0 to
configure what we want. We already saw multiple times how to manipulate system
registers in Aarch64, so the following code should look very familiar:

```asm
hcr_el2_store:
    msr hcr_el2, x0
    ret

hcr_el2_load:
    mrs x0, hcr_el2
    ret
```

Register HCR\_EL2 is a 64 bit register, and `E2H` is bit 34 of that register.


Let's take a look at the translation granules now. Section D8.1.1 Translation
granules of the Arm Architecture Reference Manual for A-profile architecture
implies that not all translation granule sizes may be supported on each Arm
implementation.

We can check if 64 KiB translation granule is supported by looking at
ID\_AA64MMFR0\_EL1 register. This is a readonly feature register. It exists to
report what optional features are supported by the underliyng hardware.

To check that 64 KiB granule is supported we need to look at bits from 24 to
27 (also known as TGran64). When all bits are zeros it means that 64 KiB
granule is supported. So we need one additional function to read this register:

```asm
id_aa64mmfr0_el1_load:
    mrs x0, id_aa64mmfr0_el1
    ret
```

Assuming that we confirmed that 64 KiB granule size is actually supported by
hardware and we want to enable it, how do we do that? For that we need to
program bits TG0 (bits from 14 to 15) of TCR\_EL2 register. To configure 64 KiB
translation granule we need to write there values `0b01`.

There are a couple of additional things that I would like to cover here:

1. Size of the physical address space;
2. Size of the virtual address space.

ARM basically can support different sizes of VA address and physical address.
When it comes to the translation table it basically controls how many bits of
the virtual address CPU will consider when doing a lookup in the translation
table. It specifically affects lookups at the *level 1*.

The size of the physical address space affects how CPU will interpret physical
addresses in the translation table and how many bits it will take into
consideration.

For the translation table we created in the example above we assumed both
virtual and physical address spaces to be 48 bit, so we want configuration
compatible with that.

ARM requires that the size of the address produced by address translation cannot
be larger than the size of the supported physical address.

To find the maximum supported size of the physical address space we need to
look at bits PARange (bits from 0 to 3) of the ID\_AA64MMFR0\_EL1 register.

And the size of the output address of the translation scheme is configured via
bits PS (bits from 16 to 18) of TCR\_EL2 register. The TCR\_EL2.PS bits have
the same encoding as ID\_AA64MMFR0\_EL1.PARange bits, so in principle we can
just copy those.

> NOTE: it's ok to have physical addresses outside of the physical address
> space in the translation table, as long as we don't actually access them, so
> even if the translation table we created above uses physical addresses that
> are too large, we should still be fine, since we should never need to access
> those.

How many bits of the virtual address the translation scheme can use depends on
multiple paramters, but for basically for 64 KiB granules it should always be
possible to support at least 48 bit virtual addresses.

To actually configure the number of significant bits of the virtual address we
should write to bits T0SZ (bits from 0 to 5) of the TCR\_EL2 register. And the
number of significant bits of the virtual address is determined as
`64 - T0SZ`.

So if we want to configure 48 bit virtual addresses, then in bits T0SZ we need
to write the value 16.

There are some other configuration bits, but let's ingore them for now and try
to put all the things together in one example. The final bit that we need in
order to make it happen is to actually enable address translation.

For that we need one more system register SCTLR\_EL2. There are couple of
relevant bits in the register:

1. M (bit 0) - when set to 1 it enables address translation;
2. EE (bit 25) - endianness of data access, when 0 CPU uses little-endian (and
   that's what we are going to use), otherwise the CPU uses big-endian.

So we need a couple more rather familiar functions:

```asm
sctlr_el2_store:
    msr sctlr_el2, x0
    ret

sctlr_el2_load:
    mrs x0, sctlr_el2
    ret
```

And we now are ready to put all things together. I will update
memory\_cpu\_setup function introduced earlier as follows:

```c
static const unsigned HCR_EL2_E2H_OFFSET = 34;

static const uint64_t TCR_EL2_TG0_MASK = 0xc000ull;
static const uint64_t TCR_EL2_TG0_64KIB = 0x4000ull;
static const uint64_t TCR_EL2_PS_MASK = 0x70000ull;
static const unsigned TCR_EL2_PS_OFFSET = 16;
static const uint64_t TCR_EL2_T0SZ = 0x1full;
static const uint64_t TCR_EL2_T0SZ_48BITS = 0x10ull;
static const uint64_t TCR_EL2_HA_MASK = 0x200000ull;
static const uint64_t TCR_EL2_HA_DISABLED = 0x000000ull;

static const uint64_t SCTLR_EL2_M_MASK = 0x1ull;
static const uint64_t SCTLR_EL2_M_ENABLE = 0x1ull;
static const uint64_t SCTLR_EL2_EE_MASK = 0x2000000ull;
static const uint64_t SCTLR_EL2_EE_LITTLE_ENDIAN = 0x0ull;

static const uint64_t ID_AA64MMFR0_EL1_TGRAN64_MASK = 0xf000000ull;
static const uint64_t ID_AA64MMFR0_EL1_TGRAN64_ENABLED = 0x0ull;
static const uint64_t ID_AA64MMFR0_EL1_PARANGE_MASK = 0xfull;

enum error_code memory_cpu_setup(void)
{
    void mair_el2_store(uint64_t mair);
    uint64_t hcr_el2_load(void);
    void hcr_el2_store(uint64_t hcr);
    uint64_t tcr_el2_load(void);
    void tcr_el2_store(uint64_t tcr);
    uint64_t sctlr_el2_load(void);
    void sctlr_el2_store(uint64_t sctlr);
    uint64_t id_aa64mmfr0_el1_load(void);

    const uint64_t mair =
        mair_attr(NORMAL_MEMORY, MT_NORMAL) |
        mair_attr(NORMAL_MEMORY_NO_CACHING, MT_NORMAL_NO_CACHING) |
        mair_attr(DEVICE_NGNRNE, MT_DEVICE_NGNRNE) |
        mair_attr(DEVICE_NGNRE, MT_DEVICE_NGNRE);

    mair_el2_store(mair);
    paging_idmap_setup();

    const uint64_t mmfr0 = id_aa64_mmfr0_el1_load();
    if ((mmfr0 & ID_AA64MMFR0_EL1_TGRAN64_MASK) !=
            ID_AA64MMFR0_EL1_TGRAN_ENABLED) {
        return ERR_NOT_SUPPORTED;
    }

    uint64_t tcr = tcr_el2_load();
    uint64_t hcr = hcr_el2_load();
    uint64_t sctlr = sctlr_el2_load();

    hcr &= ~((uint64_t)1 << HCR_EL2_E2H_OFFSET);
    tcr = (tcr & ~TCR_EL2_TG0_MASK) | TCR_EL2_TG0_64KIB;
    tcr = (tcr & ~TCR_EL2_PS_MASK) |
            ((mmfr0 & ID_AA64MMFR0_EL1_PARANGE_MASK) << TCR_EL2_PS_OFFSET);
    tcr = (tcr & ~TCR_EL2_T0SZ) | TCR_EL2_T0SZ_48BIT;
    tcr = (tcr & ~TCR_EL2_HA_MASK) | TCR_EL2_HA_DISABLED;

    sctlr = (scltr & ~SCTLR_EL2_EE_MASK) | SCTLR_EL2_EE_LITTLE_ENDIAN;
    sctlr = (sctlr & ~SCTLR_EL2_M_MASK) | SCTLR_EL2_M_ENABLE;

    tcr_el2_store(tcr);
    hcr_el2_store(hcr);
    sctlr_el2_store(sctlr);

    return OK;
}
```

# Testing

How can we test that address translation was enabled and indeed works? One way
to quickly verify that address translation is happening is to map two different
ranges of the virtual address space to the same range of the physical address
space. This way you would essentially create two different views of the same
physical memory, and to test it you can write something using one virtual
address and read the same data back using a different virtual address.

Let's try to do that in a quick, dirty and not very generic way. This is also
going to be a good exercise to explain effects of the TLB cache along the way.

I'm not going to change fundamentally the structure of the translation table -
it will still remain a simple array of 64 entries each describing how a
contiguous 4TiB long region of virtual address space maps on a contiguous 4TiB
long region of the physical memory. Instead, I'm going to pick one entry out of
64 and point it to a different area of the physical memory.

The way my test system is configured is as follows:

1. It has 128 MiB of actual physical memory (not memory mapped registers);
2. The physical memory is located in the *physical* address space in
   [0x40000000; 0x48000000).

Looking at the bits [47;42] of 0x48000000, you can see that those bits contain
only zeros. That means that the whole memory range in our identity translation
table, is currently mapped by the entry with the index 0. That's the entry I
will duplicate.

Now, we need to find where exactly in our virtual address space we want to
duplicate. Any range that the code currently does not use will work fine for
this, so I will use entry with the index 1.

> NOTE: I used the entry that does not contain any physical memory or any
> memory mapped device registers that I'm using, though strictly speaking for
> memory mapped device registers the identity mapping I created is not really
> good as it maps all the memory using the same memory attributes.

What would we expect to see when we copy entry 0 to entry 1 in our simplistic
translation table?

Entry 0 is responsible for mapping all the virtual addresses in range
[0x0; 0x40000000000). Entry 1 is responsible for mapping all the virtual
addresses in range [0x40000000000; 0x80000000000). And both will point to the
same physical address range [0x0; 0x40000000000).

So anything we write by address X in range [0x0; 0x40000000000), can be read
back by address X + 0x40000000000. And opposite is also true, anything we write
by address Y in the range [0x40000000000; 0x80000000000), can be read back by
address Y - 0x40000000000.

So let's create a simple test function:

```c
enum error_code memory_cpu_setup_test(void)
{
    /*
     * Given our setup from program point of view this array will be located
     * in memory in the virtual address range [0x0; 0x40000000000).
     */
    static char message[32];

    /*
     * Let's make changes to our translation table so that both
     * [0x0; 0x40000000000) and [0x40000000000; 0x80000000000) virtual address
     * ranges point to the same physical memory.
     */
    uint64_t *translation_table = (uint64_t *)ttbr0_el2_load();
    uint64_t old_entry = translation_table[1];

    translation_table[1] = translation_table[0];

    /*
     * Let's verify that now both [0x0; 0x40000000000) and
     * [0x40000000000; 0x80000000000) point to the same physical memory by
     * writing data using one of the two address ranges and reading it back
     * using a different address range.
     */

    static const uint64_t shift = 0x40000000000ull;
    char *original = message;
    char *mirrored = (char *)((uintptr_t)message + shift);

    strncpy(original, "Ping!", sizeof(message));
    if (strncmp(mirrored, "Ping!", sizeof(message)) != 0) {
        return ERR_INVALID_ARGUMENT;
    }

    strncpy(mirrored, "Pong!", sizeof(message));
    if (strncmp(original, "Pong!", sizeof(message)) != 0) {
        return ERR_INVALID_ARGUMENT;
    }

    translation_table[1] = old_entry; 

    return OK;
}
```

Hopefully the code of the function is self explanatory. And probably if you run
this function you will indeed observe the desirable results of the translation
table manipulations.

However, I will use this opportunity to draw attention to yet another caching
side effect that is important. The test function modified the content of the
translation table while it was being used.

Leaving aside potential race conditions associated with this, the content of the
translation table is cached by the processor using it. If you think about it,
enabling address translation without any caching would means that any access to
memory will require several additional access to memory just to resolve the
physical address.

For example, with the translation table as I set it up earlier, every memory
access will translate to two memory acesses:

1. Read the translation table entry corresponding to the virtual address to find
   the physical address;
2. Read the data at the given physical address.

With more realistic translation tables, you may have 3, 4 or 5 levels in the
translation table, so each memory access would require 5 memory accesses just to
translate virtual address to a physical address. That's way too much overhead!
So to reduce the impact, processors do use a special cache for the address
translation.

So when we update translation table like I did above, we need to make sure that
the translation cache, also commonly known as Translation Lookaside Buffer or
TLB, has to be updated accordingly as well. And, at least for ARM, that does not
happen automatically, and the code has to inform the processor, that TLB has
to be updated after updating translation table.

With that in mind, we can refer to the section D8.14.1 "Using break-before-make
when updating translation table entries" of the Arm Architecture Reference
Manual for A-profile architecture for the recommended way of doing it.

The reference manual suggests to replace the entry with an invalid entry first
and then with the entry we actually want to see. And on top of that the
procedure is heavily reinforced with memory barrier instructions along the way.
This procedure does seem somewhat excessive for our use case, but let's
implement it nontheless.

First, we need a few support functions. A few functions implementing the right
memory barriers and another invalidating TLB.

```asm
pt_update_barrier:
    dsb ishst
    ret

pt_tlbi_barrier:
    dsb ish
    ret
```

> NOTE: It's a bit excessive to create a whole function for just a varian of
> `dsb` instruction, but it does not require to introduce inline assembly or
> try to map standard C memory barrier functions to ARM instructions, so it's
> simpler this way.

The argument of the `dsb` instruction specifies what type of a barrier behavior
we want. The argument `ishst`, to put it simplistically, requires the `dsb`
instruction to wait for all the previous memory writes to complete and their
effects to become visible to everyone in the same inner shareable domain
(specifically, other CPUs that potentially may access the same memory).

We will call pt\_update\_barrier after we modify the translation table entry to
make sure that all the updates are visible.

The argument `ish` expands the scope of the `dsb` instruction a little bit by
requiring all reads to be completed in addition to writes, so `dsb #ish` is a
"stronger" barrier compared to `dsb #ishst`. According to the paragraph "Data
Synchronization Barrier (DSB)" in section B2.3.12 Memory Barriers of the
reference manual, it takes a `dsb` instruction with both read and write access
considered to make sure that TLB invalidation completed. So that's the barrier
we will use after actually invalidating TLB entries to make sure that the
effects of it are visible.

Second, we need a way to invalidate potential TLB entries. Section D8.14.5 "TLB
maintenance instructions" describes the set of available ARM commands that do
this for a wide variety of potential use cases. That's quite a complicated
reading, but here is a clue that might help to find a simple version of the
`tlbi` instruction that works for our use case: it does not make much of a
difference to us whether we consider just the last level of the translation
table walk or all the levels, since we have just a single level anyway. So we
can use the version of the instruction that takes just a single virtual address.

> NOTE: Also keep in mind that we are working with a single stage translation
> table, in EL2 exception level and care only about internal shareability
> domain.

TL;DR: We need to use `tlbi vale2is` instruction and wrapping it into a
function that takes a VA address gives us the following code:

```asm
tlb_invalidate_page:
    lsr x0, x0, 12
    tlbi vale2is, x0
    ret
```

This function takes just one argument - virtual address within the range that
we want to invalidate. It will invalidate cache for the last level of the
translation table walk, keeping all the previous levels in cache. As I mentioned
earlier it does not make much of a difference to us, but it's quite important to
avoid dropping from the cache what can stay there in the case of real
translation tables that may have multiple levels.

With those helper functions, the test function will change as follows:

```c
static void tlb_invalidate_address(uint64_t virtual_address)
{
    void pt_update_barrier(void);
    void pt_tlbi_barrier(void);
    void tlb_invalidate_page(uint64_t);

    pt_update_barrier();
    tlb_invalidate_page(virtual_address);
    pt_tlbi_barrier();
}

enum error_code memory_cpu_setup_test(void)
{
    /*
     * Given our setup from program point of view this array will be located
     * in memory in the virtual address range [0x0; 0x40000000000).
     */
    static char message[32];

    /*
     * Let's make changes to our translation table so that both
     * [0x0; 0x40000000000) and [0x40000000000; 0x80000000000) virtual address
     * ranges point to the same physical memory.
     */
    const uint64_t invalid_entry = 0x0ull;
    static const uint64_t shift = 0x40000000000ull;
    uint64_t *translation_table = (uint64_t *)ttbr0_el2_load();
    uint64_t old_entry = translation_table[1];

    translation_table[1] = invalid_entry;
    tlb_invalidate_address(shift);
    
    translation_table[1] = translation_table[0];
    tlb_invalidate_address(shift);

    /*
     * Let's verify that now both [0x0; 0x40000000000) and
     * [0x40000000000; 0x80000000000) point to the same physical memory by
     * writing data using one of the two address ranges and reading it back
     * using a different address range.
     */

    char *original = message;
    char *mirrored = (char *)((uintptr_t)message + shift);

    strncpy(original, "Ping!", sizeof(message));
    if (strncmp(mirrored, "Ping!", sizeof(message)) != 0) {
        return ERR_INVALID_ARGUMENT;
    }

    strncpy(mirrored, "Pong!", sizeof(message));
    if (strncmp(original, "Pong!", sizeof(message)) != 0) {
        return ERR_INVALID_ARGUMENT;
    }

    translation_table[1] = invalid_entry;
    tlb_invalidate_address(shift);

    translation_table[1] = old_entry; 
    tlb_invalidate_address(shift);

    return OK;
}
```

> NOTE: I'm pretty sure that most of those manipulations are quite excessive in
> my simple case just because there is no concurrency involved here, but even
> without concurency in the picture, we do have to tell the CPU when we change
> the mapping to update the TLB accordingly.

> NOTE: If you're curious about TLB manipulations you can also refer to Linux
> Kernel as an example. One place to start would be looking at the comments in
> [tlbflush.h](https://elixir.bootlin.com/linux/v6.8-rc3/source/arch/arm64/include/asm/tlbflush.h).


# Instead of conclusion

This post took quite some time for me to prepare and write. As a result, it
ended up being rather lengthy and the example in this post is not that
practical in the end (and frankly speaking, I probably made a few mistakes here
and there).

That being said, I think it was overall a useful exercise to get a level of
understanding of address translation tables in ARM. And I hope that provided
examples (both my code snippets and references to Linux Kernel implementation)
are good enough to get some people started.
