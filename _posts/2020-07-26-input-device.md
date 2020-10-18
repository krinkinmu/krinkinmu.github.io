---
layout: post
title: Nintendo Wiichuk Joystick
excerpt_separator: <!--more-->
tags: beaglebone linux-kernel i2c input-device
---
[Bootlin]: https://bootlin.com/ "Bootlin"
[Linux Kernel]: https://www.kernel.org/ "Linux Kernel"
[BeagleBone Black]: https://beagleboard.org/black "BeagleBone Black"
[BeagleBone Black Wireless]: https://beagleboard.org/black-wireless "BeagleBone Black Wireless"
[Nintendo Wiichuk]: https://www.olimex.com/Products/Modules/Sensors/MOD-WII/MOD-Wii-UEXT-NUNCHUCK/open-source-hardware "Nintendo Wiichuk"
[the previous post]: {% post_url 2020-07-25-input-device %} "the previous post"
[evtest]: https://cgit.freedesktop.org/evtest/ "evtest"

In this short note we will build on top of [the previous post] and add support
for the joystick in our [Nintendo Wiichuck] device.

All the sources used in this article are available on
[GitHub.](https://github.com/krinkinmu/bootlin)

<!--more-->

# Joystick

In [Linux Kernel] input framework joystick is considered an absolute pointing
device, meaning that it will report absolute events with absolute coordinates
in the known range.

To tell [Linux Kernel] that our driver can report absolute coordinates we need
to initialize *struct input_dev* representing our device properly:

```c
set_bit(ABS_X, input->absbit);
set_bit(ABS_Y, input->absbit);
```

In addition to that we also need to tell the kernel the range of coordinates
for our device. In case of [Nintendo Wiichuk] the coordinates are in range
starting from 0 and up to 255 both inclusive.

> *NOTE:* according to [input-programming.html](https://www.kernel.org/doc/html/latest/input/input-programming.html),
  it's ok to occasionally report events outside the range, but it's not ok if
  the input device cannot reach the lower and upper bounds on the coordinates.
  It makes sense for an absolute pointing device. Imagine if on your phone you
  could not never click one of the corners of the touch screen.

Besides the range there are two additional parameters that we can specify that
are called *fuzz* and *flat*. Parameter *fuzz* specify how much noice the
device can produce and allows to filter it.

> *NOTE:* I don't really know anything about the algorithm used to filter the
  noice, but you can specify the value 0 relatively safely and increase it if
  the noice will prove to be a problem.

To explain what the parameter *flat* means I should first point out that
joystick always returns to the central position when you stop touching it. So
one question we might ask is what coordinates does that central position has?

Naive guess would be something around 128, but in my case the *X* of the central
position is 126 and the *Y* of the central postiion is 124. So you can see that
the device is not perfectly centered. The parameter *flat* tells the kernel the
size of the "center".

> *NOTE:* touch screens seem to also be absolute pointing devices, but unlike
  with joysticks they don't have a special central position, so I suspect that
  for the touch screens the *flat* parameter does not make much sense.

Here is how you can specify the parameters of the absolute pointing device:

```c
input_set_abs_params(input, ABS_X, 0, 255, 4, 8);
input_set_abs_params(input, ABS_Y, 0, 255, 4, 8);
```

The code above tells the kernel that *X* and *Y* cooridnates reported by the
device change in the range from 0 up to 255, *fuzz* is 4 and *flat* is 8.

To report the events in the polling function all we need to do is:

```c
input_report_abs(polled->input, ABS_X, regs.x);
input_report_abs(polled->input, ABS_Y, regs.y);
```

# Instead of the conclusion

[Linux Kernel] input framework consits of two parts: device drivers and event
handlers. Device drivers communicate with hardware and report events. Event
handlers take the events from the device drivers and present them to the
userspace in the appropriate format.

This post covers the device driver that communicates with hardware that has
a joystick attached to it. However for the userspace to recoginze the device
as a joystick we also need an event handler driver that takes the events from
our driver and makes them available to the userspace via one of the joystick
device files (they are commonly named as *jsX*).

Fortunately enough there are already [generic event handler for joysticks]
(https://elixir.bootlin.com/linux/v5.7.8/source/drivers/input/joydev.c)
provided by the kernel, so there is no need for us to create one. If you create
a device driver file it should be able to match it agains the [Nintendo Wiichuk]
device and expose it to the userspace as a joystick.
