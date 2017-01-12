(current format is moniwiki)

http://mgalgs.github.io/2015/05/16/how-to-build-a-custom-linux-kernel-for-qemu-2015-edition.html


minimal kernel booting with initramfs of busybox


= build busybox and generate initramfs =

kernel v4.4 needs more recent busybox or fails to execute init.

{{{
$ curl http://busybox.net/downloads/busybox-1.23.2.tar.bz2 | tar xjf -
$ make defconfig
$ make O=../obj/busybox-x86 menuconfig
}}}


'''IMPORTANT: type /, search for “static”. You’ll see that the option is located at:'''

{{{
-> Busybox Settings
  -> Build Options
[ ] Build BusyBox as a static binary (no shared libs)

$ make -j8
$ make install

$ mkdir -p ./initramfs/x86-busybox
$ cd ./initramfs/x86-busybox
$ mkdir -pv {bin,sbin,etc,proc,sys,usr/{bin,sbin}}
$ cp -av ./busybox-x86/_install/* .

$ vim init
#!/bin/sh
 
mount -t proc none /proc
mount -t sysfs none /sys

#Create device nodes
mknod /dev/null c 1 3
mknod /dev/tty c 5 0
mdev -s
 
echo -e "\nBoot took $(cut -d' ' -f1 /proc/uptime) seconds\n"
 
exec /bin/sh

$ chmod +x init
The Gentoo wiki’s Custom Initramfs page is a great reference for building a minimalistic initramfs if you’d like to learn more.

We’re now ready to cpio everything up:

$ find . -print0 \
    | cpio --null -ov --format=newc \
    | gzip -9 > $TOP/obj/initramfs-busybox-x86.cpio.gz
}}}

= build kernel with kvm features =

https://github.com/teobaluta/qr-linux-kernel/commit/46ff53874bd935ab9955dee56d60212857e89bf3

{{{
$ make x86_64_defconfig
$ make kvmconfig
add some features for kvm

$ make -j8
}}}

= boot qemu with virtual disk =

{{{
$ qemu-img create disk_data.img 16G
$ qemu-system-x86_64 -kernel arch/x86/boot/bzImage \
-initrd ../qemu_initramfs/initramfs_dir/initramfs-busybox-x86.cpio.gz \
-nographic -append "console=ttyS0" -enable-kvm \
-drive file=disk_data.img,if=virtio,cache=none
}}}

= monitor mode =

enter monitor more: ctrl+a c

quit command to quit qemu


= format disk =

{{{
/ # fdisk /dev/vda
/ # mkfs.ext2 /dev/vda
Filesystem label=
OS type: Linux
Block size=4096 (log=2)
Fragment size=4096 (log=2)
1048576 inodes, 4194304 blocks
209715 blocks (5%) reserved for the super user
First data block=0
Maximum filesystem blocks=4194304
128 block groups
32768 blocks per group, 32768 fragments per group
8192 inodes per group
Superblock backups stored on blocks:
	32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208, 4096000
/ # mount -t ext4 /dev/vda ./mnt
[  120.864918] EXT4-fs (vda): mounted filesystem without journal. Opts: (null)
/ # ls mnt
lost+found
/ # mount
rootfs on / type rootfs (rw,size=55128k,nr_inodes=13782)
none on /proc type proc (rw,relatime)
none on /sys type sysfs (rw,relatime)
/dev/vda on /mnt type ext4 (rw,relatime)
}}}

Now we can save non-volatile data in /mnt.
