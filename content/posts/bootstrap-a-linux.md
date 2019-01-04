---
title: "Bootstrap a Linux"
date: 2019-01-03T21:42:31+02:00
draft: true
---

# How to bootstrap your a baby linux operating system using Ubuntu.

## Why would I do this? 
I want to understand the tools I use. Docker is an extension of an idea that has actually been in the kernel for a while using cgroups etc. (Just a nicer package)
BSD jails etc.
What other treasures exists in the kernel? The only way to know is to learn more about linux.
How do you learn about a project? Kafka? Kubernetes? Install it! But oh, how does one install linux? You can do it using your package manager. But you didnt learn much did you?

## Install from scratch?
Why not linux from scratch? I have a life. gcc to cross compile gcc to build something else etc etc. Good reasons(Name them) but it takes SO long! Any educational value this project has get's lost in the waiting time. The reasons *I* stopped was that *I* actually ended up getting compilation errors after hours of waiting and copy and pasting. I followed instructions, but the results was discouraging.

## Why not gentoo/arch?
Another, copy and paste, follow this method etc etc. better than LFS but still pretty tricky. I have a colleague who actually said "I'm getting better at installing Gentoo!" Pros the bootstrapping compilation steps are done for you. Cons: Still assumes a lot of the user.

## Just why?
I would like to learn more about Linux tools/systems:
dd? kpartx loopback devices parted/fdisk the linux kernel itself?
Maybe I'm a masochist. 

# Assumptions
- I'm going to use a virtual machine
- I'm going to assume that we still use a BIOS based machine (What's a BIOS? How old are you? Have you even Doom'ed bro?)

# Method
- create a virtual disk
- partition into a bootable and system partitions
- format both partitions as ext2
- install a boot loader on the boot partition
- copy from your ubuntu kernel and initrd files.
- boot the virtual machine using the new virtual disk

## create the virtual disk
The following creates a file called "mink.img" filled with 200Megs worth of zeros.

```bash
dd if=/dev/zero of=mink.img count=200 bs=1M
```
Here is some magic I'd like to explain. `dd` is just a very DUMB copy command. It takes blocks from one file and plop it to another. In this case it copies data from the special `zero` file and puts it into the new `.img` file.

## partition into bootable and system partitions
```bash
parted --script mink.img mklabel msdos mkpart p ext2 1 20 set 1 boot on mkpart p ext2 21 200
```
`parted` is a tool you can use to partition the harddisk. There's a graphical version (gparted) also but to be succicnt I'm using the encantation.

Let's break it up:
```bash
mklabel msdos  # tells parted we're going to use the msdos partition table scheme. This is a legacy thing as far as I understand it.
mkpart p ext2 1 20  # make a 20Meg partition from Meg 1 to Meg 20.
set 1 boot on # make partition 1 bootable
mkpart p ext2 21 200 # make another partition from Meg 21 and Meg 200 
```

## format partitions as ext2
To format a file (in this case "mink.img") I need to be able to treat it like a drive. That's what loopback devices are. They make the file look like a disk drive that can be mounted or even formatted! So let's see how we format a file like a drive:

```bash
kpartx -av mink.img  # create the loopback device and split it over 2 partitions (kpartx splits it up for you)
sleep 10  # this isnt instant, so wait 10 seconds (overkill)
mkfs.ext2  /dev/mapper/loop0p1 # format the boot partition
mkfs.ext2  /dev/mapper/loop0p2 # format the system partition
# Finally mount both files so we can access and work on it.
mount /dev/mapper/loop0p1 /mnt/pocket/boot_mount 
mount /dev/mapper/loop0p2 /mnt/pocket/root_mount
```
Now I've mounted a file as 2 separate partitions. The virtual drive is finally useful. Now we can start installing things.

## install a boot loader on the boot partition
The below is still a little fuzzy to me, but so far this seems to have worked:
```bash
grub-install --no-floppy  --modules="biosdisk part_msdos ext2 configfile normal multiboot" --root-directory=/mnt/pocket/boot_mount/ /dev/loop0
```
From what i understand I'm install a bios based boot loader to my virtual drive's boot partition

## copy from your ubuntu kernel and initrd files.
So first we make a place for the new files in our drive's system partition.
```bash
mkdir -p /mnt/pocket/root_mount/kernels/
```
then we copy the linux kernel
```bash
cp /boot/vmlinuz-4.10.0-27-generic /mnt/pocket/root_mount/kernels/
```
then we copy the matching "initial ram disk" file. This is like a small linux root filesystem that we can use to access the system before the real system is loaded
```bash
cp /boot/initrd.img-4.10.0-27-generic /mnt/pocket/root_mount/kernels/
```

## boot the virtual machine using the new virtual disk
So here we boot into the virtual machine using "Qemu".
```bash
qemu-system-x86_64 -drive format=raw,file=mink.img -m 1G
```
The thing is if everything goes okay the only thing you'll be seeing is the following:

![Grub Prompt](/grub-init.png)

Not very helpful. What now?

Well, it looks like grub prompt is actually pretty simple to use. For example, we can list all the partitions available to the system.
```bash
grub> ls 
(hd0) (hd0, msdos2) (hd0, msdos1) (fd0)
```
The important entries here would be `(hd0, msdos2)` and `(hd0, msdos1)`. Those would be the `system` and `boot` partitions respectively. To allow us toe boot the first thing that must be done is to set the current partition to where the kernel is located. Grub allows us to set this using the `root` command. This is similar to the way one can set DOS's current drive to `C:` for harddrive or `A:` for floppy drive.

```bash
grub> set root=(hd0, msdos1)
grub> ls /
lost+found/ kernels/
```
Our files lives in the `kernels` directory..

![Grub Navigation](/grub-navigate.png)

To start the os we need to tell grub where the kernel and the initrd files are located. For grub you specify it as follows.

![Grub Starting Linux](/grub-start-os.png)

The `boot` keyword triggers the actual booting process. `Tab` actually autocompletes filenames!

## Booting success!

![Booted Ubuntu Linux Kernel](/itworked.png)
