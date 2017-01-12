# How to use ftrace

```
echo alloc_iova > /sys/kernel/debug/tracing/set_graph_function
echo function_graph > /sys/kernel/debug/tracing/current_tracer
fio fio_test.fio
cat /sys/kernel/debug/tracing/trace > result.txt
echo nop > /sys/kernel/debug/tracing/current_tracer
```

# trace-cmd tool

https://lwn.net/Articles/410200/

record
```
root@ws00837:/home/gohkim/work/tools/intel-iommu-perf# trace-cmd record -p function_graph -g intel_alloc_iova fio fio_test.fio
  plugin 'function_graph'
job: (g=0): rw=randrw, bs=512-64K/512-64K/512-64K, ioengine=libaio, iodepth=64
...
job: (g=0): rw=randrw, bs=512-64K/512-64K/512-64K, ioengine=libaio, iodepth=64
...
job: (g=0): rw=randrw, bs=512-64K/512-64K/512-64K, ioengine=libaio, iodepth=64
...
job: (g=0): rw=randrw, bs=512-64K/512-64K/512-64K, ioengine=libaio, iodepth=64
...
job: (g=0): rw=randrw, bs=512-64K/512-64K/512-64K, ioengine=libaio, iodepth=64
...
fio-2.1.11
Starting 40 processes
...
```

report
```
root@ws00837:/home/gohkim/work/tools/intel-iommu-perf# trace-cmd report | less

...
     usb-storage-14188 [007] 84301.833429: funcgraph_entry:                   |  intel_alloc_iova() {
     usb-storage-14188 [007] 84301.833429: funcgraph_entry:                   |    alloc_iova() {
     usb-storage-14188 [007] 84301.833429: funcgraph_entry:        0.035 us   |      kmem_cache_alloc();
     usb-storage-14188 [007] 84301.833429: funcgraph_entry:        0.028 us   |      _raw_spin_lock_irqsave();
     usb-storage-14188 [007] 84301.833429: funcgraph_entry:        0.025 us   |      iova_get_pad_size();
     usb-storage-14188 [007] 84301.833430: funcgraph_entry:        0.031 us   |      _raw_spin_unlock_irqrestore();
     usb-storage-14188 [007] 84301.833430: funcgraph_exit:         1.080 us   |    }
     usb-storage-14188 [007] 84301.833430: funcgraph_exit:         1.376 us   |  }
     usb-storage-14188 [007] 84301.845267: funcgraph_entry:                   |  intel_alloc_iova() {
     usb-storage-14188 [007] 84301.845267: funcgraph_entry:                   |    alloc_iova() {
     usb-storage-14188 [007] 84301.845267: funcgraph_entry:        0.051 us   |      kmem_cache_alloc();
     usb-storage-14188 [007] 84301.845267: funcgraph_entry:        0.018 us   |      _raw_spin_lock_irqsave();
     usb-storage-14188 [007] 84301.845267: funcgraph_entry:        0.024 us   |      iova_get_pad_size();
     usb-storage-14188 [007] 84301.845268: funcgraph_entry:        0.030 us   |      _raw_spin_unlock_irqrestore();
     usb-storage-14188 [007] 84301.845268: funcgraph_exit:         0.993 us   |    }
     usb-storage-14188 [007] 84301.845268: funcgraph_exit:         1.279 us   |  }

...
```


DON't forget to reset the setting for ftrace
```
# trace-cmd reset
```
