---
layout: default
title: Kernel modules, device drivers and Device Tree
excerpt_separator: <!--more-->
tags: beaglebone linux-kernel
---
[Bootlin]: https://bootlin.com/ "Bootlin"
[Linux Kernel]: https://www.kernel.org/ "Linux Kernel"
[BeagleBone Black]: https://beagleboard.org/black "BeagleBone Black"
[BeagleBone Black Wireless]: https://beagleboard.org/black-wireless "BeagleBone Black Wireless"
[Device Tree]: https://en.wikipedia.org/wiki/Device_tree "Device Tree"
[U-Boot]: https://github.com/u-boot/u-boot "U-Boot"
[the previous post]: {% post_url 2020-07-05-beaglebone-software-update %} "the previous post"
[Bazel]: https://en.wikipedia.org/wiki/Bazel_(software) "Bazel"
[Maven]: https://en.wikipedia.org/wiki/Apache_Maven "Maven"
[CMake]: https://www.kernel.org/doc/html/latest/kbuild/kbuild.html "CMake"
[USB]: https://en.wikipedia.org/wiki/USB "USB"
[I2C]: https://en.wikipedia.org/wiki/I%C2%B2C "I2C"
[ARM]: https://en.wikipedia.org/wiki/ARM_architecture "ARM"
[x86]: https://en.wikipedia.org/wiki/X86 "x86"

I continue going through [Bootlin] training materials on embedded systems and
[Linux Kernel]. In [the previous post] I covered the environment setup, so now
we should be able to access the board and share files between the board and the
host.

In this article I'm going to try to actually create a few simple Linux Kernel
modules, build them for the [BeagleBone Black] or [BeagleBone Black Wireless]
board and test them on the actual hardware.

All the sources used in this article are available on
[GitHub.](https://github.com/krinkinmu/bootlin)

<!--more-->

# Creating and building a module

I will start with the simplest module to show how to build the modules and then
load them on the board. The module will do nothing useful, the only thing we'd
need from it is to be able to verify that it loads and unloads successfully. In
order to do that we will just print some log messages when the module loads and
unloads.

Here is the code:

```c
// SPDX-License-Identifier: GPL-2.0
#include <linux/init.h>
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/utsname.h>

static int __init version_init(void)
{
	pr_alert("%s version %s: %s\n",
		utsname()->sysname,
		utsname()->release,
		utsname()->version);
	return 0;
}

static void __exit version_exit(void)
{
	pr_alert("Goodbye, World!\n");
}

module_init(version_init);
module_exit(version_exit);
MODULE_LICENSE("GPL");
MODULE_DESCRIPTION("Linux Kernel Version module");
MODULE_AUTHOR("Krinkin Mike <krinkin.m.u@gmail.com>");
```

There is just a couple of things in this code relevant for now:

 * functions *version_init* and *version_exit* that are called when the module
   is loaded and unloaded correspondingly
 * macroses *module_init* and *module_exit* is how we tell the kernel what
   function have to be called when the module is loaded and unloaded

Inside the *version_init* and *version_exit* function we simply print some
messages to the kernel log using *pr_alert*.

We have the code ready and can move to the main part - building and loading the
module.

Linux kernel uses a bit involved build system called *Kbuild*. Kbuild does not
appear to be a completely new build system (like [Bazel], [Maven] or [CMake])
and instead it's more of an organization of [Make] files. Therefore it's driven
by makefiles that follow a certain structure. Let's take a look at the Makefile
for our simple module:

```make
ifneq ($(KERNELRELEASE),)

obj-m := version.o

else

KDIR ?= /lib/modules/`uname -r`/build

default:
	$(MAKE) -C $(KDIR) M=$$PWD

endif
```

The part relevant for *Kbuild* itself is just one line:

```make
obj-m := version.o
```

This one line magically tells the kernel build system that it should build a
module and *version.c* is the only source file for the module. However as I
mentioned before, kernel build system is a bit involved, so just calling make
on that would not work.

The build system is expected to build all kernel sources, even if they are part
of a module, together. So individual make files are expeted to be included by
the *Kbuild* when you run make in the top level directory of the kernel sources.

That *default* target in the *Makefile* above executes a command that can build
the module outside of the kernel source tree by itself. All you need to specify
is the path to the kernel sources in the *KDIR* variable.

In order to use this *Makefile* to compile for the [BeagleBone Black] we'd need
to specify the architecture for the build, the toolchain and where the rest of
the kernel code lives:

```sh
export ARCH=arm
export CROSS_COMPILE=arm-linux-gnueabi-
export KDIR=/home/kmu/ws/linux
make
```

> *NOTE:* */home/kmu/ws/linux* is where I store the Linux source code I used to
  build the kernel for the board.

If build is successful we should see quite a few build artifacts in the
directory with the module:

```sh
ls -al
total 328
drwxr-xr-x 3 kmu kmu  4096 Jul 12 14:58 .
drwxr-xr-x 4 kmu kmu  4096 Jul 12 13:04 ..
-rw-r--r-- 1 kmu kmu     8 Jul 12 13:07 built-in.a
-rw-r--r-- 1 kmu kmu   163 Jul 12 13:07 .built-in.a.cmd
-rw-r--r-- 1 kmu kmu 73044 Jul 12 12:39 .cache.mk
-rw-r--r-- 1 kmu kmu   136 Jul 12 12:59 Makefile
-rw-r--r-- 1 kmu kmu    40 Jul 12 13:07 modules.order
-rw-r--r-- 1 kmu kmu   170 Jul 12 13:07 .modules.order.cmd
-rw-r--r-- 1 kmu kmu     0 Jul 12 13:07 Module.symvers
-rw-r--r-- 1 kmu kmu   215 Jul 12 13:07 .Module.symvers.cmd
drwxr-xr-x 2 kmu kmu  4096 Jul 12 12:59 .tmp_versions
-rw-r--r-- 1 kmu kmu   551 Jul 12 12:45 version.c
-rw-r--r-- 1 kmu kmu 49848 Jul 12 13:07 version.dwo
-rw-r--r-- 1 kmu kmu 21640 Jul 12 13:07 version.ko
-rw-r--r-- 1 kmu kmu   284 Jul 12 13:07 .version.ko.cmd
-rw-r--r-- 1 kmu kmu    40 Jul 12 13:07 version.mod
-rw-r--r-- 1 kmu kmu   777 Jul 12 13:07 version.mod.c
-rw-r--r-- 1 kmu kmu   147 Jul 12 13:07 .version.mod.cmd
-rw-r--r-- 1 kmu kmu 33416 Jul 12 13:07 version.mod.dwo
-rw-r--r-- 1 kmu kmu 10004 Jul 12 13:07 version.mod.o
-rw-r--r-- 1 kmu kmu 26183 Jul 12 13:07 .version.mod.o.cmd
-rw-r--r-- 1 kmu kmu 12944 Jul 12 13:07 version.o
-rw-r--r-- 1 kmu kmu 30688 Jul 12 13:07 .version.o.cmd
```

The most important file for us at this stage is the *version.ko*. That's the
actual kernel module that we can load on the board:

```sh
mkdir /home/kmu/ws/nfsroot/modules
cp version.ko /home/kmu/ws/nfsroot/modules
```

> *NOTE:* */home/kmu/ws/nfsroot* is the path to the root file system that I
  share with the board over NFS.

We can now access *version.ko* file on the board itslef and can even load it and
check if the module works:

```sh
cd /modules
insmod version.ko
rmmod version.ko
dmes | tail
[    4.273057]     TERM=linux
[    5.822049] g_ether gadget: packet filter 0e
[    5.822064] g_ether gadget: ecm req21.43 v000e i0000 l0
[   33.778447] wlan-en-regulator: disabling
[  110.158443] random: crng init done
[  220.218673] version: loading out-of-tree module taints kernel.
[  220.225058] Linux version 5.8.0-rc3-next-20200703: #1 SMP Sat Jul 4 15:15:49 IST 2020
[  240.576688] Goodbye, World!
```

If everything worked out, then we should see the log messages that our module
prints during load and unload in the output of the *dmesg* command.

You can find more details about *Kbuild*, the structure of *Makefile* for the
Linux kernel modules and much more in the documentation in the kernel source
code. The relevant parts are available in [Documentation/kbuild]
(https://www.kernel.org/doc/html/latest/kbuild/index.html.)

# Driver model

Kernel modules allow to exectute almost arbitrary code in the kernel space.
However for us the primary interest at this point is to create a driver. In the
simplest terms the driver is just a piece of code that makes a device (real
hardware device or some pseudo device) available to the users one way or
another.

Linux kernel provides a framework for creating device drivers. The goals of the
framework is to separate platfrom dependent code from platform indpendent code,
so that platform independent code can be reused between platforms.

A typical example here would be some [USB] device. The same [USB] device (like
a storage stick, keyboard, mouse, etc) can be used on [x86], [ARM] or other
architectures. While the details of implementation of [USB] support may change
between platforms and boards, the driver for the same device ideally shoud not
need to change.

The driver model is organaized around three structures:

 * [struct bus_type](https://www.kernel.org/doc/html/latest/driver-api/driver-model/bus.html)
   represents a one bus type (like *USB*, *PCI*, *I2C*, etc);
 * [struct device_driver](https://www.kernel.org/doc/html/latest/driver-api/driver-model/driver.html)
   represents a driver that can handle a prticular type of devices on a bus;
 * [struct device](https://www.kernel.org/doc/html/latest/driver-api/driver-model/device.html)
   represents one connected device.

Those structures are exteneded by driver developers normally using a particular
way of implementing inheritance in *C*. This pattern is even documented in the
*container_of()* secion of
[Documentation/driver-api/driver-model/design-patterns.html.]
(https://www.kernel.org/doc/html/latest/driver-api/driver-model/design-patterns.html)

In practice while developing drivers using the supported buses, you will be
interacting with the bus specific API and not with the three structures above
directly. However the structure stays roughly the same: there will be bus
specific driver and device structures.

# Device Tree

Bus is supposed to be responsible for discovering new devices and finding the
right driver to handle the device. However, not all buses are capable of
automatically discovering devices connected to it. In this case we need to tell
the kernel what devices are there. One commonly used way to do that is
[Device Tree].

Baiscally [Device Tree] statically describes the devices and connections
between them. Compiled version of such description is provided to the kernel as
input. The kernel parses and instantiates the devices described in the
[Device Tree].

For the [BeagleBone Black Wireless] the top level [Device Tree] file lives in
*arch/arm/boot/dts/am335x-boneblack-wireless.dts* in the Linux source code. This
file is compiled into *arch/arm/boot/dts/am335x-boneblack-wireless.dtb*. The
compiled file is what we provide to [U-Boot] via TFTP together with the kernel
image itself.

For a node in the [Device Tree] we can specify the *compatible* attribute. The
string value of the attribute is used to find the right driver for the device.
The way it works is that the driver itself encodes supported device identifiers
that are checked against the value of the *compatible* attribute in the
[Device Tree].

More information about using [Device Tree] in the Linux is available in the
[Documentation/devicetree](https://www.kernel.org/doc/html/latest/devicetree/index.html.)

# I2C

Let's throw in a bit of practice now to link all the pieces together. [I2C] is
a very simple bus, physically it has just a couple of wires: one for data and
one for clock signals.

[I2C] is a master/slave bus, meaning that devices connected via [I2C] are not
equal. All transactions are always initiated by the master, so no slave device
will send any signals until asked.

Each [I2C] device must have a unique address, which allows to connect multiple
devices to the same bus. However the bus does not provide any capabilities to
discover connected devices. Therefore, we'd need to provide some [Device Tree]
description for the connected [I2C] devices.

We will start with creating a new [Device Tree] file inside the
*arch/arm/boot/dts* directory that contains exactly the same configuration as
the [Device Tree] file for the [BeagleBone Black Wireless] board that I use.

I'll call the file *am335x-boneblack-wireless-custom.dts*:

```c
#include "am335x-boneblack-wireless-custom.dts"
```

To build this file we'd need to update the *Makefile* in the same directory:

```diff
diff --git a/arch/arm/boot/dts/Makefile b/arch/arm/boot/dts/Makefile
index 74dd94e72848..e939150c4672 100644
--- a/arch/arm/boot/dts/Makefile
+++ b/arch/arm/boot/dts/Makefile
@@ -782,6 +782,7 @@ dtb-$(CONFIG_SOC_AM33XX) += \
        am335x-bone.dtb \
        am335x-boneblack.dtb \
        am335x-boneblack-wireless.dtb \
+       am335x-boneblack-wireless-custom.dtb \
        am335x-boneblue.dtb \
        am335x-bonegreen.dtb \
        am335x-bonegreen-wireless.dtb \
```

Now we should be able to build the kernel and generate the binary [Device Tree]:

```sh
cd /home/kmu/ws/linux
export ARCH=arm
export CROSS_COMPILE=arm-linux-gnueabi-
make dtbs
```

> *NOTE:* again */home/kmu/ws/linux* is where I store the kernel source code.

If everything is ok, we should have a compiled [Device Tree] file:

```sh
ls -l arch/arm/boot/dts/am335x-boneblack-wireless-custom.dtb
-rw-r--r-- 1 kmu kmu 62048 Jul 12 16:55 arch/arm/boot/dts/am335x-boneblack-wireless-custom.dtb
```

Since we didn't make any changes to the [Device Tree] yet, the content of the
compiled file should be very similar to the *am335x-boneblack-wireless.dtb*:

```sh
ls -l arch/arm/boot/dts/am335x-boneblack-wireless.dtb
-rw-r--r-- 1 kmu kmu 62048 Jul  4 15:15 arch/arm/boot/dts/am335x-boneblack-wireless.dtb
```

# I2C controller and Nunchuk device in Device Tree

Now let's try to modify the [Device Tree] and introduce a configuration for a
new [I2C] device. As a device for our experiments I will use the
[Nintendo Wiichuk](https://www.olimex.com/Products/Modules/Sensors/MOD-WII/MOD-Wii-UEXT-NUNCHUCK/open-source-hardware),
the device used in the [Bootlin] materials.

> *NOTE:* in this post we will not communicate with the device itself, that
  will covered in the future posts. So it doesn't really matter if you have
  a device or not at this point, I'm only refering to it here to be specific
  and setup a base for the future posts.

The device [I2C] address is *0x52*. The address is hardcoded and enforced by the
device itself. How do I know that the address is *0x52*? This information is
available in the [Bootlin materials](https://bootlin.com/labs/doc/nunchuk.pdf).
Also, on the
[olimex.com](https://www.olimex.com/Products/Modules/Sensors/MOD-WII/MOD-Wii-UEXT-NUNCHUCK/open-source-hardware)
where you can purchase the device there is a link to Arduino examples, where you
can also find the [I2C] address of the device.

[BeagleBone Black] and [BeagleBone Black Wireless] have three [I2C] controllers.
We will connect *Wiichuk* to the second [I2C] controller. This controller is
not used for any builtin devices on the board and is not even properly
configured in the default [Device Tree], so we'd need to add the configuration
for the controller and the device.

Let's modify our custom [Device Tree] file to include the [I2C] controller
node and a child node for the [I2C] controller with the *Wiichuk* device:

```c
#include "am335x-boneblack-wireless.dts"

&i2c1 {
        status = "okay";
        clock-frequency = <100000>;

        nunchuk@52 {
                compatible = "nintendo,nunchuk";
		reg = <0x52>;
        };
};
```

*i2c1* is a shortcut for the second [I2C] controller node in the [Device Tree].
It's defined in the *aliases* section in the *arch/arm/boot/dts/am33xx.dtsi*
file that is included indirectly by our [Device Tree].

Normally [Device Tree] configuration is structured as a tree, thus the name.
Creating an alias allows us to refer to a particular node without specifying
the complete path to that node. That's why in the [Device Tree] file above we
could create the node for the second [I2C] controller directly without any
parent nodes.

Also, [Device Tree] compiler merges trees together node by node. So it would be
more correct to say that the file above does not create a node for the second
[I2C] controller, but it modifes the existing node for the [I2C] controller
adding *status* and *clock-frequency* properties as well as a child node
*nunchuk@52*.

You may notice that the name of the node for the *Wiichuk* device has *@52*
suffix. This part of the name is called a unit address. Normally, the kernel
should not depend on the value of the unit address. The unit address is mostly
a convention that allows to generate a unique name for the [Device Tree] node.
For example, in this particular case we used the [I2C] address of the *Wiichuk*
device.

> *NOTE:* even though unit address is not supposed to be used, it's still
  available to the kernel code, so there is no hard restriction that prevents
  device drivers from depending on the unit address somehow.

This is by no means a complete [I2C] configuration, but let's try to build it
and test on the device:

```sh
cd /home/kmu/ws/linux
export ARCH=arm
export CROSS_COMPILE=arm-linux-gnueabi-
make dtbs
sudo cp arch/arm/boot/dts/am335x-boneblack-wireless-custom.dtb /var/lib/tftpboot/am335x-boneblack-wireless.dtb
```

Hopefully the board still can boot with the new compiled [Device Tree] file.
After loading the board you should be able to check if it actually uses the
newly build [Device Tree] file like this:

```sh
find /sys/firmware/devicetree -name "*nunchuk*"
/sys/firmware/devicetree/base/ocp/interconnect@48000000/segment@0/target-module@2a000/i2c@0/nunchuk@52
```

> *NOTE:* it does not mean that the [I2C] controller or the device actually
  work. As a matter of fact we don't even need to connect the *Wiichuk* to the
  board. This test just shows that the [Device Tree] used by the kernel contains
  the node for the *Wiichuk* device.

# I2C device driver

With the [Device Tree] in place we can now create a template of the [I2C]
driver. At this stage the driver will not do anything useful and is only needed
to check that our [Device Tree] is correct and the driver code is called by
the kernel when it "detects" the device.

> *NOTE:* as it was mentioned above [I2C] is a very simple bus and does not
  support enumerating devices connected to the bus. So when it comes to [I2C]
  detecting device means that the kernel found it in the [Device Tree] or
  somehow else was instructed that the device should be there.

[I2C] driver is required to prepare and register
[struct i2c_driver structure](https://elixir.bootlin.com/linux/v5.8-rc4/source/include/linux/i2c.h#L256).
This structure contains the information required to match the [Device Tree]
nodes to the driver that can serve them and a few callbacks for the kernel to
call during the device and device driver lifesycle.

For now we will use just two calbacks: *probe* and *remove*. Those are called
when the device is "detected" and "removed". In those callbacks we will do
nothing except logging, so at this point it's a bit too early to explain the
parameters of the functions.

What is more important for us the is *driver* field of the
[struct i2c_driver structure](https://elixir.bootlin.com/linux/v5.8-rc4/source/include/linux/i2c.h#L256).
This is a field of type *struct device_driver* that we mentioned above. It
contains parts common for all kinds of drivers (*USB*, *I2C*, *PCI*, etc). And
information required to match [Device Tree] nodes to the device drivers fits
that category.

*of_match_table* field of the *struct device_driver* points to the array of
[struct of_device_id](https://elixir.bootlin.com/linux/v5.8-rc4/source/include/linux/mod_devicetable.h#L260)
that should hold the information to match against the *compatible* attribute in
the [Device Tree] node. For example, for the *Wiichuk* in the device tree we
specified:

```c
nunchuk@52 {
	compatible = "nintendo,nunchuk";
	reg = <0x52>;
};
```

> *NOTE:* the *of* prefix stands for *Open Firmware*. *Open Firmware* is a
  standard that defines [Device Tree].

Therefore if we specify *nintendo,nunchuk* in the *compatible* attribute of the
[struct of_device_id](https://elixir.bootlin.com/linux/v5.8-rc4/source/include/linux/mod_devicetable.h#L260)
the kernel will know that it's this particular driver that handles the device
specified in the [Device Tree].

> *NOTE:* the same driver can handle multiple different devices with different
  values for the *compatible* properly, therefore in the driver we don't just
  have one *struct of_device_id* but an array of structures instead.

> *NOTE:* we don't specify the size of the array anywhere, instead to signal the
  kernel how many entires is there the array must end with a zero-filled entry.

The complete code is available in [krinkinmu/bootlin](https://github.com/krinkinmu/bootlin/tree/0bb0bc566eeff321267d3e15fa2ee4ccf5289c43/nunchuk)
repository. Here is the kernel module code:

```c
// SPDX-License-Identifier: GPL-2.0
#include <linux/init.h>
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/i2c.h>

static int wiichuk_i2c_probe(struct i2c_client *client,
			     const struct i2c_device_id *id)
{
	(void) client;
	(void) id;

	pr_alert("Wiichuk device \"detected\".\n");
	return 0;
}

static int wiichuk_i2c_remove(struct i2c_client *client)
{
	(void) client;

	pr_alert("Wiichuk device \"removed\".\n");
	return 0;
}

static const struct of_device_id wiichuk_of_match[] = {
	{ .compatible = "nintendo,nunchuk" },
	{ },
};

MODULE_DEVICE_TABLE(of, wiichuk_of_match);

static const struct i2c_device_id wiichuk_i2c_id[] = {
	{ "wiichuk_i2c", 0 },
	{ },
};

MODULE_DEVICE_TABLE(i2c, wiichuk_i2c_id);

static struct i2c_driver wiichuk_i2c_driver = {
	.driver = {
		.name = "wiichuk_i2c",
		.of_match_table = wiichuk_of_match
	},
	.probe = wiichuk_i2c_probe,
	.remove = wiichuk_i2c_remove,
	.id_table = wiichuk_i2c_id,
};

module_i2c_driver(wiichuk_i2c_driver);
MODULE_LICENSE("GPL");
MODULE_DESCRIPTION("Nintendo Wiichuk I2C driver");
MODULE_AUTHOR("Krinkin Mike <krinkin.m.u@gmail.com>");
```

It's very similar to the code for the simple kernel module above. However you
may notice that there is no *init* and *exit* functions. For common bus types
kernel provides helper macroses for drivers if those *init* and *exit*
functions are trivial.

For [I2C] such a helper macros is *module_i2c_driver*. All we need is to point
the macros to the *struct i2c_driver* for our driver and it will generate the
appropriate *init* and *exit* functions for our module.

You can also see in the code above the *wiichuk_of_match* array of
*struct of_device_id* that is used to match [Device Tree] nodes against this
driver as described above.

> *NOTE:* the *wiichuk_i2c_id* array is used for purposes very similar to the
  *wiichuk_of_match*. It's just a different mechanism to match the device with
  a driver that doesn't use [Device Tree]. I'm not going to cover it here.

*Makefile* for the [I2C] driver is not different from the *Makefile* for the
simple kernel module that was provided above (though, you may want to change
the name of the *C* file with the code).

The procedure to build, share and load/unload the module is also exactly the
same as with the simple kernel module that was shown at the very begining.

# Instead of the conclusion

It appear to be often the case that configuration for the code is more
complicated than the code itself. That's what we see here as well:

  * the code for the kernel module is simple, which is not surprising given
    that it doesn't really do anything useful yet
  * however to make it work we need to go through quite involved process of
    creating the right [Device Tree] to match the driver with the devices.

[I2C] is a simple bus, but that simplicity results in the more complicated
configuration. More complicated buses like *USB* allow to automatically
discover new devices connected to the bus, and, therefore, are more flexible
as we don't need to provide configuration enumerating all the devices in
advance.
