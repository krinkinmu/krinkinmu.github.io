---
layout: post
title: U-boot boot script
excerpt_separator: <!--more-->
tags: u-boot quemu
---

To wrap up my explorations of U-boot I'd like show how to automatically load
a kernel with U-boot on startup.

<!--more-->

# Boot script

In the past posts to load my toy binary using U-boot I had to run the
following commands in the U-boot shell:

```sh
fatload virtio 0:1 0x40000000 kernel.bin
fatload virtio 0:1 0x41000000 virt.dtb
booti 0x40000000 - 0x41000000
```

That's not a big deal, but it would be better if U-boot could do all that
automatically and, naturally, U-boot should be able to do that.

> NOTE: U-boot is used in multiple consumer hardware systems, for example,
> some Wi-Fi routers use U-boot as a bootloader and you don't have to type
> some shell commands when using those - you just power up the device and
> then U-boot does the rest automatically.

So how can we tell U-boot to execute command above automatically?

First, we need to prepare the script. When I built U-boot with it I also
built a bunch of useful tools, one of them is `mkimage`. This tool takes
some image or data as an input, attaches a header to it and produces an
image in format that U-boot is familiar with.

Let's save the commands above in a text file, let's call it `boot.txt`,
though the name does not matter as much. Now, we can use `mkimage` to
produce a U-boot image from it as follows:

```sh
~/ws/u-boot/tools/mkimage -T script -d boot.txt boot.scr
Image Name:   
Created:      Sun Nov 19 20:54:48 2023
Image Type:   PowerPC Linux Script (gzip compressed)
Data Size:    118 Bytes = 0.12 KiB = 0.00 MiB
Load Address: 00000000
Entry Point:  00000000
Contents:
   Image 0: 110 Bytes = 0.11 KiB = 0.00 MiB
```

> NOTE: `~/ws/u-boot/` is where U-boot sources I downloaded live and where
> the U-boot build results are, so that's where `mkimage` is as well.

This command should have prodiced a file `boot.scr`. The content of this file
is mostly the text of our script, but it also contains an additional binary
header:

```sh
cat boot.scr
'V�$��eZvPv�jպnfatload virtio 0:1 0x40000000 kernel.bin
fatload virtio 0:1 0x41000000 virt.dtb
booti 0x40000000 - 0x41000000
```

So now, I will place this boot.scr file in the root file system used by QEMU:

```sh
mkdir rootfs/boot
cp boot.scr rootfs/boot
```

> NOTE: `rootfs` is the directory that QEMU will use as a root file system on
> the emulated device.

So if we start QEMU now, it will load U-boot and U-boot will load our toy
binary automatically auto some wait time unless we interrupt it:

```sh
qemu-system-aarch64 \
    -machine virt,virtualization=on,secure=off \
    -cpu max \
    -bios u-boot.bin \
    -nographic \
    -drive file=fat:rw:./rootfs,format=raw,media=disk


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
Scanning for bootflows in all bootdevs
Seq  Method       State   Uclass    Part  Name                      Filename
---  -----------  ------  --------  ----  ------------------------  ----------------
Scanning global bootmeth 'efi_mgr':
Scanning bootdev 'fw-cfg@9020000.bootdev':
fatal: no kernel available
No working controllers found
scanning bus for devices...
Scanning bootdev 'virtio-blk#33.bootdev':
  0  script       ready   virtio       1  virtio-blk#33.bootdev.par /boot/boot.scr
** Booting bootflow 'virtio-blk#33.bootdev.part_1' with script
4608 bytes read in 1 ms (4.4 MiB/s)
1048576 bytes read in 1 ms (1000 MiB/s)
## Flattened Device Tree blob at 41000000
   Booting using the fdt blob at 0x41000000
Working FDT set to 41000000
   Loading Device Tree to 0000000045c8c000, end 0000000045d8efff ... OK
Working FDT set to 45c8c000

Starting kernel ...

Hello, World
```

# Standard Boot Process

So let's try to understand a little bit more how U-boot actually finds the
kernel to boot and the specifc commands. U-boot has a few relevant concepts
that are relevant to the boot process:

1. Boot Device - disk, memory card, network device, etc from which we get the
   binaries, device tree, scripts and other things we need to boot the system;
2. Boot Method - it's a method/algorithm/process used to scan the Boot Device
   extract the configs and binaries we need;
3. Boot Flow - a configuration of how to actually boot an OS, I will ignore it
   for my simplistic example as I don't actually need it.

U-boot supports multiple differnt types of boot devices and boot methods. For
example, in the U-boot build that I use on QEMU I can see the following boot
devices:

```sh
bootdev list
Seq  Probed  Status  Uclass    Name
---  ------  ------  --------  ------------------
  0   [   ]      OK  qfw       fw-cfg@9020000.bootdev
  1   [   ]      OK  ethernet  virtio-net#32.bootdev
  2   [   ]      OK  virtio    virtio-blk#33.bootdev
---  ------  ------  --------  ------------------
(3 bootdevs)
```

And here are the available boot methods:

```sh
bootmeth list -a
Order  Seq  Name                Description
-----  ---  ------------------  ------------------
    0    0  efi                 EFI boot from an .efi file
 glob    1  efi_mgr             EFI bootmgr flow
    2    2  extlinux            Extlinux boot from a block device
    3    3  pxe                 PXE boot from a network device
    4    4  qfw                 QEMU boot using firmware interface
    5    5  script              Script boot from a block device
 glob    6  vbe_simple          VBE simple
-----  ---  ------------------  ------------------
(7 bootmeths)
```

In the example above I used `virtio-blk#33.bootdev` as a boot device and
`script` as a boot method. Though, U-boot tried multiple different options
before it figured out that those boot device and method work.

# Boot Order

As you saw above `virtio-blk#33.bootdev` was not the first device in the list
and `script` was far from the first method, so U-boot tried multiple different
options before it found that that combination worked.

We can actually see what things and in what order U-boot would try on startup
using the following command in U-boot shell:

```sh
bootflow scan -lae
Scanning for bootflows in all bootdevs
Seq  Method       State   Uclass    Part  Name                      Filename
---  -----------  ------  --------  ----  ------------------------  ----------------
Scanning global bootmeth 'efi_mgr':
  0  efi_mgr      base    (none)       0  <NULL>                    <NULL>
     ** No media/partition found, err=-22E
Scanning bootdev 'fw-cfg@9020000.bootdev':
  1  efi          base    qfw          0  <NULL>                    <NULL>
     ** No media/partition found, err=-524E
  2  extlinux     base    qfw          0  <NULL>                    <NULL>
     ** No media/partition found, err=-524E
  3  pxe          base    qfw          0  <NULL>                    <NULL>
     ** No media/partition found, err=-524E
fatal: no kernel available
  4  qfw          base    qfw          0  qfw                       <NULL>
     ** No media/partition found, err=-5E
  5  script       base    qfw          0  <NULL>                    /boot/boot.scr
     ** No media/partition found, err=-1E
No working controllers found
scanning bus for devices...
Scanning bootdev 'virtio-blk#33.bootdev':
  6  efi          media   virtio       0  virtio-blk#33.bootdev.who <NULL>
     ** No partition found, err=-2E
  7  extlinux     media   virtio       0  virtio-blk#33.bootdev.who <NULL>
     ** No partition found, err=-2E
  8  pxe          base    virtio       0  virtio-blk#33.bootdev.who <NULL>
     ** No media/partition found, err=-524E
  9  qfw          base    virtio       0  virtio-blk#33.bootdev.who <NULL>
     ** No media/partition found, err=-524E
  a  script       media   virtio       0  virtio-blk#33.bootdev.who <NULL>
     ** No partition found, err=-2E
  b  efi          fs      virtio       1  virtio-blk#33.bootdev.par efi/boot/bootaa64.efi
     ** File not found, err=-2E
  c  extlinux     fs      virtio       1  virtio-blk#33.bootdev.par /boot/extlinux/extlinux.conf
     ** File not found, err=-2E
  d  pxe          base    virtio       1  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
  e  qfw          base    virtio       1  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
  f  script       ready   virtio       1  virtio-blk#33.bootdev.par /boot/boot.scr
 10  efi          media   virtio       2  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 11  extlinux     media   virtio       2  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 12  pxe          base    virtio       2  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 13  qfw          base    virtio       2  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 14  script       media   virtio       2  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 15  efi          media   virtio       3  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 16  extlinux     media   virtio       3  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 17  pxe          base    virtio       3  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 18  qfw          base    virtio       3  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 19  script       media   virtio       3  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 1a  efi          media   virtio       4  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 1b  extlinux     media   virtio       4  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 1c  pxe          base    virtio       4  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 1d  qfw          base    virtio       4  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 1e  script       media   virtio       4  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 1f  efi          media   virtio       5  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 20  extlinux     media   virtio       5  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 21  pxe          base    virtio       5  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 22  qfw          base    virtio       5  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 23  script       media   virtio       5  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 24  efi          media   virtio       6  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 25  extlinux     media   virtio       6  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 26  pxe          base    virtio       6  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 27  qfw          base    virtio       6  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 28  script       media   virtio       6  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 29  efi          media   virtio       7  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 2a  extlinux     media   virtio       7  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 2b  pxe          base    virtio       7  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 2c  qfw          base    virtio       7  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 2d  script       media   virtio       7  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 2e  efi          media   virtio       8  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 2f  extlinux     media   virtio       8  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 30  pxe          base    virtio       8  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 31  qfw          base    virtio       8  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 32  script       media   virtio       8  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 33  efi          media   virtio       9  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 34  extlinux     media   virtio       9  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 35  pxe          base    virtio       9  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 36  qfw          base    virtio       9  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 37  script       media   virtio       9  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 38  efi          media   virtio       a  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 39  extlinux     media   virtio       a  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 3a  pxe          base    virtio       a  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 3b  qfw          base    virtio       a  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 3c  script       media   virtio       a  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 3d  efi          media   virtio       b  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 3e  extlinux     media   virtio       b  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 3f  pxe          base    virtio       b  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 40  qfw          base    virtio       b  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 41  script       media   virtio       b  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 42  efi          media   virtio       c  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 43  extlinux     media   virtio       c  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 44  pxe          base    virtio       c  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 45  qfw          base    virtio       c  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 46  script       media   virtio       c  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 47  efi          media   virtio       d  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 48  extlinux     media   virtio       d  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 49  pxe          base    virtio       d  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 4a  qfw          base    virtio       d  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 4b  script       media   virtio       d  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 4c  efi          media   virtio       e  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 4d  extlinux     media   virtio       e  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 4e  pxe          base    virtio       e  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 4f  qfw          base    virtio       e  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 50  script       media   virtio       e  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 51  efi          media   virtio       f  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 52  extlinux     media   virtio       f  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 53  pxe          base    virtio       f  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 54  qfw          base    virtio       f  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 55  script       media   virtio       f  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 56  efi          media   virtio      10  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 57  extlinux     media   virtio      10  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 58  pxe          base    virtio      10  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 59  qfw          base    virtio      10  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 5a  script       media   virtio      10  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 5b  efi          media   virtio      11  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 5c  extlinux     media   virtio      11  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 5d  pxe          base    virtio      11  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 5e  qfw          base    virtio      11  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 5f  script       media   virtio      11  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 60  efi          media   virtio      12  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 61  extlinux     media   virtio      12  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 62  pxe          base    virtio      12  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 63  qfw          base    virtio      12  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 64  script       media   virtio      12  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 65  efi          media   virtio      13  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 66  extlinux     media   virtio      13  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 67  pxe          base    virtio      13  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 68  qfw          base    virtio      13  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 69  script       media   virtio      13  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 6a  efi          media   virtio      14  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 6b  extlinux     media   virtio      14  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 6c  pxe          base    virtio      14  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 6d  qfw          base    virtio      14  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 6e  script       media   virtio      14  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 6f  efi          media   virtio      15  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 70  extlinux     media   virtio      15  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 71  pxe          base    virtio      15  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 72  qfw          base    virtio      15  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 73  script       media   virtio      15  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 74  efi          media   virtio      16  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 75  extlinux     media   virtio      16  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 76  pxe          base    virtio      16  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 77  qfw          base    virtio      16  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 78  script       media   virtio      16  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 79  efi          media   virtio      17  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 7a  extlinux     media   virtio      17  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 7b  pxe          base    virtio      17  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 7c  qfw          base    virtio      17  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 7d  script       media   virtio      17  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 7e  efi          media   virtio      18  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 7f  extlinux     media   virtio      18  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 80  pxe          base    virtio      18  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 81  qfw          base    virtio      18  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 82  script       media   virtio      18  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 83  efi          media   virtio      19  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 84  extlinux     media   virtio      19  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 85  pxe          base    virtio      19  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 86  qfw          base    virtio      19  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 87  script       media   virtio      19  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 88  efi          media   virtio      1a  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 89  extlinux     media   virtio      1a  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 8a  pxe          base    virtio      1a  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 8b  qfw          base    virtio      1a  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 8c  script       media   virtio      1a  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 8d  efi          media   virtio      1b  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 8e  extlinux     media   virtio      1b  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 8f  pxe          base    virtio      1b  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 90  qfw          base    virtio      1b  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 91  script       media   virtio      1b  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 92  efi          media   virtio      1c  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 93  extlinux     media   virtio      1c  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 94  pxe          base    virtio      1c  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 95  qfw          base    virtio      1c  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 96  script       media   virtio      1c  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 97  efi          media   virtio      1d  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 98  extlinux     media   virtio      1d  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 99  pxe          base    virtio      1d  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 9a  qfw          base    virtio      1d  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 9b  script       media   virtio      1d  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
BOOTP broadcast 1
DHCP client bound to address 10.0.2.15 (2 ms)
Scanning bootdev 'virtio-net#32.bootdev':
BOOTP broadcast 1
DHCP client bound to address 10.0.2.15 (0 ms)
*** Warning: no boot file name; using '0A00020F.img'
Using virtio-net#32 device
TFTP from server 10.0.2.2; our IP address is 10.0.2.15
Filename '0A00020F.img'.
Load address: 0x40400000
Loading: *
TFTP error: 'Access violation' (2)
Not retrying...
 9c  efi          base    ethernet     0  virtio-net#32.bootdev.0   <NULL>
     ** No media/partition found, err=-2E
 9d  extlinux     base    ethernet     0  <NULL>                    <NULL>
     ** No media/partition found, err=-524E
 9e  pxe          base    ethernet     0  <NULL>                    <NULL>
     ** No media/partition found, err=-524E
 9f  qfw          base    ethernet     0  <NULL>                    <NULL>
     ** No media/partition found, err=-524E
 a0  script       base    ethernet     0  virtio-net#32.bootdev.0   <NULL>
     ** No media/partition found, err=-22E
No more bootdevs
---  -----------  ------  --------  ----  ------------------------  ----------------
(161 bootflows, 1 valid)
```

Excluding so called global boot methods, the logic U-boot uses is pretty
straighforward - it just itreatates over all the possible boot devices and tries
all the possible boot methods on each device in order.

So what is the order of those boot devices? It appears that the order is
controlled by an environment variable `boot_targets`. It seems that the default
value of the `boot_targets` variable is board specific, but it can be changed.

For example, in my case the default value looks like:

```sh
env print boot_targets
boot_targets=qfw usb scsi virtio nvme dhcp
```

And given that I know that my boot script lives on a Virtio block device I can
override it this way:

```sh
setenv boot_targets "virtio"
```

And once I do that, the boot process will actually start by looking at the
`virtio` block device first reducing the number of options to consider:

```sh
bootflow scan -lae          
Scanning for bootflows in all bootdevs
Seq  Method       State   Uclass    Part  Name                      Filename
---  -----------  ------  --------  ----  ------------------------  ----------------
Scanning global bootmeth 'efi_mgr':
  0  efi_mgr      base    (none)       0  <NULL>                    <NULL>
     ** No media/partition found, err=-22E
Scanning bootdev 'virtio-blk#33.bootdev':
  1  efi          media   virtio       0  virtio-blk#33.bootdev.who <NULL>
     ** No partition found, err=-2E
  2  extlinux     media   virtio       0  virtio-blk#33.bootdev.who <NULL>
     ** No partition found, err=-2E
  3  pxe          base    virtio       0  virtio-blk#33.bootdev.who <NULL>
     ** No media/partition found, err=-524E
  4  qfw          base    virtio       0  virtio-blk#33.bootdev.who <NULL>
     ** No media/partition found, err=-524E
  5  script       media   virtio       0  virtio-blk#33.bootdev.who <NULL>
     ** No partition found, err=-2E
  6  efi          fs      virtio       1  virtio-blk#33.bootdev.par efi/boot/bootaa64.efi
     ** File not found, err=-2E
  7  extlinux     fs      virtio       1  virtio-blk#33.bootdev.par /boot/extlinux/extlinux.conf
     ** File not found, err=-2E
  8  pxe          base    virtio       1  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
  9  qfw          base    virtio       1  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
  a  script       ready   virtio       1  virtio-blk#33.bootdev.par /boot/boot.scr
  b  efi          media   virtio       2  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
  c  extlinux     media   virtio       2  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
  d  pxe          base    virtio       2  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
  e  qfw          base    virtio       2  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
  f  script       media   virtio       2  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 10  efi          media   virtio       3  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 11  extlinux     media   virtio       3  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 12  pxe          base    virtio       3  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 13  qfw          base    virtio       3  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 14  script       media   virtio       3  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 15  efi          media   virtio       4  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 16  extlinux     media   virtio       4  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 17  pxe          base    virtio       4  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 18  qfw          base    virtio       4  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 19  script       media   virtio       4  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 1a  efi          media   virtio       5  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 1b  extlinux     media   virtio       5  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 1c  pxe          base    virtio       5  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 1d  qfw          base    virtio       5  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 1e  script       media   virtio       5  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 1f  efi          media   virtio       6  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 20  extlinux     media   virtio       6  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 21  pxe          base    virtio       6  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 22  qfw          base    virtio       6  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 23  script       media   virtio       6  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 24  efi          media   virtio       7  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 25  extlinux     media   virtio       7  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 26  pxe          base    virtio       7  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 27  qfw          base    virtio       7  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 28  script       media   virtio       7  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 29  efi          media   virtio       8  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 2a  extlinux     media   virtio       8  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 2b  pxe          base    virtio       8  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 2c  qfw          base    virtio       8  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 2d  script       media   virtio       8  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 2e  efi          media   virtio       9  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 2f  extlinux     media   virtio       9  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 30  pxe          base    virtio       9  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 31  qfw          base    virtio       9  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 32  script       media   virtio       9  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 33  efi          media   virtio       a  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 34  extlinux     media   virtio       a  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 35  pxe          base    virtio       a  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 36  qfw          base    virtio       a  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 37  script       media   virtio       a  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 38  efi          media   virtio       b  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 39  extlinux     media   virtio       b  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 3a  pxe          base    virtio       b  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 3b  qfw          base    virtio       b  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 3c  script       media   virtio       b  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 3d  efi          media   virtio       c  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 3e  extlinux     media   virtio       c  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 3f  pxe          base    virtio       c  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 40  qfw          base    virtio       c  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 41  script       media   virtio       c  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 42  efi          media   virtio       d  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 43  extlinux     media   virtio       d  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 44  pxe          base    virtio       d  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 45  qfw          base    virtio       d  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 46  script       media   virtio       d  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 47  efi          media   virtio       e  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 48  extlinux     media   virtio       e  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 49  pxe          base    virtio       e  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 4a  qfw          base    virtio       e  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 4b  script       media   virtio       e  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 4c  efi          media   virtio       f  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 4d  extlinux     media   virtio       f  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 4e  pxe          base    virtio       f  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 4f  qfw          base    virtio       f  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 50  script       media   virtio       f  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 51  efi          media   virtio      10  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 52  extlinux     media   virtio      10  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 53  pxe          base    virtio      10  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 54  qfw          base    virtio      10  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 55  script       media   virtio      10  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 56  efi          media   virtio      11  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 57  extlinux     media   virtio      11  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 58  pxe          base    virtio      11  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 59  qfw          base    virtio      11  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 5a  script       media   virtio      11  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 5b  efi          media   virtio      12  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 5c  extlinux     media   virtio      12  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 5d  pxe          base    virtio      12  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 5e  qfw          base    virtio      12  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 5f  script       media   virtio      12  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 60  efi          media   virtio      13  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 61  extlinux     media   virtio      13  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 62  pxe          base    virtio      13  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 63  qfw          base    virtio      13  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 64  script       media   virtio      13  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 65  efi          media   virtio      14  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 66  extlinux     media   virtio      14  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 67  pxe          base    virtio      14  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 68  qfw          base    virtio      14  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 69  script       media   virtio      14  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 6a  efi          media   virtio      15  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 6b  extlinux     media   virtio      15  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 6c  pxe          base    virtio      15  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 6d  qfw          base    virtio      15  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 6e  script       media   virtio      15  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 6f  efi          media   virtio      16  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 70  extlinux     media   virtio      16  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 71  pxe          base    virtio      16  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 72  qfw          base    virtio      16  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 73  script       media   virtio      16  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 74  efi          media   virtio      17  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 75  extlinux     media   virtio      17  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 76  pxe          base    virtio      17  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 77  qfw          base    virtio      17  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 78  script       media   virtio      17  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 79  efi          media   virtio      18  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 7a  extlinux     media   virtio      18  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 7b  pxe          base    virtio      18  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 7c  qfw          base    virtio      18  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 7d  script       media   virtio      18  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 7e  efi          media   virtio      19  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 7f  extlinux     media   virtio      19  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 80  pxe          base    virtio      19  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 81  qfw          base    virtio      19  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 82  script       media   virtio      19  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 83  efi          media   virtio      1a  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 84  extlinux     media   virtio      1a  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 85  pxe          base    virtio      1a  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 86  qfw          base    virtio      1a  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 87  script       media   virtio      1a  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 88  efi          media   virtio      1b  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 89  extlinux     media   virtio      1b  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 8a  pxe          base    virtio      1b  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 8b  qfw          base    virtio      1b  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 8c  script       media   virtio      1b  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 8d  efi          media   virtio      1c  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 8e  extlinux     media   virtio      1c  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 8f  pxe          base    virtio      1c  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 90  qfw          base    virtio      1c  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 91  script       media   virtio      1c  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 92  efi          media   virtio      1d  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 93  extlinux     media   virtio      1d  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 94  pxe          base    virtio      1d  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 95  qfw          base    virtio      1d  virtio-blk#33.bootdev.par <NULL>
     ** No media/partition found, err=-524E
 96  script       media   virtio      1d  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
No more bootdevs
---  -----------  ------  --------  ----  ------------------------  ----------------
(151 bootflows, 1 valid)
```

Addtionally, I can change the order of boot methods as well even further
reducing the number of options U-boot will have to try:

```sh
setenv bootmeths "script"
bootflow scan -lae       
Scanning for bootflows in all bootdevs
Seq  Method       State   Uclass    Part  Name                      Filename
---  -----------  ------  --------  ----  ------------------------  ----------------
Scanning bootdev 'virtio-blk#33.bootdev':
  0  script       media   virtio       0  virtio-blk#33.bootdev.who <NULL>
     ** No partition found, err=-2E
  1  script       ready   virtio       1  virtio-blk#33.bootdev.par /boot/boot.scr
  2  script       media   virtio       2  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
  3  script       media   virtio       3  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
  4  script       media   virtio       4  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
  5  script       media   virtio       5  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
  6  script       media   virtio       6  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
  7  script       media   virtio       7  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
  8  script       media   virtio       8  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
  9  script       media   virtio       9  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
  a  script       media   virtio       a  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
  b  script       media   virtio       b  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
  c  script       media   virtio       c  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
  d  script       media   virtio       d  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
  e  script       media   virtio       e  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
  f  script       media   virtio       f  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 10  script       media   virtio      10  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 11  script       media   virtio      11  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 12  script       media   virtio      12  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 13  script       media   virtio      13  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 14  script       media   virtio      14  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 15  script       media   virtio      15  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 16  script       media   virtio      16  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 17  script       media   virtio      17  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 18  script       media   virtio      18  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 19  script       media   virtio      19  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 1a  script       media   virtio      1a  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 1b  script       media   virtio      1b  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 1c  script       media   virtio      1c  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
 1d  script       media   virtio      1d  virtio-blk#33.bootdev.par <NULL>
     ** No partition found, err=-2E
No more bootdevs
---  -----------  ------  --------  ----  ------------------------  ----------------
(30 bootflows, 1 valid)
```

# Instead of conclusion

Instead of the conclusion I will provide some code references to where this
boot logic implementation lives in U-boot:

1. the implementation of the `bootflow` shell command lives in
   [`u-boot/cmd/bootflow.c`](https://github.com/u-boot/u-boot/blob/master/cmd/bootflow.c)
2. the implementation of the boot device iterators is located in
   [`u-boot/boot/bootflow.c`](https://github.com/u-boot/u-boot/blob/master/boot/bootflow.c)
3. logic of different boot methods is split bettwen multiple files in
   `u-boot/boot` directory, but the common routines for iteration through the
   available boot methods live in
   [`u-boot/boot/bootmeth-uclass.c`](https://github.com/u-boot/u-boot/blob/master/boot/bootmeth-uclass.c)
4. spefically `script` boot method implementation details live in
   [`u-boot/boot/bootmeth-script.c`](https://github.com/u-boot/u-boot/blob/master/boot/bootmeth_script.c)
   and by looking around for similarly named files you can find what other boot
   methods can U-boot support in principle.

Finally, environment variables `boot_targets` and `bootmeths` that control what
boot devices and methods to try and in which order handled a little bit
differently from each other.

`boot_targets` environment variable is read directly when needed and that's why
it's relatively easy to find a reference to it in
[`u-boot/boot/bootstd-uclass.c`](https://github.com/u-boot/u-boot/blob/master/boot/bootstd-uclass.c).

`bootmeths` variable is not accessed directly and instead there is a callback
that gets called when this variable get changed. It's registered via
`U_BOOT_ENV_CALLBACK` macro in `u-boot/boot/bootmeth-uclass.c`.

