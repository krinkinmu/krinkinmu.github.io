---
layout: post
title: Getting started with U-Boot
excerpt_separator: <!--more-->
tags: u-boot quemu gdb aarch64
---

A lot of time has passed since I posted last time. Old hobby projects were
long forgotten and it's time to start from scratch, but do it right this time
(I wonder if that's the reason I never finish my hobby projects).

In the past posts I covered already EFI, Qemu, Aarch64. I tried to create a
simple EFI bootloader and a simplistic Aarch64 kernel from scratch.

I still didn't quite abandon the idea of playing with virtualization on
Aarch64, but creating everything from scratch, as fun as it is, probably
going to delay things even further.

So a new me wants to learn and use some of the existing tools and in this
post I will cover the most basic things related to U-boot.

<!--more-->

# Pre-requisites

We will start with creating a simple binary that we will load with U-boot.
For now this binary will not do anything useful or exciting:

```gas
.text
.global start

start:
    b start
```

This is basically an unconditional infinite loop - instruction `b` is just
a jump to a given label.

We now need to compile and link it in such a way that does not pull in
various unncessary dependencies (e.g. libc and what not). In order to do
that I used a linker script from one of my hobby projects:

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

    .init_array : ALIGN(0x1000) {
        _INIT_BEGIN = .;
        KEEP(*(SORT(.init_array.*)))
        KEEP(*(.init_array))
        _INIT_END = .;
    } :rodata

    .rodata : {
        _RODATA_BEGIN = .;
        *(.rodata)
        _RODATA_END = .;
    } :rodata

    .data.rel.ro : {
        _RELRO_BEGIN = .;
        *(.data.rel.ro)
        _RELRO_END = .;
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

What this linker script instructs linker to do is not super relevant for this
post. It's sufficient to say, that with this script linker is expected to
produce and Aarch64 ELF binary with an entry point in function with name
`start` (it's the name of the label in the code above).

Now let's build that. I use LLVM toolchain to build things because I've had it
installed already:

```sh
clang -fPIE -target aarch64-unknown-none -c start.S -o start.o
lld -flavor ld -maarch64elf --pie --static --nostdlib --script=kernel.lds start.o -o kernel.elf
```

These command produce a rather simple ELF binary with the name `kernel.elf`.
We can take a look at the internals of the file briefly to make sure that it
contains what we expect:

```sh
llvm-objdump --disassemble kernel.elf 

kernel.elf:	file format elf64-littleaarch64

Disassembly of section .text:

0000000000001000 <start>:
    1000: 00 00 00 14  	b	0x1000 <start>
```

The output tells us that it's indeed a Aarch64 ELF binary (specifically, little
endian Aarch64 ELF binary) and the binary code there matches what I wrote
before.

Now, I don't actually want an ELF binary, I actually want the raw binary that
just contains the code and data without any ELF metadata. The reason I want a
raw binary is because later in this post I'm going to load it in memory, using
U-boot, and tell U-boot to execute it. All the ELF metadata will just stand in
the way.

So to conver this ELF binary into a raw binary we can use `objdump`:

```sh
llvm-objcopy -O binary kernel.elf kernel.bin
```

We cannot disassemable the new file using `llvm-objdump` as we did with the ELF
file above (or at least I didn't find a good way to do it), but given that our
binary is simple, we can just use `hexdump`:

```sh
hexdump -C -n 4 kernel.bin 
00000000  00 00 00 14                                       |....|
00000004
```

If you squint a little bit you will notice the four bytes (`00 00 00 14`) that
we saw above - that's just the binary encoding of the `b` instruction in our
code and it's located right at the beginning of the file.

So now I have a binary I can load using U-boot in our experiments.

# Downloading and building U-boot

The main star of the show is next now - let's donwload and build U-boot.
I again will be using Qemu instead of real hardware for my experiments because
it's easy, so we will be building U-boot for hardware emulated by Qemu:

```sh
sudo apt-get install binutils-aarch64-linux-gnu
git clone git@github.com:u-boot/u-boot.git
cd u-boot
make HOST=clang qemu_arm64_defconfig
export TRIPLET=aarch64-linux-gnu
make HOSTCC=clang CROSS_COMPILE=${TRIPLET}- CC="clang -target ${TRIPLET}" -j8
```

> NOTE: Even though I'm mostly using LLVM tools to build U-boot I still had to
> install GNU binutils.

`qemu_arm64_defconfig` is a predefined configuration for the Aarch64 hardware
emulated by Qemu, so I didn't need to configure U-boot build in any way for
this.

Successful build should produce multiple different artifacts, but for now I
care just about `u-boot.bin` - that's our U-boot image that we will run.

We can test that it works in Qemu:

```sh
qemu-system-aarch64 -machine virt,virtualization=on,secure=off -cpu max -bios u-boot.bin -nographic


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
=>
```

So I have Qemu running the U-boot image I just built.

# Qemu setup

Now I want to load and run the simple binary I created earlier using U-boot.
For that, I'm going to add a FAT disk to the machine Qemu emulates. I don't
need to create a disk image, Qemu can actually take a content of a directory
and present it as a FAT disk to the emulated hardware, which is nice.

Let's prepare it:

```sh
mkdir rootfs
cp kernel.bin rootfs/
```

And now we can start Qemu again, but this time with a disk:

```sh
qemu-system-aarch64 -machine virt,virtualization=on,secure=off -cpu max -bios u-boot.bin -nographic -drive file=fat:rw:./rootfs,format=raw,media=disk


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
=> 
```

Still the same output, but now we will explore a little bit. U-boot shell has
multiple commands (and what commands available depends on the specific
configuration), you can see available commands by running `help`:

```sh
=> help
?         - alias for 'help'
base      - print or set address offset
bdinfo    - print Board Info structure
blkcache  - block cache diagnostics and control
boot      - boot default, i.e., run 'bootcmd'
bootd     - boot default, i.e., run 'bootcmd'
bootdev   - Boot devices
bootefi   - Boots an EFI payload from memory
bootelf   - Boot from an ELF image in memory
bootflow  - Boot flows
booti     - boot Linux kernel 'Image' format from memory
bootm     - boot application image from memory
bootmeth  - Boot methods
bootp     - boot image via network using BOOTP/TFTP protocol
bootvx    - Boot vxWorks from an ELF image
bootz     - boot Linux zImage image from memory
chpart    - change active partition of a MTD device
cmp       - memory compare
coninfo   - print console devices and information
cp        - memory copy
crc32     - checksum calculation
date      - get/set/reset date & time
dfu       - Device Firmware Upgrade
dhcp      - boot image via network using DHCP/TFTP protocol
dm        - Driver model low level access
echo      - echo args to console
editenv   - edit environment variable
eficonfig - provide menu-driven UEFI variable maintenance interface
env       - environment handling commands
erase     - erase FLASH memory
exit      - exit script
ext2load  - load binary file from a Ext2 filesystem
ext2ls    - list files in a directory (default /)
ext4load  - load binary file from a Ext4 filesystem
ext4ls    - list files in a directory (default /)
ext4size  - determine a file's size
false     - do nothing, unsuccessfully
fatinfo   - print information about filesystem
fatload   - load binary file from a dos filesystem
fatls     - list files in a directory (default /)
fatmkdir  - create a directory
fatrm     - delete a file
fatsize   - determine a file's size
fatwrite  - write file into a dos filesystem
fdt       - flattened device tree utility commands
flinfo    - print FLASH memory information
fstype    - Look up a filesystem type
fstypes   - List supported filesystem types
go        - start application at address 'addr'
gzwrite   - unzip and write memory to block device
help      - print command description/usage
iminfo    - print header information for application image
imxtract  - extract a part of a multi-image
itest     - return true/false on integer compare
ln        - Create a symbolic link
load      - load binary file from a filesystem
loadb     - load binary file over serial line (kermit mode)
loads     - load S-Record file over serial line
loadx     - load binary file over serial line (xmodem mode)
loady     - load binary file over serial line (ymodem mode)
loop      - infinite loop on address range
ls        - list files in a directory (default /)
lzmadec   - lzma uncompress a memory region
md        - memory display
mii       - MII utility commands
mm        - memory modify (auto-incrementing address)
mtd       - MTD utils
mtdparts  - define flash/nand partitions
mw        - memory write (fill)
net       - NET sub-system
nm        - memory modify (constant address)
nvme      - NVM Express sub-system
panic     - Panic with optional message
part      - disk partition related commands
pci       - list and access PCI Configuration Space
ping      - send ICMP ECHO_REQUEST to network host
poweroff  - Perform POWEROFF of the device
printenv  - print environment variables
protect   - enable or disable FLASH write protection
pxe       - commands to get and boot from pxe files
To use IPv6 add -ipv6 parameter
qfw       - QEMU firmware interface
random    - fill memory with random pattern
reset     - Perform RESET of the CPU
run       - run commands in an environment variable
save      - save file to a filesystem
saveenv   - save environment variables to persistent storage
scsi      - SCSI sub-system
scsiboot  - boot from SCSI device
setenv    - set environment variables
setexpr   - set environment variable as the result of eval expression
showvar   - print local hushshell variables
size      - determine a file's size
sleep     - delay execution for some time
source    - run script from memory
test      - minimal test like /bin/sh
tftpboot  - load file via network using TFTP protocol
tpm       - Issue a TPMv1.x command
tpm2      - Issue a TPMv2.x command
true      - do nothing, successfully
unlz4     - lz4 uncompress a memory region
unzip     - unzip a memory region
usb       - USB sub-system
usbboot   - boot from USB device
vbe       - Verified Boot for Embedded
version   - print monitor, compiler and linker version
virtio    - virtio block devices sub-system
```

I want to explore the attached disk a little bit. Qemu relies on `virtio`
specification to emulate disk in this case, so `virtio` command is what I need:

```sh
=> virtio info
Device 0: 1af4 VirtIO Block Device
            Type: Hard Disk
            Capacity: 504.0 MB = 0.4 GB (1032192 x 512)
=> virtio part 0

Partition Map for VirtIO device 0  --   Partition Type: DOS

Part	Start Sector	Num Sectors	UUID		Type
  1	63        	1032129   	be1afdfa-01	06 Boot
```

`virtio info` shows that we have one `virtio` disk with index 0, so using
`virtio part 0` we can see what partitions this disk has.

Given that disk "formatted" using FAT filesystem I will use `fatls` to actually
look at the files on the disk:

```sh
=> fatls virtio 0:1
     4344   kernel.bin

1 file(s), 0 dir(s)

```

In this command `virtio` is the "interface" or device subsystem and `0:1` are
device and partition on that device.

Now I have my binary available to U-boot.

# Loading and running the binary

Since my binary is raw, loading it basically means copying it from disk to
memory. I want to know where in memory I can put it, let's take a look at what
is available:

```sh
=> bdinfo
boot_params = 0x0000000000000000
DRAM bank   = 0x0000000000000000
-> start    = 0x0000000040000000
-> size     = 0x0000000008000000
flashstart  = 0x0000000000000000
flashsize   = 0x0000000004000000
flashoffset = 0x00000000000fc5e0
baudrate    = 115200 bps
relocaddr   = 0x0000000047ed8000
reloc off   = 0x0000000047ed8000
Build       = 64-bit
current eth = virtio-net#32
ethaddr     = 52:52:52:52:52:52
IP addr     = <NULL>
fdt_blob    = 0x0000000046d97db0
new_fdt     = 0x0000000046d97db0
fdt_size    = 0x0000000000100000
lmb_dump_all:
 memory.cnt = 0x1 / max = 0x10
 memory[0]	[0x40000000-0x47ffffff], 0x08000000 bytes flags: 0
 reserved.cnt = 0x2 / max = 0x10
 reserved[0]	[0x45d8f000-0x47ffffff], 0x02271000 bytes flags: 0
 reserved[1]	[0x46d937b0-0x47ffffff], 0x0126c850 bytes flags: 0
devicetree  = board
arch_number = 0x0000000000000000
TLB addr    = 0x0000000047fe0000
irq_sp      = 0x0000000046d97da0
sp start    = 0x0000000046d97da0
Early malloc usage: 508 / 2000
```

I don't know exactly how to interpret all this output, but `lmb_dump_all` looks
like what I need. And if I understand it right, I have enough memory available
starting from address `0x40000000` and all the memory that was already reserved
for something else is way above that address.

Now I can copy the binary from the disk to memory:

```sh
=> fatload virtio 0:1 0x40000000 kernel.bin
4344 bytes read in 1 ms (4.1 MiB/s)
```

Let's quickly check what I now have in memory at `0x40000000`:

```sh
=> md.b 0x40000000 0x04   
40000000: 00 00 00 14
```

And yes that's the same 4 magic bytes of the `b` instruction, so far so good.
So now I have my binary in memory I can actually tell U-boot to jump there and
start running it:

```sh
=> go 0x40000000
## Starting application at 0x40000000 ...
```

And at this point everything hangs. That's because the binary I created does
not do anything useful it just goes into an infinite loop.

# Verifying

The things hanged, but how do I know that they hanged in my code and not
because something else failed? Well, I don't know that for sure, but let's
confirm.

Let's kill the Qemu and start it somewhat differently to get more visibility
into what's happenning. Now I'm going to run Qemu with `monitor` enabled:

```sh
qemu-system-aarch64 -machine virt,virtualization=on,secure=off -cpu max -bios u-boot.bin -nographic -drive file=fat:rw:./rootfs,format=raw,media=disk -monitor telnet:localhost:1234,server,nowait


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
4344 bytes read in 2 ms (2.1 MiB/s)
=> go 0x40000000
## Starting application at 0x40000000 ...
```

I have Qemu running and want to confirm that U-boot indeed started my binary.
I started Qemu with `monitor` listening on `locahost:1234`, so I'm going to
connect to it using `telnet`:

```sh
elnet localhost 1234
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
QEMU 6.2.0 monitor - type 'help' for more information
(qemu)
```

`monitor` allows quite a bit of introspection into the virtual machine, but for
now all I need `monitor` for is to start `gdb`, so that I can connect with
debugger to the virtual machine and see what exactly it executes:

```sh
(qemu) gdbserver tcp::1235
Waiting for gdb connection on device 'tcp::1235'
```

Now when Qemu is listening we can connect to port `1235` with `gdb`:

```sh
gdb-multiarch 
GNU gdb (Ubuntu 12.1-0ubuntu1~22.04) 12.1
Copyright (C) 2022 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.
Type "show copying" and "show warranty" for details.
This GDB was configured as "x86_64-linux-gnu".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<https://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
    <http://www.gnu.org/software/gdb/documentation/>.

For help, type "help".
Type "apropos word" to search for commands related to "word".
(gdb) set architecture aarch64
The target architecture is set to "aarch64".
(gdb) target remote localhost:1235
Remote debugging using localhost:1235
warning: No executable has been specified and target does not support
determining executable automatically.  Try using the "file" command.
0x0000000040000000 in ?? ()
```

So `gdb` is now connected and right away I can see that the code in the virtual
machine is running at address `0x40000000` - that's exactly where I loaded my
little binary.

Let's take a look at what command is actually there:

```sh
(gdb) display/i $pc
1: x/i $pc
=> 0x40000000:	b	0x40000000
```

And here I can see my `b` instruction again.

# Instead of conclusion

Not much to say here, it's probably the most basic use of U-boot out there.
I hope to return to this topic and see how U-boot loads proper kernel (e.g.
Linux kernel) and some other nice things that U-boot has in other posts.
