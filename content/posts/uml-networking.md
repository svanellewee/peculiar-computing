---
title: "Uml Networking"
date: 2019-07-09T19:54:31+02:00
draft: true
---

Having a virtual machine just sitting there isn't terribly exciting. How about we get networking going? 

## The filesystem (again)
In my previous entries I've tried to build a filesystem from scratch, multiple times. I was concerned I might leave some essential device or module out, but I also wanted the most minimal filesystem I could get to prevent unnecessary bloat.

I then turned to [Buildroot](https://buildroot.org/) to automate most of the work of filesystem creation that I was doing manually in the other posts. Figuring out how to configure it correctly was quite tricky. After a lot of googling later I found [this post](https://unix.stackexchange.com/questions/73203/how-to-create-rootfs-for-user-mode-linux-on-fedora-18/372207#372207) as a good starting point. I'll reproduce it here, just in case, with a few of my own notes and changes.

Firstly we set up the buildroot configuration with the qemu x86 configuration as base. The `BR2_ROOTFS_OVERLAY` setting was meant to allow you to add files to your new filesystem after it has been created, in this case, it will be the init system's configuration (see later)

```bash
git clone git://git.buildroot.net/buildroot
cd buildroot
git checkout 2019.05.1
make qemu_x86_64_defconfig
echo 'BR2_ROOTFS_OVERLAY="rootfs_overlay"' >>.config
make olddefconfig
```

Getting sysinit configuration working is usually quite challenging, so I'm quite greatful SO's Mr Santilli was so generious with his information.

```bash
# Make the custom inittab inside the rootfs overlay directory
mkdir -p rootfs_overlay/etc
printf '
::sysinit:/bin/mount -t proc proc /proc
::sysinit:/bin/mount -o remount,rw /
::sysinit:/bin/mkdir -p /dev/pts
::sysinit:/bin/mkdir -p /dev/shm
::sysinit:/bin/mount -a
::sysinit:/sbin/mdev -s
::sysinit:/bin/hostname -F /etc/hostname
::sysinit:/etc/init.d/rcS
console::respawn:/sbin/getty -n -L console 0 vt100
::ctrlaltdel:/sbin/reboot
::shutdown:/etc/init.d/rcK
::shutdown:/sbin/swapoff -a
::shutdown:/bin/umount -a -r
' > rootfs_overlay/etc/inittab
```
I have also manually selected the latest linux kernel and headers (at time of writing, that would be `5.1.x`) 

```bash
# Build image.
make BR2_JLEVEL=$(($(nproc)-2))
```

For future reference I've added some notes on a [gist to keep me honest](https://gist.github.com/svanellewee/22414b800b320e40445a431c732f29fe) 

Anyway this should get the filesystem working. 

## Building yet another kernel

To build the kernel, I've cheated a little, using Arch Linux's AUR magic. What I like about Arch's Linux approach to packages, is that it's so minimalist. It's basically just bash. Even if you run another distro, you can still infer how to build the package just by looking at the bash script.

```bash
pkgname=linux-usermode
true && pkgname=(linux-usermode linux-usermode-modules)
pkgbase=linux-usermode
_kernelname=-usermodelinux
_srcname=linux-5.1.16
pkgver=5.1.16
pkgrel=1
pkgdesc="User mode Linux kernel and modules"
arch=('x86_64')
license=('GPL2')
url="http://user-mode-linux.sourceforge.net/"
depends=('coreutils')
makedepends=('bc' 'inetutils' 'gcc')
source=(
  http://www.kernel.org/pub/linux/kernel/v5.x/${_srcname}.tar.{xz,sign}
  #http://www.kernel.org/pub/linux/kernel/v5.x/patch-${pkgver}.{xz,sign}
  http://www.kernel.org/pub/linux/kernel/v5.x/patch-${pkgver}.xz
  config)

sha256sums=('8a3e55be3e788700836db6f75875b4d3b824a581d1eacfc2fcd29ed4e727ba3e' 
            'cb1fbbe695c7d2fd39b9e36fd23c5256d1d65c7dd813ec30e2ecfc0ae2a9f333'
            '62521554a7c13151a6b7d58e0093656b45fa9a6de06e127e96725a897dc2813a'
            '0c6ca2df8b072b1fa1c13d290e2c1f0c97d872419f4bf8c2fd813a29e79c5626')

validpgpkeys=(
  'ABAF11C65A2970B130ABE3C479BE3E4300411886'  # Linus Torvalds
  '647F28654894E3BD457199BE38DBBDC86092693E'  # Greg Kroah-Hartman
)

prepare() {
  cd "${srcdir}/${_srcname}"

  # add upstream patch
  #patch -p1 -i "${srcdir}/patch-${pkgver}"

  cat ../config - >.config <<END
CONFIG_LOCALVERSION="${_kernelname}"
CONFIG_LOCALVERSION_AUTO=n
END

  # set extraversion to pkgrel
  sed -i "/^EXTRAVERSION =/s/=.*/= -${pkgrel}/" Makefile

  # rewrite configuration
  yes "" | make ARCH=um config >/dev/null
}

build() {
  cd "${srcdir}/${_srcname}"
  unset LDFLAGS CFLAGS

  make ARCH=um CC=gcc vmlinux modules 
}

package_linux-usermode() {
  cd "${srcdir}/${_srcname}"
  mkdir -p "$pkgdir/usr/bin" "$pkgdir/usr/share/kernel-usermode"
  install -m 644 System.map ${pkgdir}/usr/share/kernel-usermode/System.map
  install -m 755 vmlinux ${pkgdir}/usr/bin/
}

package_linux-usermode-modules() {
  install=modules.install

  cd "${srcdir}/${_srcname}"

  # get kernel version, but discard the first result
  make ARCH=um kernelrelease > /dev/null
  _kernver="$(make ARCH=um kernelrelease)"

  #  make ARCH=um INSTALL_MOD_PATH="${pkgdir}/usr" modules_install
  make ARCH=um INSTALL_MOD_PATH="${pkgdir}/usr" _modinst_
  rm -f $pkgdir/usr/lib/modules/${_kernver}/{source,build}
  sed \
    -e "s/KERNEL_VERSION=.*/KERNEL_VERSION=${_kernver}/g" \
    -i "${startdir}/modules.install"
}

```
Of course, building and installing the 2 new packages are as usual for Arch:

```bash
makepkg
sudo pacman -U linux-usermode-5.1.16-1-x86_64.pkg.tar.xz linux-usermode-modules-5.1.16-1-x86_64.pkg.tar.xz

```

## Running one Usermode Linux Instance

After installing the usermode linux kernel next we should run an instance. On the host's side a 'tapX' device will appear with ip `192.168.0.254`. In this case it is `tap0`.

```bash
cd ~/source/buildroot
vmlinux mem=1G ubd0=output/images/rootfs.ext2 eth0=tuntap,,,192.168.0.254
```

There should a long string of startup logs, followed by
```
....
Starting network: 
* modprobe tun
* ifconfig tap0 192.168.0.254 netmask 255.255.255.255 up
* bash -c echo 1 > /proc/sys/net/ipv4/ip_forward
udhcpc: started, v1.30.1
udhcpc: sending discover
udhcpc: sending discover
udhcpc: sending discover
udhcpc: no lease, failing
FAIL
Starting dockerd: OK

buildroot login: root
```

Further, on the UML (guest's) side add an address to the local `eth0` network interface, then add a default route outward.

```bash
> ip addr add 192.168.0.253 dev eth0
* route add -host 192.168.0.253 dev tap0
* bash -c echo 1 > /proc/sys/net/ipv4/conf/tap0/proxy_arp
> ip route add default  via 192.168.0.253 dev eth0 
```

So there is a link between the internal(guest) eth0 device and the external(host) tun device.

Now we want to make sure your host will know where to respond to if your UML guest makes an request.
```bash
HOST> sudo iptables -t nat -I POSTROUTING -o wlp2s0  -j MASQUERADE
HOST> sudo iptables -I FORWARD -o tap0 -j ACCEPT
HOST> sudo iptables -I FORWARD -i tap0 -j ACCEPT
```
My interface `wlp2s0` is my host's physical wifi adapter. Could also have been something like `eth0` so you should just be aware how you're connected in real life. Another thing your virtual adapter might not be called `tap0`. It could be `tap1`, or if you made a custom one it could be `stephan-uml`. _The name is just a convention._

To test the configuration you should be able to ping the following:
```bash
UML> ping 192.168.0.254
UML> ping 8.8.8.8  # Google's DNS
```

If you don't get a response, try checking the traffic on your virtual network adapter:
```bash
HOST> sudo tcpdump  -i tap0 -l -n
```

A big help to understanding how to debug UML networking was the [User Mode Linux Book by Jeff Dike](http://ptgmedia.pearsoncmg.com/images/9780131865051/downloads/013865056_Dike_book.pdf) What was great though was that none of the networking debugging hints was special to UML. It was all just stock standard methods for debugging Linux networking. In this sense User Mode Linux is immencely helpful.

## Extra credit: A second UML instance

```bash
cd ~/source/buildroot
cp output/images/rootfs{,-alt}.ext2     # copy the rootfs.ext2 file and give it a `-alt` suffix.
vmlinux mem=1G ubd0=output/images/rootfs-alt.ext2 eth0=tuntap,,,192.168.0.252   # start another tap device (tap1?) with ip 192.168.0.252
```

Then after logging in as `root` on the UML do the same as previous instance (but with different IP)
```bash
UML> ip addr add 192.168.0.251 dev eth0
UML> ip route add default  via 192.168.0.251 dev eth0
UML> ping 192.168.0.253   # can you ping the other UML ? Probably not
UML> ping 8.8.8.8 # Won't work.. why?
```

Why won't this ping yet? Because you forgot to update the new tap devices iptables. Assuming your new tap device is called `tap1`..
```bash
HOST> sudo iptables -t nat -I POSTROUTING -o wlp2s0  -j MASQUERADE
HOST> sudo iptables -I FORWARD -o tap1 -j ACCEPT
HOST> sudo iptables -I FORWARD -i tap1 -j ACCEPT
```

NOW! Pinging will happen. Pinging between UML guests, pinging between hosts and UML's.
