---
title: "Network_upgrade"
tags: [networking,unifi,esxi]
date: 2024-09-29T20:52:06+03:00
draft: true
---

There's been many changes in my internal network in the past few months. For several years I've wanted to build a NAS in order to start self-hosting more and more things and rely less on the "cloud", and finally did so. I bought a setup as follows:

- **Case:** Fractal Design Define 7 (lots of space for HDDs)
- **Motherboard:** Asrock Z790 Pro RS/D4 (mostly because of the crazy amount of 8 sata ports, making it so I don't have to buy any SATA expansion cards yet.)
- **CPU:** Intel Core i5-13400F (This one I chose mostly due to it having a nice amount of cores/threads for virtualization, 6 performance cores and 4 efficiency-cores)
- **RAM:** Kingston FURY Beast DDR4 3200 MHz CL16 32 Gt (I thought 32 GB would be more than enough but turns out that if you want to run any VMs, more is probably good as ZFS enjoys RAM)
- **PSU:** Asus Rog Strix 750W Gold Aura Edition 80+ Gold (This is really good due to it being really quiet and power efficient according to Cybenetics test; it's also made by Seasonic which has been a very reputable PSU manufacturer for quite a while)
- **HDD:** 6 x 16 TB X18 Exos drives (might as well have enough space to have a big enough buffer for the foreseeable future)

I installed the latest stable version of TrueNAS SCALE and the installation was very simple; you just have to decide what drive to install the OS to, choose a password and choose whether or not you want to have a swap partition. I chose to go with Scale due to it being the newer version of TrueNAS which is built on Linux, whereas the older version, CORE is based on BSD. There's several features such as virtualization which can only be achieved on SCALE, so that and it being updated more frequently, I chose SCALE. Both utilize the OpenZFS file system which will protect my data with features such as self-healing features, incremental snapshots, software-defined raid and even replication to off-site targets. Awesome.

After installation I checked for the latest updates and then already looked into creating my first pool. I chose to go with a Raid-z2 pool for redundancy purposes (meaning that I can lose two entire drives until data loss occurs) and I'm really glad that I did because while writing this, one drive already started failing S.M.A.R.T tests consistently, and during the writing of this I ended up buying a new backup drive already which I truthfully should have just done when I bought the entire setup. Better late than never!

Initially I didn't realise that were I to want to encrypt my pool, I would need to do that before making the pool, so I had to actually start over even though I had already transferred several terabytes of data. After transferring my data back to somewhere I would then move it back from, I re-made the entire pool this time with ZFS encryption enabled, and then went onto actually using the NAS, of course after double and triple-checking I have exported the necessary encryption keys off-site in a secure manner. This leaves me with a pool whose data is encrypted at rest, meaning that if someone were to physically steal the drives in the NAS, they could not read the data without the necessary keys. This encryption does not, however, prevent access to the data on a network level if the device is already turned on, since TrueNAS SCALE decrypts the pool at boot by default, meaning that you still have to properly secure your environment around the NAS.

Despite having never used TrueNAS before, using it is quite straightforward indeed. You can effortlessly create either NFS or SMB shares (or iSCSI targets if your heart so desires) for which you can then manage permissions for, either in a Unix permissions editor or in an Access Control List editor. For NFS shares it's also possible to do more granular permissions handling in the built-in shell of the software either from the web UI or by connecting to TrueNAS over SSH (which is not turned on by default.)

![unixpermissions](/network_upgrade/unixpermissions.png)

