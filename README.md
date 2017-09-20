# Tips for Linux kernel & driver development

How to use iostat
* https://bartsjerps.wordpress.com/2011/03/04/io-bottleneck-linux/
* host wait time = storage wait * queue-size
```
$ iostat -xk 2 /dev/sd[bc]
```

Check processor, vsr, rss and pid with ps command
```
gohkim@ws00837:~$ ps -ao pid,psr,user,vsz,rss,comm,args
  PID PSR USER        VSZ   RSS COMMAND         COMMAND
 9734   7 gohkim    42648  3724 ps              ps -ao pid,psr,user,vsz,rss,comm,args
 ```
 
Summation of integer values from file which has multi lines of integer.
* $ awk '{s+=$1} END {print s}' < file.text
```
sudo cat /proc/$i/smaps | grep Anon | awk '{ print $2 }' | awk '{s+=$1} END {print s}'
```

emacs: replace something into the newline
* ; -> newline: M-x replace-string RET ; RET C-q C-j
* =y -> newline bool newline default y: alt+shift+5, =y, C-q C-j bool C-q C-j default y C-q C-j
```
config COMPAT_IS___SKB_TX_HASH=y

config COMPAT_IS___SKB_TX_HASH
	bool
	default y
```


git sub-option for --pretty:
* format:"%s" print only subject
```
git log --pretty=format:"%s" ...mdadm-3.3.2-5
```

kernel build failed with "error: code model kernel does not support PIC mode"
```
diff --git a/Makefile b/Makefile
index 0f9cb36d45c2c..b95a6774e4600 100644
--- a/Makefile
+++ b/Makefile
@@ -341,8 +341,8 @@ include scripts/Kbuild.include
 
 # Make variables (CC, etc...)
 AS             = $(CROSS_COMPILE)as
-LD             = $(CROSS_COMPILE)ld
-CC             = $(CROSS_COMPILE)gcc
+LD             = $(CROSS_COMPILE)ld -no-pie
+CC             = $(CROSS_COMPILE)gcc -no-pie
 CPP            = $(CC) -E
 AR             = $(CROSS_COMPILE)ar
 NM             = $(CROSS_COMPILE)nm
```
 
check source difference between two branches
* ``git diff -w branchAAA branchBBB path``
* ``git diff -w origin/master feature/fix-lock drivers/scsi``
* use -w to ignore spaces
* ``git diff commit1234 commit5677 [path]``

resolve failure of 'git am'
* ``git am --reject patch.patch``
* source.c.rej is generated.
* resolve only source.c file

find kernel config option at once
```
make menuconfig
press '/' to search the option
press number of the option
(1)   -> Compile-time checks and compiler options  
 (2) -> Kernel hacking  
```

build .deb package of kernel
```
make -j `getconf _NPROCESSORS_ONLN` deb-pkg LOCALVERSION=-custom
cd ..
sudo dpkg -i linux-firmware-image-4.11.1-custom_4.11.1-custom-1_amd64.deb
sudo dpkg -i linux-libc-dev_4.11.1-custom-1_amd64.deb
sudo dpkg -i linux-headers-4.11.1-custom_4.11.1-custom-1_amd64.deb
sudo dpkg -i linux-image-4.11.1-custom-dbg_4.11.1-custom-1_amd64.deb
sudo dpkg -i linux-image-4.11.1-custom_4.11.1-custom-1_amd64.deb
```

delay function for C language
* not sleep, no context switch
* there is no delay in glibc, but it has CLOCKS_PER_SEC and time() function.
```
#include <time.h>

void delay(int milliseconds)
{
        long pause;
	clock_t now,then;

	pause = milliseconds*(CLOCKS_PER_SEC/1000);
	now = then = clock();
	while( (now-then) < pause )
		now = clock();
}
```

tip for strace: ALWAYS USE -f option
* -f: trace all children
* -f -p <PID>: trace all threads of <PID>
```
# strace -p <PID> -f
```

Kernel compile error with GCC-6.x: "code model kernel does not support PIC mode"
```
diff --git a/Makefile b/Makefile
index dda982c..f96b174 100644
--- a/Makefile
+++ b/Makefile
@@ -608,6 +608,12 @@ endif # $(dot-config)
 # Defaults to vmlinux, but the arch makefile usually adds further targets
 all: vmlinux
 
+# force no-pie for distro compilers that enable pie by default
+KBUILD_CFLAGS += $(call cc-option, -fno-pie)
+KBUILD_CFLAGS += $(call cc-option, -no-pie)
+KBUILD_AFLAGS += $(call cc-option, -fno-pie)
+KBUILD_CPPFLAGS += $(call cc-option, -fno-pie)
+
 # The arch Makefile can set ARCH_{CPP,A,C}FLAGS to override the default
 # values of the respective KBUILD_* variables
 ARCH_CPPFLAGS :=
 ```

Keep ssh connection from being frozen: add following lines in /etc/ssh/sshd_config
```
--------- client side ----------
Host *
ServerAliveInterval 100

-------- server side ------------
ClientAliveInterval 60
TCPKeepAlive yes
ClientAliveCountMax 10000
```

Including kernel header directly in application cannot succeed. Two ways to install kernel headers into /usr/src/linux directory
* install linux-libc-dev package for normal distribution kernel
* run ``make headers_install`` for custom kernel
```
gkim@ib1:~/linux-pserver-future$ gcc -Wp,-MD,samples/bpf/.test_verifier.o.d -Wall -Wmissing-prototypes -Wstrict-prototypes -O2 -fomit-frame-pointer -std=gnu89 -I./usr/include -I./include/uapi    -c -o samples/bpf/test_verifier.o samples/bpf/test_verifier.c
In file included from ./include/uapi/linux/bpf.h:10:0,
                 from samples/bpf/test_verifier.c:12:
./include/uapi/linux/types.h:9:2: warning: #warning "Attempt to use kernel headers from user space, see http://kernelnewbies.org/KernelHeaders" [-Wcpp]
 #warning "Attempt to use kernel headers from user space, see http://kernelnewbies.org/KernelHeaders"
  ^
In file included from ./include/uapi/linux/posix_types.h:4:0,
                 from ./include/uapi/linux/types.h:13,
                 from ./include/uapi/linux/bpf.h:10,
                 from samples/bpf/test_verifier.c:12:
./include/uapi/linux/stddef.h:1:28: fatal error: linux/compiler.h: No such file or directory
 #include <linux/compiler.h>
                            ^
compilation terminated.
```

Create screen session and run a command in the screen without attaching the screen
* "-d -m": Start screen in detached mode. 
* "-S sessionname": set session name as sessionname
```
# screen -dmS ddd stress --cpu 2 --io 2 --hdd 2 --vm 2
# sshpass -p passwd ssh root@1.2.3.4 'screen -dmS ddd stress --cpu 2 --io 2 --hdd 2 --vm 2'
```

strip modules when linux kernel installs modules
```
# make INSTALL_MOD_STRIP=1 modules_install
```

checkpatch.pl usage
* show type of issue: --show-types
* show only specified types: --types
```
$ ./scripts/checkpatch.pl --file --show-types drivers/staging/greybus/*.c
----------------------------------------
drivers/staging/greybus/arche-apb-ctrl.c
----------------------------------------
CHECK:LINE_SPACING: Please don't use multiple blank lines
#24: FILE: drivers/staging/greybus/arche-apb-ctrl.c:24:
+
+

CHECK:PARENTHESIS_ALIGNMENT: Alignment should match open parenthesis
#74: FILE: drivers/staging/greybus/arche-apb-ctrl.c:74:
+	if (apb->init_disabled ||
+			apb->state == ARCHE_PLATFORM_STATE_ACTIVE)

$./scripts/checkpatch.pl --file --types="LONG_LINE" drivers/staging/greybus/*.c | less
```

change keyboard layout for console
```
try...
# sudo dpkg-reconfigure console-setup
# dpkg-reconfigure keyboard-configuration
# reboot
```

checkout a remote branch
```
git fetch
git checkout test
```
But there are multiple remote, it doesn't download the branch, then:
```
git checkout -b test <name of remote>/test
```

check the coding style of external driver source file with checkpatch.pl
* --file: check regular file, not patch
* --no-tree: no kernel source tree
* --help prints other options
```
gohkim@ws00837:~/study/little_challenge/task05$ ~/work/linux-torvalds/scripts/checkpatch.pl --file --no-tree drv.c
WARNING: please, no spaces at the start of a line
#25: FILE: drv.c:25:
+    printk(KERN_EMERG "usb_task05_probe\n");$
...
```

build a debian package from source
```
$ dpkb-buildpackage -d
-d: 의존성/충돌 체크 안함
-b: 소스 체크 안함 
```

change console keyboard layout: ``sudo apt-get install console-common``

git command for prerry: ``git log --pretty=format:"%h %ad%x09%s"``

get guid of dm-13: ``pbkvm dm2guid dm-13``

fio: generate IOs with various types, size, interval and so on
```
[global]
description=Emulation of Storage Server Access Pattern
bssplit=512/20:1k/16:2k/9:4k/12:8k/19:16k/10:32k/8:64k/4
fadvise_hint=0
#rw=randrw:2
rw=write
direct=1

ioengine=libaio
iodepth=64
iodepth_batch_submit=64
iodepth_batch_complete=64
numjobs=4
gtod_reduce=1
group_reporting=1

time_based=1
runtime=30

[job]
filename=./test1
```

network stress test
```
netperf -H 10.66.83.169 -l 600
iperf with multithreads
iperf -c <IP> -P 16 -t 3600
```

slub debugging detail: https://www.kernel.org/doc/Documentation/vm/slub.txt

SCST: http://scst.sourceforge.net/

iostat
* https://www.kernel.org/doc/Documentation/block/stat.txt
* https://www.kernel.org/doc/Documentation/iostats.txt
* ``dstat -df``: check amount of read/write for block device at each second

pkill: kill process with name
* ``sleep $((60*10)) & pkill mprime`` -> kill mprime after 60-min

pgrep: list processes of specific user
* ``pgrep -u gohkim -l`` -> print pid of gohkim as weel as command

flush buffer of disks: ``blockdev --flushbufs``

perf report cannot print the kernel symbols even-if the kernel is built with CONFIG_KALLSYMS and CONFIG_KALLSYMS_ALL
```
gohkim@ws00837:~/hdd/intel-iommu-perf$ ./perf report -k /proc/kallsyms 
Warning:
Kernel address maps (/proc/{kallsyms,modules}) were restricted.

Check /proc/sys/kernel/kptr_restrict before running 'perf record'.

gohkim@ws00837:~/hdd/intel-iommu-perf$ cat /proc/sys/kernel/kptr_restrict 
1

root@ws00837:/usr/src/linux-source-4.2.0/linux-source-4.2.0# echo 0 > /proc/sys/kernel/kptr_restrict 

gohkim@ws00837:~/hdd/intel-iommu-perf$ cat /proc/sys/kernel/kptr_restrict 
0
```

How to develop against linux-next: https://lwn.net/Articles/289245/

check the event-log of Megaraid device
* ``$ sudo storcli /c0 show events``

run shell command via ssh
* ``$ ssh systemsboy@rhost.systemsboy.edu 'ls -l; ps -aux; whoami'``

gvncviewer
* port: ``ps aux | grep kvm | grep <job-name>``  ===> -vnc 0.0.0.0:50  ===> 50 is port number
* ``$ gvncviewer <pserver-ip>:<port>``

linux-next 업데이트
```
$git remote update
$git reset --hard origin/master
```

패치 파일 생성
* ``git format-patch -3 -s --cover-letter --subject-prefix=PATCHv2``
* -3: 가장 최신 커밋부터 3개까지
* --cover-letter옵션: 패치 파일 통계 등 자동 생성됨
* --subject-prefix: 제목에 PATCH대신 들어갈 말머리
* -s: signed off by를 자동으로 추가)

패치파일 전송
* -–thread --no-chain-reply-to 0000-cover-letter를 가장 먼저 전송하고 reply로 0001~0002로 전송함
```
git send-email --to=gioh.kim@lge.com --thread --no-chain-reply-to 0000-cover-letter.patch 0001-patch-name.patch 0002-2nd.patch
```
여러 patch 파일들의 통계 : diffstat 패치파일이름

```
sudo echo 3 > /proc/sys/vm/drop_caches는 permission denied가 발생한다.
이렇게 할것
echo 1 | sudo tee /proc/sys/vm/drop_caches
```

defconfig 파일 만드는 방법: make savedefconfig

Howto cherry-pick patch from mainline kernel:
```
$ git remote add mainline git://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git
$ git fetch mainline
```
Then cherry pick the commits you need, which for this Haswell HDMI audio bug meant these three commits (as best I can tell):
```
$ git cherry-pick 9419ab6b72325e20789a61004cf68dc9e909a009
$ git cherry-pick c88d4e84e639df9a9640ecff71de2501a84d1f48
$ git cherry-pick 17df3f55652f7ea8fb1197b5c32e227b3da9f215
```

브랜치를 만들었는데 remote에 있는 브랜치랑 연결이 안될때
```
git branch --set-upstream-to=origin/feature/jessie-builds-3.3.2-5pb6-storage  feature/jessie-builds-3.3.2-5pb6-storage
```

awk pattern maching example
* ``'/<pattern>/{<action>}'``
```
gohkim@ws00837:~$ cat /proc/meminfo | awk '/MemFree/{print $0}'
MemFree:         1458512 kB
gohkim@ws00837:~$ cat /proc/meminfo | awk '/MemFree/{print $1}'
MemFree:
gohkim@ws00837:~$ cat /proc/meminfo | awk '/MemFree/{print $2}'
1458528
gohkim@ws00837:~$ cat /proc/meminfo | awk '/MemFree/{printf "%d\n", $2 * 0.9 / 64 ;}'
27592
```

gitconfig: pretty log printing
```
gohkim@ws00837:~/work/linux-stable$ cat ~/.gitconfig
...

[alias]
    lg1 = log --graph --abbrev-commit --decorate --format=format:'%C(bold blue)%h%C(reset) - %C(bold green)(%ar)%C(reset) %C(white)%s%C(reset) %C(dim white)- %an%C(reset)%C(bold yellow)%d%C(reset)'
    lg2 = log --graph --abbrev-commit --decorate --format=format:'%C(bold blue)%h%C(reset) - %C(bold cyan)%aD%C(reset) %C(bold green)(%ar)%C(reset)%C(bold yellow)%d%C(reset)%n''          %C(white)%s%C(reset) %C(dim white)- %an%C(reset)' --all
    lg = !"git lg1"
```

debug udevd: show what rules are applied
```
sudo pkill udevd
sudo udevd --debug-trace --verbose --suppress-syslog
```
