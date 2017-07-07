
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

print start time and finish time of a request.

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

# 2nd example


## result

It prints the statistics of latency of each requests of all disk.
* "id=fd00010" is device number of a disk, (253,16)
* "read X X X" is latency of read request, 0~10ms, 10~100ms, >100ms
  * "read 0 1 0" means: no request took 0~10ms, 1 request took 10~100ms, no request took >100ms to be finished.

```
root@pserver:~/hdd/linux-pserver-future# ./samples/bpf/biolatency 
          <idle>-0     [000] dNh. 94665.801747: : id=fd00010 rw=1 ms=0

[ id=fd00000 read 4202512 0 0, write 4197773 140727536943792 0] ------------> not initialized yet


          <idle>-0     [000] d.h. 94665.801827: : id=fd00010 rw=1 ms=0

[ id=fd00000 read 4202512 0 0, write 4197773 140727536943792 0]


          <idle>-0     [000] d.h. 94665.883213: : id=fd00010 rw=1 ms=0

[ id=fd00000 read 4202512 0 0, write 4197773 140727536943792 0]
[ id=fd00010 read 0 0 0, write 3 0 0]


          <idle>-0     [000] d.h. 94665.922332: : id=0 rw=1 ms=39

[ id=fd00000 read 0 0 0, write 3 0 0]
[ id=fd00010 read 0 0 0, write 3 0 0]


          <idle>-0     [002] dNh. 94679.041308: : id=fd00001 rw=0 ms=62

[ id=fd00000 read 0 0 0, write 3 0 0]
[ id=fd00010 read 0 0 0, write 3 0 0]
[ id=fd00001 read 0 1 0, write 0 0 0]


              ls-28927 [002] d.h. 94679.041714: : id=fd00001 rw=0 ms=0

[ id=fd00000 read 0 1 0, write 0 0 0]
[ id=fd00010 read 0 0 0, write 3 0 0]
[ id=fd00001 read 1 1 0, write 0 0 0]
```

## source files

```
root@pserver:~/hdd/linux-pserver-future# cat samples/bpf/biolatency_user.c
#include <stdio.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <libelf.h>
#include <gelf.h>
#include <errno.h>
#include <unistd.h>
#include <string.h>
#include <stdbool.h>
#include <stdlib.h>
#include <linux/bpf.h>
#include <linux/filter.h>
#include <linux/perf_event.h>
#include <sys/syscall.h>
#include <sys/ioctl.h>
#include <sys/mman.h>
#include <poll.h>
#include <ctype.h>
#include "libbpf.h"
#include "bpf_load.h"

#define DEBUGFS "/sys/kernel/debug/tracing/"


struct lat_table {
	/*
	 * bpf program use restrict C language
	 * - pointer must be read by bpf_probe_read()
	 * - writing to pointer is not possible
	 * - array and array of pointer cannot be here: counters[rw][15]
	 */
	__u64 counters_r_less_10;
	__u64 counters_r_10_100;
	__u64 counters_r_more_100;
	__u64 counters_w_less_10;
	__u64 counters_w_10_100;
	__u64 counters_w_more_100;
};

void show_trace(void)
{
	int trace_fd;
	__u32 key = 0, next_key;
	struct lat_table table;

	trace_fd = open(DEBUGFS "trace_pipe", O_RDONLY, 0);
	if (trace_fd < 0)
		exit(1);

	while (1) {
		static char buf[4096];
		ssize_t sz;

		sz = read(trace_fd, buf, sizeof(buf));
		if (sz > 0) {
			buf[sz] = 0;
			puts(buf);
		}

		/* device number of virtblk: 253,0 */
		key = 0xfd00000;

		/* map_fd is array of fd of maps.
		 * For example, latency_map is the second maps
		 * in biolatency_kern.c file.
		 * So fd according to latency_map is map_fd[1].
		 */
		while (bpf_get_next_key(map_fd[1], &key, &next_key) == 0) {
			bpf_lookup_elem(map_fd[1], &key, &table);
			printf("[ id=%x read %llu %llu %llu, write %llu %llu %llu]\n",
			       key,
			       table.counters_r_less_10,
			       table.counters_r_10_100,
			       table.counters_r_more_100,
			       table.counters_w_less_10,
			       table.counters_w_10_100,
			       table.counters_w_more_100);
			key = next_key;
		}
		printf("\n\n");
	}
}

int main(int ac, char **argv)
{
	char filename[256];

	snprintf(filename, sizeof(filename), "%s_kern.o", argv[0]);

	/* if loading bpf fails, increase rlimit with:
	 * ulimit -l 10240
	 * ulimit -n 65536
	 */
	if (load_bpf_file(filename)) {
		printf("fail to load bpf file: %s", bpf_log_buf);
		return 1;
	}

	/* read trace message infinitely */
	show_trace();
	return 0;
}
root@pserver:~/hdd/linux-pserver-future# cat samples/bpf/biolatency_kern.c
#include <linux/skbuff.h>
#include <linux/netdevice.h>
#include <uapi/linux/bpf.h>
#include <linux/version.h>
#include <linux/blkdev.h>
#include "bpf_helpers.h"


/*
 * NEVER READ any field of a structure directly
 */
#define _(P) ({typeof(P) val = 0; bpf_probe_read(&val, sizeof(val), &P); val; })


struct bpf_map_def SEC("maps") bio_latency_map = {
	.type = BPF_MAP_TYPE_HASH,
	.key_size = sizeof(long),
	.value_size = sizeof(u64),
	.max_entries = 65536,
};

void record_start(struct pt_regs *ctx)
{
	struct request *req;
	long req_key;
	u64 ktime_start = bpf_ktime_get_ns();

	req = (struct request *)PT_REGS_PARM1(ctx);
	req_key = (long)req;
	bpf_map_update_elem(&bio_latency_map, &req_key, &ktime_start, BPF_ANY);
}

SEC("kprobe/blk_start_request")
int bpf_prog1(struct pt_regs *ctx)
{
	record_start(ctx);
	return 0;
}

SEC("kprobe/blk_mq_start_request")
int bpf_prog2(struct pt_regs *ctx)
{
	record_start(ctx);
	return 0;
}

struct lat_table {
	/*
	 * bpf program use restrict C language
	 * - pointer must be read by bpf_probe_read()
	 * - writing to pointer is not possible
	 * - array and array of pointer cannot be here: counters[rw][15]
	 */
	u64 counters_r_less_10;
	u64 counters_r_10_100;
	u64 counters_r_more_100;
	u64 counters_w_less_10;
	u64 counters_w_10_100;
	u64 counters_w_more_100;
};


struct bpf_map_def SEC("maps") latency_map = {
	.type = BPF_MAP_TYPE_HASH,
	.key_size = sizeof(u32),
	.value_size = sizeof(struct lat_table),
	 /* if it's too big, it will fail to load bpf program */
	.max_entries = 1024,
};

SEC("kprobe/blk_account_io_done")
int bpf_prog3(struct pt_regs *ctx)
{
	struct request *req = (struct request *)PT_REGS_PARM1(ctx);
	long req_key = (long)req;
	u64 ktime_end = bpf_ktime_get_ns();
	char message[] = "ms=%d idx=%d\n";
	u64 *ktime_start;
	unsigned long ms;
	int rw;
	struct lat_table *table;
	struct hd_struct *hdp;
	dev_t dev;
	int major, minor;
	u32 dev_id;
	char rw_msg[] = "id=%x rw=%d ms=%llu\n";

	ktime_start = bpf_map_lookup_elem(&bio_latency_map, &req_key);
	if (!ktime_start) {
		/* skip not traced request */
		return 0;
	}

	ms = (ktime_end - *ktime_start) / 1000000L;
	rw = (int)_(req->cmd_flags) & 1;
	hdp = _(req->part);
	dev_id = (u32)_(hdp->__dev.devt);
	/* req is not necessary anymore */
	bpf_map_delete_elem(&bio_latency_map, &req_key);

	table = bpf_map_lookup_elem(&latency_map, &dev_id);
	if (table) {
		if (rw) {
			if (ms < 10UL)
				__sync_fetch_and_add((long *)&table->counters_w_less_10, 1);
			else if (ms < 100UL)
				__sync_fetch_and_add((long *)&table->counters_w_10_100, 1);
			else
				__sync_fetch_and_add((long *)&table->counters_w_more_100, 1);
		} else {
			if (ms < 10UL)
				__sync_fetch_and_add((long *)&table->counters_r_less_10, 1);
			else if (ms < 100UL)
				__sync_fetch_and_add((long *)&table->counters_r_10_100, 1);
			else
				__sync_fetch_and_add((long *)&table->counters_r_more_100, 1);
		}
		bpf_trace_printk(rw_msg, sizeof(rw_msg), dev_id, rw, ms);
	} else {
		struct lat_table clear;

		memset(&clear, 0, sizeof(struct lat_table));
		/* create table for dev_id device */
		bpf_map_update_elem(&latency_map, &dev_id, &clear, BPF_NOEXIST);
	}

	return 0;
}

char _license[] SEC("license") = "GPL";
u32 _version SEC("version") = LINUX_VERSION_CODE;
```


# Performance degradation


## setup null_blk module

1. build null_blk.ko 
```make drivers/block/null_blk.ko queue_mode=2
or 
insmod /lib/modules/4.10.0-26-generic/kernel/drivers/block/null_blk.ko queue_mode=2
```

2. device files
```
root@ws00837:/usr/src/linux-source-4.10.0/linux-source-4.10.0# ls /dev/nullb*
/dev/nullb0  /dev/nullb1  /dev/nullb1p1  /dev/nullb1p3
```

3. run fio with following configuration
```
root@ws00837:/usr/src/linux-source-4.10.0/linux-source-4.10.0# cat fio.conf 
[global]
description=Emulation of Storage Server Access Pattern
bssplit=512/20:1k/16:2k/9:4k/12:8k/19:16k/10:32k/8:64k/4
fadvise_hint=0
rw=randrw:2
#rw=write
direct=1

ioengine=libaio
iodepth=64
iodepth_batch_submit=64
iodepth_batch_complete=64
numjobs=4
gtod_reduce=1
group_reporting=1

time_based=1
runtime=10

[job]
filename=/dev/nullb0
root@ws00837:/usr/src/linux-source-4.10.0/linux-source-4.10.0# fio fio.conf 
```

## before bpf


```
root@ws00837:/usr/src/linux-source-4.10.0/linux-source-4.10.0# bash -x go.bash 
+ echo 3
+ fio fio.conf
job: (g=0): rw=randrw, bs=512-64K/512-64K/512-64K, ioengine=libaio, iodepth=64
...
fio-2.16
Starting 4 processes
Jobs: 4 (f=4): [m(4)] [100.0% done] [13315MB/13308MB/0KB /s] [1591K/1591K/0 iops] [eta 00m:00s]
job: (groupid=0, jobs=4): err= 0: pid=16760: Fri Jul  7 17:27:45 2017
  Description  : [Emulation of Storage Server Access Pattern]
  read : io=136315MB, bw=13630MB/s, iops=1581.3K, runt= 10001msec
  write: io=136474MB, bw=13646MB/s, iops=1581.7K, runt= 10001msec
  cpu          : usr=29.57%, sys=70.30%, ctx=15611, majf=0, minf=34
  IO depths    : 1=0.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=100.0%
     submit    : 0=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=100.0%, >=64=0.0%
     complete  : 0=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=100.0%, >=64=0.0%
     issued    : total=r=15813620/w=15818308/d=0, short=r=0/w=0/d=0, drop=r=0/w=0/d=0
     latency   : target=0, window=0, percentile=100.00%, depth=64

Run status group 0 (all jobs):
   READ: io=136315MB, aggrb=13630MB/s, minb=13630MB/s, maxb=13630MB/s, mint=10001msec, maxt=10001msec
  WRITE: io=136474MB, aggrb=13646MB/s, minb=13646MB/s, maxb=13646MB/s, mint=10001msec, maxt=10001msec

Disk stats (read/write):
  nullb0: ios=8414306/8417055, merge=7198075/7199570, ticks=122628/122148, in_queue=249776, util=98.83%
+ echo 3
+ fio fio.conf
job: (g=0): rw=randrw, bs=512-64K/512-64K/512-64K, ioengine=libaio, iodepth=64
...
fio-2.16
Starting 4 processes
Jobs: 4 (f=4): [m(4)] [100.0% done] [13082MB/13080MB/0KB /s] [1563K/1564K/0 iops] [eta 00m:00s]
job: (groupid=0, jobs=4): err= 0: pid=16774: Fri Jul  7 17:27:55 2017
  Description  : [Emulation of Storage Server Access Pattern]
  read : io=136536MB, bw=13652MB/s, iops=1583.9K, runt= 10001msec
  write: io=136693MB, bw=13668MB/s, iops=1584.4K, runt= 10001msec
  cpu          : usr=30.04%, sys=69.80%, ctx=14752, majf=0, minf=39
  IO depths    : 1=0.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=100.0%
     submit    : 0=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=100.0%, >=64=0.0%
     complete  : 0=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=100.0%, >=64=0.0%
     issued    : total=r=15839622/w=15844914/d=0, short=r=0/w=0/d=0, drop=r=0/w=0/d=0
     latency   : target=0, window=0, percentile=100.00%, depth=64

Run status group 0 (all jobs):
   READ: io=136536MB, aggrb=13652MB/s, minb=13652MB/s, maxb=13652MB/s, mint=10001msec, maxt=10001msec
  WRITE: io=136693MB, aggrb=13668MB/s, minb=13668MB/s, maxb=13668MB/s, mint=10001msec, maxt=10001msec

Disk stats (read/write):
  nullb0: ios=8436286/8438950, merge=7216617/7218577, ticks=121888/123236, in_queue=247228, util=98.03%
+ echo 3
+ fio fio.conf
job: (g=0): rw=randrw, bs=512-64K/512-64K/512-64K, ioengine=libaio, iodepth=64
...
fio-2.16
Starting 4 processes
Jobs: 4 (f=4): [m(4)] [100.0% done] [13012MB/13034MB/0KB /s] [1557K/1560K/0 iops] [eta 00m:00s]
job: (groupid=0, jobs=4): err= 0: pid=16790: Fri Jul  7 17:28:06 2017
  Description  : [Emulation of Storage Server Access Pattern]
  read : io=138497MB, bw=13848MB/s, iops=1607.4K, runt= 10001msec
  write: io=138660MB, bw=13865MB/s, iops=1608.1K, runt= 10001msec
  cpu          : usr=29.89%, sys=69.98%, ctx=15264, majf=0, minf=31
  IO depths    : 1=0.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=100.0%
     submit    : 0=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=100.0%, >=64=0.0%
     complete  : 0=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=100.0%, >=64=0.0%
     issued    : total=r=16075589/w=16081715/d=0, short=r=0/w=0/d=0, drop=r=0/w=0/d=0
     latency   : target=0, window=0, percentile=100.00%, depth=64

Run status group 0 (all jobs):
   READ: io=138497MB, aggrb=13848MB/s, minb=13848MB/s, maxb=13848MB/s, mint=10001msec, maxt=10001msec
  WRITE: io=138660MB, aggrb=13865MB/s, minb=13865MB/s, maxb=13865MB/s, mint=10001msec, maxt=10001msec

Disk stats (read/write):
  nullb0: ios=8554673/8557790, merge=7318157/7320348, ticks=121004/121888, in_queue=245948, util=99.02%

```

## after bpf

```
+ fio fio.conf
job: (g=0): rw=randrw, bs=512-64K/512-64K/512-64K, ioengine=libaio, iodepth=64
...
fio-2.16
Starting 4 processes
Jobs: 4 (f=4): [m(4)] [100.0% done] [9905MB/9902MB/0KB /s] [1167K/1165K/0 iops] [eta 00m:00s] 
job: (groupid=0, jobs=4): err= 0: pid=16810: Fri Jul  7 17:28:16 2017
  Description  : [Emulation of Storage Server Access Pattern]
  read : io=100011MB, bw=10000MB/s, iops=1149.7K, runt= 10001msec
  write: io=100127MB, bw=10012MB/s, iops=1149.7K, runt= 10001msec
  cpu          : usr=22.82%, sys=77.05%, ctx=17738, majf=0, minf=29
  IO depths    : 1=0.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=100.0%
     submit    : 0=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=100.0%, >=64=0.0%
     complete  : 0=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=100.0%, >=64=0.0%
     issued    : total=r=11497837/w=11497611/d=0, short=r=0/w=0/d=0, drop=r=0/w=0/d=0
     latency   : target=0, window=0, percentile=100.00%, depth=64

Run status group 0 (all jobs):
   READ: io=100011MB, aggrb=10000MB/s, minb=10000MB/s, maxb=10000MB/s, mint=10001msec, maxt=10001msec
  WRITE: io=100127MB, aggrb=10012MB/s, minb=10012MB/s, maxb=10012MB/s, mint=10001msec, maxt=10001msec

Disk stats (read/write):
  nullb0: ios=6123579/6123209, merge=5236715/5236455, ticks=133884/133692, in_queue=273224, util=99.20%
+ echo 3
+ fio fio.conf
job: (g=0): rw=randrw, bs=512-64K/512-64K/512-64K, ioengine=libaio, iodepth=64
...
fio-2.16
Starting 4 processes
Jobs: 4 (f=4): [m(4)] [100.0% done] [10034MB/10039MB/0KB /s] [1183K/1181K/0 iops] [eta 00m:00s]
job: (groupid=0, jobs=4): err= 0: pid=16824: Fri Jul  7 17:28:27 2017
  Description  : [Emulation of Storage Server Access Pattern]
  read : io=101393MB, bw=10138MB/s, iops=1165.1K, runt= 10001msec
  write: io=101512MB, bw=10150MB/s, iops=1166.3K, runt= 10001msec
  cpu          : usr=21.43%, sys=78.46%, ctx=17819, majf=0, minf=31
  IO depths    : 1=0.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=100.0%
     submit    : 0=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=100.0%, >=64=0.0%
     complete  : 0=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=100.0%, >=64=0.0%
     issued    : total=r=11660778/w=11661198/d=0, short=r=0/w=0/d=0, drop=r=0/w=0/d=0
     latency   : target=0, window=0, percentile=100.00%, depth=64

Run status group 0 (all jobs):
   READ: io=101393MB, aggrb=10138MB/s, minb=10138MB/s, maxb=10138MB/s, mint=10001msec, maxt=10001msec
  WRITE: io=101512MB, aggrb=10150MB/s, minb=10150MB/s, maxb=10150MB/s, mint=10001msec, maxt=10001msec

Disk stats (read/write):
  nullb0: ios=6211522/6211511, merge=5312289/5311734, ticks=138884/140316, in_queue=285192, util=99.28%
+ echo 3
+ fio fio.conf
job: (g=0): rw=randrw, bs=512-64K/512-64K/512-64K, ioengine=libaio, iodepth=64
...
fio-2.16
Starting 4 processes
Jobs: 4 (f=4): [m(4)] [100.0% done] [9766MB/9771MB/0KB /s] [1150K/1148K/0 iops] [eta 00m:00s] 
job: (groupid=0, jobs=4): err= 0: pid=16838: Fri Jul  7 17:28:37 2017
  Description  : [Emulation of Storage Server Access Pattern]
  read : io=99847MB, bw=9983.7MB/s, iops=1147.7K, runt= 10001msec
  write: io=99960MB, bw=9995.3MB/s, iops=1147.7K, runt= 10001msec
  cpu          : usr=22.39%, sys=77.48%, ctx=16992, majf=0, minf=38
  IO depths    : 1=0.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=100.0%
     submit    : 0=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=100.0%, >=64=0.0%
     complete  : 0=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=100.0%, >=64=0.0%
     issued    : total=r=11478058/w=11477966/d=0, short=r=0/w=0/d=0, drop=r=0/w=0/d=0
     latency   : target=0, window=0, percentile=100.00%, depth=64

Run status group 0 (all jobs):
   READ: io=99847MB, aggrb=9983.7MB/s, minb=9983.7MB/s, maxb=9983.7MB/s, mint=10001msec, maxt=10001msec
  WRITE: io=99960MB, aggrb=9995.3MB/s, minb=9995.3MB/s, maxb=9995.3MB/s, mint=10001msec, maxt=10001msec

Disk stats (read/write):
  nullb0: ios=6107827/6107585, merge=5223197/5223228, ticks=136116/135452, in_queue=277376, util=99.07%

```

## conclusion

performace drop = 27%
* before: 13600MB/s
* after: 10000MB/s

