# introduction to eBPF

Good document: http://www.brendangregg.com/ebpf.html

This document is based on Debuan/Ubuntu distribution.

kernel options for eBPF:
```
root@ws00837:/home/gohkim/work/linux-torvalds# grep BPF .config
CONFIG_BPF=y
CONFIG_BPF_SYSCALL=y
CONFIG_NETFILTER_XT_MATCH_BPF=m
CONFIG_NET_CLS_BPF=m
CONFIG_NET_ACT_BPF=m
CONFIG_BPF_JIT=y
CONFIG_HAVE_BPF_JIT=y
CONFIG_BPF_EVENTS=y
CONFIG_TEST_BPF=m
```

# install bcc-tools using eBPF

WARNING! Tools are installed in ``/usr/share/bcc/tools`` directory.
* not /usr/bin
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


# ubuntu tools of perf-tools-unstable package

## kprobe

manual: http://manpages.ubuntu.com/manpages/zesty/man8/kprobe-perf.8.html


## functrace

manual: http://manpages.ubuntu.com/manpages/zesty/man8/functrace.8.html

### simple trace a kernel function

```
root@ws00837:/home/gohkim/work/linux-torvalds# functrace do_nanosleep
Tracing "do_nanosleep"... Ctrl-C to end.
        redshift-2190  [006] .... 91973.831185: do_nanosleep <-hrtimer_nanosleep
      irqbalance-1110  [001] .... 91976.338481: do_nanosleep <-hrtimer_nanosleep
        redshift-2190  [006] .... 91978.833109: do_nanosleep <-hrtimer_nanosleep
 gnome-terminal--2291  [002] .... 91978.892798: do_nanosleep <-hrtimer_nanosleep
 gnome-terminal--2291  [002] .... 91978.927036: do_nanosleep <-hrtimer_nanosleep
 gnome-terminal--2291  [002] .... 91978.960710: do_nanosleep <-hrtimer_nanosleep
^C
Ending tracing...
```

## 
