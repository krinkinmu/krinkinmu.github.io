---
layout: default
title: Nintendo Wiichuk interface and input devices
excerpt_separator: <!--more-->
tags: beaglebone linux-kernel i2c input-device
---
[Bootlin]: https://bootlin.com/ "Bootlin"
[Linux Kernel]: https://www.kernel.org/ "Linux Kernel"
[BeagleBone Black]: https://beagleboard.org/black "BeagleBone Black"
[BeagleBone Black Wireless]: https://beagleboard.org/black-wireless "BeagleBone Black Wireless"
[Nintendo Wiichuk]: https://www.olimex.com/Products/Modules/Sensors/MOD-WII/MOD-Wii-UEXT-NUNCHUCK/open-source-hardware "Nintendo Wiichuk"
[Device Tree]: https://en.wikipedia.org/wiki/Device_tree "Device Tree"
[U-Boot]: https://github.com/u-boot/u-boot "U-Boot"
[I2C tools]: https://i2c.wiki.kernel.org/index.php/I2C_Tools "I2C Tools"
[the previous post]: {% post_url 2020-07-18-nintendo-wiichuk-i2c %} "the previous post"
[Bazel]: https://en.wikipedia.org/wiki/Bazel_(software) "Bazel"
[Maven]: https://en.wikipedia.org/wiki/Apache_Maven "Maven"
[CMake]: https://www.kernel.org/doc/html/latest/kbuild/kbuild.html "CMake"
[USB]: https://en.wikipedia.org/wiki/USB "USB"
[I2C]: https://en.wikipedia.org/wiki/I%C2%B2C "I2C"
[ARM]: https://en.wikipedia.org/wiki/ARM_architecture "ARM"
[x86]: https://en.wikipedia.org/wiki/X86 "x86"
[evtest]: https://cgit.freedesktop.org/evtest/ "evtest"

I continue going through [Bootlin] training materials on embedded systems and
[Linux Kernel]. In [the previous post] we configured the second [I2C]
controller the [BeagleBone Black] or [BeagleBone Black Wireless] and connected
the [Nintendo Wiichuk] device to the board.

In this article I will look a bit deaper into the [Nintendo Wiichuk] device
interface and how can our driver present the device as an input device.

All the sources used in this article are available on
[GitHub.](https://github.com/krinkinmu/bootlin)

<!--more-->

# Preparations

Before we jump into the device driver itself we should start with figuring out
how to test that it works. I will use [evtest] utility to test that our new
device driver actually reports events.

Here is how to download the code and cross-compile it for the board:

```sh
cd ~/ws
git clone https://gitlab.freedesktop.org/libevdev/evtest
cd evtest
autoreconf -iv
./configure LDFLAGS="-static" --host=arm-linux-gnueabi
make
```

> *NOTE:* you would need Autotools installed in your system for the commands
  above to work.

> *NOTE:* as with other articles *~/ws* is my workspace directory where I store
  all the code and where the code for [evtest] is downloaded.

> *NOTE:* I use static compilation so that the resulting binary is self
  sufficient and easy to install by just coping it to the board.

If the build is succesful you should have the binary named *evtest* in the
current directory. All it remains to do is to share it with the board:

```sh
cp evtest ~/ws/nfsroot/bin
```

> *NOTE:* *~/ws/nfsroot* is the root file system directory that I share with the
  device over NFS.

# Nintendo Wiichuk protocol

Now we have a tool we can use to test our input device and we know how to
connect the device to the board. The next step would be to understand how to
communicate with the [Nintendo Wiichuk].

First of all [Nintendo Wiichuk] needs some initialization, a handshake signal.

> *NOTE:* I failed to find any form of an official datasheet that would describe
  the protocol used. There are still plenty of examples that cover the protocol,
  but this part of the article doesn't really provide any information not
  covered by other materials on the Internet.

The handshake consists of two commands each of them are two bytes. We can think
of those two commands as opaque strings, but it appears that the first byte of
the command identifies a register internal to the [Nintendo Wiichuk] device and
the second byte is a value we want to write to that register.

The commands we need to send to the [Nintendo Wiichuk] device are:

 * *0xf0*, *0x55*
 * *0xfb*, *0x00*

We can interpret it as setting register *0xf0* of the [Nintendo Wiichuk] device
to *0x55* and setting register *0xfb* to *0x00*. This interpretation might seem hardly better than looking at those two commands as opaque strings because we
still don't really know what registers *0xf0* and *0xfb* are and what values
*0x55* and *0x00* actually mean. However, the language of reading and writing
registers is slightly easier to understand, therefore that's what I'm going to
use.

The next important part of the protocol is to actually read the data from
[Nintendo Wiichuk] device. As you may have figured out [Nintendo Wiichuk] is
a joistic of some sort or, in other way, it's a pointer device. So we should be
able to read some kind of coordinates from the device.

The complete state that describes the coordinates, acceleration and state of the
buttons on [Nintendo Wiichuk] is six bytes long and is stored in register
*0x00*. So to get the state we need to read register *0x00* that stores a six
byte value.

There is a quirk however. It appears that [Nintendo Wiichuk] only updates the
value of the *0x00* register after it's been read. That means that in order to
get the latest state we'd need to read the register twice: first time to tell
the device to update the state, and the second time to get the newly updated
state. We will deal with this quirk when we will talk about the [Linux Kernel]
input framework, but for initial testing we will just read the register twice.

Finally, we need to understand how to interpret those size bytes of the state.
You can find the detailed description in
[the bootlin materials,](https://bootlin.com/labs/doc/nunchuk.pdf) There you
can see that the state encodes:

 * whether buttons *Z* and *C* are pressed or released
 * [Nintendo Wiichuk] joistic position in the form of *y* and *x* coordinates
 * acceleration along *X*, *Y* and *Z* axises.

> *NOTE:* I don't yet have an idea of how those *X*, *Y* and *Z* axises are
  defined for the accelerometer inside the [Nintendo Wiichuk] device.

# Linux Kernel interface for I2C

So now we know the protocol, but that's not enough. In addition to knowing the
protocol we also need to know how to implement it inside the [Linux Kernel]. In
other words we need to know how to actually send and receive bytes to and from
the [I2C] device.

That's not really hard to do. There are just a couple of function that we need
to know:

 * *i2c_master_send* to send some bytes to an [I2C] device
 * *i2c_master_recv* to receive some bytes from an [I2C] device.

Both functions return a negative value, if the operation failed. Otherwise it
returns the number of bytes sent/received. The [I2C] protocol itself doesn't
mandate a fixed message size, so potentially you can receive (or send) an
arbitrary amount of data (with some limitations). That's why the interface looks
like a generic read/write interface.

In our case however for [Nintendo Wiichuk] device we know the response size. For
example, the size of *0x00* register is six bytes and, therefore when we read
the *0x00* register we always expect to receive exactly six bytes when we use
the *i2c_master_recv* function.

So while the interface seem to allow variable size of messages and responses,
the sizes of messages and responses for [Nintendo Wiichuk] devices are actually
known and fixed in advance and we do not need to care about partial reads and
writes.

> *NOTE:* I will skip the handshake part of the device initialization as it
  also boils down just using the functions presented above and does not provide
  any additional insight. The full code still can be found on
  [GitHub.](https://github.com/krinkinmu/bootlin)

So let us take a look at how we can read the data from the [Nintendo Wiichuk]:

```c
struct wiichuk_state {
	char data[6];
};

static int wiichuk_i2c_read_state(
	const struct i2c_client *client, struct wiichuk_state *state)
{
	const char msg[] = {0x00};
	int ret;

	ret = i2c_master_send(client, msg, sizeof(msg));
	if (ret < 0)
		return ret;

	ret = i2c_master_recv(client, state->data, sizeof(state->data));
	if (ret < 0)
		return ret;

	return 0;
}
```

The two important parts in the code above are calls to *i2c_master_send* and
*i2c_master_recv*. The *i2c_master_send* call tells the device that we want to
read the content of the register *0x00* by literally sending *0x00* to the
device. The *i2c_master_recv* is used to read the response from the device.

> *NOTE:* I create *struct wiichuk_state* to avoid using raw pointers to the
  receive buffer. Using pointers just raises questions regarding the size of
  the buffer and whether it's enough or not. We can make it work even with raw
  pointers, but I prefer the code that raises less questions even if it's
  correct.

In the code above we didn't explicitly specify the [I2C] address of the device
we are communicating with. The *i2c_client* structure contains the [I2C]
address among other things. The structure is actually provided by the kernel
(look at the signature of the *probe* method for our driver) and we don't need
to create it and the kernel knows the address of the device from the
[Device Tree] node for the device, specifically, property *reg*.

What remains is to parse the result and take into account the quirk of
[Nintendo Wiichuk] device that requires us to read the *0x00* register twice:

```c
struct wiichuk_registers {
	unsigned x;
	unsigned y;
	unsigned x_acc;
	unsigned y_acc;
	unsigned z_acc;
	bool c_pressed;
	bool z_pressed;
};

static int wiichuk_i2c_read_registers(
	const struct i2c_client *client, struct wiichuk_registers *regs)
{
	struct wiichuk_state state;
	int err;

	err = wiichuk_i2c_read_state(client, &state);
	if (err)
		return err;

	usleep_range(10000, 20000);

	err = wiichuk_i2c_read_state(client, &state);
	if (err)
		return err;

	wiichuk_parse_registers(&state, regs);

	return 0;
}
```

> *NOTE:* I will not provide the implementation of the *wiichuk_parse_registers*
  as the function is of limited interest and you can find one possible
  implementation on [GitHub.](https://github.com/krinkinmu/bootlin)

As was mentioned before we read the *0x00* registers twice and therefore the
function *wiichuk_i2c_read_state* is also called twice, but results of the first
call are discarded.

Intersting thing is the call to *usleep_range* between the calls to
*wiichuk_i2c_read_state*. As you may guess from the name, the function
introduces a delay. In this particular case the delay will be somewhere
beetween 10 and 20 ms.

Experimenting with the device it appears that it takes some time after the
first read of the *0x00* register for the device to actually update its
internal state. So if you execute the two reads too fast you may see stale
results. That delay there is to avoid specifically that situation.

> *NOTE:* 10ms delay is probably too much, especially if you consider that
  [Nintendo Wiichuk] is an imput device for a gaming console where delays might
  be important. I didn't try to figure out what would be a minimum delay
  required and just used the same value as in the [Bootlin] materials. We can
  reconsider the value later when we try to expose the device as an input
  device to the system.

# Input framework

[Linux Kernel] has a framework that unifies all the input devices (mice,
keyboards, touchpad and plenty of other weird input devices). That framework
allows driver to specify what kind of events the device can generate (for
example, button push, button release, mouse move, etc) and provides an
interface for the driver to report those events. The input framework takes care
of exposing those events to the userspace in a uniform and organized manner,
so we don't really need to think about it.

> *NOTE:* You can find what kinds of even types the input framework supports
  in [Documentation/input/event-codes.txt.](https://www.kernel.org/doc/Documentation/input/event-codes.txt)

The primary data structure that we need to be familiar with when working with
the input framework is [struct input_dev.](https://elixir.bootlin.com/linux/v5.7.8/source/include/linux/input.h#L131)

Because we are working with a device that cannot generate interupts, there is
another structure that we also should get familiar with: [struct input_polled_dev.](https://elixir.bootlin.com/linux/v5.7.8/source/include/linux/input-polldev.h#L34)

As the name suggests the *struct input_polled_dev* is used for devices that
cannot generate signals and, therefore, have to be queried regularly to
generate input events.

Inside *struct input_polled_dev* you, at the very least, need to initialize the
*poll* field that stores the polling callback. This callback will be called
periodically to query the device. Additionally you can specify the polling
interval, because by default it's half a second, which is quite large for an
input device.

> *NOTE:* it's my understanding that human reaction is on order of couple of
hundreds of milliseconds, so half a second is quite noticable compared to
typica human reaction time.

Let's for now just initialize our structures to be able to report button events:

```c
static void wiichuk_input_dev_init(struct input_polled_dev *polled_input)
{
	struct input_dev *input = polled_input->input;

	polled_input->poll = &wiichuck_poll;
	polled_input->poll_interval = 5;

	input->name = "Nintendo Wiichuk";
	input->id.bustype = BUS_I2C;

	set_bit(EV_KEY, input->evbit);
	set_bit(BTN_C, input->keybit);
	set_bit(BTN_Z, input->keybit);
}
```

As you can see the *struct input_polled_dev* contains pointer to the
*struct input_dev*. One way to think about relations between
*struct input_polled_dev* and *struct input_dev* is that
*struct input_polled_dev* represents a special case of *struct input_dev*.

When we initialize the *struct input_dev* we specify what types of events the
device can generate. *EV_KEY* represents all kinds of button press/release
events (like keyboard button presses and releases). More specifically, we
specify that the device has two buttons: *BTN_C* and *BTN_Z*.

> *NOTE:* *BTN_C* and *BTN_Z* are already defined in the [Linux Kernel] source
  code. I can imagine that if you have a completely new device it might have
  buttons that would require a new definition.

Outside of *struct input_dev* part of the *struct input_polled_dev* the code
above also specifies the poll callback *wiichuk_poll* and the polling interval
in milliseconds.

> *NOTE:* because the [Linux Kernel] will query the device regularly we may drop
  the logic that reads the registers multiple times to account for the quirk of
  [Nintendo Wiichuk].

The signature of the polling callback is defined as follows:

```c
void wiichuk_poll(struct input_polled_dev *polled)
```

One important thing to mention here is that we need a pointer to a valid
*struct i2c_client* to communicate with the [Nintendo Wiichuk] device and the
callback function does not have it as an argument. So we need to think of a way
to link *struct i2c_client* pointer to the *struct input_polled_dev*.

Generally speaking a driver might need more than just *struct i2c_client* in
general. Some drivers may need to communicate with multiple physical devices and
keep track of some state. So it's not uncommon to see a sort of a driver
structure that holds all the things the driver needs. Let's create a structure
like that:

```c
struct wiichuk_dev {
	struct i2c_client *i2c_client;
};
```

Inside the *probe* function we can allocate and initialize this structure (we
do have a pointer to *struct i2c_client* inside the *probe* function).

Then to link this structure *struct input_polled_dev* has a special field
called *private*. The field type is just a *void* pointer, so you can store
there a pointer to *struct wiichuk_dev* to later use it inside the polling
callback.

With this information we now can take a look at how the polling callback might
look like in our case:

```c
static void wiichuk_poll(struct input_polled_dev *polled)
{
	struct wiichuk_dev *wiichuk = (struct wiichuk_dev *)polled->private;
	struct wiichuk_registers regs;
	int err;

	err = wiichuk_i2c_read_registers(wiichuk->i2c_client, &regs);
	if (err) {
		pr_err("Failed to read the Nintendo Wiichuk state: %d\n", err);
		return;
	}

	input_event(polled->input, EV_KEY, BTN_Z, regs.z_pressed);
	input_event(polled->input, EV_KEY, BTN_C, regs.c_pressed);
	input_sync(polled->input);
}
```

> *NOTE:* we would need to modify *wiichuk_input_dev_init* to set the *private*
  field to point to the *struct wiichuk_dev*.

A final note on the input framework is that once the *struct input_polled_dev*
and *struct input_dev* are initialized we need to register the new input device
in the framework.

For polled devices like [Nintendo Nunchuk] we should use
[input_register_polled_device.](https://elixir.bootlin.com/linux/v5.7.8/source/include/linux/input-polldev.h#L53)

# A short note on memory allocation

As you may have noticed, we need to allocate quite a few various structures for
the driver. Depending on the structure a different way is used. For example,
*struct input_polled_dev* is a generic structure and [Linux Kernel] provides
[devm_input_allocate_polled_device](https://elixir.bootlin.com/linux/v5.7.8/source/include/linux/input-polldev.h#L52)
function to do that.

Naturally, for *struct wiichuk_dev* that we defined ourself [Linux Kernel] does
not have a special function, so we will use a generic
[devm_kzalloc](https://elixir.bootlin.com/linux/v5.7.8/source/include/linux/device.h#L210)
function.

All *devm_* allocation functions require to provide an instance of
*struct device* that this allocated memory will be associated with. The memory
will be automatically deallocated when the device goes away. This simplifies
memory management and error handling a bit.

> *NOTE:* [Linux Kernel] has generic memory allocation functions that are not
  linked with the kernel driver framework as well.

# Testing

Once the code is ready, the module is compiled and shared with the board we can
test if it generates input events using [evtest]. Just on the board do this:

```sh
evtest
No device specified, trying to scan all of /dev/input/event*
Available devices:
/dev/input/event0:	Nintendo Wiichuk
Select the device event number [0-0]: 0
```

The command will ask you yo pick one of the input devices. In my case I had
only one device attached, so there isn't much to choose from. Once you picked
the [Nintendo Wiichuk] try to push buttons on the device. If everything is
correct you should be able to see the generated inputs in the [evtest] output:

```sh
Input driver version is 1.0.1
Input device ID: bus 0x18 vendor 0x0 product 0x0 version 0x0
Input device name: "Nintendo Wiichuk"
Supported events:
  Event type 0 (EV_SYN)
  Event type 1 (EV_KEY)
    Event code 306 (BTN_C)
    Event code 309 (BTN_Z)
Properties:
Testing ... (interrupt to exit)
Event: time 3980.650990, type 1 (EV_KEY), code 306 (BTN_C), value 1
Event: time 3980.650990, -------------- SYN_REPORT ------------
Event: time 3980.990982, type 1 (EV_KEY), code 306 (BTN_C), value 0
Event: time 3980.990982, -------------- SYN_REPORT ------------
```

# Instead of the conclusion

This was quite a long one, but it covered quite a few things as well. It's
by no means an exhaustive introduction into the [Linux Kernel] input framework,
but that's a starting point. I will try to follow up this article with a
shorter article that would enable support for the joystick on the
[Nintendo Nunchuk].
