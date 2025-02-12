---
title: "Network_upgrade_pt2"
date: 2024-02-12T14:30:52+02:00
draft: true
---

A while ago I built a NAS, discovered I want faster than gigabit speeds so I then bought myself a switch which has 1 GbE, 2.5 GbE and even 10 GbE SFP+ ports. Many devices in my home network already support 2.5 GbE speeds, so it was about time I be able to actually utilize that speed as well. 

However, when switching from an old HP switch with VLAN capabilities to a Unifi one, it wasn't very straightforward to have all my VLANS from ESXi accessible on the switch, nor to even setup the switch itself. Compared to the HP one where one could simply plug it in and log into the switch and begin configuring it, in the Unifi world things aren't as simple: to even begin configuring the thing, you have to either have a Unifi controller enabled device such as a Unifi Dream Machine or Dream Machine Pro and if you don't, you'll have to install the controller software elsewhere. You can't simply log into the switch and start configuring it; the only thing you can do is connect it to a controller. Bit weird if you ask me.

