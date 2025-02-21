---
title: "Installing GUI in Arch Linux"
slug: "installing_gui_in_arch_linux"
tags: [arch,linux]
date: 2024-08-01T14:44:09+03:00
draft: false
---

A while ago I installed Arch Linux, but have since been pre-occupied with other tasks and a vacation to the Canary Islands (highly recommend Tenerife with a rented car!) so I didn't have a chance to post this until now. As promised, I will now be installing the GUI portion of the OS, and what I like to use for that specifically is called Hyprland. 

## **Nvidia required steps** 

Now, a problem is that my discrete GPU is an NVIDIA one, and Hyprland isn't even officially supported on NVIDIA cards. I might end up getting a cheap AMD card later on and use the AMD one for Linux and NVIDIA one for a Windows VM (for certain games) if this proves to be a nuisance down the road, but I have previously been able to use Hyprland just fine with my existing hardware. 

Last time I did this I remember having to follow several guides online, but I'll try to begin with the official documentation for Hyprland at https://wiki.hyprland.org/Nvidia/. 

After running a `# pacman -Syu` to update the system, first steps are to download the correct headers package for the current kernel. `uname -a` returns an expected result; when I installed Arch, I specified `linux` as the kernel, and Linux is the kernel I am running. Thus I'll `# pacman -S linux-headers`.

Next, I'll need to install the necessary nvidia packages. I believe last time it was `nvidia-dkms`, so I'll go with that one, and I will also apparently need to get `nvidia-utils` and also `lib32-nvidia-utils` in order to be able to play games via Wine and natively as well, however I can't immediately find any package for the latter. I'll get back to it if required. Finally I also need to install `egl-wayland`.

Now I'll need to "enable modeset for nvidia", which is done by editing the kernel parameters of the respective bootloader, and for me that is grub, so I'll add `nvidia_drm.modeset=1` to the very end of `GRUB_CMDLINE_LINUX_DEFAULT` in `/etc/default/grub`, and from a quick google search it seems multiple parameters are separated by spaces, so after the word "quiet" I have there, I'll add in the parameter, save and then run `# grub-mkconfig -o /boot/grub/grub.cfg` for the changes to take effect in grub. 

![drm_modeset](/arch_linux/IMG_0531.JPEG)

I then need to add similar things to `/etc/mkinitcpio.conf`:

![drm_modeset](/arch_linux/IMG_0530.JPEG)

... And have the changes take effect by running `# mkinitcpio -P`. The guide points out that if there were any errors mentioned about missing nvidia modules, the aforementioned headers package probably wasn't installed, but I followed the guide and did as asked, so I didn't get any such error.

The guide also mentions to export certain variables to the hyprland config, and at this stage I have not even installed hyprland itself yet, so I'll do just that now.

## Installing Hyprland 

Apparently you can automatically compile the latest hyprland package from source if installing from the AUR, Arch User repository, so I'll install `Aura`, another package manager for Arch Linux which can detect fishy behaviour in packages downloaded from AUR, and notify you if something seems suspicious:

`git clone https://aur.archlinux.org/aura-bin.git`

`cd aura-bin`

`makepkg`

And then the next command which proved troublesome on my newly created account; I had to follow [guidance on the Arch wiki](https://wiki.archlinux.org/title/Sudo) and uncomment `# %wheel ALL=(ALL:ALL) ALL` which grants all users in the group `wheel` access to `sudo`, after I remembered that simply adding a user to `wheel` on a fresh Arch installation doesn´t automatically allow them to use `sudo`: 

`sudo pacman -U aura-bin-3.2.9-1-x86_64.pkg.tar.zst`, the .pkg.tar.zst file having been created by the `makepkg` command. 

Next I can run `sudo aura -A hyprland-git`; the `-A` switch will list all the dependencies required to install that package and ask me to confirm. All looks good for me, so I'll go ahead and say `Y` , and `Y` once more to install all the packages. 

## **Installation problems** 

The install here actually failed, the error being `ERROR: A failure occured in build().` Aura asks if I'd like to continue anyway but I decide not to. I go on to try the manual build method, where a slew of packages are listed. I tried to install them with Aura, and was met with "no valid packages specified", so I decide to use yay as suggested by the Hyprland wiki:

`git clone https://aur.archlinux.org/yay.git`

`cd yay`

`makepkg -si`

And then: 

`yay -S gdb ninja gcc cmake meson libxcb xcb-proto xcb-util xcb-util-keysyms libxfixes libx11 libxcomposite xorg-xinput libxrender pixman wayland-protocols cairo pango seatd libxkbcommon xcb-util-wm xorg-xwayland libinput libliftoff libdisplay-info cpio tomlplusplus hyprlang hyprcursor hyprwayland-scanner xcb-util-errors` which seems to have run without errors, yay!

Next:

```git clone --recursive https://github.com/hyprwm/Hyprland
git clone --recursive https://github.com/hyprwm/Hyprland
cd Hyprland
make all && sudo make install
```

That one throws an error about missing a required package `hyprutils`. I download it with `sudo pacman -S hyprutils` and try again. Still the same problem, seems pacman got `hyprutils-0.1.3-1` and the one required is `hyprutils-0.1.4` or newer. `yay hyprutils` seems to also return older versions than requested by the installation script. 

I decide to look at the issues of the package to see if anyone else has the same issue, and find [this](https://github.com/hyprwm/Hyprland/issues/6596) issue for NixOS, but no issue has been made for Arch Linux. I make a comment in this issue and boot back to Windows while waiting. 

After waiting a while, I decided to open a new issue of my own, to which I promptly got a one word response; hyprutils-git. I assumed installing this would not work due to it being older than what the error specifies, but sure enough, it works. Nice! I also installed `kitty` which is the recommended terminal to be used with Hyprland. I also exported variables to the hyprland.conf mentioned on the Hyprland wiki:

```
env = LIBVA_DRIVER_NAME,nvidia
env = XDG_SESSION_TYPE,wayland
env = GBM_BACKEND,nvidia-drm
env = __GLX_VENDOR_LIBRARY_NAME,nvidia

cursor {
    no_hardware_cursors = true
}
```

After doing all this, simply running `Hyprland` should start Hyprland. I did however have issues doing this, and kept getting the following error:

```
Hyprland threw in ctor: filesystem error: status: Permission denied [/run/user/0/hypr]

Cannot continue.
```

I believe the issue had to do with polkit and seatd not having been started; these are services used by Hyprland. After I enabled both services with systemctl, re-did the mkinitcpio and grub steps and rebooted, running `Hyprland` finally started working. By default you are automatically made an example hyprland.conf (which was where I added those environment variables above) file in `~/.config/hypr/hyprland.conf`, and by default this file includes keybindings to start things like `kitty`, and the default shortcut for that is the so-called SUPERKEY, so usually Windows key in a Windows keyboard + Q. Initially it would not launch, so I googled and found [this Reddit thread](https://www.reddit.com/r/hyprland/comments/zrw9zq/kitty_not_launching/) which mentions that simply installing gtk3 should be enough, and it certainly was. Another user mentioned that adding `KITTY_DISABLE_WAYLAND=0` to `/etc/environment` would also fix the issue but oh well, this one already works for me. I am now able to open a terminal window or ten, and I also changed the keyboard layout to FI in the config file, which immediately changed it to the Finnish layout; changes like this immediately take effect, including ones that can mess up things. Still, I find this very nice, and always make backups of dotfiles if I am not sure about something.

## **Wrapping up** 

At this stage I have a barebones Hyprland setup on Arch Linux, and the next steps are to configure the relevant dotfiles to reflect how I want my system to run, starting from simple things such as making the play/pause button on my keyboard actually do something to things like taking screenshots. 

I shall keep documenting my experiences regarding this and other technical topics I wish to cover. If you got this far, I highly appreciate you taking the time to read this!





