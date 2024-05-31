---
title: "Returning to Arch Linux"
tags: [arch,linux]
date: 2024-05-31T03:33:58+03:00
draft: false
---

As a sort of rite of passage for many IT professionals out there dabbling with Linux, I think many at some point end up installing Arch Linux, including myself. I have previously used Arch Linux multiple times, and the last time that I ended up going back to Windows was because of getting a new HDR enabled monitor which is something that isn't really supported on Linux, and that was the excuse to never boot back to Arch. However, especially after hearing about Microsoft's Recall feature, I decided that now is the time to return. Thus I am making this post explaining how I install Arch Linux. I will have a dualboot setup, so I can hopefully spend most of my time in Linux, and only use Windows when absolutely necessary.

## **Installing Arch Linux** 

First thing to do is naturally to grab the latest .iso for the installer from archlinux.org and making a bootable USB drive with it, and then booting into the installer. A former colleague and a good friend of mine once told me about Rufus, and ever since then I have always used it for this purpose. 

**Getting started, choosing and partitioning the disk** 

While I am able to use the US keyboard layout if necessary, especially if for whatever reason the Finnish one is not available, I much prefer to use the Finnish layout, and thus I change to that with ``loadkeys fi``.

Next, I enable ntp with `timedatectl set-ntp true`. After that, I list my available drives using lsblk, and not remembering for certain which drive it was that I wanted to install Arch on, I quickly go back to Windows to doublecheck. Now that I know the drive I want to install on, I run cfdisk /dev/mydrive, in this instance ``cfdisk /dev/nvme1n1`` , and it's named as such due to it being an nvme ssd. This drive already has another partition that is used as a shared drive between my previous Linux and Windows installation to move some files around each OS, so I have to be careful not to wipe that backup partition. 

I choose 1 GB for what will be my boot drive (I was initially going to go for less space, but then remembered I might want to have multiple kernels installed), 80 GB for what will be my root drive and finally about 1 TB for what will be my /home/ partition. Finally in the cfdisk menu I choose write, enter "yes" to confirm the changes and go to quit to get out of cfdisk. I will not make a swap partition as I don't see a need for it with 64GB of useable RAM. 

![IMG_0390.JPEG](/arch_linux/IMG_0390.JPEG)

Now funnily enough, my existing exfat partition is between what will be my root and home partition, so I have to take that into consideration when formatting the partitions.

**Formatting the partitions** 

Next, I format the /home and root partitions with ext4 as I have no need to access this drive on anything but this Arch installation, and the command to do this is ``mkfs.ext4``. The boot drive is recommended to be in FAT32 format, and that format can be achieved with ``mkfs -t vfat``. 

``mkfs.ext4 /dev/nvme1n1p2`` 

``mkfs.ext4 /dev/nvme1n1p4`` 

``mkfs -t vfat /dev/nvme1n1p1`` 

**Mounting the disk to prepare for installing** 

After struggling a bit here I remembered the importance of mounting the root partition FIRST, otherwise things just won't work. I did so by running

``mount /dev/nvme1n1p2 /mnt`` 

Then I mounted the rest of the partitions (and learned about the --mkdir switch that makes a folder before mounting):

`mount --mkdir /dev/nvme1n1p1 /mnt/boot/efi` (after trying and having issues with just /mnt/boot, more on that later)

`mount --mkdir /dev/nvme1n1p4 /mnt/home`

**Pacstrappin'** 

Now it's time to install some initial packages to get started with. Pacstrap is the tool used for this, and I install the following packages with it to /mnt where all of my partitions are currently mounted:

`pacstrap /mnt base base-devel linux linux-firmware vim`

- Base is a minimal package to set to define a basic Arch Linux installation, base-devel has some useful basic tools such as grep and pacman (the package manager of Arch), linux has the linux kernel and modules and linux-firmware has linux firmware files as the name suggests. As said in the Arch linux wiki, the linux package can also be substituted with any kernel package of your choice, but this is an easy way to get started and more can be added later. Vim I'm installing now as it's the text editor I prefer to use. 

**Setting up drives to load at boot**

Next, I have to make it so that the partitions as I have made them will always be loaded upon boot as they are supposed to, and this can be accomplished with the `genfstab` command. It is good to note here that by default, the command would list the device node instead of the UUID of each drive, and that in unfortunate circumstances could make it so that if there were any issues with any of the partitions, one device could change and take another ones place in the naming scheme, meaning that the partitions might load wrong. However, UUID, or a Universally unique identifier does not change, as its name implies, and by running `genfstab -U` instead, the UUID will be used instead of the device node name. Thanks for pointing this out, Mental Outlaw!

So what needs to be done now is for me to run `genfstab -U >> /mnt/etc/fstab`, which pastes out the partitions, their respective devices and their UUIDs, and puts this information into /mnt/etc/fstab. Fstab takes care of mounting drives in a Linux environment.

It is also important to note that for the `genfstab -U` command to show expected results, e.g. the root, boot and home partitions for me, they need to have been mounted correctly beforehand; you must mount the root partition first as noted before. Otherwise you might have unexpected results and things will not work as they are supposed to. I could not see the /boot/efi partition mentioned until I mounted the root partition before anything else.

![IMG_0399.JPEG](/arch_linux/IMG_0399.JPEG)

**Using arch-chroot, installing some packages** 

Next up is switching root to the actual Arch installation, away from the bootable USB drive, and this can be accomplished with `arch-chroot /mnt`. After running it, I am now in what will be the actual installation of the OS. 

Now I will install a network manager aptly called NetworkManager, as it is what I have the most experience with, and the default network manager, systemd-networkd doesn't work as nicely in my experience. Additionally, I will need to install grub and efibootmgr (this last one I learned the hard way after having issues running grub-install; had this issue previously as well but didn't remember it this time.)

Thus, I run `pacman -S networkmanager grub efibootmgr`

Finally, I'll need to enable networkmanager to start at boot with `systemctl enable NetworkManager`. 

**Configuring grub** 

Now comes configuring grub that will help us boot into Arch Linux. I run the following command:

``grub-install --target=x86_64-efi --efi-directory=/boot/efi`` 

As you can see, no errors were found.

![IMG_0400.JPEG](/arch_linux/IMG_0400.JPEG)

Next, I run 

`grub-mkconfig -o /boot/grub/grub.cfg`

![IMG_0402.JPEG](/arch_linux/IMG_0402.JPEG)

That's all that is needed for grub.

**Changing root password**

I quickly change the root password with `passwd root`.

**Setting locale**

Almost done. Next up is setting the locale, and for simplicity's sake I'll go with en_US; can't actually remember if I needed to ever change this to something else later on, but if so, that can always be done later.

The locale settings are set by editing `/etc/locale.gen`, uncommenting the relevant locales, saving the changes and then running `locale-gen`. This for me was just the en-US UTF and ISO options.

I will also set the default keymap to be used in the console by entering `KEYMAP=FI` to a file I will have to make at `/etc/vconsole.conf`. 

**Finishing up**

Mental Outlaw also points out that one can set their desired timezone by running `ln -sf /usr/share/zoneinfo/Region/City /etc/localtime`. Very handy.

Finally before rebooting into the installation, I have to set a hostname by just entering my desired hostname to `/etc/hostname`. 

Now I can exit the arch-chroot, unmount everything with `umount -R /mnt` and reboot into the OS.

Thats it! Last time I remember this only using under a gig of RAM...

![IMG_0403.JPEG](/arch_linux/IMG_0403.JPEG)

## **Problems faced, lessons learned** 

I reminded myself that before mounting any other partitions, the root partition has to be mounted first, otherwise unexpected things happen. From the Arch wiki I also learned that while mounting a drive, you can have the same command make the directory where the drive will be mounted with `mount --mkdir /dev/drivename /new/path` which is very handy indeed. I also remembered that simply trying to run `grub-install` to my drive directly will not work, and instead I have to run the command I did after also having the `efibootmgr` installed. Finally I was reminded that the boot partition should be in FAT32 file format, and the boot folder should be /mnt/boot/efi instead of just /mnt/boot. 

Next up I will install [Hyprland](https://hyprland.org), a very appealing tiling compositor that I very much enjoy to use, and I will be making a post on this topic later on. I'll also be installing this specifically in a setup using an NVIDIA GPU which has always been a bit of a hassle on the Linux side of things, so that should be fun. Until then, thank you for reading!
