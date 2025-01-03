---
title: "Creating a ext4 Virtual Hard Drive (vhd) for use in WSL"
date: 2025-01-03
layout: post
tags: jekyll blog github VHD vhdx ext4 linux wsl development
---

## Why do you want another VHD for WSL?

The default WSL image installs itself somewhere into your C drive under Program Files and is provided as a linux file system image, ext4. This is nice and performant for WSL but not without issues: your Windows OS is likely to be installed on a SSD or a M.2 NVMe and those devices are likely to have a storage limitation, being premium devices.

You could create a VHD under Windows, but if you use this under WSL it will rightly warn you that its performance will be compromised. So you need to create a new ext4 file system. This perhaps wasn't obvious to me (having failed to RTFM) but once I'd understood the problem I quickly realised Windows isn't going to be able to create a ext4 VHD.

The other motivation for having a VHD is being able to copy this to a USB and mount/copy it onto other devices, or eventually, I would like to build a full VHD image with containers of my choice pre-installed, and treat it a little like a virtual box. For now though the sanity of compiling code on a ext4 file system that isn't my C drive is the main target.

## Commands

I have a SSD device enumerated as E: in Windows where I perform Windows-related development. I will create a small VHD on that device. I am not explaining how to install WSL, enable WSL2 or any mount commands to make the E: device accessible (that should automatically happen anyway). I prefer OpenSuSE Tumbleweed for my WSL (and indeed stand-alone Linux) installs.

'''
richard@Nightmare:/> sudo dd if=/dev/zero of=/mnt/e/dev.vhdx bs=10M count=1024
1024+0 records in
1024+0 records out
10737418240 bytes (11 GB, 10 GiB) copied, 26.7963 s, 401 MB/s
''''

The above command creates the VHD container and stuffs it full of zero bytes, bs parameter refers to the block size so you don't want anything excessive to protect system performance whilst giving good user performance, and the count parameter is instructing the number of these blocks, so in this case, a 10Gb vhdx and the above showed that this took < 27 seconds on my machine and this is on a SSD device. Don't agonise over block size - this amount is sane for modern machines.

'''
sudo mkfs -t ext4 /mnt/e/dev.vhdx
sudo mkdir /mnt/dev
sudo mount -o loop /mnt/e/dev.vhdx /mnt/dev
''''

The remaining commands create the ext4 file system, somewhere to mount it and finally the mount command itself. However you are likely to want to auto-mount this for subsequent restarts of WSL:

'''
tmpfs /var/tmp tmpfs defaults 0 0
/mnt/e/dev.vhdx /mnt/dev ext4 defaults,nofail 0 0
'''

That's the contents of my /etc/fstab so feel free to edit yours and add in that critical line.

## Inspecting contents from Windows

You don't really want to be performing development on Windows outside of the WSL on a etx4 file system and indeed most tools handle it rather badly. However there are many cases for just wanting to inspect files in explorer, or copy stuff around.

'''
\\wsl.localhost\openSUSE-Tumbleweed\mnt\dev
'''

That's the exact path you can use in Explorer to inspect the WSL-mounted container I created above from within my OpenSuSE WSL install.


'''
\\wsl$
'''

Is the more generic path which will list all of your WSL mounts.
