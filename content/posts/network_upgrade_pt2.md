---
title: "Network upgrade part 2: Unifi switch in non Unifi environment"
tags: [networking,unifi,esxi]
slug: "network_upgrade_pt2"
date: 2025-02-12T14:30:52+02:00
draft: false
---

A while ago I built a [NAS](https://blog.nmrs.fi/posts/network_upgrade/), discovered I want faster than gigabit speeds so I then bought myself a switch which has 1 GbE, 2.5 GbE and even 10 GbE SFP+ ports. Many devices in my home network already support 2.5 GbE speeds, so it was about time I be able to actually utilize that speed as well. 

However, when switching from an old HP switch with VLAN capabilities to a Unifi one, it wasn't very straightforward to have all my VLANS from ESXi accessible on the switch, nor to even setup the switch itself. Compared to the HP one where one could simply plug it in and log into the switch and begin configuring it, in the Unifi world things aren't as simple: to even begin configuring the thing, you have to either have a Unifi controller enabled device such as a Unifi Dream Machine or Dream Machine Pro and if you don't, you'll have to install the controller software elsewhere. You can't simply log into the switch and start configuring it; the only thing you can do is to SSH to it in order to connect it to a controller. Bit weird if you ask me. 

(Upon closer examination it seems as though you can actually do things when SSH'd in, but this isn't really openly explained by Ubiquiti themselves https://lazyadmin.nl/home-network/unifi-ssh-commands/)

## Plan

Now my ultimate goal was to install the controller software in a virtual machine, which funnily enough would run on the ESXi that has all the VLANs that the switch  should have access to.  While writing this, I'm starting to realise how ridiculously complex my network is but I suppose it's just hectic enough to learn a lot, which I have. Below is a very simplified topology of what I wanted to achieve:

![topology](/network_upgrade/topology.png)

## Configuration (and issues with it)

I first had to install the controller software to my PC and connect the switch to it by SSHing into it with `ssh ubnt@ipaddress` and using the default password `ubnt`.  After that, I had to run `set-inform http://myipaddress:8080/inform`, and I could already adapt the thing to my network on the controller side of things; as I installed the controller software on my PC, I had to open it up and head to 127.0.0.1:8080 in my browser to see the controller software.

The idea was to first configure the switch through the locally installed controller software so that I could setup all the necessary VLANs, make sure they all work and then install the controller software on a barebones Ubuntu server and connect the switch to it, I remember it was either impossible or very difficult to make this work without installing the "local" controller first, because of the existence of several VLANs and not just one single one, and especially wanting to have the Unifi switch be in a separate management VLAN.

![unificontroller](/network_upgrade/unificontroller.png)

## Desperation & salvation

At this point things got tricky. I thought I could just have one port be tagged with all of my VLANs (which you have to create in the Settings > Network section of the software at the time of writing, set the router as a third party one in a similar setup) coming from the ESXi server and that being it in terms of getting all my VLANS on the switch, but that was not the case. I could not for the life of me figure out how to put the switch into the management VLAN and have it only be accessible there, and noticed many others having this same problem before, such as one self-proclaimed network engineer of 20 years. It really started to seem like it would not be possible to have a standalone Unifi switch in a non Unifi environment, at least in the way I wanted, and everywhere I could see mostly fans of the product line saying that it won't work, that I should have a Unifi firewall as well because that is "how it's designed". Helpful.

After spending a few days on trying to figure this out and even recruiting a friend to help me solve this, we stumbled upon a Reddit comment saying that in order to accomplish this, you have to dedicate a **single port just for the management VLAN** alone, in addition to having **another** port tagged for **all the other desired VLANs** like so:

![salvation](/network_upgrade/salvation.png)

In addition to this, you have to configure the IP settings to reflect the fact that the switch will indeed reside in a management VLAN from UniFi devices > Switch > Cog wheel > IP Settings at the time of writing (and no, these are not my real settings):

![ipsettings](/network_upgrade/ipsettings.png)

It's good to note here that I needed to put in another NIC to the ESXi machine for just the management VLAN for this all to work, since I had no free ports available.

## Success! 

After the above realisation (thank you yet again random Reddit stranger!) I was finally able to not only configure all the necessary VLANs on the switch but also able to install and configure a barebones Ubuntu server to work as the Unifi controller (thanks Ubiquiti for [helping out](https://community.ui.com/questions/UniFi-Installation-Scripts-or-UniFi-Easy-Update-Script-or-UniFi-Lets-Encrypt-or-UniFi-Easy-Encrypt-/ccbc7530-dd61-40a7-82ec-22b17f027776) with this) which was very straightforward and hopefully remains that way; the script took care of everything including automated updates. It is a bit concerning that the scripts seem to be hosted on a single employee's website, however... What happens if they leave the company? Or if they get hacked themselves? 

In any case, if you're reading this, know that I truly appreciate it! I have a ton of ideas for more content soon and even discovered recently that a CISO enjoyed my posts as well - no pressure, right? 
