---
title: "Virtualbox Efi Linux"
date: 2019-03-20T06:00:19+02:00
draft: true
---

I've been using [QEMU](https://www.qemu.org/) for my previous posts, but since I'd like to experiment with EFI a bit more, I've decided to move my efforts to a [VirtualBox](https://www.virtualbox.org/), since it seems to have EFI support baked in.

In order to make this post a little less of a repeat of the previous posts, I've attempted to write more of the post, *in code* and just explain how each snippet works and can be used.

## Making the blank image file

I try to write all my bash code as a library for later reuse. That way I have more primitives to work with when I want to retry certain parts of the process again. The following snippet creates an `INFO` function that writes to `stderr` so that the `stdout` stuff can be piped without adding human level logging into the mix.

```bash
#!/usr/bin/env bash

function info() {
	local GREEN="\e[32m"
	echo -e "\e[32m[INFO]\e[0m $*" >&2
}
```

Next I create a function that will create the blank image given the required filename and size in Megs:

```bash
function make-blank-image() {
	local OUTPUT_FILE="${1}"
	local SIZE="${2}"
	info "making a image file "${OUTPUT_FILE}" of ${SIZE}Mbytes"
	dd if=/dev/zero \
		of="${OUTPUT_FILE}" \
		bs=1M count="${SIZE}"
}
```

Using `parted` I then create a function that can partition the file into the 
- EFI,
- Swap and
- Main partitions

```bash

function make-paritions() {
	local INPUT_FILE="${1}"
	local SWAP_SIZE=512
	local GPT_START="0%"
	local GPT_END=257
	local SWAP_START=$((${GPT_END} + 1))
	local SWAP_END=$((${SWAP_START} + ${SWAP_SIZE}))
	local MAIN_START=$((${SWAP_END} + 1))
	local MAIN_END_KBYTES=$(du ${INTPUT_FILE} | awk '{ print $1 }') 
	local MAIN_END=$(bc <<<"${MAIN_END_KBYTES} / 1024")
	info "Making GPT from ${GPT_START} ${GPT_END}"
	parted --script "${INPUT_FILE}" mklabel gpt mkpart p fat32 ${GPT_START} ${GPT_END} set 1 boot on
	info "Making swap from ${SWAP_START} ${SWAP_END}"
	parted --script "${INPUT_FILE}" mkpart p linux-swap ${SWAP_START} ${SWAP_END}
	info "Making main partition from ${MAIN_START} ${MAIN_END}"
	parted --script "${INPUT_FILE}" mkpart p ext4 ${MAIN_START} ${MAIN_END}
}
```

Now here it gets a little ugly. Once the image file has been partitioned we have to map the image is seperate loopback devices so that they can be formatted. For instance, since the `EFI` partition needs to be `FAT` filesystem and I've chosen to format the `main` partition as `EXT4` filesystem, they have to be applied separately to each partition.

```bash
function format-partitions() {
	local INPUT_FILE="${1}"
	sudo kpartx -av "${INPUT_FILE}"
	sleep 1
	sudo mkfs.vfat /dev/mapper/loop0p1
	sleep 1
	sudo mkfs.ext4 /dev/mapper/loop0p3
	sleep 1
	sudo kpartx -d "${INPUT_FILE}"
}
```
Note, I've added sleep, since it seems that formatting has some sort of asynchronous aspect to it. I'm open to suggestions on why this is and how to improve this. The *really* ugly part is that I still need to apply sudo to access a file that merely lives in my own home directory. Sure, the /dev/ directory is restricted, but I would have liked to be able to perform operations on my own files without sudo access as a requirement. Perhaps projects such as [libguestfs](http://libguestfs.org/) could help, but I'm not quite sure.

Next I wrote a function that will install files to the root partition of my image.

```bash
function install-to-root() {
	local INPUT_FILE="${1}"
	local KERNEL_IMAGE_LOCATION="${2}"
	mkdir -p /tmp/root
	sudo kpartx -av "${INPUT_FILE}"
	sleep 1
	sudo mount /dev/mapper/loop0p3 /tmp/root
	sudo mkdir /tmp/root/boot
	sudo cp ${KERNEL_IMAGE_LOCATION} /tmp/root/boot
	sudo umount /tmp/root
	sudo kpartx -d "${INPUT_FILE}"
}
```

```bash
function install-to-boot() {
	set -x
	local INPUT_FILE="${1}"
	local BOOT_IMAGE_LOCATION="${2}"
	mkdir -p /tmp/boot
	sudo kpartx -av "${INPUT_FILE}"
	sleep 1
	sudo mount /dev/mapper/loop0p1 /tmp/boot
	sudo mkdir -p /tmp/boot/EFI/boot/
	sudo cp ${BOOT_IMAGE_LOCATION} /tmp/boot/EFI/boot/
	if [[ ! -f /tmp/boot/EFI/boot/startup.nsh ]]
	then
		cat <<- "EOF" > /tmp/boot/EFI/boot/startup.nsh
		\EFI\BOOT\BOOTX64.EFI
		EOF
	fi
	sudo umount /tmp/boot
	sudo kpartx -d "${INPUT_FILE}"
	set +x
}
```

```bash
function setup-grub(){
	local KERNEL_PATH="${1}"
	local INITRAMFS_PATH="${2}"
	cat <<- EOF > /tmp/grub.cfg
	insmod part_gpt
	insmod part_msdos
	insmod fat
	insmod efi_gop
	insmod efi_uga
	insmod video_bochs
	insmod video_cirrus

	menuentry "Start The Magic.." {
	  set root=(hd1,gpt3)
	  linux ${KERNEL_PATH} loglevel=8 acpi=off
	  initrd ${INITRAMFS_PATH}
	}
	EOF
	grub-mkstandalone -d /usr/lib/grub/x86_64-efi \
		-O x86_64-efi \
		--modules="part_gpt part_msdos" \
		--fonts="unicode" \
		-o "./bootx64.efi" "boot/grub/grub.cfg=/tmp/grub.cfg" -v
}
```

```bash
function build-initramfs() {
	local BUSYBOX_DIR=${1}
	if [[ -d /tmp/busybox ]]
	then
		rm -fr /tmp/busybox
	fi
	cp -ar "${BUSYBOX_DIR}"/_install /tmp/busybox
	cat <<- "EOF" > /tmp/busybox/init
	#!/bin/busybox sh
	mount -t proc none /proc
	mount -t sysfs none /sys
	echo "Starting kernel $(uname -r)"
	exec /bin/sh
	EOF
	local KERNEL_DIR="${2}"
	(
	cat <<- EOF
	# A simple initramfs
	dir /dev 0755 0 0
	nod /dev/console 0600 0 0 c 5 1
	nod /dev/tty0 0600 0 0 c 4 0
	dir /sys 0755 0 0
	dir /proc 0755 0 0
	dir /root 0700 0 0
	dir /sbin 0755 0 0
	dir /bin 0755 0 0
	file /bin/busybox /tmp/busybox/bin/busybox 755 0 0
	EOF

	for i in $(find /tmp/busybox/sbin/ -type l )
	do 
		cur_link="${i##/tmp/busybox/sbin/}"
		echo "slink /sbin/${cur_link} /bin/busybox 777 0 0"
	done
	for i in $(find /tmp/busybox/bin/* -type l )
	do
		cur_link="${i##/tmp/busybox/bin/}"
		echo "slink /bin/${cur_link} /bin/busybox 777 0 0"
	done
	echo "file /init /tmp/busybox/init 755 0 0"
	) > /tmp/initramfs-list
	${KERNEL_DIR}/usr/gen_init_cpio /tmp/initramfs-list > custom-initramfs.cpio.gz
}
```

```bash
function convert-raw-to-vdi() {
    local INPUT_FILE="${1}"
    VBoxManage convertdd ${INPUT_FILE}.img ${INPUT_FILE}.vdi --format VDI
}
```
