http://phwl.org/2022/qemu-aarch64-debian/


install qemu package with brew

Download debian netinstall ISO

Download initrd and linux image

다음 파일들을 받는다

```
$ qemu-img create -f qcow2 debian-3607-aarch64.qcow2 32G 
$ wget http://ftp.au.debian.org/debian/dists/bullseye/main/installer-arm64/current/images/netboot/debian-installer/arm64/initrd.gz
$ wget http://ftp.au.debian.org/debian/dists/bullseye/main/installer-arm64/current/images/netboot/debian-installer/arm64/linux
```

install debian

다음과 같이 데비안을 설치한다

```
qemu-system-aarch64 \
    -accel hvf \
    -M virt,highmem=on \
    -cpu host \
    -smp 4 \
    -m 4096 \
    -net nic -net user \
    -kernel ./linux -initrd ./initrd.gz -append "console=ttyAMA0" \
    -hda ./disk.qcow2 \
    -drive file=./debian-11.5.0-arm64-netinst.iso,id=cdrom,if=none,media=cdrom \
    -device virtio-scsi-device -device scsi-cd,drive=cdrom -nographic
```

MOVE to another linux machine because MACOS does not have libguestfs-tools.
And extract kernel and initrd images from the disk image.

데비안이 설치된 qcow2 파일에서 커널과 initrd를 복사한다. 이 명령은 맥에서는 실행안되고 리눅스 환경에서만 실행된다.

```
$ sudo apt install libguestfs-tools
$ sudo virt-ls -a debian-3607-aarch64.qcow2 /boot/
System.map-5.10.0-10-arm64
System.map-5.10.0-11-arm64
config-5.10.0-10-arm64
config-5.10.0-11-arm64
initrd.img
initrd.img-5.10.0-10-arm64
initrd.img-5.10.0-11-arm64
initrd.img.old
lost+found
vmlinuz
vmlinuz-5.10.0-10-arm64
vmlinuz-5.10.0-11-arm64
vmlinuz.old
$ sudo virt-copy-out -a debian-3607-aarch64.qcow2 /boot/vmlinuz-5.10.0-11-arm64 /boot/initrd.img-5.10.0-11-arm64 .
```

COME back to MACOS.
Boot the VM.
* I don't know why the kernel and initrd images are specified for qemu parameters.
* I guess that's because I did not install the GUI. It requires the kernel and initrd to use terminal interface.
* Use VNCserver or Xserver for GUI

가져온 커널/initrd로 데비안 부팅한다.

```
$ qemu-system-aarch64 \
    -accel hvf \
    -M virt,highmem=on \
    -cpu host \
    -smp 4 \
    -m 4096 \
    -net nic -net user,hostfwd=tcp::10022-:22 \
    -kernel ./vmlinuz-5.10.0-18-arm64 -initrd ./initrd.img-5.10.0-18-arm64 \
    -append "root=/dev/vda2 console=ttyAMA0" \
    -drive if=virtio,file=./disk.qcow2,format=qcow2,id=hd \
    -device intel-hda -device hda-duplex -nographic
```

Log in
```
$ ssh gurugio@localhost -p 10022
```

Stop the VM
* Ctrl + c, a: get back to Qemu monitor
* (qemu) q: stop VM

Upgrade kernel
```
$ scp -P 10022 gurugio@localhost:/boot/vmlinux-<NEW> gurugio@localhost:/boot/initrd.img-<NEW> .
boot with new kernel and initrd images
```
