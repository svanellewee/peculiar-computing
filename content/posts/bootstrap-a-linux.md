---
title: "Creating a minimal environment to bootstrap the Linux Kernel"
date: 2019-01-03T21:42:31+02:00
draft: true
---

## Intro
Here are some notes for creating a very minimal test environment for the linux kernel. This version will re-use the host operating system's kernel to get things going. Follow-up posts will compile the kernel and initial ramdisk from scratch.

## Assumptions
- We're using an Ubuntu 16.04 as our host machine. It comes with almost all the applications we need to build the virtual disk and machine. (We'll also be "borrowing" the host's kernel and init ram disk files.)
- Qemu will be used as the virtualisation software.
- For simplicity a BIOS based architecture will be assumed.

## Method
To summarize here's the steps that we'd be following:

- create a virtual disk
- partition into a bootable and system partitions
- format both partitions as ext2
- install a boot loader on the boot partition
- copy from your ubuntu kernel and initrd files.
- boot the virtual machine using the new virtual disk

### create the virtual disk
In order to create our virtual "hard drive" we need to create a blank file that can be used to simulate an empty hard disk. This can be accomplished using the `dd` command. 

`dd` is quite a dumb command that just moves blocks from one file to another. To illustrate this look at the following:
```bash
dd if=/dev/zero of=virtual-drive.img count=200 bs=1M   # create  file called "virtual-drive.img" that is 200Mbytes big
```
The above copies data from the special `zero` file (provided in most unix like operating systems) and puts it into the newly created `.img` file.

### partition into bootable and system partitions
Our "hard drive" is empty and still pretty useless. To fix that we need to split it up into 2 parts

- the boot partition (the bootloader can live there)
- the system partition (the kernel files will live here)

Here is a little problem the more astute reader will notice. _How does one partition a file_? Remember, in linux everything is treated like files anyway. For example `/dev/sda1` is a file in the `/dev` directory that represents 1 partition on a hard drive. So we can use the same tools we use to partition the real hard drive on our fake, virtual drive.

There are a few tools that can partition the drive, but in this example we'll be using `parted`. To split the `.img` file into the required boot and system partitions the following incantation can be used:
```bash
parted --script virtual-drive.img mklabel msdos mkpart p ext2 1 20 set 1 boot on mkpart p ext2 21 200
```

Let's break it up:
```bash
mklabel msdos  # tells parted we're going to use the msdos partition table scheme. This is a legacy thing as far as I understand it.
mkpart p ext2 1 20  # make a 20Meg partition from Meg 1 to Meg 20.
set 1 boot on # make partition 1 bootable
mkpart p ext2 21 200 # make another partition from Meg 21 and Meg 200 
```

### format partitions as ext2
To format our partitions we need to turn them into [loopback devices](https://wiki.osdev.org/Loopback_Device). This is a way "to interpret files as real devices."

```bash
kpartx -av virtual-drive.img  # create the loopback device and split it over 2 partitions (kpartx splits it up for you)
sleep 10  # this isnt instant, so wait 10 seconds (overkill)
```
In our case this creates 2 files:

- `/dev/mapper/loop0p1` - represents the boot partition
- `/dev/mapper/loop0p2` - represents the system partition

Which represents the partitions `parted` created on the virtual drive. Now that we have 2 "devices" we can legitimately format them as if they're partitions on a real drive.
```bash
mkfs.ext2  /dev/mapper/loop0p1 # format the boot partition
mkfs.ext2  /dev/mapper/loop0p2 # format the system partition
```
Finally mount both files so we can access and work on it.
```bash
mount /dev/mapper/loop0p1 /mnt/boot_mount 
mount /dev/mapper/loop0p2 /mnt/root_mount
```
Now we can start installing things.

### install a boot loader on the boot partition
A boot loader is the first program that runs when the computer starts. In our case we'll be using the GRand Unified Bootloader or [GRUB](https://www.gnu.org/software/grub/)

This installs the boot loader to the mounted boot partition of our file. `/dev/loop0` is the loopback representation of the entire file (disregarding partitions)
```bash
grub-install --no-floppy  --modules="biosdisk part_msdos ext2 configfile normal multiboot" --root-directory=/mnt/boot_mount/ /dev/loop0
```
I found the original version of this command [here](https://roscopeco.com/2013/08/12/creating-a-bootable-hard-disk-image-with-grub2/). It's still not 100% clear to me what each module does.

### copy from your ubuntu kernel and initrd files.
So first we make a place for the new files in our drive's system partition.
```bash
mkdir -p /mnt/root_mount/kernels/
```
then we copy the linux kernel
```bash
cp /boot/vmlinuz-4.10.0-27-generic /mnt/root_mount/kernels/
```
then we copy the matching "initial ram disk" file. This is like a small linux root filesystem that we can use to access the system before the real system is loaded
```bash
cp /boot/initrd.img-4.10.0-27-generic /mnt/root_mount/kernels/
```

### boot the virtual machine using the new virtual disk
So here we boot into the virtual machine using "Qemu".
```bash
qemu-system-x86_64 -drive format=raw,file=virtual-drive.img -m 1G
```
The thing is if everything goes okay the only thing you'll be seeing is the following:

![Grub Prompt](/grub-init.png)

Not very helpful. What now?

Well, it looks like grub prompt is actually pretty simple to use. For example, we can list all the partitions available to the system.
```bash
grub> ls 
(hd0) (hd0, msdos2) (hd0, msdos1) (fd0)
```
The important entries here would be `(hd0, msdos2)` and `(hd0, msdos1)`. Those would be the `system` and `boot` partitions respectively. To allow us to boot the first thing that must be done is to set the current partition to where the kernel is located. Grub allows us to set this using the `root` command. This is similar to the way one can set DOS's current drive to `C:` for harddrive or `A:` for floppy drive.

```bash
grub> set root=(hd0, msdos2)
grub> ls /
lost+found/ kernels/
```
As expected, our files lives are were we left them in the `kernels` directory..

![Grub Navigation](/grub-navigate.png)

To start the os we need to tell grub where the kernel and the initrd files are located. For grub you specify it as follows.

![Grub Starting Linux](/grub-start-os.png)

The `boot` keyword triggers the actual booting process. `Tab` actually autocompletes filenames!

### Booting success!

![Booted Ubuntu Linux Kernel](/itworked.png)

We are now running the kernel with a filesystem in memory (initramfs). Here is what is currently mounted:

![Mounted Initial file system](/initramfs-mount.png)

### The full script for reference

```bash
#!/usr/bin/env bash
set -e
dd if=/dev/zero of=virtual-disk.img count=200 bs=1M
parted --script virtual-disk.img mklabel msdos mkpart p ext2 1 20 set 1 boot on mkpart p ext2 21 200
sudo kpartx -av virtual-disk.img
sleep 10
sudo mkfs.ext2  /dev/mapper/loop0p1
sudo mkfs.ext2  /dev/mapper/loop0p2
sudo mount /dev/mapper/loop0p1 /mnt/boot_mount
sudo mount /dev/mapper/loop0p2 /mnt/root_mount
sudo grub-install --no-floppy  --modules="biosdisk part_msdos ext2 configfile normal multiboot" --root-directory=/mnt/boot_mount/ /dev/loop0
sudo mkdir -p /mnt/root_mount/kernels/
sudo cp /boot/vmlinuz-4.10.0-27-generic /mnt/root_mount/kernels/
sudo cp /boot/initrd.img-4.10.0-27-generic /mnt/root_mount/kernels/
sudo umount /mnt/root_mount/
qemu-system-x86_64 -drive format=raw,file=virtual-disk.img -m 1G
set +e

```

## Next Up..

- [Setting up a grub menu]({{< ref "posts/create-a-grub-file" >}})
- Setting up a custom Kernel and initramfs/initrd

# References

- [Ross Bamford's site](https://roscopeco.com/2013/08/12/creating-a-bootable-hard-disk-image-with-grub2/)
- [OrangePi.org](http://www.orangepi.org/Docs/Makingabootable.html)
