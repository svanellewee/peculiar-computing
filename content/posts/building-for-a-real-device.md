---
title: "Building for a Real Device"
date: 2019-01-29T21:52:23+02:00
draft: true
---

So the other day my buddy Shane gifted me an old ASUS transformer (T100TA). Very generious of him! Since it's a windows laptop and I'm a masochist, I obviously wanted to install Linux on it. Luckily a lot of the work was done already by [this gentleman](http://www.jfwhome.com/2016/01/04/latest-steps-to-install-ubuntu-on-the-asus-t100ta/). Some additional notes was found [here](https://wiki.debian.org/InstallingDebianOn/Asus/T100TA) After some wrestling[^1]  with the debian [non-free multi-arch netinstaller](https://cdimage.debian.org/cdimage/unofficial/non-free/cd-including-firmware/), I managed to get a running 32bit debian installation! 

I got a few very simple apps running.. and [Quake](http://quakespasm.sourceforge.net/). You know, the essentials. However, despite the documentation I *still* couldn't make sound work.

After doing some more reading I realised that the device was able to handle 64bit kernels (the boot infrastructure still ran 32bit for some reason) I had hoped that maybe the drivers I was trying was perhaps better maintained for the 64bit case, therefor, I attempted to install the 64bit version of the same debian distro. 

Failure! Sound still didn't work on the 64bit version and now I had a funny screen glitch that occurred. None of the web resources seemed to have had any effective advice for making it work. This set me off on a mission: 

1. Build my own linux kernel for an actual piece of hardware and..
2. Figure out how to debug hardware behaviour/drivers on Linux.

# Build my own linux kernel for an actual piece of hardware

## Step zero: Download Linux Source
I decided to get the latest greatest kernel, because why not? (Maybe we'll see "why not" later..) Just like I did [here]({{< ref "posts/build-a-kernel-from-scratch" >}}). I was hoping to use the currently installed debian 64bit kernel configuration to base my kernel build on it (copy, paste and adapt.) Fortunately the linux make-system makes this config migration/upgrade very simple (more on that later)!

```bash
 wget https://cdn.kernel.org/pub/linux/kernel/v4.x/linux-4.20.2.tar.xz
 tar -xvf linux-4.20.2.tar.xz -C ~/ # Will create "linux-4.20.2" in your ~/ dir
```

## Step one: Steal the laptop's current Debian installation's config
On the ASUS after installing Debian, I found the config file in the /boot/ directory. I copied that to the system where I was going to build the kernel.

In my case the previous kernel for my current version of Debian was `4.9.0-8-amd64`, but it will probably look different on other systems. Then I just had to plug in my flashstick and copy:

```bash 
# On the ASUS debian distro..
cp /boot/config-4.9.0-8-amd64 /to/usb/drive
umount /to/usb/drive

# plug usb into build pc...
```

Let's call my distro "morty", because I plan on doing horrible things to him, in the name of science or something..
```bash
# On the build pc..
# /from/usb/drive is a placeholder for wherever you mount your usb drive.
mkdir ~/morty
cp /from/usb/drive/config-4.9.0-8-amd64 ~/morty/.config   # Copy and rename
cd ~/linux-4.20.2                   # where linux kernel source was extracted too
make O=~/morty olddefconfig         # Migrate old asus config-4.9.0-8-amd64 to new source config format (adds new config switches)
```
*Here note the last line referencing "olddefconfig". This is the migration make target that takes old kernel settings and inserts the new values with defaults. This is a Linux build system not-so-secret sauce!* 

## Step two: Use the config to rebuild the latest kernel
```bash
make O=~/morty menuconfig    # Select/Deselect things... more on this later.
```
This was a lot of trial and error. Remember to favour built-in kernel modules `<*>` rather than loadable `<M>` because installing modules during init is tricky, especially if you don't know what you're doing. Making the modules "built-in" means they are installed into the kernel and therefor ready to be used from bootup, instead of modules being separate and inert until you write the correct `insmod` or `modprobe` incantation.

Set all things in the following list to be built-in(pre-installed in the kernel) `{*}` and not a module(to be installed at some later time) `{M}` in order to simplify the things you need to have installed at initramfs (the temporary startup environment).

Assuming you are using kernel 4.20 then the changes that I made can be summarised by the following list of config settings (*Don't panic, I'll show how the way to find these configs yourself*)
```
CONFIG_CRC16=y
CONFIG_CRYPTO_AEAD=y
CONFIG_CRYPTO_BLKCIPHER=y
CONFIG_CRYPTO_CBC=y
CONFIG_CRYPTO_CRC32C=y
CONFIG_CRYPTO_CTR=y
CONFIG_CRYPTO_CTS=y
CONFIG_CRYPTO_DRBG_MENU=y
CONFIG_CRYPTO_DRBG=y
CONFIG_CRYPTO_ECB=y
CONFIG_CRYPTO_JITTERENTROPY=y
CONFIG_CRYPTO_NULL=y
CONFIG_CRYPTO_RNG_DEFAULT=y
CONFIG_CRYPTO_RNG=y
CONFIG_CRYPTO_SEQIV=y
CONFIG_CRYPTO_XTS=y
CONFIG_DEVTMPFS_MOUNT=y
CONFIG_EXT4_FS=y
CONFIG_FS_ENCRYPTION=y
CONFIG_FS_MBCACHE=y
CONFIG_HID_GENERIC=y
CONFIG_HID=y
CONFIG_I2C_HID=y
CONFIG_INPUT_MATRIXKMAP=y
CONFIG_INPUT_POLLDEV=y
CONFIG_INTEL_ISH_HID=y
CONFIG_JBD2=y
CONFIG_KEYBOARD_ADP5588=y
CONFIG_KEYBOARD_ADP5589=y
CONFIG_KEYBOARD_DLINK_DIR685=y
CONFIG_KEYBOARD_GPIO_POLLED=y
CONFIG_KEYBOARD_GPIO=y
CONFIG_KEYBOARD_LKKBD=y
CONFIG_KEYBOARD_LM8323=y
CONFIG_KEYBOARD_LM8333=y
CONFIG_KEYBOARD_MATRIX=y
CONFIG_KEYBOARD_MAX7359=y
CONFIG_KEYBOARD_MCS=y
CONFIG_KEYBOARD_MPR121=y
CONFIG_KEYBOARD_NEWTON=y
CONFIG_KEYBOARD_OPENCORES=y
CONFIG_KEYBOARD_QT1070=y
CONFIG_KEYBOARD_QT2160=y
CONFIG_KEYBOARD_SAMSUNG=y
CONFIG_KEYBOARD_STOWAWAY=y
CONFIG_KEYBOARD_SUNKBD=y
CONFIG_KEYBOARD_TCA6416=y
CONFIG_KEYBOARD_TCA8418=y
CONFIG_KEYBOARD_TM2_TOUCHKEY=y
CONFIG_KEYBOARD_XTKBD=y
CONFIG_UHID=y
CONFIG_USB_EHCI_HCD=y
CONFIG_USB_EHCI_PCI=y
CONFIG_USB_HID=y
CONFIG_USB_OHCI_HCD_PCI=y
CONFIG_USB_OHCI_HCD_PLATFORM=y
CONFIG_USB_OHCI_HCD=y
CONFIG_USB_UHCI_HCD=y
CONFIG_USB_XHCI_DBGCAP=y
CONFIG_USB_XHCI_HCD=y
CONFIG_USB_XHCI_PCI=y
CONFIG_USB_XHCI_PLATFORM=y
CONFIG_USB=y
```
To find the value let's assume you have just finished upgrading the ASUS config to the new 4.20 kernel config.

```bash
make O=~/morty olddefconfig
```
Now some of the above `CRYPTO` settings have been done for you! The main stuff I needed was all related to USB/Human Interface Device(HID)/Keyboards. Start menuconfig:

```bash
make O=~/morty menuconfig
```

- Let's say we want to figure out where `CONFIG_USB` exists. Type the `/`-key, type `USB` and the `Location` should be given as `Device Drivers --> USB support --> {*} Support for Host-side USB`. Follow this route through the menu system. When you find it ensure it is set to "built-in"/`{*}` to ensure the setting `CONFIG_USB=y`
- Similarly to find the `CONFIG_OHCI_HCD` setting, type the `/`-key and search `OHCI_HCD` and follow the route specified by the `Location`-details. This should be `Device Drivers --> USB support --> {*} OHCI HCD (USB 1.1) support` and set this `{*}` so that it will match the setting `CONFIG_OHCI_HCD=y`
- For the `CONFIG_USB_OHCI_HCD` setting search `OHCI_HCD` and the `Location` should be `Device Drivers --> USB support --> {*} OHCI support for the PCI-bus USB controllers` and also set to 'built-in'.

And so on, if you'd like to know more, or you'd like me to explain this a bit more please get in touch!

I suspect you can also deactivate a lot of unnecessary things currently enabled in your config. Examples of things you can remove are:

- KVM/virtual machine support
- Weird filesystem support...

and the list goes on. To be fair I'm not sure that everything in my config is required. But this is what worked. If you find a config setting that is unnecessary, let me know. I'll also be setting/amending my list as time goes on.

## Step three: Make a custom initramfs
Initramfs is the initial filesystem that linux mounts. It exists only in memory and allows you the flexibility to do some things before the main filesystem is mounted. I'm going to base my one on Busybox, which is all your normal Unix tools in a convenient package.  Grab and build our busybox image as follows

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
Now to create the `./init` file. The kernel is usually inclined to looks for this script in the initial ramfs, in order to know what code to execute.. All the busybox utilities are basically there to allow the `./init` to do it's job (ie start Linux)

I'm going to try keep mine, barebones..
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

On the ASUS I modified the grub.cfg as follows:

```
vim /boot/grub/grub.cfg
```

I just copied one of the `menuconfig {}` blocks and substituted my own kernel values. Something along the lines of:

```
menuconfig "My awesome config" {
  linux /boot/bzImage root=UUID=...some long value....  ro loglevel=8
  initrd /boot custom-initramfs.cpio.gz
}
```
The UUID value should be available in the other configs there, but apparently there are other ways of finding that value... I won't go into that here. Save and restart.

When Mr ASUS boots again, be sure to select "My awesome config" and see if that works for you. I had quite a few times where it didn't work. The key here is to keep trying. Recompile, boot the ASUS into the current Debian install,  copy over to the new kernel `bzImage` to the ASUS /boot directory, reboot and try "My awesome config" again. 

## Conclusion:

- Question: Did I manage to fix my sound issues?
- Answer: Not yet. I feel this is just the first step to learning more about a Linux. The sad thing is I'm one of *many* people out there who don't actually know much about Linux, despite relying on a stack that is built on top of it. I would love to know how the OS sausage is made. And I'm hoping in my journey to understand Linux better I will be a better engineer in the longrun. 

- Question: In that case, why not go with [Linux from scratch](http://www.linuxfromscratch.org/lfs/view/stable/)?
- Answer: Oh I tried, a couple of times on a VM. It was so boring and I found that the lack of having some indication of progress after *hours* of building, only to be snookered by some arbitrary build error that slipped in due to reasons unknown, very frustrating. I think I'll still try it again at some stage, but I think knowing where to go is something that might help a project such as Linux from scratch. I'm trying to answer that need with these blogs.

- Question: In that case, why not go with [Archlinux](https://archlinux.org) or [Gentoo](https://gentoo.org)?
- Answer: Great projects, gives you a good idea of how the OS's work. But I was concerned that I'd become too dependant the abstractions the tools of these distro's might include. Example being `archroot`. Why that? Why not regular everyday `chroot` ? Oh I see there are niceties...Why? What makes these things essential? Why not just add it to upstream `chroot`? So I will get to these distros one day, I just feel I'm not going to learn as much if I used them right now.

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

## Refernces
https://cdimage.debian.org/cdimage/unofficial/non-free/cd-including-firmware/9.7.0+nonfree/multi-arch/iso-cd/

## Ways to debug
- grub kernel parameter snippet, set loglevel=8 # all the debug (must be > 7)
- USB_TEST/TEST_USB
Run the VM using qemu.. snippet

For some reason USB keyboards don't emulate in QEMU unless you do this, can't explain why..:
```bash
qemu-system-x86_64    -usb   -device usb-host,hostbus=2,hostaddr=1  -drive format=raw,file=/path/to/raw/disk/virtual-disk.img  -m 1G
```
[^1]: I had to quit out of the normal installer process to install the settings file the wifi required to work..
