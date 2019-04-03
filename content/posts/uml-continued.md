---
title: "Uml Continued"
date: 2019-04-02T21:44:52+02:00
draft: true
---


Previously I managed to build UML (User-mode Linux) but using the host filesystem as it's filesystem. I wanted to go a step further and replace that as well.

Here goes. So far the best example of how to do this lives in the busybox repositories' own git repo, in the [Examples Directory](https://git.busybox.net/busybox/tree/examples/bootfloppy?id=2f28b2bdbbe229b760e7c2a271d73a19f929ca76) I deviated from it however, since some aspects doesn't seem required anymore (such as the uClibc compilation step)

To build this you need:

- The kernel source (obviously)
- The busybox source

## The kernel source

To build everything this time we need to support the special block device (drive) that User-Mode Linux exposes to the world namely `UBD?`. 

```bash
# The kernel source tree that I tested this process on.
cd linux-5.0.3/         

# setup the mini config for the kernel setup
cat <<EOF > mini.config
CONFIG_64BIT=y
CONFIG_X86_64=y
CONFIG_BINFMT_ELF=y
CONFIG_HOSTFS=y
CONFIG_BLOCK=y
CONFIG_LBD=y
CONFIG_BLK_DEV=y
CONFIG_BLK_DEV_LOOP=y
CONFIG_BLK_DEV_UBD=y   # to allow UML to have a ubda ubdb.. block device. This will reference our image.
CONFIG_STDERR_CONSOLE=y
CONFIG_UNIX98_PTYS=y
CONFIG_MODULES=y
CONFIG_INOTIFY_USER=y
CONFIG_NET=y
CONFIG_UNIX=y
CONFIG_EXT2_FS=y
CONFIG_EXT3_FS=y
CONFIG_EXT4_FS=y
CONFIG_PROC_FS=y      # for /proc pseudo filesystem support
CONFIG_SYSFS=y        # for /sys pseudo filesystem support
CONFIG_PROC_SYSCTL=y  # for /proc/sys support (device auto creation)
CONFIG_TTY_CHAN=y
EOF

# make a place to put our UserModeLinux (example ~/uml)
mkdir ~/uml
# set every value to NO except for what's specified in mini.config
make O=~/uml ARCH=um allnoconfig KCONFIG_ALLCONFIG=~/linux-5.0.3/mini.config
# -j specifies the number of processors to compile on. $(nproc) provides the core count.
make O=~/uml ARCH=um -j$(nproc)       
```

This should mean that you end up with an executable at `~/uml/linux`. Next we'll need to build the filesystem and that's where busybox's `bootfloppy` example comes in handy for clues.

## The busybox source

Firstly make a root filesystem at `/tmp/rootfs` for example:

```bash
# make a 400Meg image (Arbitrarily chosen number)
dd if=/dev/zero of=/tmp/rootfs bs=1M count=400

# I want the latest EXT filesystem..
mkfs.ext4 rootfs

# Make a mount point and mount via loopback
mkdir ~/rootfs_mnt 
sudo mount -o loop /tmp/rootfs ~/rootfs_mnt
```

Download and extract busybox

```bash

cd busybox-1.30.1

make menuconfig   # remember to choose "Settings --> Build static binary"

make -j$(nproc)


```
Finally to install busybox to our new image we do the following (this assumes `~/rootfs_mnt` is still mounted)

```
make CONFIG_PREFIX=~/rootfs_mnt install

```

This should leave you with an incomplete filesystem:

```
cd ~/rootfs_mnt
mkdir dev sys proc
```
Here I cheated. I used the etc from the `example/bootfloppy` directory to make my own.

```
cp -a ~/busybox-1.30.1/examples/bootfloppy/etc/ ~/rootfs_mnt/etc

# take out the tty2::askfirst part. Not sure why this breaks.
grep -v tty2::askfirst ~/rootfs_mnt/etc/inittab  > /tmp/inittab
cp /tmp/inittab ~/rootfs_mnt/etc/inittab
```

I use fstab to set up my `/sys` and `/proc` directories. The proc line should already be there but just in case here's the whole file as I have it currently

```
cat <<EOF > ~/rootfs_mnt/etc/fstab
proc /proc proc defaults 0 0
sysfs /sys sysfs defaults 0 0
EOF
```
The reason we want `sysfs` to be available is so that we can automate the device creation using `mdev`. There is a manual script in the `bootfloppy` example but I'd like to see how hard `mdev` would be given that it's built into `busybox`. I followed the example [here](https://git.busybox.net/busybox/plain/docs/mdev.txt).

*Note* there's a note on using he command ` echo /sbin/mdev > /proc/sys/kernel/hotplug` to allow hotplugging of devices. So far I've not been able to make the `hotplug` file appear on /proc/sys. I'll keep trying though.

Now we can unmount our image

```
umount ~/rootfs_mnt/
```

Run our image that we made (I suggested `/tmp/rootfs` as a name and path)
```
~/uml/linux ubda=/tmp/rootfs root=/dev/ubda rw mem=1G
```

Now since we've set up the `sys` we can just call `mdev -s` to setup our `/dev` devices directory!

## What to do next?

How about we run a couple of User-mode linuxes on one machine?

```

cp /tmp/rootfs /tmp/rootfs1
cp /tmp/rootfs /tmp/rootfs2
cp /tmp/rootfs /tmp/rootfs3
cp /tmp/rootfs /tmp/rootfs4
cp /tmp/rootfs /tmp/rootfs5

cat <<"EOF" >demo-uml.sh
for index in {1..5}
do
   xterm -e ~/uml/linux ubda=/tmp/rootfs${index} root=/dev/ubda rw mem=500M &
done
EOF
bash ./demo-uml.sh
```

Here are the windows, after I moved them around a bit

![Many Linuxes](/many-linuxes.png)

