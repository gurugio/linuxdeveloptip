(current format is moniwiki)

run my custom kernel with qemu


= install ubuntu on qemu =

refer: http://www.ubuntugeek.com/install-qemu-on-ubuntu-14-10-and-mange-qemu-with-virt-manager.html 


create vm disk

{{{
$ qemu-img create ubuntu11-server.img 128G
}}}


Size should be at least 64GB to build kernel inside of qemu.


install ubuntu in qemu

{{{
$ qemu-system-x86_64 -smp 4 -cpu host -m 1024 -enable-kvm -cdrom ubuntu-11.04-server-i386.iso ubuntu11-server.img -boot d
}}}

이 명령으로 디스크 이미지 ubuntu11-server.img에 우분투를 설치한다.

옵션 설명

{{{
-smp 4: 4코어 실행
-cpu host: 호스트의 cpu를 그대로 사용해서 성능이 최대한 나오게함
-m 1024: 메모리 크기 지정을 안하면 부팅 후 패닉 발생
-enable-kvm: 이 옵션이 없으면 소프트웨어 가상화를 해서 엄청 느려짐 꼭 필요함
}}}

{{{
qemu-system-x86_64 -enable-kvm -smp 4 -m 1024 -drive file=ubuntu.img,if=virtio,cache=none -redir tcp:7777::22 
}}}


run ubuntu on terminal:
{{{
qemu-system-x86_64 -enable-kvm -smp 4 -m 1024 -drive file=ubuntu.img,if=virtio,cache=none -nographic -redir tcp:7777::22 
}}}

connect target via ssh:
{{{
ssh -p 7777 localhost
}}}

= 타겟에서 커널 빌드 =

타겟에서도 커널을 빌드해야한다.
정확히 말하면 커널이 아니라 드라이버를 빌드해서 /lib/modules에 드라이버들이 있어야 호스트에 있는 커널로 부팅했을 때 initrd가 드라이버를 로딩해서 부팅이 완료될 수 있다.
만약에 드라이버가 없으면 initrd가 rootfs를 못읽는 등 부팅이 완료되지 않는다.

타겟에서 호스트와 동일한 커널 버전을 받아서 동일한 옵션으로 빌드하고 make modules_install을 실행한다.

{{{
git에서 linux-next 받기 https://github.com/torvalds/linux.git
cp /boot/config~~ .config OR make oldconfig
make localmodconfig
make menuconfig -> General setup -> Local version 에 새로운 이름을 넣어서 기존 커널과 구분함
make all -j8
sudo make modules_install 드라이버를 /lib/modules에 복사해놓아야 호스트에서 v4.0으로 부팅했을 때 드라이버가 로드됨
}}}

 * cp /boot/config~~ .config; make oldconfig: adjust current kernel-config into my kernel
 * make localmodconfig: activate only modules that are loaded currently


= 호스트에서 커널 빌드 =

호스트에서 타겟에서 빌드한 것과 동일한 버전의 커널을 빌드한다.

'''주의할 것:'''
 * config는 타겟에서 빌드한 config를 복사해서 사용한다.
 * 타겟에서 빌드한 config를 .config로 복사한 후 make kvmconfig를 실행해서 qemu 실행에 필요한 옵션들을 추가한다.
 * CONFIG_DEVTMPFS, CONFIG_DEVTMPFS_MOUNT 옵션을 확인해서 없으면 켜줘야 /dev 디렉토리가 생성된다.
'''안되던 커널 이미지도 make kvmconfig해서 DEVTMPFS옵션을 켜주면 부팅되기도한다. 꼭 실행할 것'''

빌드 방법도 동일하다.

빌드한 커널을 테스트하는 방법은

{{{
qemu-system-x86_64 -smp 4 -cpu host -m 1024 -enable-kvm -vga qxl -kernel vmlinuz~~ -initrd initrd~~ -append root=UUID=~~~ -drive file=ubuntu.img,if=virtio,cache=none -redir tcp:7777::22 
}}}

initrd는 새로 빌드한 버전으로 할 필요는 없을 수도 있다. 먼저 기본으로 깔려있는걸 해보고 안되면 새로 빌드한 걸로 해본다.


다음은 커널 테스트에 필여한 옵션

{{{
-kernel: 커널 이미지 bzImage도 됨
-initrd: 타겟의 /boot에 이미 있는 initrd를 써도 되고, 타겟에서 make install 해서 생성된 initrd 써도 되고. 반드시 타겟에 있는 initrd를 쓸 것. 호스트에 있는걸 쓰면 rootfs를 못읽기도 함
-append: root=UUID=~~~ 꼭 써줘야함 이게 안맞으면 initrom만 부팅됨. 보통 cat /proc/cmdline 으로 root=값이 뭔지 알아내면 된다. LVM으로 설치하면 /dev/mapper/~~같이 되는데 상관없이 똑같이 써주면 됨
ubuntu.img: 최초로 우분투를 설치한 이미지
}}}

= ubuntu 환경 설정 =

== ttyS0 실행 ==

타겟에 ttyS0을 활성화해야 qemu에서 그래픽을 실행하지않고 터미널로만 실행할 수 있다. ttyS0이 없으면 부팅 중간에 멈춘것 처럼 보이게 된다. 사실은 부팅 중간에 멈춘게 아니라 ttyS0이 없어서 로그인 프롬프트가 출력이 안되는 것이다.

타겟에서 아래와 같이 ttyS0.conf를 생성하면됨
{{{
$ cat /etc/init/ttyS0.conf 
# ttyS0 - getty
#
# This service maintains a getty on ttyS0 from the point the system is
# started until it is shut down again.

start on stopped rc or RUNLEVEL=[12345]
stop on runlevel [!12345]

respawn
exec /sbin/getty -L 115200 ttyS0 vt102
}}}


ttyS0.conf를 만들고 이렇게 실행하면 커널 부팅 과정은 안나오고 로그인 프롬프트만 출력됨
{{{
qemu-system-x86_64 -smp 8 -cpu host -m 8092 -enable-kvm -drive \
file=disk_boot.img,if=virtio,cache=none,format=raw \
-nographic -redir tcp:7777::22
}}}


커널 부팅 메세지로 출력하려면 GRUB_CMDLINE_LINUX_DEFAULT에 console=ttyS0을 추가해야함
{{{
gurugio@gioh-pserver:~$ cat /etc/default/grub 
...
GRUB_CMDLINE_LINUX_DEFAULT="console=ttyS0"
...
}}}

sudo update-grub 실행해야 grub 메뉴에서 커널 옵션이 바뀜
{{{
$ vi /boot/grub/grub.cfg
...

        linux   /boot/vmlinuz-3.19.0-25-generic root=UUID=73bd8c0d-8e16-4f9b-8277
0-0769249d30a8 ro  console=ttyS0 nomdmonddf nomdmonisw crashkernel=384M-:128M
}}}
= 우분투를 터미널로 실행해서 보드에 터미널로 접속한 것과 동일한 환경으로 실행하기 =

최종 옵션 정리: ubuntu-server를 설치했으면 굳이 GUI창으로 우분투를 실행할 필요가 없이 터미널로만 실행하면 된다.

터미널에서 실행
{{{
qemu-system-x86_64 -smp 4 -cpu host -m 1024 -enable-kvm \
-kernel arch/x86/boot/bzImage \
-initrd ./initrd.img-4.0.0 \
-append "root=/dev/mapper/userver--vg-root ro \
earlyprintk console=ttyS0" \
-drive file=~/qemu/ubuntu-server.img,if=virtio,cache=none \
-redir tcp:7777::22 -device virtio-balloon -nographic
}}}

개별 윈도우 창으로 실행
{{{
qemu-system-x86_64 -smp 4 -cpu host -m 1024 -enable-kvm \
-kernel arch/x86/boot/bzImage \
-initrd ./initrd.img-4.0.0 \
-append "root=/dev/mapper/userver--vg-root ro \
earlyprintk console=ttyS0" \
-drive file=~/qemu/ubuntu-server.img,if=virtio,cache=none \
-redir tcp:7777::22 -device virtio-balloon -graphic qxl -stdio
}}}


{{{
if=virtio,cache=none
디스크를 읽을 때 io가상화를 사용하고 디스크 캐시를 사용하지 않는다.

-redir tcp:7777::22
타겟에 ssh서버가 깔려있다면 타겟의 22포트와 호스트의 7777 포트를 연결한다. ssh -p 7777 localhost 명령으로 타겟에 접속할 수 있다.
타겟에 드라이버가 잘 설치되서 타겟에서 ifconfig를 실행했을 때 eth0 인터페이스가 있어야 접속이 된다.

console=ttyS0
현재 터미널에 커널 로그부터 출력하고 쉘까지 실행한다. 터미널 접속과 동일하다.

-nographic
터미널로 실행하도록 한다.

-vga qxl
비디오 카드 선택 qxl이나 std 등이 있는데 해보고 괸찬은 걸로 고르면 된다.
}}}

'''qemu 모니터를 실행하려면 ctrl-c a 를 입력한다. 모니터에서 다시 타겟으로 돌아가는 것도 ctrl-c a 이다'''

= balloon 드라이버 사용 =

게스트 커널 옵션에서 virtio, balloon 관련한 옵션 CONFIG_VIRTIO_BALLOON 등등을 다 켜고 qemu 옵션에 -balloon virtio을 추가하면 balloon 드라이버가 로딩된다.

ctrl-alt-2를 누르면 qemu 모니터로 진입한다. 모니터에서 balloon 1024 명령은 게스트의 메모리 크기를 1024로 바꾸는 것이다.

qemu> balloon 1024
qemu> balloon 512

1024로 설정한 후 커널 빌드를 실행하거나 해서 단편화를 만들고 512로 설정하면 512M의 페이지를 줄이면서 게스트의 커널이 페이지 migration을 실행하는데 이때 balloon migration이 발생한다.


sudo echo 3 > /proc/sys/vm/drop_caches는 permission denied가 발생한다.
이렇게 할것
echo 1 | sudo tee /proc/sys/vm/drop_caches

= example =

옵션 하나씩 확인할 것
{{{
kvm -name Serverwittchene3c1782b-83a3-4669-8312-861061f321d7 -m 1024 -dimms pfx=p1024,size=1024M,num=255 -dimmpop pfx=p1024,num=3 -M pc-1.0 -enable-kvm -nodefconfig -nodefaults -rtc base=utc -netdev tap,ifname=n0201c3f00f17,id=hostnet6,vhost=on,vhostforce=on,vnet_hdr=off,script=no,downscript=no -device virtio-net-pci,netdev=hostnet6,id=net6,mac=02:01:c3:f0:0f:17,bus=pci.0,addr=0x6 -chardev pty,id=charserial0 -device isa-serial,chardev=charserial0,id=serial0 -usb -device usb-tablet,id=input0 -vnc 0.0.0.0:36 -vga std -device virtio-balloon-pci,id=balloon0,bus=pci.0,addr=0x3 -cpu Opteron_G4,+cmp_legacy,+mmxext,+nrip_save,+osvw,+vme,+npt -smp 2,sockets=32,cores=1,maxcpus=64,threads=1 -drive file=/dev/md20,if=none,id=drive-virtio-disk5,format=raw,cache=none -device virtio-blk-pci,bus=pci.0,addr=0x5,drive=drive-virtio-disk5,id=virtio-disk5,bootindex=1 -drive file=/dev/md25,if=none,id=drive-virtio-disk7,format=raw,cache=none -device virtio-blk-pci,bus=pci.0,addr=0x7,drive=drive-virtio-disk7,id=virtio-disk7 -drive file=/dev/md43,if=none,id=drive-virtio-disk8,format=raw,cache=none -device virtio-blk-pci,bus=pci.0,addr=0x8,drive=drive-virtio-disk8,id=virtio-disk8 -drive file=/dev/md48,if=none,id=drive-virtio-disk9,format=raw,cache=none -device virtio-blk-pci,bus=pci.0,addr=0x9,drive=drive-virtio-disk9,id=virtio-disk9 -drive file=/dev/md49,if=none,id=drive-virtio-disk10,format=raw,cache=none -device virtio-blk-pci,bus=pci.0,addr=0xa,drive=drive-virtio-disk10,id=virtio-disk10 -drive file=/dev/md50,if=none,id=drive-virtio-disk11,format=raw,cache=none -device virtio-blk-pci,bus=pci.0,addr=0xb,drive=drive-virtio-disk11,id=virtio-disk11 -drive file=/dev/md51,if=none,id=drive-virtio-disk12,format=raw,cache=none -device virtio-blk-pci,bus=pci.0,addr=0xc,drive=drive-virtio-disk12,id=virtio-disk12 -S -qmp unix:/opt/profitbricks/vcb/pbkvm/mon/e3c1782b-83a3-4669-8312-861061f321d7.sock,server,nowait -uuid e3c1782b-83a3-4669-8312-861061f321d7
}}}


= networking =

 * http://www.linux-kvm.org/page/Networking
 * http://anddev.tistory.com/108

여러개의 VM을 실행하고 각각 다른 ip를 할당받도록 하기

VM외부에서 VM으로도 접속이 가능해지고 VM끼리도 통신이 가능해짐

== bridge 네트워크 설정 ==

bridge-utils 설치
{{{
$ apt-get install bridge-utils
}}}

/etc/network/interface 파일에서 기존의 eth0나 eno1 등을 지우고 br0을 추가
{{{
gohkim@ws00837:~/hdd/qemu_linux_pserver$ cat /etc/network/interfaces
# interfaces(5) file used by ifup(8) and ifdown(8)
auto lo
iface lo inet loopback
#auto eno1
#iface eno1 inet manual
# for VMs
auto br0
iface br0 inet dhcp
 bridge_ports eno1
 bridge_stp off
 bridge_maxwait 0
 bridge_fd 0 
}}}

br0 활성화: networking 재시작이 오래 걸릴 수 있음
{{{
$ brctl addbr br0
$ /etc/init.d/networking restart
}}}

== qemu 설정 ==

qemu에서 사용할 /etc/qemu-ifup-br 파일 만들기
{{{
#!/bin/sh

set -x

switch=br0

if [ -n "$1" ];then
        /usr/bin/sudo /usr/sbin/tunctl -u `whoami` -t $1
        /usr/bin/sudo /sbin/ip link set $1 up
        sleep 0.5s
        /usr/bin/sudo /sbin/brctl addif $switch $1
        exit 0
else
        echo "Error: no interface specified"
        exit 1
fi
}}}
{{{
$ chmod +x /etc/network/interfaces
}}}


qemu에 옵션 추가
 * -device e1000,netdev=net0,mac=$mac: mac 주소 반드시 지정해야 dhcp서버에서 VM마다 서로 다른 ip 받아옴
 * mac 주소가 같으면 동일한 IP를 또 받아옴
 * -netdev tap,id=net0,script=/etc/qemu-ifup-br
{{{
gohkim@ws00837:~/hdd/qemu_linux_pserver$ cat go_terminal.sh 
#!/bin/bash
mac=$(printf "DE:AD:BE:EF:%02X:%02X" $((RANDOM%256)) $((RANDOM%256)))
/usr/bin/sudo qemu-system-x86_64 -smp 4 -cpu host -m 8092 -enable-kvm \
-drive file=disk_boot.img,if=virtio,cache=none,format=raw \
-device e1000,netdev=net0,mac=$mac \
-netdev tap,id=net0,script=/etc/qemu-ifup-br \
-nographic
}}}
