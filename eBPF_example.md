
*This is based on v4.4. The newer version should be different and have less restriction.*

# Preparation
* kernel versions of source tree and system MUST be same
  * fail to load kernel program
* install libelf-dev libssl-dev
* install kernel headers into /usr/include/linux: ``make headers_install``
  * ``samples/bpf/test_verifier.c:12:23: fatal error: linux/bpf.h: No such file or directory``
  * make a usr/include directory inside of kernel source tree and install user header files into it
* clang >= v3.4.0 and llam >= v3.7.1
  * download llvm package from llvm.org, no need to install
* ulimit -l 10240 (Debian)
  * For Ubuntu, fix /etc/security/limits.conf
* mount kernel debugfs: ``mount -t debugfs none /sys/kernel/debug/``


## How to increase rlimit

**If rlimit is too low, loading program or creating map fails without any error message (kernel v4.4).**

### max open files

If there is an error like ``failed to create a map: 1 Operation not permitted``, we should increase "maximum number of open file" like following.

1. /etc/security/limits.conf 
```
vi /etc/security/limits.conf 
*               soft    nofile            10240
*               hard    nofile            10240
```
2. add one line into /etc/pam.d/common-session and /etc/pam.d/common-session-noninteractive 
```
session required pam_limits.so
```
3. reboot
```
gurugio@giohnote:~/kernel/linux-source-4.4.0$ ulimit -n
10240
```

### max locked memory

Kernel uses locked memory to create map.
So we should increase max size of locked memory.

1. add /etc/security/limits.conf
```
*               soft    memlock           unlimited
*               hard    memlock           unlimited
```

2. or use ulimit command
```
## Show the current Hard limit for "memlock"
$ ulimit -H -l
64

## Show the current Soft limit for "memlock"
$ ulimit -S -l
64

## Set the current Soft "memlock" limit to 48KiB
$ ulimit -H -l 1024
$ ulimit -S -l 1024
```

How to check new setup.
```
root@pserver:~/hdd/linux-pserver-future# ulimit -a
core file size          (blocks, -c) 0
data seg size           (kbytes, -d) unlimited
scheduling priority             (-e) 0
file size               (blocks, -f) unlimited
pending signals                 (-i) 3948
max locked memory       (kbytes, -l) 1024
max memory size         (kbytes, -m) unlimited
open files                      (-n) 65536
pipe size            (512 bytes, -p) 8
POSIX message queues     (bytes, -q) 819200
real-time priority              (-r) 0
stack size              (kbytes, -s) 8192
cpu time               (seconds, -t) unlimited
max user processes              (-u) 3948
virtual memory          (kbytes, -v) unlimited
file locks                      (-x) unlimited
```


# build

Do ``make samples/bpf/``

Fix Makefile of v4.4
* change the path of llc in Makefile
* clang should be in $PATH
```
diff --git a/samples/bpf/Makefile b/samples/bpf/Makefile
index edd638b..cabdb29 100644
--- a/samples/bpf/Makefile
+++ b/samples/bpf/Makefile
@@ -65,15 +65,16 @@ HOSTLOADLIBES_trace_output += -lelf -lrt
 HOSTLOADLIBES_lathist += -lelf
 
 # point this to your LLVM backend with bpf support
-LLC=$(srctree)/tools/bpf/llvm/bld/Debug+Asserts/bin/llc
+#LLC=$(srctree)/tools/bpf/llvm/bld/Debug+Asserts/bin/llc
+LLC=/home/gkim/llvm/bin/llc
 
 # asm/sysreg.h inline assmbly used by it is incompatible with llvm.
 # But, ehere is not easy way to fix it, so just exclude it since it is
 # useless for BPF samples.
 $(obj)/%.o: $(src)/%.c
-       clang $(NOSTDINC_FLAGS) $(LINUXINCLUDE) $(EXTRA_CFLAGS) \
+       /home/gkim/llvm/bin/clang $(NOSTDINC_FLAGS) $(LINUXINCLUDE) $(EXTRA_CFLAGS) \
                -D__KERNEL__ -D__ASM_SYSREG_H -Wno-unused-value -Wno-pointer-sign \
                -O2 -emit-llvm -c $< -o -| $(LLC) -march=bpf -filetype=obj -o $@
-       clang $(NOSTDINC_FLAGS) $(LINUXINCLUDE) $(EXTRA_CFLAGS) \
+       /home/gkim/llvm/bin/clang $(NOSTDINC_FLAGS) $(LINUXINCLUDE) $(EXTRA_CFLAGS) \
                -D__KERNEL__ -D__ASM_SYSREG_H -Wno-unused-value -Wno-pointer-sign \
                -O2 -emit-llvm -c $< -o -| $(LLC) -march=bpf -filetype=asm -o $@.s
```

Run each sample program.
```
gurugio@giohnote:~/kernel/linux-source-4.4.0$ sudo ./samples/bpf/sockex1
TCP 0 UDP 0 ICMP 0 bytes
TCP 0 UDP 0 ICMP 196 bytes
```
```
gurugio@giohnote:~/kernel/linux-source-4.4.0$ sudo ./samples/bpf/tracex1
            ping-5802  [000] d.s1  1134.435854: : skb ffff88021576a600 len 84
            ping-5802  [000] d.s1  1134.435881: : skb ffff88021576ae00 len 84

            ping-5802  [000] d.s1  1135.435228: : skb ffff88021576a300 len 84
            ping-5802  [000] d.s1  1135.435273: : skb ffff88021576bb00 len 84
```

```
root@pserver:~/hdd/linux-pserver-future# samples/bpf/tracex2
location 0xffffffff8178433d count 1

5000000+0 records in
5000000+0 records out
2560000000 bytes (2.6 GB) copied, 1.76873 s, 1.4 GB/s
location 0xffffffff8178433d count 2

location 0xffffffff8178433d count 3

location 0xffffffff8178433d count 4


pid 1333 cmd sshd uid 0
           syscall write() stats
     byte_size       : count     distribution
       1 -> 1        : 0        |                                      |
       2 -> 3        : 0        |                                      |
       4 -> 7        : 0        |                                      |
       8 -> 15       : 0        |                                      |
      16 -> 31       : 0        |                                      |
      32 -> 63       : 0        |                                      |
      64 -> 127      : 4        |************************************* |
     128 -> 255      : 1        |********                              |
```

# example source.

```
root@debianvm:/usr/src/linux-source-4.9# cat samples/bpf/gioh1_user.c
#include <stdio.h>
#include <linux/bpf.h>
#include <unistd.h>
#include "libbpf.h"
#include "bpf_load.h"


/*
 * If it failed with "failed to create a map: 1 Operation not permitted" error,
 * set rlimit with "ulimit -l 10240".
 */


int main(int ac, char **argv)
{
	//FILE *f;
	char filename[256];

	snprintf(filename, sizeof(filename), "%s_kern.o", argv[0]);

	if (load_bpf_file(filename)) {
		printf("%s", bpf_log_buf);
		return 1;
	}

	/* f = popen("dd if=/dev/zero of=./big bs=512 count=100", "r"); */
	/* (void) f; */

	/* read trace message infinitely */
	read_trace_pipe();
	
	return 0;
}
root@debianvm:/usr/src/linux-source-4.9# cat samples/bpf/gioh1_kern.c
/* Copyright (c) 2013-2015 PLUMgrid, http://plumgrid.com
 *
 * This program is free software; you can redistribute it and/or
 * modify it under the terms of version 2 of the GNU General Public
 * License as published by the Free Software Foundation.
 */
#include <linux/skbuff.h>
#include <linux/netdevice.h>
#include <uapi/linux/bpf.h>
#include <linux/version.h>
#include <linux/blkdev.h>
#include "bpf_helpers.h"

#define _(P) ({typeof(P) val = 0; bpf_probe_read(&val, sizeof(val), &P); val;})


struct bpf_map_def SEC("maps") bio_latency_map = {
	.type = BPF_MAP_TYPE_HASH,
	.key_size = sizeof(long),
	.value_size = sizeof(u64),
	.max_entries = 4096,
};

/* kprobe is NOT a stable ABI
 * kernel functions can be removed, renamed or completely change semantics.
 * Number of arguments and their positions can change, etc.
 * In such case this bpf+kprobe example will no longer be meaningful
 */
void print_request(struct pt_regs *ctx)
{
	struct request *req;
	int cpu;
	unsigned long start_time;
	u64 ktime_val = bpf_ktime_get_ns();
	char fmt[] = "req=%p time=%llu\n";
	/* char time_fmt[] = "time=%llu\n"; */
	long req_key;
	

	req = (struct request *)PT_REGS_PARM1(ctx);
	/* cpu = _(req->cpu); */
	/* start_time = _(req->start_time); */
	
	/* using bpf_trace_printk() for DEBUG ONLY */
	bpf_trace_printk(fmt, sizeof(fmt), req, ktime_val);
	/* bpf_trace_printk(time_fmt, sizeof(time_fmt), ktime_val); */

	req_key = (long)req;
	bpf_map_update_elem(&bio_latency_map, &req_key, &ktime_val, BPF_ANY);
}


SEC("kprobe/blk_start_request")
int bpf_prog1(struct pt_regs *ctx)
{
	print_request(ctx);
	return 0;
}

SEC("kprobe/blk_mq_start_request")
int bpf_prog2(struct pt_regs *ctx)
{
	print_request(ctx);
	return 0;
}


SEC("kprobe/blk_account_io_completion")
int bpf_prog3(struct pt_regs *ctx)
{
	struct request *req = (struct request *)PT_REGS_PARM1(ctx);
	long req_key = (long)req;
	u64 *ktime_start;
	u64 ktime_end = bpf_ktime_get_ns();
	char message[] = "req=%p start=%llu end=%llu\n";
	u64 delta;
	char error_message[] = "skip not-recorded request\n";
	
	ktime_start = bpf_map_lookup_elem(&bio_latency_map, &req_key);

	if (!ktime_start) {
		bpf_trace_printk(error_message, sizeof(error_message));
		return 0;
	}
	
	delta = *ktime_start;
	bpf_trace_printk(message, sizeof(message), req, delta, ktime_end);
	
	bpf_map_delete_elem(&bio_latency_map, &req_key);
	return 0;
}

char _license[] SEC("license") = "GPL";
u32 _version SEC("version") = LINUX_VERSION_CODE;
```
