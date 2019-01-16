---
title: "Create a Grub File For Our Toy Linux"
date: 2019-01-16T20:48:58+02:00
draft: true
---

So after [the previous post]({{< ref "posts/bootstrap-a-linux" >}}) I promised I was going to show how to make a nifty grub menu so that we won't have to keep typing all the commands manually to boot the kernel.

Turns out it's quite simple:

```bash
cat <<EOF | sudo tee /mnt/boot_mount/boot/grub/grub.cfg
 menuentry "Boot up the kernel" {
    set root="(hd0,msdos2)"
    linux /kernels/vmlinuz-4.10.0-27-generic root=/dev/sda1
    initrd /kernels/initrd.img-4.10.0-27-generic
    boot
}
EOF

sync   # you need to do this to commit the changes to the virtual drive

# Now you can run the emulator again
qemu-system-x86_64 -drive format=raw,file=virtual-disk.img -m 1G
```

You should be greeted by the following

![Grub Menu](/grub-menu.png)
