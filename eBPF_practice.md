# introduction to eBPF

Good document: http://www.brendangregg.com/ebpf.html

This document is based on Debuan/Ubuntu distribution.

kernel options for eBPF:
```
root@ws00837:/home/gohkim/work/linux-torvalds# grep BPF .config
CONFIG_BPF=y  --------> mandatory
CONFIG_BPF_SYSCALL=y -> mandatory
CONFIG_NET_CLS_BPF=m
CONFIG_NET_ACT_BPF=m
CONFIG_BPF_JIT=y
CONFIG_HAVE_BPF_JIT=y
CONFIG_BPF_EVENTS=y
CONFIG_TEST_BPF=m
```

# install bcc-tools using eBPF

WARNING! Tools are installed in ``/usr/share/bcc/tools`` directory, not /usr/bin
And every tool is Python or Bash script, so each tool has a manual inside of file.

```
$ echo "deb [trusted=yes] https://repo.iovisor.org/apt/xenial xenial-nightly main" | \
    sudo tee /etc/apt/sources.list.d/iovisor.list
$ sudo apt-get update
$ sudo apt-get install bcc-tools
$ ls /usr/share/bcc/tools/
argdist       dbstat               hardirqs        offcputime   rubyflow     tcpconnlat
bashreadline  dcsnoop              javacalls       offwaketime  rubygc       tcplife
biolatency    dcstat               javaflow        old          rubyobjnew   tcpretrans
biosnoop      deadlock_detector    javagc          oomkill      rubystat     tcptop
biotop        deadlock_detector.c  javaobjnew      opensnoop    runqlat      tcptracer
bitesize      doc                  javastat        phpcalls     runqlen      tplist
bpflist       execsnoop            javathreads     phpflow      slabratetop  trace
btrfsdist     ext4dist             killsnoop       phpstat      softirqs     ttysnoop
btrfsslower   ext4slower           lib             pidpersec    solisten     vfscount
cachestat     filelife             llcstat         profile      sslsniff     vfsstat
cachetop      fileslower           mdflush         pythoncalls  stackcount   wakeuptime
capable       filetop              memleak         pythonflow   statsnoop    xfsdist
cobjnew       funccount            mountsnoop      pythongc     syncsnoop    xfsslower
cpudist       funclatency          mysqld_qslower  pythonstat   syscount     zfsdist
cpuunclaimed  funcslower           nodegc          reset-trace  tcpaccept    zfsslower
dbslower      gethostlatency       nodestat        rubycalls    tcpconnect
```

And doc directory has how use manuals for each tool:
```
gohkim@ws00837:~$ ls /usr/share/bcc/tools/doc
argdist_example.txt            javacalls_example.txt       rubygc_example.txt
bashreadline_example.txt       javaflow_example.txt        rubyobjnew_example.txt
biolatency_example.txt         javagc_example.txt          rubystat_example.txt
biosnoop_example.txt           javaobjnew_example.txt      runqlat_example.txt
biotop_example.txt             javastat_example.txt        runqlen_example.txt
bitesize_example.txt           javathreads_example.txt     slabratetop_example.txt
bpflist_example.txt            killsnoop_example.txt       softirqs_example.txt
btrfsdist_example.txt          lib                         solisten_example.txt
btrfsslower_example.txt        llcstat_example.txt         sslsniff_example.txt
cachestat_example.txt          mdflush_example.txt         stackcount_example.txt
cachetop_example.txt           memleak_example.txt         statsnoop_example.txt
capable_example.txt            mountsnoop_example.txt      syncsnoop_example.txt
cobjnew_example.txt            mysqld_qslower_example.txt  syscount_example.txt
cpudist_example.txt            nodegc_example.txt          tcpaccept_example.txt
cpuunclaimed_example.txt       nodestat_example.txt        tcpconnect_example.txt
dbslower_example.txt           offcputime_example.txt      tcpconnlat_example.txt
dbstat_example.txt             offwaketime_example.txt     tcplife_example.txt
dcsnoop_example.txt            oomkill_example.txt         tcpretrans_example.txt
dcstat_example.txt             opensnoop_example.txt       tcptop_example.txt
deadlock_detector_example.txt  phpcalls_example.txt        tcptracer_example.txt
execsnoop_example.txt          phpflow_example.txt         tplist_example.txt
ext4dist_example.txt           phpstat_example.txt         trace_example.txt
ext4slower_example.txt         pidpersec_example.txt       ttysnoop_example.txt
filelife_example.txt           profile_example.txt         vfscount_example.txt
fileslower_example.txt         pythoncalls_example.txt     vfsstat_example.txt
filetop_example.txt            pythonflow_example.txt      wakeuptime_example.txt
funccount_example.txt          pythongc_example.txt        xfsdist_example.txt
funclatency_example.txt        pythonstat_example.txt      xfsslower_example.txt
funcslower_example.txt         reset-trace_example.txt     zfsdist_example.txt
gethostlatency_example.txt     rubycalls_example.txt       zfsslower_example.txt
hardirqs_example.txt           rubyflow_example.txt
```


# practices

## biolatency

gather bio latency before exit:
```
gohkim@ws00837:~/work/tmp$ sudo /usr/share/bcc/tools/biolatency 
[sudo] password for gohkim: 
Tracing block device I/O... Hit Ctrl-C to end.
^C
     usecs               : count     distribution
         0 -> 1          : 0        |                                        |
         2 -> 3          : 0        |                                        |
         4 -> 7          : 0        |                                        |
         8 -> 15         : 0        |                                        |
        16 -> 31         : 2        |                                        |
        32 -> 63         : 5        |**                                      |
        64 -> 127        : 16       |*******                                 |
       128 -> 255        : 6        |**                                      |
       256 -> 511        : 84       |****************************************|
       512 -> 1023       : 44       |********************                    |
      1024 -> 2047       : 39       |******************                      |
      2048 -> 4095       : 1        |                                        |
```

options:
* ``gohkim@ws00837:~/work/tmp$ sudo /usr/share/bcc/tools/biolatency  3``: print every 3 seconds until ctrl+c
* ``gohkim@ws00837:~/work/tmp$ sudo /usr/share/bcc/tools/biolatency  3 2``: every 3 seconds, 2 times
* ``-D``: for each devices
* ``-Q``: include queued time

## biosnoop

Trace block device I/O with process, disk, and latency details: no option
* use grep to filter
```
gohkim@ws00837:~/work/tmp$ sudo /usr/share/bcc/tools/biosnoop 
TIME(s)        COMM           PID    DISK    T  SECTOR    BYTES   LAT(ms)
0.000000000    ?              0              R  -1        8          1.09
2.000026000    ?              0              R  -1        8          1.11
3.094404000    BrowserBlockin 2465   sda     W  148372480 143360     1.72
3.094592000    jbd2/dm-0-8    271    sda     W  105815096 36864      0.10
3.105634000    kworker/5:1    17199  sda     W  105815168 4096       0.04
3.999985000    ?              0              R  -1        8          1.13
5.999989000    ?              0              R  -1        8          1.12
8.000018000    ?              0              R  -1        8          1.10
8.424784000    jbd2/dm-0-8    271    sda     W  68169456  4096       1.53
8.424800000    jbd2/dm-0-8    271    sda     W  68168248  4096       1.53
8.424800000    jbd2/dm-0-8    271    sda     W  69198784  8192       1.52
gohkim@ws00837:~/work/tmp$ sudo /usr/share/bcc/tools/biosnoop | grep Chrome
0.000000000    Chrome_SyncThr 2592   sda     W  65321472  200704     1.87
0.011592000    Chrome_SyncThr 2592   sda     W  65321472  4096       0.03
0.017208000    Chrome_SyncThr 2592   sda     W  3223632   65536      0.15
0.017269000    Chrome_SyncThr 2592   sda     W  3223824   32768      0.20
```

## profile

profile prints callstack of each CPU. 
https://github.com/iovisor/bcc/blob/master/tools/profile_example.txt

*I cannot run profile on kernel v4.4. It requires newer kernel.*

# perf-tools-unstable package

kprobe: http://manpages.ubuntu.com/manpages/zesty/man8/kprobe-perf.8.html
functrace: http://manpages.ubuntu.com/manpages/zesty/man8/functrace.8.html
