---
title: "User Mode Linux"
date: 2019-03-19T11:22:25+02:00
draft: true
---
## Intro
In my study of the linux kernel I recently learned about the concept of "User-mode Linux". This allows you to run a linux kernel as an executable. (In other words, Linux running on Linux.)

### Why run Linux inside of Linux ?
There are various reasons. You can run `gdb` to validate the latest kernel code changes, conveniently from within your current system and using your old kernel. You can setup your own virtualization, even create a network of these linux executables. You can even create [honeypots](http://user-mode-linux.sourceforge.net/old/honeypots.html)

The documentation on this seems not to follow the current state of things so much. (The [official site](http://user-mode-linux.sourceforge.net/) doesn't even reflect that this feature has been merged into the main Linux kernel source tree since version 2.6.0! (_At the time I'm writing this the kernel has just reached version 5 now!_)

Clearly there is a  *lot* of web posts covering this and I know I'm probably adding to the noise, but for me this post is to document my method of getting a user mode linux (aka UML) working, based on various sources (Which I will list in the references below.)

In order to build a user mode kernel I learnt about yet another way to set the configuration of the Linux build system. 

### Environment variables `ARCH=um` and `KCONFIG_ALLCONFIG`
The linux build system allows you to overlay a small subset of options over another standard config. For example:

```bash
make  ARCH=um  KCONFIG_ALLCONFIG=./mini.config allnoconfig
```

Here, the file `mini.config` sets some kernel build options, while `allnoconfig` unsets all the other options. It's similar to applying a bitmask operation on the options.

A post from 2013 specified the most helpful get started I could find. [Techoverflow.net](https://techoverflow.net/2013/07/09/user-mode-linux-for-beginners-setup-and-first-vm/) specified a nice succinct list of config options to get you going. Unfortunately something must have changed in the Linux kernel build system since it was written, because as-is, it doesn't work on kernel 4.20.2.


*Tip*: When setting a `mini.config` option remember to also add the option it's dependant on. The build system doesn't seem to do that for you. In the above posts' case, adding `CONFIG_BLOCK=y` seemed to allow the EXT2,3,4 options to work. **Options seems to be structured in a dependency tree**. After adding this `CONFIG_BLOCK` setting to the Techoverflow example and deciding I wanted to build all the settings into the kernel (rather than separate modules) I ended up with a `mini.config` as follows:

```bash
cat <<EOF > mini.config
CONFIG_64BIT=y
CONFIG_X86_64=y
CONFIG_BINFMT_ELF=y
CONFIG_HOSTFS=y
CONFIG_BLOCK=y
CONFIG_LBD=y
CONFIG_BLK_DEV=y
CONFIG_BLK_DEV_LOOP=y
CONFIG_STDERR_CONSOLE=y
CONFIG_UNIX98_PTYS=y
CONFIG_EXT2_FS=y
CONFIG_MODULES=y
CONFIG_INOTIFY_USER=y
CONFIG_NET=y
CONFIG_UNIX=y
CONFIG_EXT3_FS=y
CONFIG_EXT4_FS=y
EOF
```

Finally when your new `mini.config` is done then let the build system know by assigning the filename to the special `KCONFIG_ALLCONFIG` environment variable. Ie:

```bash
KCONFIG_ALLCONFIG=./mini.config
```
Finally, the `ARCH=um` setting tells the build system to build a "user mode" kernel that can run as an executable (read "guest") on a host Linux system.

### The whole process from start to finish
#### Make a directory (I'll choose `~/uml` as mine)

```bash
mkdir ~/uml
```
#### Download and extract linux source somewhere

```bash
curl https://cdn.kernel.org/pub/linux/kernel/v5.x/linux-5.0.3.tar.xz -o ~/Downloads/linux-5.0.3.tar.xz
tar -xvf ~/Downloads/linux-5.0.3.tar.xz -C ~/
cd ~/linux-5.0.3
```

#### Make your `mini.config` 

```bash
cat <<EOF > ~/linux-5.0.3/mini.config
CONFIG_64BIT=y
CONFIG_X86_64=y
CONFIG_BINFMT_ELF=y
CONFIG_HOSTFS=y
CONFIG_BLOCK=y
CONFIG_LBD=y
CONFIG_BLK_DEV=y
CONFIG_BLK_DEV_LOOP=y
CONFIG_STDERR_CONSOLE=y
CONFIG_UNIX98_PTYS=y
CONFIG_EXT2_FS=y
CONFIG_MODULES=y
CONFIG_INOTIFY_USER=y
CONFIG_NET=y
CONFIG_UNIX=y
CONFIG_EXT3_FS=y
CONFIG_EXT4_FS=y
EOF
```

#### Setup the build system with the new config. (The following sets nothing but what's provided in the `mini.config`)

```bash
make O=~/uml ARCH=um allnoconfig KCONFIG_ALLCONFIG=~/linux-5.0.3/mini.config
```
You can use `grep` to confirm that the EXT3, EXT4 settings have been applied successfully, for example:

```bash
stephan:~/linux-5.0.3 $ grep EXT ~/uml/.config
# CONFIG_EXTCON is not set
CONFIG_EXT2_FS=y
# CONFIG_EXT2_FS_XATTR is not set
CONFIG_EXT3_FS=y
# CONFIG_EXT3_FS_POSIX_ACL is not set
# CONFIG_EXT3_FS_SECURITY is not set
CONFIG_EXT4_FS=y
# CONFIG_EXT4_FS_POSIX_ACL is not set
# CONFIG_EXT4_FS_SECURITY is not set
# CONFIG_EXT4_ENCRYPTION is not set
# CONFIG_EXT4_DEBUG is not set
# CONFIG_PAGE_EXTENSION is not set
# CONFIG_DEBUG_BLOCK_EXT_DEVT is not set
```

#### Build!

```bash
make O=~/uml ARCH=um -j$(nproc)   # nproc gives the number of cpu's we can work with
```
The build step shouldn't take too long(approximately 3 mins on an oldish laptop), since most build options have been disabled.

#### Run!

You can build your own filesystem apparently (though I'm still unsure how) but for now the simplest would just be to use the host filesystem and host's `bash` as the init process.

```bash
cd ~/uml
./linux rootfstype=hostfs rw init=/bin/bash
```

You should then be able to navigate using the new kernel. To validate that you're *not* using the host kernel you can issue a `uname -sr`.

![Successful UML Run](/uml-works.png)


### Conclusion

I look forward to playing with more user mode linux kernels! I am amazed at the wonderful features the Linux kernel and it's build system allows one to do. Given how fast things move I have no doubt this post will be date soon (as most UML tutorials seem to end up being). Perhaps this can act as a way-marker for those who's trying to make sense of this great (but strangely unknown) feature of the Linux kernel.
