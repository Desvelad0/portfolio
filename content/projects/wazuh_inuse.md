---
title: "Finding and fixing vulnerabilities with Wazuh"
date: 2024-05-21T16:33:27+03:00
draft: false
---

Since the last post got a bit long, I thought I would make this one just to show how Wazuh works in practise, hopefully in a more digestible manner. 

## **Drilling into vulnerabilities of an agent**

I have slowly been rolling out Wazuh to my servers, starting with an important Ubuntu Server, which is my only Linux server that has a GUI for website access locally on the machine. As it is a still essentially a server, despite it having a desktop installation, there are many things that come pre-installed that I don't personally need, such as bluetooth drivers.

Looking at this server, there are some medium level vulnerabilities on those very bluetooth drivers:

![image](/wazuh_inuse/cve.png)

I shall see if I can easily just uninstall the bluez packages entirely without breaking the system. From personal experience using Arch, it should be quite easy to get rid of these and shouldn't cause problems, but in any case I'll just take a snapshot of the machine as a precaution. I have also full backups on Veeam from this machine.

Seems I have already uninstalled the main bluez package, but some additional bluez packages still exist:

```
root@mysteryserver:~# apt list --installed bluez*
Listing... Done
bluez-cups/jammy-updates,jammy-security,now 5.64-0ubuntu1.1 amd64 [installed,automatic]
bluez-obexd/jammy-updates,jammy-security,now 5.64-0ubuntu1.1 amd64 [installed,automatic]
```

bluez-cups is apparently a Bluetooth printer driver for CUPS as one might guess, while bluez-obexd "contains a OBEX(OBject EXchange) daemon", and OBEX is a communication protocol that facilitates "the exchange of the binary object between the devices." This didn't quite explain to me what this is used for, so I asked for a translation from Copilot that says to think of it as a standardized way for devices to talk to each other and transfer files, and judging by the prefix bluez, it is by using bluetooth. 

I don't need either of these packages, so I'll just `apt remove --purge` them to get rid of some easy CVEs and reduce the attack surface.

## **Learning about BusyBox and its vulnerabilities**

While I wait for the changes to reflect on the Wazuh side, I shall take a look at some far more critical vulnerabilities with packages `busybox-initramfs` (several vulns ranging from 7.8 to 8.9 CVSS3 score) and `busybox-static` (vulns from 7.8 to a 9.8!)

BusyBox, I just learned, is a lightweight software suite that provides simplified versions of standard Unix tools such as `ls`, `cp`, `mv`, `grep` and many, many more, being "The Swiss Army Knife of Embedded Linux". The `busybox-static` package meanwhile is a simple stand alone shell which provides all the utilities available in Busybox, intended to be used as a rescue shell "in the event that you screw up your system", and it is there to "save your system from certain destruction" [according to debian.org](https://packages.debian.org/en/buster/busybox-static). 

`busybox-initramfs` is a crucial component which the Linux kernel essentially  loads during boot and provides a stand alone shell providing just the basic utilities needed for the [initramfs](https://wiki.ubuntu.com/Initramfs).

The most critical CVE present in both packages, CVE-2022-48174, is a stack overflow vulnerability which could be executed from command to arbitrary code execution in "Internet of Vehicles", a term I hadn't personally heard before the writing of this. Arbitrary code execution in vehicles, doesn't sound alarming at all!

```
root@server:~# apt upgrade busybox-static
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
busybox-static is already the newest version (1:1.30.1-7ubuntu3).
busybox-static set to manually installed.
Calculating upgrade... Done
```

The vulnerability seems to be present in versions earlier than 1.35, so I definitely have a vulnerable version even if I can't directly upgrade it with apt here. It seems that as of the writing of this, there is still no upgrade available from Canonical for jammy, which is my current version, so I have to look into alternatives. Canonical does link to patches [upstream](https://git.busybox.net/busybox/commit/?id=d417193cf37ca1005830d7e16f5fa7e1d8a44209). 

Having never done this sort of a thing before (upgrading packages manually), I thought to see if Copilot could help me out here. I am recommended to backup the current busybox binary from /usr/bin/busybox and replacing this with an updated one directly from BusyBox. I might as well try to go directly to the newest stable release, which at the time of writing is 1.36.

I `wget` and `tar -xvf` the 1.36 version from https://git.busybox.net/busybox/commit/?h=1_36_stable. I `cd` to the unpacked folder and with Copilot's instructions, `make menuconfig` to choose what applets I want to be in the final binary. This initially didn't work and I had to install `libc6-dev` and `libncurses5-dev`, and then I could finally do it. I go with the default options. Next, I `make` the binary which takes quite a long time, and at the end of it `cp busybox /usr/bin`. I test it out with a few commands such as `ls` , `cp`, `mv` and all seem to work just fine. Since I manually did all of this, `apt` still shows the old version installed. I also discover from further troubleshooting, that I should have probably done `make install` which takes care of putting busybox in all the relevant places as well as the symbolic linking, so I did that as well just to stay on the safe side.

I then looked a bit deeper into the CVE on Canonical's side, and saw this:

![ubuntu](/wazuh_inuse/ubuntucve.png)

So Canonical essentially gives a low priority to fixing this, and thus it has not been fixed for jammy at the time of writing, and specifically for jammy the status is still "Needed" at https://ubuntu.com/security/CVE-2022-48174. Additionally, Canonical mentions that "This vulnerability is mitigated in part by the use of gcc’s [stack protector](https://wiki.ubuntu.com/Security/Features#stack-protector) in Ubuntu", which seems to be a component that protects against stack overflows, which is exactly the type of vulnerability in question here.

At the very least, at this stage, the bluez packages that I removed before don't show up in Wazuh anymore. I decided to return to the snapshot, as I was unable to make the changes made reflect on Wazuh's and the server's end; I tried to even make a dummy package with `equivs` with the correct version and "install it", but kept running into an error saying `syntax error in control file: !<arch>` when trying to install the dummy package. I copied the same line directly from the real package with the same exact architecture, amd64, but alas, it would not work.

This along with the fact that even Canonical doesn't seem too concerned with the vulnerability, I thought it best to move on, especially since this is in my own home environment with me and my partner as the primary users of the server. 

## **More vulnerabilities!**

Additionally I find some very fresh Windows 10 vulnerabilities, the most severe one being CVE-2024-30040, CVSS3 8.8, which could be used in a social engineering attack, allowing an unauthenticated attacker to execute arbitrary code, IF the victim happened to open a malicious document. Since this vulnerability was found on my Veeam machine, such an event couldn't really take place in any case. I have blocked the machine from even accessing the internet, and it is alone in its very own VLAN, making it even more unlikely anything like this could take place, but in any case I temporarily allow the server access to the internet to get the latest updates.

## **Lessons learned**

For starters with Linux distributions, I need to pay closer attention to what each distribution's maintainers say about CVEs that Wazuh might be concerned about. In this case Canonical does not find the busybox vulnerabilities that Wazuh flagged to be a big cause for concern, whereas other websites covering this CVE seem very concerned indeed about the vulnerabilities found. Ubuntu specifically has a "stack protector" component which protects against stack overflow vulnerabilities such as CVE-2022-48174. 

I did, however, learn about BusyBox and what it does, and it seems like a really handy Swiss Army Knife indeed, can definitely see why it would be used in embbeded systems and elsewhere. It apparently has over 400 commands built-in that can be used, though the tools included do have fewer options available than the standalone, full-featured tools themselves. Additionally I learned about a tool called `equivs` that can be used to install dummy packages for cases like this where I manually upgrade a package on my own without the help of a package manager. 

I also re-learned how to compile programs written in C. I have done this previously for other tools, but it has been a while since I last had to do so. Not very challenging indeed.

