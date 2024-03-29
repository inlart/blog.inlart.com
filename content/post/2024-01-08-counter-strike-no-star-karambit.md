+++
title = "The no-star Karambit in Counter-Strike"
date = 2024-01-08

[taxonomies]
categories = ["Technical"]
tags = ["Counter-Strike"]
+++

Counter-Strike is a very popular first-person shooter.
I’m playing the game since 2017 and it’s definitely one of my favorite games since then.
Since the Counter-Strike: Global Offensive [Arms Deal update](https://www.counter-strike.net/armsdeal) in 2013 the game includes weapon skins. Weapon skins only change the appearance of a weapon but do not provide any in-game advantages. Since the Arms Deal update in 2013 there are new weapon skins added regularly to the game.
In 2023 the game's successor Counter-Strike 2 was released which allowed players to also use weapon skins obtained in Counter-Strike: Global Offensive.
Although not necessary for reading this post, there is an interesting [GDC talk](https://www.youtube.com/watch?v=gd_QeY9uATA) about why and how weapon skins were added to the game.

Weapon skins can be obtained by in-game drops or by unboxing cases for real money.
Whenever a new weapon skin is created the weapon skin properties are randomly generated. It’s random what skin you get but also how used it looks (wear) and some weapon skin textures are even randomly put on the weapon (seed).
This leads to some very rare combinations which get traded for a lot of money.

This post isn’t about a weapon skin that is just super rare to be dropped. It’s about a weapon skin that is impossible to unbox. The no-star Karambit.

## No-star Karambit

What makes the no-star Karambit different to all other Karambits is, that when inspecting it in game the displayed name is just “Karambit” instead of the usual name "★ Karambit". In Counter-Strike all knives and gloves usually have a "★" preceding their displayed name while other weapon skins do not have the preceding "★".
The no-star Karambit was created by steam support and is different in some other ways as well as we will see in the [technical analysis](#any-difference).

The weapon skin can be inspected in-game if you have Counter-Strike 2 installed using this link: [steam://rungame/730/76561202255233023/+csgo_econ_action_preview S76561198218243965A4667288050D12026300745402096731](steam://rungame/730/76561202255233023/+csgo_econ_action_preview%20S76561198218243965A4667288050D12026300745402096731)

## Counter-Strike Weapon Skin Information

If you want to check out a weapon skin you would usually open an inspect link to look at it in-game just like the link in the previous section.
As the game hides some information we need to check the actual properties of the weapon. There are libraries that do exactly that. For python there is the [csgo](https://github.com/ValvePython/csgo) module which I used to retrieve the weapon skin properties.
The data exchange is done using protobuf. The relevant protobuf message for us is [CEconItemPreviewDataBlock](https://github.com/ValvePython/csgo/blob/ed81efa8c36122e882ffa5247be1b327dbd20850/protobufs/cstrike15_gcmessages.proto#L801).

A few months ago there was also a [tool](https://github.com/dr3fty/cs2-inspect-gen) published which allows to generate inspect links for weapon skins that do not actually exist. This tool allows to inspect skins in-game which do not exist.

### Any difference?

The first obvious thing to try now is to get the no-star Karambit weapon skin information and compare it to a regular Karambit.
Requesting the information for the no-star Karambit returns the following:
```
itemid: 4667288050
defindex: 507
paintindex: 0
rarity: 6
quality: 4
inventory: 3221225479
origin: 7
```

The information of a regular Karambit contains:
```
itemid: 25750358286
defindex: 507
paintindex: 0
rarity: 6
quality: 3
paintwear: 1031179971
paintseed: 434
inventory: 1
origin: 8
```

The difference of the `itemid` and `inventory` are not relevant.

The `quality` difference is what makes the Karambit actually show up without the star but more on that [later](#quality).

Unboxed weapon skins have an `origin` of `8` while weapon skins generated by the steam support have an `origin` of `7`.

The no-star Karambit also doesn't include a `paintwear` and `paintseed`. The missing `paintwear` makes the weapon skin show up as the lowest wear item in weapon skin databases like [csfloat.com](https://csfloat.com/db).
There is also a [Bayonet](steam://rungame/730/76561202255233023/+csgo_econ_action_preview%20S76561198076597766A99309927D758371152553511212) with a star but with the missing `paintwear`. 
This Bayonet also misses the `paintwear` and `paintseed`.

### Quality

The different `quality` makes the no-star not show the "★" in its name.
This theory can be easily tested with the tool to generate inspect links for non-existing weapon skins.
You can create one [inspect link](steam://rungame/730/76561202255233023/+csgo_econ_action_preview%200018FB03200028063004A9B4D108) with `quality = 4` and one [inspect link](steam://rungame/730/76561202255233023/+csgo_econ_action_preview%200018FB03200028063003842B1C10) with `quality = 3` but otherwise have the exact same properties.

While knife weapon skins come with the "★" in the displayed name the displayed name for weapons like the AK-47 doesn't include it because the `quality` is not `3`.
By generating an inspect link and setting the `quality` to `3` we can create [inspect links](steam://rungame/730/76561202255233023/+csgo_econ_action_preview%20001807202C28053003409505AF5D32D4) for weapon skins like the AK-47 which do display its name with a "★" although this usually wouldn't show.
