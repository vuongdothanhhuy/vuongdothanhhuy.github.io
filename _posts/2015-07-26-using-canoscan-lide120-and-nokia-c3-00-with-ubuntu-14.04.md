---
layout: post
title: Using CanoScan LiDE120 and Nokia C3-00 with Ubuntu 14.04
excerpt: "My own experience on using CanoScan LiDE120 and Nokia C3-00 whose drivers and applications are for Windows only, with Ubuntu 14.04.2 LTS"
modified: 2015-07-26
tags: [linux, ubuntu, virtualbox, CanoScan, LiDE120, Nokia C3-00]
comments: true
image:
  feature: banner.jpg
  credit: WeGraphics
  creditlink: http://wegraphics.net/downloads/free-ultimate-blurred-background-pack/
---

Hi there,

Recently, I have bought a scanner in order to digitalize my old documents and papers. It's a cheap Canon scanner: *CanoScan LiDE120*.

*CanoScan LiDE120* is a cheap but good quality scanner for university students and people working with a lot of papers like me. It can deliver good scanning with acceptable color accuracy for photo archiving and outstanding cripsed looking for documents. However, this scanner only has drivers for Windows and Mac, and the scanning speed is a bit slow.

I'm a web developer and I make extensive use on Ubuntu on my personal laptop. I also dual boot Windows 7 (Yeah, it's 7, not 8 or 8.1, since I kinda dislike the tablet and touch UI of those versions) along the side, for some Windows-only tasks that I don't want to or can't run inside a virtual machine. The scanner is installed and worked flawlessly in Windows environment. However, it's a pain and a huge discomfort having to leave Ubuntu with lots of works in progress, restart the computer and jump over Windows, only to do some scanning tasks.

I use vagrant with Virtualbox, so I think I could install a Windows 7 guest and somehow pass the USB devices from my Ubuntu host to the Windows guest. Here is how I did:

1. Installed *DMKS*. This is optional, but I like to install it so as I upgrade Virtualbox, my kernel will be recompiled automatically and I don't need to run `/etc/init.d/vboxdrv setup` by myself.
2. Installed latest *Virtualbox* from [https://www.virtualbox.org/wiki/Downloads](https://www.virtualbox.org/wiki/Downloads). You can also install from Ubuntu repo. However, the version in there might be a bit outdated.
3. Installed the *Extension Pack* corresponding to the Virtualbox version. This extension pack adds supports for using USB devices, so installing this is compulsory. Remember to match the version of Virtualbox to the Extension Pack.
4. Created a virtual machine profile. I go with 1024MB RAM and 20GB HDD (.vdi format).
5. Went to the Settings of the newly created VM, USB section, and enabled the USB controller with USB 2.0 version.
6. Booted the virtual machine and installed Windows 7.
7. Installed the Guest Additions.
8. Then I had a Windows 7 VM with Guest Addtions installed and USB controller enabled. Booted the VM up, plugged the scanner in, then at the bottom right corner, I right-clicked to the USB icon, only to find that there's nothing to choose from.

After performing all those steps, I thought I should have been able to see and choose my scanner from the list. However, the list was empty. A quick check on the host machine with `lsusb` confirmed that my scanner was picked up successfully. It's just the Virtualbox that somehow can't see it and allows me to choose it from the list.

**TL;DR - Solution:**

Hours of Googling revealed a vital missing steps: add my current host machine user to the group `vboxusers`. Do it by running **`sudo usermod -a -G vboxusers your_current_username`**. Then logout and log back in. Confirm by running `VBoxManage list usbhost` and it should list all USB devices currently being connected to the host machine.

Now I can boot up the VM, plug the scanner in then right-click and choose my scanner from the list. Windows guest automatically pick up the scanner. Install ScanGear and I was good to go and scan!

This trick can be used to use any devices that only supports Windows. Beside my _CanoScan LiDE120_, I also managed to connect my _Nokia C3-00_ and use Nokia Suite inside VM to backup and restore my phone.

Good luck!