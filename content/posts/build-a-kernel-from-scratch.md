---
title: "Build a Kernel From Scratch"
date: 2019-01-16T21:34:16+02:00
draft: true
---

In the [previous post]({{< ref "posts/bootstrap-a-linux" >}}) I borrowed the kernel files from my current Ubuntu 16.04 distribution. After that I [set up a grub menu]({{< ref "posts/create-a-grub-file" >}}) to speed up testing time.

Let's grab some kernel sourcecode

```bash
wget https://cdn.kernel.org/pub/linux/kernel/v4.x/linux-4.20.2.tar.xz
```
The new goal is to compile the smallest most bare bones kernel we can muster. This according to Mitchel Humphrey's excellent 2015 post on this `allnoconfig` is just the thing we need. I'm going to follow his example.

```bash
tar xvf linux-4.20.2.tar.xz
cd linux-4.20.2
make O=../outputfile allnoconfig    # O= specifies the output directory
make O=../outputfile nconfig        # Enable a few things since allnoconfig is literally nothing!
```
Just for simplicity I'm pasting his configuration here, but I still think you should read his post from top to bottom. It's really a great resource!

The first section just enables the `printk` call and notes that we want a 64-bit kernel
```
[*] 64-bit kernel

-> General setup
  -> Configure standard kernel features
[*] Enable support for printk
```

The next is interesting. This tells the compiler that the kernel must support an initial ram filesystem (Just like the one I borrowed from my linux /boot directory "initrd*")

```
-> General setup
[*] Initial RAM filesystem and RAM disk (initramfs/initrd) support
```

The next is surprising. It seems the kernel is the thing that enables shebang (`#!`) lines in scripts? I always thought that was a bash function. 
```
-> Executable file formats / Emulations
[*] Kernel support for ELF binaries
[*] Kernel support for scripts starting with #!
```

Serial console is enabled as follows.. 
```
-> Device Drivers
  -> Character devices
[*] Enable TTY

-> Device Drivers
  -> Character devices
    -> Serial drivers
[*] 8250/16550 and compatible serial support
[*]   Console on 8250/16550 and compatible serial port
```
Now to enable some special purpose file systems..
```
-> File systems
  -> Pseudo filesystems
[*] /proc file system support
[*] sysfs file system support
```
Save the settings and compile to the directory `../outputfile` as follows:
```
make O=../outputfile -j$(nproc)
```
The end result should appear when you see this:

![Kernel Compiled](/new-kernel-noconfig.png)

Then the new kernel needs to be copied to our image (I'm assuming it's mounted as per the [original post]({{< ref "posts/bootstrap-a-linux" >}}). If this is true you should be able to see the following mapped loop devices:

```bash
ls /dev/mapper/
control loop0p1 loop0p2
```
If you don't see those loop devices, just change to the directory of our `virtual-disk.img` and re-run:
```bash
sudo kpartx -av virtual-disk.img
``` 
This should make the loop devices appear. Given that my loop devices are called `loop0p1` and `loop0p2` and I know the second partition of my file was the root filesystem I'm going to mount it as follows:


```bash
sudo mount /dev/mapper/loop0p2 /mnt/root_mount/
sudo mount /dev/mapper/loop0p1 /mnt/boot_mount/
sudo cp ../outputfile/arch/x86_64/boot/bzImage /mnt/root_mount/kernels/
```
Given that in the first post we already put the Ubuntu kernel and ramdisk we should see the following:

```bash
ls /mnt/root_mount/kernels/
bzImage  initrd.img-4.10.0-27-generic  vmlinuz-4.10.0-27-generic
```
This will "boot" if you specify the same boot parameters. Just like the [first grub post]({{< ref "posts/create-a-grub-file.md" >}}) we can just make another entry to bottom of the grub file at `/mnt/boot_mount/boot/grub/grub.cfg`

```
menuentry "Boot up the minimal" {
   set root="(hd0,msdos2)"
   linux /kernels/bzImage root=/dev/sda1
   initrd /kernels/initrd.img-4.10.0-27-generic
   boot
}
```
and then try running it.

```bash
qemu-system-x86_64 -drive format=raw,file=virtual-disk.img -m 1G
```
But I suspect because I borrowed the ramdisk the initramdisk from another distro, it is not meant for this kernel you're not going to get past bootup with it unfortunately. That's why we go back to Mitchel's blog to create our own initramfs.

Unlike Mitchel I'm not really worried about building busybox from scratch. So I'm just going to grab a pre-compiled version:

```bash
cd /tmp/
wget https://busybox.net/downloads/busybox-1.30.0.tar.bz2
tar -xvf busybox-1.30.0
cd busybox-1.30.0
mkdir /tmp/busybox_output
make O=/tmp/busybox_output defconfig
make O=/tmp/busybox_output menuconfig  # ensure static binary
make O=/tmp/busybox_output -j2 && make O=/tmp/busybox_output install
```
Here's something that differs from the original post. The position of the menuitem to toggle is now:
```
Settings
   [*] Build static binary (no shared libs)
```

```bash
mkdir -p /tmp/initramfs/x86-busybox
cd /tmp/initramfs/x86-busybox
mkdir -pv {bin,sbin,etc,proc,sys,usr/{bin,sbin}}
cp -av /tmp/busybox_output/_install/* .  # this actually installs busybox
```
The init program he used needs to be places in your busybox root:
```bash
cat <<"EOF" > /tmp/initramfs/x86-busybox/init
#!/bin/sh

mount -t proc none /proc
mount -t sysfs none /sys
echo -e "\nBoot took $(cut -d' ' -f1 /proc/uptime) seconds\n"

exec /bin/sh
EOF


# Make executable...
chmod a+x /tmp/initramfs/x86-busybox/init
```
To build the init ramfs file itself there seems to be a standard way of doing this

```bash
cd /tmp/initramfs/x86-busybox/
find . -print0 | cpio --null --create --verbose --format=newc | gzip --best > /tmp/my-initramfs.cpio.gz
sudo cp /tmp/my-initramfs.cpio.gz  /mnt/root_mount/kernels/   # Copy the file to our "virtual-disk.img"
```
Now to adjust the grubfile entry to use our new init ramfs
```
menuentry "Boot up the minimal" {
   set root="(hd0,msdos2)"
   linux /kernels/bzImage root=/dev/sda1
   initrd /kernels/my-initramfs.cpio.gz
   boot
}
```
Remember to call `sync` after these changes.

Now we should be ready to start our VM again:

```bash
qemu-system-x86_64 -drive format=raw,file=virtual-disk.img -m 1G
```
![Successful init start](/custom-initramfs.png)

## Summary

This blog is basically a copy of Mitchel Humphrey's Blog post. I do however put my own take on bootloading (I use grub) and I've also added some updates to settings that has changed since 2015. If you read this Mr Humphrey, I thank you for your work. It's inspired me to also try write about this topic.

Next I would like to see if we can boot using an EFI system instead of BIOS. Then I would also like to switch root from the init ramfs to the root filesystem on a virtual harddrive!

# References

- [Mitchel Humphreys: How to build a custom kernel for Qemu 2015 Edition](http://mgalgs.github.io/2015/05/16/how-to-build-a-custom-linux-kernel-for-qemu-2015-edition.html)

