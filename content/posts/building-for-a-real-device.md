---
title: "Building for a Real Device"
date: 2019-01-29T21:52:23+02:00
draft: true
---

So the other day my buddy Shane gifted me an old ASUS transformer (T100TA). Very generious of him! Since it's a windows laptop and I'm a masochist, obviously I wanted to install Linux on it. Luckly a lot of the work was done already by [this gentleman](http://www.jfwhome.com/2016/01/04/latest-steps-to-install-ubuntu-on-the-asus-t100ta/). Some additional notes was found [here](https://wiki.debian.org/InstallingDebianOn/Asus/T100TA) After some restling with the debian non-free netinstaller(add link), I managed to get a running 32bit debian installation! 

I got a few very simple apps running.. and [Quakespasm](http://quakespasm.sourceforge.net/). You know, the essentials. But despite the documentation found, I still couldn't make sound work.

After doing some more reading I realised that the device was able to handle 64bit kernels (the boot infrastructure still ran 32bit for some reason) I had hoped that maybe the drivers I was trying was perhaps only available for 64bit, therefor, I attempted to install the 64bit version of the same debian distro. 

Failure! Sound still didn't work on the 64bit version. None of the web resources seemed to have had any effective advice for making it work. This set me off on a mission: _Build my own kernel, and figure out how to debug hardware on Linux_.

## Step zero: Download Linux Source
I decided to get the latest greatest kernel, because why not? (Maybe we'll see why not later..) You can check my previous post [here]({{< ref "posts/build-a-kernel-from-scratch" >}}). I was hoping to use the currently installed debian 64bit kernel configuration to base my kernel build on it (copy, paste and adapt.)

```bash
 wget https://cdn.kernel.org/pub/linux/kernel/v4.x/linux-4.20.2.tar.xz
 tar -xvf linux-4.20.2.tar.xz -C ~/ # Will create "linux-4.20.2" in your ~/ dir
```

## Step one: Steal The Laptop's current Debian's Config
On the ASUS, find the config file in the /boot/ directory. Copy it to the system where you're going to build the kernel.

In my case the previous kernel for my current version of Debian was `4.9.0-8-amd64`, but it will probably look different on other systems. Then I just had to plug in my flashstick and copy:

```bash 
# On the ASUS debian distro..
cp /boot/config-4.9.0-8-amd64 /to/usb/drive
umount /to/usb/drive

# plug usb into build pc...
```

Let's call my distro "morty", because I plan on messing with this poor little thing's brain a bit.

```bash
# On the build pc..
# /from/usb/drive is a placeholder for wherever you mount your usb drive.
mkdir ~/morty
cp /from/usb/drive/config-4.9.0-8-amd64 ~/morty/.config   # Copy and rename
cd ~/linux-4.20.2                   # where linux kernel source was extracted too
make O=~/morty olddefconfig         # Migrate old asus config-4.9.0-8-amd64 to new source config format (adds new config switches)
```

## Step two: Use the Config to rebuild the latest kernel
```bash
make O=~/morty menuconfig    # Select/Deselect things... more on this later.
```
This was a lot of trial and error. Remember to favour built-in kernel modules `<*>` rather than loadable `<M>` because your initial ramfs will be vastly simplified that way.

Set all things in the folling list to be builtin `{*}` and not a module `{M}` in order to simplify the things you need to have installed at initramfs (the temporary startup environment).

- `Device Drivers --> USB support --> {*} Support for Host-side USB`
- `Device Drivers --> USB support --> {*} OHCI HCD (USB 1.1) support`
- `Device Drivers --> USB support --> {*} OHCI support for the PCI-bus USB controllers`
- `Device Drivers --> HID support --> USB HID Support --> <*> USB HID transport layer`
- `Device Drivers --> HID support --> HID bus support {*}` 
- `Device Drivers --> HID support --> <*> Generic HID driver` 

Those are the important ones. I suspect you can also deactivate a lot of rubish in your config. Examples of things you can remove are:

- KVM/virtual machine support
- Weird filesystem support
- A lot of other thing..
### What to enable?
Problems encountered. 
[Solved](https://wiki.archlinux.org/index.php/Minimal_initramfs#Keyboard_modules)
Bake in the requirements..

## Step three: Make a custom initramfs
Initramfs is the initial filesystem that linux mounts. It exists only in memory and allows you the flexibility to do some things before the main filesystem is mounted. Grab and build our busybox image as follows

```bash
cd /tmp/
wget https://busybox.net/downloads/busybox-1.30.0.tar.bz2
tar -xvf busybox-1.30.0.tar.gz           # xzvf won't work for bz2
cd /tmp/busybox-1.30.0
mkdir ~/busybox                          # Now let's build to ~/busybox
make O=~/busybox menuconfig
```
Set the `Settings --> [*] Build static binary (No shared libs)` to enabled `[*]`, exit, compile and install

```bash
make O=~/busybox -j2
make O=~/busybox install
```

You should end up with the following built structure
```bash
stephan:/tmp/busybox-1.30.0 $ ls ~/busybox/_install/ -ahltr
total 20K
drwxrwxr-x 32 stephan stephan 4.0K Jan 29 22:19 ..
drwxrwxr-x  2 stephan stephan 4.0K Jan 29 22:19 bin
lrwxrwxrwx  1 stephan stephan   11 Jan 29 22:19 linuxrc -> bin/busybox
drwxrwxr-x  2 stephan stephan 4.0K Jan 29 22:19 sbin
drwxrwxr-x  5 stephan stephan 4.0K Jan 29 22:19 .
drwxrwxr-x  4 stephan stephan 4.0K Jan 29 22:19 usr
```
Make a workspace to put your initramfs. I called mine `~/initramfs`

```bash
# Make a new directory to build our initramfs in
mkdir ~/initramfs
cd ~/initramfs
```
Now to create the `./init` file. This script tells the kernel what the things are we want to mount and start. I'm going to try keep mine, barebones..
```bash
# edit the "init" file
cat <<"EOF" > init
#!/bin/busybox sh
echo "Starting kernel $(uname -r)"
echo "lsusb says:"
lsusb
echo "done."
exec /bin/sh
EOF
# Make executable
chmod a+x init
```
Now let's construct the initramfs using the Linux kernel tools:
Then let's copy the busybox files recursively:
```bash
cd ~/initramfs
cp -rav ~/busybox/_install/* .
```
You should see these items
```bash
stephan:~/initramfs $ ls
bin  init  linuxrc  sbin  usr
```
After that, we need to use the linux tree tool `gen_init_cpio` by specifying what we want in the initramfs. The very simple one is as follows:

```bash
cat <<EOF > initramfs-list
# A simple initramfs
dir /dev 0755 0 0
nod /dev/console 0600 0 0 c 5 1
dir /root 0700 0 0
dir /sbin 0755 0 0
dir /bin 0755 0 0
file /bin/busybox ${HOME}/initramfs/bin/busybox 755 0 0
slink /sbin/lsusb /bin/busybox 777 0 0
slink /bin/sh busybox 777 0 0
file /init ${HOME}/initramfs/init 755 0 0
EOF
```

You can replace references `${HOME}/initramfs/` with the absolute path of your actual `./init` file we've made previously. For example mine was
`/home/stephan/morty/`

The symlink entries (`slink`) is very important since our first process will use `sh`- and `lsusb`-applets to work. Now we need to call `gen_init_cpio` from where we compiled it earlier (should be `~/morty/`)

```bash
~/morty/usr/gen_init_cpio initramfs-list > custom-initramfs.cpio.gz
```
Now the plan is to copy both the kernel and the initramfs to a flashstick to install to our ASUS

```bash
cp ~/morty/arch/x86/boot/bzImage         /wherever/flashdrive/is/mounted
cp ~/initramfs/custom-initramfs.cpio.gz /wherever/flashdrive/is/mounted
```

## Step four: copy back to the ASUS
```bash
# On the ASUS laptop
# mount your flashstick (please let me know if you need help here)
cp /mnt/bzImage /boot/
cp /mnt/custom-initramfs.cpio.gz /boot/
```

## Step five: Grub update

## Conclusion: Did I manage to fix my sound issues?
Answer: Not yet. Why bother with all this then?......
## Other notes:

The current initramfs doesn't actually add all the symlinks we imported from busybox in our copy. This might fix that:

```bash
#!/usr/bin/env bash
(
printf "
# A simple initramfs
dir /dev 0755 0 0
nod /dev/console 0600 0 0 c 5 1
dir /root 0700 0 0
dir /sbin 0755 0 0
dir /bin 0755 0 0
file /bin/busybox ${HOME}/initramfs/bin/busybox 755 0 0
"
for i in $(find sbin/* -type l )
do
        cur_link="${i##sbin/}"
        echo "slink /sbin/${cur_link} busybox 777 0 0"
done
for i in $(find bin/* -type l )
do
        cur_link="${i##bin/}"
        echo "slink /bin/${cur_link} busybox 777 0 0"
done
printf "
file /init ${HOME}/initramfs/init 755 0 0
"
) > initramfs-list
~/morty/usr/gen_init_cpio initramfs-list > custom-initramfs.cpio.gz
```
Just put this into file in your initramfs workspace (`~/initramfs`) after busybox has been copied


## Ways to debug
grub kernel parameter snippet:.....loglevel=8 # all the debug (must be > 7)
USB_TEST/TEST_USB
Run the VM using qemu.. snippet

For some reason USB keyboards don't emulate in QEMU unless you do this, can't explain why..:
```bash
qemu-system-x86_64    -usb   -device usb-host,hostbus=2,hostaddr=1  -drive format=raw,file=/path/to/raw/disk/virtual-disk.img  -m 1G
```
