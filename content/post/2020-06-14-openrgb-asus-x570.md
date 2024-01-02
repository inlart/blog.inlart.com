+++
title = "Adding Asus X570 Support To OpenRGB"
date = 2020-06-14
updated = 2023-07-17

[taxonomies]
tags = ["OpenRGB", "Reverse Engineering", "RGB", "Hardware"]
categories = ["Technical"]
+++

The story of me adding a driver to control Asus X570 RGB starts somewhere in late 2019 when I created a PC parts list to replace my seven years old PC. Since I didn't have any issues with my Asus mainboard in my old PC I went with Asus again and picked the Asus Prime X570-Pro.

After building the PC and setting everything up I quickly wondered if there was a way to control my lighting. I looked at the official Asus site and was quickly disappointed since they only support Windows. After that I took a look at open source projects to control RGB devices. There are actually quite a lot of them. Most of them only supporting a single device or a collection of similar devices. But OpenRGB was different. After originally only supporting Asus mainboards the project added drivers for other devices to provide a single place for all your RGB controls.

I tinkered around with the project and added support for the water cooler I use, the NZXT Kraken X62. It was actually pretty easy to do so. There were already other projects that had support for it and in addition there was a nice summary with captures of the protocol which made it really easy to add a driver to OpenRGB. I had not worked with USB communication before that, but since other drivers already did communicate with the device via USB I just followed what they did at that point.

After adding support for my water cooler the next on my list was the mainboard. In comparison to the NZXT Kraken driver it was much more challenging than I originally anticipated. At the point of creating the driver there were no other projects I could rely on. The driver also had to support different mainboards. They can have a different number of integrated LEDs or addressable headers which actually reflects in the protocol. In addition to that the Asus software to control the mainboard didn't find any devices in my Windows VM. Thus, I had to rely on captures and the really good analysis done by other members of the community to figure out the protocol and implementation.

## Protocol

The communication is done via USB and the RGB controller has VID `0B05` and PID `18F3`.
All sent and received messages are 65 byte long. If the message doesn't fill the 65 byte the rest is zero-filled.

### Mainboard Information

#### LED EC1 Version

To request a LED EC1 Version response from the controller the message `0xEC82` is sent.
The controller responds with a `0xEC02` message followed by an ASCII string.
It only contains upper case letters, numbers and hyphens.
This version string for my Prime X570-Pro is `AULA3-6K75-0109` which can also be found in the BIOS. The string is currently not used by the driver but shown in the information tab in OpenRGB.

#### Configuration Table

To request the configuration table from the controller the message `0xECB0` is sent.
The controller returns a message starting with `0xEC30`.
Each line in the config table is 6 bytes long with the first entry starting at address `0x04` of the response message.

The first two bytes are always set to `0x1E9F` for all X570 boards I have seen.
The important values taken from the configuration table are located at `0x02` for the number of addressable channels and `0x1B` for the number of mainboard LEDs (this includes integrated LEDs and the RGB headers).

The response by my mainboard looks like this:
```
1E 9F 01 01 00 00
78 3C 00 00 00 00
00 00 00 00 00 00
00 00 00 00 00 00
00 00 00 08 09 02
00 00 00 00 00 00
00 00 00 00 00 00
00 00 00 00 00 00
00 00 00 00 00 00
00 00 00 00 00 00
```

### Setting Colors and Effects

#### Channel ID

There are generally two different types of channels.
A type for a _fixed_ number of LEDs (i.e. the mainboard, there is only a single fixed channel) and a channel for each _addressable_ RGB header.
For different messages in the protocol there are different ids sent to refer to certain channels.

The channel id sent for effects:

| channel type | channel_effect id | comment |
| --- | --- | --- |
| fixed | 0x00 ||
| addressable | 0x01 | For mainboards that only provide addressable RGB headers the addressable channel_effect id is `0x00` |

Since the effect can only be changed for each type and not actual channel one could also refer to this as a channel type instead of an id.

For the direct mode the fixed channel has id `0x04` and the addressable ids start at `0x00`:

| channel type | channel_direct id |
| --- | --- |
| fixed | 0x04 |
| addressable #n | n-1 |

#### Set Effect Mode

Modes that are currently supported by OpenRGB:

| id | mode name | number of colors |
| --- | --- | --- |
| 0x00 | off | 0 |
| 0x01 | static | 1 |
| 0x02 | breathing | 1 |
| 0x03 | flashing | 1 |
| 0x04 | spectrum cycle | 0 |
| 0x05 | rainbow | 0 |
| 0x07 | chase fade | 1 |
| 0x09 | chase | 1 |
| 0xff | direct | see direct mode colors |

To switch the mode for a channel the following message is sent to the mainboard controller:

| index | mode |
| --- | --- |
| 0x00 | 0xEC |
| 0x01 | 0x35 |
| 0x02 | channel_effect id |
| 0x03 | 0x00 |
| 0x04 | 0x00 |
| 0x05 | mode id |

#### Colors

This messages are used to change the color for every mode except the direct mode.

| index | mode |
| --- | --- |
| 0x00 | 0xEC |
| 0x01 | 0x36 |
| 0x02 | LED mask |
| 0x03 | LED mask |
| 0x04 | 0x00 |
| 0x05 + offset | colors |

The number of colors sent depends on the channel type. For the fixed channel it is sending a value for each LED and for the addressable only a single color. Colors are sent in RGB order with 1 byte each (red, green, blue).

The color data is shifted by an `offset` depending on the channel id. It is shifted by the number of colors that would be sent by all the previous channels and the LED mask is set accordingly.
The message is only sent if the effect mode supports colors.

#### Direct Mode Colors

This message is used to set the color for the direct mode.
The direct mode lets you set the color of each LED individually.

| index | mode |
| --- | --- |
| 0x00 | 0xEC |
| 0x01 | 0x40 |
| 0x02 | apply \| channel_direct id |
| 0x03 | start led |
| 0x04 | led_count |
| 0x05 | color data |

Setting `apply` (`0x1 << 4`) makes the colors actually show up. This is usually done in the last direct mode packet. The color data contains `led_count` RGB values (max 20).

#### Commit

There is also a commit message to preserve the sate of modes/colors if the mainboard isn't powered:

| index | mode |
| --- | --- |
| 0x00 | 0xEC |
| 0x01 | 0x3F |
| 0x02 | 0x55 |
