(current format is moniwiki)


= with qemu =

https://www.kernel.org/doc/Documentation/gdb-kernel-debugging.txt

http://stackoverflow.com/questions/11408041/how-to-debug-the-linux-kernel-with-gdb-and-qemu

http://www.linux-magazine.com/Online/Features/Qemu-and-the-Kernel

http://wiki.osdev.org/Kernel_Debugging

== -gdb tcp::1234 or -s option ==

'''enable kernel option: CONFIG_DEBUG_INFO'''

-gdb tcp::1234 option is equivalent to -s option.

If you use want to use another port you would want to set -gdb tcp::<PORT> option.


run qemu with -s options:
{{{
gohkim@ws00837:~/work/linux-storage$ qemu-system-x86_64 -smp 4 -cpu host -m 2048 \
> -kernel arch/x86/boot/bzImage \
> -enable-kvm -nographic \
> -drive file=~/work/qemu_ubuntu/disk_boot.img,if=virtio,cache=none,format=raw \
> -initrd ~/work/qemu_ubuntu/initrd.img-3.12.45-1-storage+ \
> -append "root=/dev/vda1 ro splash nomodeset earlyprintk console=tty0 console=ttyS0" \
> -redir tcp:7777::22 -s
}}}
The -s option opens 1234 port for gdb connection.

run gdb:
 * gdb vmlinux
 * target remote localhost:1234
 * b rcu_process_callbacks
 * c
 * ctrl + c
{{{
gohkim@ws00837:~/work/linux-storage$ gdb vmlinux
GNU gdb (Ubuntu 7.10-1ubuntu2) 7.10
Copyright (C) 2015 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "x86_64-linux-gnu".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
<http://www.gnu.org/software/gdb/documentation/>.
For help, type "help".
Type "apropos word" to search for commands related to "word"...
Reading symbols from vmlinux...done.
(gdb) target remote localhost:1234
Remote debugging using localhost:1234
native_safe_halt ()
    at /home/gohkim/work/linux-storage/arch/x86/include/asm/irqflags.h:50
50	}
(gdb) c
Continuing.
^C
Program received signal SIGINT, Interrupt.
native_safe_halt ()
    at /home/gohkim/work/linux-storage/arch/x86/include/asm/irqflags.h:50
50	}
(gdb) b rcu_process_callbacks 
Breakpoint 1 at 0xffffffff810c2890: file kernel/rcutree.c, line 2229.
(gdb) c
Continuing.

Breakpoint 1, rcu_process_callbacks (
    unused=0xffffffff81a050c8 <softirq_vec+72>) at kernel/rcutree.c:2229
2229	{
(gdb) l
2224	
2225	/*
2226	 * Do RCU core processing for the current CPU.
2227	 */
2228	static void rcu_process_callbacks(struct softirq_action *unused)
2229	{
2230		struct rcu_state *rsp;
2231	
2232		if (cpu_is_offline(smp_processor_id()))
2233			return;
(gdb) 
}}}

Now the target kernel will run and print messages.


== -S (large S) option ==

The -S makes the qemu stop before running kernel and wait for gdb.
We can debug some functions in early booting process.
But it has a bug: "Remote 'g' packet reply is too long"

IT IS NOT SOLVED IN UBUNTU-15.04 with:

{{{
gohkim@ws00837:~/work/linux-storage$ gdb -v
GNU gdb (Ubuntu 7.10-1ubuntu2) 7.10
Copyright (C) 2015 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "x86_64-linux-gnu".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
<http://www.gnu.org/software/gdb/documentation/>.
For help, type "help".
Type "apropos word" to search for commands related to "word".
gohkim@ws00837:~/work/linux-storage$ qemu-system-x86_64 --version
QEMU emulator version 2.3.0 (Debian 1:2.3+dfsg-5ubuntu9.1), Copyright (c) 2003-2008 Fabrice Bellard
}}}


== python scripts for gdb ==


http://lxr.free-electrons.com/source/Documentation/gdb-kernel-debugging.txt
* enable CONFIG_GDB_SCRIPTS
* disable CONFIG_DEBUG_INFO_REDUCED


from Roman's email:

{{{

3.  Live debugging with the GDB is already a bit of a help, but there
    are some python scripts shipped with a kernel to make live more
    easier.  Located here:

        ./linux/scripts/gdb/

    These are python scripts to ease debugging which are quite well
    described in a kernel documentation:

        ./linux/Documentation/gdb-kernel-debugging.txt

    Even these scripts are well described the following are some tips
    to get immediately started with the kernel we have:

        $ cd linux-storage-future
        $ PYTHONPATH=./scripts/gdb gdb ./vmlinux
        (gdb) source scripts/gdb/vmlinux-gdb.py
        (gdb) target remote :1234

    Now we've attached to a QEMU process and can make some script magic:

        (gdb) lx-dmesg

            Outputs current dmesg buffer directly from the memory.

        (gdb) lx-symbols ./path/to/modules

            Loads symbols from modules it finds in specified path.
            Very convenient, replaces this ugly command:

            (gdb) add-symbol-file module.ko 0xffffffffa002d000
-readnow -s .data 0xffffffffa003ed00 -s .bss

        (gdb) lx-ps

            Iterates over the main task list in the memory and prints
            task_struct entries.

        (gdb) lx-list-check

            Macro which is located here:

                ./scripts/gdb/linux/lists.py

            Macro is not that useful, but can be manually extended to
            traverse all sorts of 'struct list_head' objects.

        (gdb) p (struct thread_info *)$lx_task_by_pid(1)->stack
        $1 = (struct thread_info *) 0xffff880139bc8000

            Returns thread_info.
}}}


= with kdump =

https://www.kernel.org/doc/Documentation/kdump/kdump.txt

https://wiki.ubuntu.com/Kernel/CrashdumpRecipe

http://superuser.com/questions/280767/how-can-i-enable-kernel-crash-dumps-in-debian

The steps are roughly,

{{{
sudo apt-get install kdump-tools  ==> ubuntu: linux-crashdump
Set USE_KDUMP=1 in /etc/defaults/kdump-tools
Add crashkernel=128M to the kernel command-line given in bootloader configuration (e.g. /etc/defaults/grub). It also doesn't hurt to pass nmi_watchdog=1 as well to ensure that hard hangs are caught.
Note that 128MB is merely a ballpark figure. It needs to be large enough to accomodate the kernel image and the associated init ramdisk.
If your initram disk is large, you might be able to shrink it by tweaking /etc/initramfs-tools/initramfs.conf
Ensure that your boot loader configuration is updated (e.g. sudo update-grub)
Ensure your kernel is built with,
CONFIG_RELOCATABLE=y
CONFIG_KEXEC=y
CONFIG_CRASH_DUMP=y
CONFIG_DEBUG_INFO=y
Reboot
Verify that the crash kernel is loaded, cat /sys/kernel/kexec_crash_loaded
Optional: Test that all of this worked,
sudo sync; echo c | sudo tee /proc/sysrq-trigger
Use the crash tool to look at the resulting crash dump
Find a handle of good whiskey to ease the pain of your future in kernel debugging.
}}}


{{{
dmesg | grep -i crash

...
[    0.000000] Reserving 64MB of memory at 800MB for crashkernel (System RAM: 1023MB)
}}}


crash V6.x.x needs to upgrade!: https://www.redhat.com/archives/crash-utility/2012-June/msg00007.html
{{{
root@ib2:~/tmp/linux-pserver# crash vmlinux /var/crash/201512231645/dump.201512231645 

crash 6.0.6
Copyright (C) 2002-2012  Red Hat, Inc.
Copyright (C) 2004, 2005, 2006  IBM Corporation
Copyright (C) 1999-2006  Hewlett-Packard Co
Copyright (C) 2005, 2006  Fujitsu Limited
Copyright (C) 2006, 2007  VA Linux Systems Japan K.K.
Copyright (C) 2005  NEC Corporation
Copyright (C) 1999, 2002, 2007  Silicon Graphics, Inc.
Copyright (C) 1999, 2000, 2001, 2002  Mission Critical Linux, Inc.
This program is free software, covered by the GNU General Public License,
and you are welcome to change it and/or distribute copies of it under
certain conditions.  Enter "help copying" to see the conditions.
This program has absolutely no warranty.  Enter "help warranty" for details.
 
GNU gdb (GDB) 7.3.1
Copyright (C) 2011 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "x86_64-unknown-linux-gnu"...

      KERNEL: vmlinux                           
    DUMPFILE: /var/crash/201512231645/dump.201512231645  [PARTIAL DUMP]
        CPUS: 4
        DATE: Thu Jan  1 01:00:00 1970
      UPTIME: 00:06:46
LOAD AVERAGE: 0.01, 0.29, 0.22
       TASKS: 129
    NODENAME: ib2
     RELEASE: 3.12.45-5-pserver-gioh+
     VERSION: #132 SMP Wed Dec 23 16:09:49 CET 2015
     MACHINE: x86_64  (3010 Mhz)
      MEMORY: 7.7 GB
       PANIC: 
crash: cannot determine length of symbol: log_end
}}}


= download a specific version of kernel =

To download another version of the kernel, not installed in the system.

 1. apt-cache search package-name : find the package name
 1. apt-cache policy package-name : find the version table
 1. apt-get download/install package-name=version : download/install the specific version of the package
 1. dpkg -x package-name dir-path : extract the package into dir

To download linux-source-4.2.0-16.19:
{{{
gohkim@ws00837:~$ apt-cache policy linux-source-4.2.0 
linux-source-4.2.0:
  Installed: 4.2.0-27.32
  Candidate: 4.2.0-27.32
  Version table:
 *** 4.2.0-27.32 0
        500 http://de.archive.ubuntu.com/ubuntu/ wily-updates/main amd64 Packages
        500 http://security.ubuntu.com/ubuntu/ wily-security/main amd64 Packages
        100 /var/lib/dpkg/status
     4.2.0-25.30 0
        500 http://de.archive.ubuntu.com/ubuntu/ wily-updates/main amd64 Packages
        500 http://security.ubuntu.com/ubuntu/ wily-security/main amd64 Packages
     4.2.0-23.28 0
        500 http://de.archive.ubuntu.com/ubuntu/ wily-updates/main amd64 Packages
        500 http://security.ubuntu.com/ubuntu/ wily-security/main amd64 Packages
     4.2.0-22.27 0
        500 http://de.archive.ubuntu.com/ubuntu/ wily-updates/main amd64 Packages
        500 http://security.ubuntu.com/ubuntu/ wily-security/main amd64 Packages
     4.2.0-21.25 0
        500 http://de.archive.ubuntu.com/ubuntu/ wily-updates/main amd64 Packages
        500 http://security.ubuntu.com/ubuntu/ wily-security/main amd64 Packages
     4.2.0-19.23 0
        500 http://de.archive.ubuntu.com/ubuntu/ wily-updates/main amd64 Packages
        500 http://security.ubuntu.com/ubuntu/ wily-security/main amd64 Packages
     4.2.0-18.22 0
        500 http://de.archive.ubuntu.com/ubuntu/ wily-updates/main amd64 Packages
        500 http://security.ubuntu.com/ubuntu/ wily-security/main amd64 Packages
     4.2.0-17.21 0
        500 http://de.archive.ubuntu.com/ubuntu/ wily-updates/main amd64 Packages
     4.2.0-16.19 0
        500 http://de.archive.ubuntu.com/ubuntu/ wily/main amd64 Packages
gohkim@ws00837:~$ apt-get download linux-source-4.2.0=4.2.0-16.19
Get:1 http://de.archive.ubuntu.com/ubuntu/ wily/main linux-source-4.2.0 all 4.2.0-16.19 [108 MB]
19% [1 linux-source-4.2.0 20,4 MB/108 MB 19%]                    1.858 kB/s 46s^C

}}}

= crash tool =

 * bt
  * bt -c 17: only task at cpu-17
  * bt -c 16-31: cpu 16 ~ 31
 * foreach bt: backtrace of all processes
 * log: kernel log
 * ps
 * disassemble <kernel-function>
  * disassemble /r <kernel-function>: print opcode and instruction
 * rd <address> <count>: read memory from address
 * gdb list *(memcpy+16): find c-code line
 * mod: print modules
  * no information for module loaded: ffffffffa0316fa0  kvm                   342174  (not loaded)  [CONFIG_KALLSYMS]
  * mod -S: find modules <path-of-vmlinux>/lib/modules/<version>/<modules-path>
  * There not only vmlinux but also modules should be copied


= debugging case =

 * qemu cannot support ACPI -> cannot debug acpi_idle, frequency setting
