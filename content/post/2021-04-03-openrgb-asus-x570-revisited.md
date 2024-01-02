+++
title = "Revisiting \"Adding Asus X570 Support To OpenRGB\""
date = 2021-05-09
updated = 2024-01-01

[taxonomies]
tags = ["OpenRGB", "Reverse Engineering", "RGB", "Hardware"]
categories = ["Technical"]
+++

It's been almost a year since I added support for Asus X570 boards to OpenRGB. I think this is a good time to recap what happened and changed within that year.

In addition to the Asus X570 mainboards, Asus also uses the same controller for its B550 and Z470 boards. This means that the driver is actually used for quite some mainboards now. To my knowledge there haven't been any major issues with the driver.

## Windows

Due to the USB controller being out-of-spec the initial version did crash in libusb on Windows. [The issue](https://github.com/libusb/libusb/issues/725) wasn't fixed in libusb because out-of-spec USB devices aren't supported. They also didn't try to fix the segfault. Since OpenRGB was already using a [forked libusb version](https://github.com/CalcProgrammer1/libusb) this version was [adjusted](https://github.com/CalcProgrammer1/libusb/commit/1c5a7d84ad5a644e913d7e29c559ea0b97d43fcc) to support the Aura USB controller. On Linux the USB controller didn't cause any issues.

## Bugs?

Besides the initial problems on Windows people usually struggle with the addressable headers. The number of LEDs on the addressable headers is set to zero by default which makes mode changes not work unless the zone is resized. This isn't really a bug but bad UX and will hopefully change at some point.

In addition there was an [actual issue](https://gitlab.com/CalcProgrammer1/OpenRGB/-/issues/361) on some of the mainboards where the colors didn't get set properly. After having a look at a USB capture I realized the bytes 3 and 4 are currently set wrong for these mainboards. In the initial implementation byte 3 was sending the channel id and byte 4 was hard coded to `0xff` and `0x00` for mainboard LEDs and addressable LEDs respectively. As it turned out the two bytes are actually a 16 bit mask to select the colors to be updated. This might seem like a weird mix up but can be easily explained when looking at the bytes for one of the mainboards:

| name | byte 3 | byte4 |
| --- | --- | --- |
| mainboard | `0x00` | `0xff` |
| addressable 1 | `0x01` | `0x00` |
| addressable 2 | `0x02` | `0x00` |

In the table byte 3 could be interpreted as an id with byte 4 being set to `0xff` for the mainboard channel and to `0x00` otherwise. When looking at other boards with a different number of LEDs it can be easily seen that the initial interpretation was wrong and the bytes actually should be interpreted as an LED mask.
The protocol is now updated in the [original blog post](@/post/2020-06-14-openrgb-asus-x570.md).

## Further Improvements

Something I want to change in the near future is the naming of the non-addressable RGB headers. They are currently part of the mainboard zone but have the same naming as the mainboard LEDs which makes it hard to find them.
