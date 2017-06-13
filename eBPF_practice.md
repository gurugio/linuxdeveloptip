# intro

introduction to eBPF: http://www.brendangregg.com/ebpf.html

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

# kprobe
manual: http://manpages.ubuntu.com/manpages/zesty/man8/kprobe-perf.8.html


# functrace

manual: http://manpages.ubuntu.com/manpages/zesty/man8/functrace.8.html

## simple trace a kernel function

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
