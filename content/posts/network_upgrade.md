---
title: "Network upgrade part 1: TrueNAS SCALE"
tags: [networking,truenas,unifi,esxi]
date: 2024-11-19T20:56:06+02:00
draft: false
---

There's been many changes in my internal network in the past few months. For several years I've wanted to build a NAS in order to start self-hosting more and more things and rely less on the "cloud", and finally did so. I bought a setup as follows:

- **Case:** Fractal Design Define 7 (lots of space for HDDs)
- **Motherboard:** Asrock Z790 Pro RS/D4 (mostly because of the crazy amount of 8 sata ports, making it so I don't have to buy any SATA expansion cards yet.)
- **CPU:** Intel Core i5-13400F (This one I chose mostly due to it having a nice amount of cores/threads for virtualization, 6 performance cores and 4 efficiency-cores)
- **RAM:** Kingston FURY Beast DDR4 3200 MHz CL16 32 Gt (I thought 32 GB would be more than enough but turns out that if you want to run any VMs, more is probably good as ZFS enjoys RAM)
- **PSU:** Asus Rog Strix 750W Gold Aura Edition 80+ Gold (This is really good due to it being really quiet and power efficient according to Cybenetics test; it's also made by Seasonic which has been a very reputable PSU manufacturer for quite a while)
- **HDD:** 6 x 16 TB X18 Exos drives (might as well have enough space to have a big enough buffer for the foreseeable future)

I installed the latest stable version of TrueNAS SCALE and the installation was very simple; you just have to decide what drive to install the OS to, choose a root password and choose whether or not you want to have a swap partition. I chose to go with SCALE due to it being the newer version of TrueNAS which is built on Linux, whereas the older version, CORE is based on BSD. There's several features such as virtualization which can only be achieved on SCALE, so that, it being updated more frequently and me being more comfortable on Linux, I chose SCALE. Both utilize the OpenZFS file system which will protect my data with features such as self-healing features, incremental snapshots, software-defined raid and even replication to off-site targets. Awesome.

After installation I checked for the latest updates and then already looked into creating my first pool. I chose to go with a Raid-z2 pool for redundancy purposes (meaning that I can lose two entire drives until data loss occurs) and I'm really glad that I did because while writing this, one drive already started failing S.M.A.R.T tests consistently, and during the writing of this I ended up buying a new backup drive already which I truthfully should have just done when I bought the entire setup. Better late than never!

![truenas_webui](/network_upgrade/truenas_webui.png)

Initially I didn't realise that were I to want to encrypt my pool, I would need to do that before making the pool, so I had to actually start over even though I had already transferred several terabytes of data. After transferring my data back to somewhere I would then move it back from, I re-made the entire pool this time with ZFS encryption enabled, and then went onto actually using the NAS, of course after double and triple-checking I have exported the necessary encryption keys off-site in a secure manner. This leaves me with a pool whose data is encrypted at rest, meaning that if someone were to physically steal the drives in the NAS, they could not read the data without the necessary keys. This encryption does not, however, prevent access to the data on a network level if the device is already turned on, since TrueNAS SCALE decrypts the pool at boot by default, meaning that you still have to properly secure your environment around the NAS.

Despite having never used TrueNAS before, using it is quite straightforward indeed. You can effortlessly create either NFS or SMB shares (or iSCSI targets if your heart so desires) for which you can then manage permissions for, either in a Unix permissions editor or in an Access Control List editor. For NFS shares it's also possible to do more granular permissions handling in the built-in shell of the software either from the web UI or by connecting to TrueNAS over SSH (which is not turned on by default.) For NFS shares it is possible to handle permissions on a group or user level in as granular a way as desired, even down to a single file in a share. In order to do that however, the user ID or group ID has to match an existing user on TrueNAS, so this kind of granular permissions can only really be achieved on Unix systems, and it works great. Windows clients however can utilize SMB shares and Active Directory for more granular permissions.

![unixpermissions](/network_upgrade/unixpermissions.png)

At this stage I have moved most of my data away from traditional cloud providers over to my NAS, and setup a cloud sync task to upload the most important data off site to an encrypted bucket automatically, with the data being encrypted using a built-in remote encryption feature where the user has to supply an encryption password and the encryption salt. With the current amount of data in the remote bucket, this is also way cheaper than having absolutely everything on say OneDrive, and as a privacy and security minded person get to have the peace of mind knowing that my most important data is safe and sound in an encrypted matter where no matter the promises of the provider, they cannot read the data even if they wished to do so, and more importantly an attacker could also not access my data, even if there was a data breach or misconfiguration of some sort on the provider's end. Even the filenames can be encrypted if desired.

The cloud sync tasks are very straightforward and easy to setup; you can choose the direction of push or pull, transfer mode of sync, copy or move and then choose any path you like to be backed up. It could be something as little as a few specific files or even an entire pool; the sky really is the limit here, as is the price to pay for the off-site backup depending on the chosen provider of which there are 18 to choose from at the time of writing, if going for a provider that is easy to setup natively in TrueNAS. This includes providers such as Dropbox, Google Cloud Storage and Amason S3, or any other provider not listed using SFTP. Additionally, you can setup Rsync tasks to sync over SSH or even replicate to another TrueNAS system. These area all in a Data Protection section of the UI where you can also setup periodic snapshots and S.M.A.R.T. tests.

If there needs to be restoring of data, it can either be done from an already setup cloud sync task by choosing "restore", or in a disaster scenario if the whole installation is destroyed, by making a new cloud sync task, populating the relevant fields in the UI and restoring the data. At this stage I am not sure on which transfer mode you would need to pick for that scenario, and hopefully will never have to find out, but since I have all necessary details safe e.g encryption password + salt + the credentials for my provider, it won't be a problem to figure that out later on if required; I have already tested restoring from an existing task and it really is as simple as pressing a button.

![dataprotection](/network_upgrade/dataprotection.png)

## **Nice, but what about making this all faster?** 

After setting everything up, I decided that a 1 GbE network is not fast enough, and bought a UniFi switch which has ports for 1 GbE, 2.5 GbE and even two 10 GbE SFP+ ports. Many of my devices already have 2.5 GbE networking built-in, meaning that I get to already enjoy over double the speeds that I had with my old switch. Setting up the switch was not a walk in a park however, at least not in my non-Unifi environment with my virtualized pfSense running on my ESXi, and it was honestly a bit of a headache to get working, compared to the old managed HP switch I had been using until now, and this is something I'll be covering in my next post. 

Overall, setting up TrueNAS SCALE is quite straightforward and is a great way to perhaps take back control over one's data and secure it in the way one sees fit, and in the long run can even be cheaper than only relying on cloud storage. Then again, RAID or even RAID-Z is not a backup, and the data that is stored on a NAS should be also backed up elsewhere if the data is important, and that is exactly what I have done with my setup as well. 

Next up: How to setup a UniFi switch in a non-Unifi environment. For now, thank you for reading my post!
