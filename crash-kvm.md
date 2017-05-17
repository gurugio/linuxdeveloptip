
# How to run crash

Run crash with kernel debug image and dump file
* download dbg package of the kernel including vmlinux file
* or if kernel is built by yourself, you can find a vmlinux file in the kernel directory
```
$ crash tmp/usr/lib/debug/vmlinux-3.16.0-4-amd64 0515.dump
...
      KERNEL: tmp/usr/lib/debug/vmlinux-3.16.0-4-amd64
    DUMPFILE: 0515.dump  [PARTIAL DUMP]
        CPUS: 1
        DATE: Sun May 14 05:05:33 2017
      UPTIME: 8 days, 12:24:20
LOAD AVERAGE: 1.07, 1.09, 1.08
       TASKS: 40
    NODENAME: debian
     RELEASE: 3.16.0-4-amd64
     VERSION: #1 SMP Debian 3.16.39-1 (2016-12-30)
     MACHINE: x86_64  (2799 Mhz)
      MEMORY: 4 GB
       PANIC: "BUG: unable to handle kernel NULL pointer dereference at           (null)"
         PID: 45075
     COMMAND: "fio"
        TASK: ffff880036c32150  [THREAD_INFO: ffff8800ba2e8000]
         CPU: 0
       STATE: TASK_RUNNING (PANIC)

crash> 
```

# get task information

A process that generated panic was fio and task_struct was at 0xffff880036c32150.
* A command to print the value of an expression
  * expression can be variable or any other expression
  * For example, p can print a element in array: ```p ((int *)0xffff88003de0b140)[0x81]```

```
crash> p/x *(struct task_struct *)0xffff880036c32150
$6 = {
  state = 0x0, 
  stack = 0xffff8800ba2e8000, 
  usage = {
    counter = 0x2
  }, 
...................
crash> p/x ((struct task_struct *)0xffff880036c32150).thread
$7 = {
  tls_array = {{
      {
        {
          a = 0x0, 
          b = 0x0
        }, 
        {
          limit0 = 0x0, 
          base0 = 0x0, 
          base1 = 0x0, 
          type = 0x0, 
......................
  sp0 = 0xffff8800ba2ec000, 
  sp = 0xffff8800ba2ebe40, 
  usersp = 0x7ffdc0388e10, 
  es = 0x0, 
  ds = 0x0, 
  fsindex = 0x0, 
  gsindex = 0x0, 
  fs = 0x7fe5abfaf7c0, 
  gs = 0x0, 
  ptrace_bps = {0x0, 0x0, 0x0, 0x0}, 
  debugreg6 = 0x0, 
  ptrace_dr7 = 0x0, 
  cr2 = 0xffff90013b02d000, 
  trap_nr = 0xe, 
  error_code = 0x0, 
  fpu = {
    last_cpu = 0x0, 
    has_fpu = 0x1, 
    state = 0xffff8801395b5740
  }, 
  io_bitmap_ptr = 0x0, 
  iopl = 0x0, 
  io_bitmap_max = 0x0, 
  fpu_counter = 0x15
}
crash> p/x ((struct task_struct *)0xffff880036c32150).thread.sp0
$10 = 0xffff8800ba2ec000
```

There is ps command to check processes when the panic happened.
```
crash> ps
   PID    PPID  CPU       TASK        ST  %MEM     VSZ    RSS  COMM
      0      0   0  ffffffff8181a460  RU   0.0       0      0  [swapper/0]
     32     -1   0  ffff8800b97a2d20  IN   0.0       0      0  [ipv6_addrconf]
...
    440     -1   0  ffff8801396cac60  IN   0.1  258672   4320  rs:main Q:Reg
    441     -1   0  ffff8800baacd630  IN   0.3   55668  13696  munin-node
    852     -1   0  ffff880139625470  IN   0.1   27088   3628  systemd
    853    852   0  ffff8800baaccce0  IN   0.0   49820   2600  (sd-pam)
    865     -1   0  ffff8801395a12f0  IN   0.1   27288   2836  screen
    866    865   0  ffff880036c3b4f0  IN   0.1   23456   4156  bash
  10850    866   0  ffff880139e062d0  IN   0.5  170440  25260  iperf
  10851    866   0  ffff880139624b20  IN   0.5  170440  25260  iperf
  10852    866   0  ffff880036f79430  RU   0.5  170440  25260  iperf
  16026     -1   0  ffff8801395b60d0  IN   0.1   27284   2696  screen
  40343     -1   0  ffff880139502ca0  IN   0.0       0      0  [kworker/0:2]
> 45075     -1   0  ffff880036c32150  RU   1.1  192840  58384  fio
  45076     -1   0  ffff8801395a09a0  IN   1.1  192840  58384  fio
  45077  45075   0  ffff880139614b60  RU   0.1  170340   3036  fio
  45088    866   0  ffff8800bb47f5b0  RU   0.5  170440  25260  iperf
```

If we know PID, we can print task_struct with task command.
```
crash> task 45075
PID: 45075  TASK: ffff880036c32150  CPU: 0   COMMAND: "fio"
struct task_struct {
  state = 0, 
  stack = 0xffff8800ba2e8000, 
  usage = {
    counter = 2
  }, 
  flags = 4202496, 
  ptrace = 0, 
  wake_entry = {
    next = 0x0
  }, 
```

# investigate stack

sp0 in thread_struct has the top address of kernel stack
* top address of kernel stack is 0xffff8800ba2ec000
  * so kernel stack is 0xffff8800ba2e8000 ~ 0xffff8800ba2ec000, if kernel stack size is 16K
* sp: current stack pointer
  * you can check the last kernel function the task ran
```
crash> p/x ((struct task_struct *)0xffff880036c32150).thread.sp0
$12 = 0xffff8800ba2ec000
crash> p/x ((struct task_struct *)0xffff880036c32150).thread.sp
$13 = 0xffff8800ba2ebe40
crash> bt -S 0xffff8800ba2ec000
PID: 45075  TASK: ffff880036c32150  CPU: 0   COMMAND: "fio"
bt: non-process stack address for this task: ffff8800ba2ec000
    (valid range: ffff8800ba2e8000 - ffff8800ba2ec000)
```


command: ps, task, x, rd

