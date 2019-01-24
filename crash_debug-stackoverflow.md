
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

In task_struct, stack field has the lowest address of kernel stack. sp0 in thread_struct has the top address of kernel stack.
* top address of kernel stack is 0xffff8800ba2ec000
  * so kernel stack is 0xffff8800ba2e8000 ~ 0xffff8800ba2ec000, if kernel stack size is 16K
* sp: current stack pointer
  * you can check the last kernel function the task ran
```
crash> task 45075
PID: 45075  TASK: ffff880036c32150  CPU: 0   COMMAND: "fio"
struct task_struct {
  state = 0, 
  stack = 0xffff8800ba2e8000, 
  ...
crash> p/x ((struct task_struct *)0xffff880036c32150).thread.sp0
$12 = 0xffff8800ba2ec000
crash> p/x ((struct task_struct *)0xffff880036c32150).thread.sp
$13 = 0xffff8800ba2ebe40
crash> bt -S 0xffff8800ba2ec000
PID: 45075  TASK: ffff880036c32150  CPU: 0   COMMAND: "fio"
bt: non-process stack address for this task: ffff8800ba2ec000
    (valid range: ffff8800ba2e8000 - ffff8800ba2ec000)
```

There are two commands for printing values in the kernel stack.

bt with -S option can callstack from the address.
* "bt -S 0xffff8800ba2e8000": print callstack from 0xffff8800ba2e8000 ~ 0xffff8800ba2ec000
* At the last, there are register values that are stored in kernel stack when user process calls the system-call.
  * RSP shows the stack address of user process
  * RIP shows the instruction address of user process
  * CS and SS has CPL(CS)=RPL(SS)=3
* "foreach bt" command show the backtrace of all processes
```
crash> bt -S 0xffff8800ba2e8000
PID: 45075  TASK: ffff880036c32150  CPU: 0   COMMAND: "fio"
 #0 [ffff8800ba2e8000] __schedule at ffffffff81517166
 #1 [ffff8800ba2eb8c8] zone_statistics at ffffffff8115d235
 #2 [ffff8800ba2eb9c0] __alloc_pages_nodemask at ffffffff81148626
 #3 [ffff8800ba2ebbe8] __d_lookup_rcu at ffffffff811c268e
 #4 [ffff8800ba2ebc58] lookup_fast at ffffffff811b45be
 #5 [ffff8800ba2ebca0] link_path_walk at ffffffff811b5701
 #6 [ffff8800ba2ebd40] path_lookupat at ffffffff811b60d2
 #7 [ffff8800ba2ebde0] pick_next_task_fair at ffffffff810a4621
 #8 [ffff8800ba2ebe48] __schedule at ffffffff815171b1
 #9 [ffff8800ba2ebea0] do_nanosleep at ffffffff81516c42
#10 [ffff8800ba2ebec8] hrtimer_nanosleep at ffffffff8108d18f
#11 [ffff8800ba2ebf60] sys_nanosleep at ffffffff8108d2ad
#12 [ffff8800ba2ebf80] system_call_fast_compare_end at ffffffff8151adcd
    RIP: 00007fe5a96c1f2d  RSP: 00007ffdc0388dd0  RFLAGS: 00000202
    RAX: 0000000000000000  RBX: ffffffff8151adcd  RCX: 00007fe5a96c1f2d
    RDX: 0000000000000000  RSI: 0000000000000000  RDI: 00007ffdc0388e20
    RBP: 00007fe5a4acd3b8   R8: 0000000000462697   R9: 00007fe5a965399a
    R10: 00007fe5a99ac460  R11: 0000000000000293  R12: 0000000000989680
    R13: 0000000000000000  R14: ffffffff8108d2ad  R15: 0000000000000001
    ORIG_RAX: 0000000000000023  CS: 0033  SS: 002b
```

You can also print the memory values directly with x or rd command. 
* rd
  * -s: print symbols
  * many other options for physical address, user virtual address, slab info and so on
* x
  * simple: x/FMT address
  
At the top of stack, we can see the register values of user program as bt command shows.
```
crash> rd -s 0xffff8800ba2ebf80 16
ffff8800ba2ebf80:  system_call_fast_compare_end+16 0000000000000293 
ffff8800ba2ebf90:  00007fe5a99ac460 00007fe5a965399a 
ffff8800ba2ebfa0:  0000000000462697 0000000000000000 
ffff8800ba2ebfb0:  00007fe5a96c1f2d 0000000000000000 
ffff8800ba2ebfc0:  0000000000000000 00007ffdc0388e20 
ffff8800ba2ebfd0:  0000000000000023 00007fe5a96c1f2d 
ffff8800ba2ebfe0:  0000000000000033 0000000000000202 
ffff8800ba2ebff0:  00007ffdc0388dd0 000000000000002b 
crash> x/10ag 0xffff8800ba2ebf80
0xffff8800ba2ebf80:     0xffffffff8151adcd <system_call_fast_compare_end+16>    0x293
0xffff8800ba2ebf90:     0x7fe5a99ac460  0x7fe5a965399a
0xffff8800ba2ebfa0:     0x462697        0x0
0xffff8800ba2ebfb0:     0x7fe5a96c1f2d  0x0
0xffff8800ba2ebfc0:     0x0     0x7ffdc0388e20
```

Check stack of another process.
* ps: prints PID
* task PID: print task_struct and base stack address
* bt TASK PID: print callstack
* bt -S BASE-STACK PID: print callstack in stack
```
crash> ps
   PID    PPID  CPU       TASK        ST  %MEM     VSZ    RSS  COMM
      0      0   0  ffffffff8181a460  RU   0.0       0      0  [swapper/0]
     32     -1   0  ffff8800b97a2d20  IN   0.0       0      0  [ipv6_addrconf]
     33     -1   0  ffff8800b97a23d0  IN   0.0       0      0  [deferwq]
...
  10851    866   0  ffff880139624b20  IN   0.5  170440  25260  iperf
  10852    866   0  ffff880036f79430  RU   0.5  170440  25260  iperf
  16026     -1   0  ffff8801395b60d0  IN   0.1   27284   2696  screen
  40343     -1   0  ffff880139502ca0  IN   0.0       0      0  [kworker/0:2]
> 45075     -1   0  ffff880036c32150  RU   1.1  192840  58384  fio
  45076     -1   0  ffff8801395a09a0  IN   1.1  192840  58384  fio
  45077  45075   0  ffff880139614b60  RU   0.1  170340   3036  fio
  45088    866   0  ffff8800bb47f5b0  RU   0.5  170440  25260  iperf
crash> task 45088
PID: 45088  TASK: ffff8800bb47f5b0  CPU: 0   COMMAND: "iperf"
struct task_struct {
  state = 0, 
  stack = 0xffff88013a114000, 
  usage = {
...
crash> bt ffff8800bb47f5b0 45088
PID: 45088  TASK: ffff8800bb47f5b0  CPU: 0   COMMAND: "iperf"
 #0 [ffff88013a117f18] __schedule at ffffffff81517166
 #1 [ffff88013a117f78] retint_careful at ffffffff8151bb3c
    RIP: 00007f7abd0b66cf  RSP: 00007f7abbcf4e58  RFLAGS: 00000246
    RAX: 0000000000000000  RBX: 000000000000fe2e  RCX: 0000000000000000
    RDX: 0000000000020000  RSI: 00007f7aac0008f0  RDI: 0000000000000005
    RBP: ffffffff8151bb3c   R8: 0000000000000000   R9: 0000000000000000
    R10: 0000000000000000  R11: 0000000000000002  R12: 0000000000020000
    R13: 00007f7aac020910  R14: 00007f7aac0008f0  R15: 00000000f248bd70
    ORIG_RAX: ffffffffffffff10  CS: 0033  SS: 002b

PID: 45088  TASK: ffff8800bb47f5b0  CPU: 0   COMMAND: "iperf"
 #0 [ffff88013a117f18] __schedule at ffffffff81517166
 #1 [ffff88013a117f78] retint_careful at ffffffff8151bb3c
    RIP: 00007f7abd0b66cf  RSP: 00007f7abbcf4e58  RFLAGS: 00000246
    RAX: 0000000000000000  RBX: 000000000000fe2e  RCX: 0000000000000000
    RDX: 0000000000020000  RSI: 00007f7aac0008f0  RDI: 0000000000000005
    RBP: ffffffff8151bb3c   R8: 0000000000000000   R9: 0000000000000000
    R10: 0000000000000000  R11: 0000000000000002  R12: 0000000000020000
    R13: 00007f7aac020910  R14: 00007f7aac0008f0  R15: 00000000f248bd70
    ORIG_RAX: ffffffffffffff10  CS: 0033  SS: 002b
crash> bt -S 0xffff88013a114000 45088
PID: 45088  TASK: ffff8800bb47f5b0  CPU: 0   COMMAND: "iperf"
 #0 [ffff88013a114000] __schedule at ffffffff81517166
 #1 [ffff88013a1176f8] __sk_run_filter at ffffffff81439d11
 #2 [ffff88013a1179c8] vp_notify at ffffffffa00ea28d [virtio_pci]
 #3 [ffff88013a1179d0] virtqueue_kick at ffffffffa00e43f4 [virtio_ring]
 #4 [ffff88013a1179e8] start_xmit at ffffffffa0129a36 [virtio_net]
 #5 [ffff88013a117a30] dev_hard_start_xmit at ffffffff81426b87
 #6 [ffff88013a117aa0] vp_notify at ffffffffa00ea28d [virtio_pci]
 #7 [ffff88013a117aa8] virtqueue_kick at ffffffffa00e43f4 [virtio_ring]
 #8 [ffff88013a117ac0] start_xmit at ffffffffa0129a36 [virtio_net]
 #9 [ffff88013a117b08] dev_hard_start_xmit at ffffffff81426b87
#10 [ffff88013a117b58] sch_direct_xmit at ffffffff81447093
#11 [ffff88013a117b90] __dev_queue_xmit at ffffffff814270ec
#12 [ffff88013a117bd0] ip_finish_output at ffffffff814606f6
#13 [ffff88013a117c10] ip_queue_xmit at ffffffff81461bf1
#14 [ffff88013a117c50] skb_copy_datagram_iovec at ffffffff81419d76
#15 [ffff88013a117ca8] tcp_recvmsg at ffffffff8146c5a5
#16 [ffff88013a117d40] inet_recvmsg at ffffffff814948fa
#17 [ffff88013a117d60] sock_recvmsg at ffffffff8140c49a
#18 [ffff88013a117e78] put_prev_entity at ffffffff8109e9e7
#19 [ffff88013a117eb8] pick_next_task_fair at ffffffff810a4621
#20 [ffff88013a117f78] retint_careful at ffffffff8151bb3c
    RIP: 00007f7abd0b66cf  RSP: 00007f7abbcf4e58  RFLAGS: 00000246
    RAX: 0000000000000000  RBX: 000000000000fe2e  RCX: 0000000000000000
    RDX: 0000000000020000  RSI: 00007f7aac0008f0  RDI: 0000000000000005
    RBP: ffffffff8151bb3c   R8: 0000000000000000   R9: 0000000000000000
    R10: 0000000000000000  R11: 0000000000000002  R12: 0000000000020000
    R13: 00007f7aac020910  R14: 00007f7aac0008f0  R15: 00000000f248bd70
    ORIG_RAX: ffffffffffffff10  CS: 0033  SS: 002b
```

# find instruction that generated panic

To find what instruction generated panic, we can use
* dmeag: show kernel message
* disassemble: show function code
* bt: print callstack
* info registers: print register values

For example, bt command shows backtrace and dmesg shows kernel message.
* There are three stack pointer in the message
  * exception stack pointer: 0xffff88013ee04ea8
    * A stack for exception handlers
  * process stack pointer: 0xffff88013a17e000
    * A stack currently used
  * current stack base: 0xffff8800ba2e8000
    * A stack of the task

I found something weird
* CPL in CS is 0 but RPS in SS is 3 -> general protection fault happended before doublefault
* process stack pointer is not in stack area
  * RSP=ffff88013a17df88
  * When double-fault generated, RSP value was 0xffff88013a17df88
  * "bt -S 0xffff88013a17d000" command shows that it was wrong stack address
* Why bt shows "process stack pointer" is ffff88013a17e000?
  * async_page_fault does "sub 0x78, rsp"
  * ffff88013a17e000 - 0x78 = ffff88013a17df88
  * therefore bt think top address of stack was ffff88013a17e000 when async_page_fault was called
* check data in stack with x command
  * area 0xffff88013a17e000 ~ 0xffff88013a17e0b8 looks like stack
  * area 0xffff88013a17df88 ~ 0xffff88013a17e000 does not look like stack
* vtop command show how an address is mapped in page-table
  * FLAGS shows 0xffff88013a17df88 is mapped as RO
  * page has lru, so it might be file data or program code loaded from program file
* **therefore we found that stack was over-flowed in exception code and generated double-fault.**
```
crash> bt
PID: 45075  TASK: ffff880036c32150  CPU: 0   COMMAND: "fio"
 #0 [ffff88013ee04ea8] panic at ffffffff81511945
 #1 [ffff88013ee04f20] df_debug at ffffffff810503dd
 #2 [ffff88013ee04f30] do_double_fault at ffffffff81014f18
 #3 [ffff88013ee04f50] double_fault at ffffffff8151c868
    [exception RIP: async_page_fault+13]
    RIP: ffffffff8151cdfd  RSP: ffff88013a17df88  RFLAGS: 00010096
    RAX: 000000000000d0c0  RBX: 0000000000000001  RCX: ffffffff8151bac7
    RDX: 000000000000d0c0  RSI: 0000000000000000  RDI: ffff88013a17e038
    RBP: 0000000000006fd2   R8: 0000000000000000   R9: 0000000000000000
    R10: 0000000000000200  R11: 0000000000001000  R12: ffff88013ee03af0
    R13: 0000000000000001  R14: 0000000000006fd2  R15: ffff88013ee03c48
    ORIG_RAX: ffffffffffffffff  CS: 0010  SS: 002b
--- <DOUBLEFAULT exception stack> ---
 #4 [ffff88013a17df88] async_page_fault at ffffffff8151cdfd
bt: cannot transition from exception stack to current process stack:
    exception stack pointer: ffff88013ee04ea8
      process stack pointer: ffff88013a17e000
         current stack base: ffff8800ba2e8000
crash> bt -S ffff88013a17e000
PID: 45075  TASK: ffff880036c32150  CPU: 0   COMMAND: "fio"
bt: non-process stack address for this task: ffff88013a17e000
    (valid range: ffff8800ba2e8000 - ffff8800ba2ec000)
crash> disassemble async_page_fault 
Dump of assembler code for function async_page_fault:
   0xffffffff8151cdf0 <+0>:     data32 xchg %ax,%ax
   0xffffffff8151cdf3 <+3>:     data32 xchg %ax,%ax
   0xffffffff8151cdf6 <+6>:     data32 xchg %ax,%ax
   0xffffffff8151cdf9 <+9>:     sub    $0x78,%rsp
   0xffffffff8151cdfd <+13>:    callq  0xffffffff8151cf80 <error_entry>
   0xffffffff8151ce02 <+18>:    mov    %rsp,%rdi
   0xffffffff8151ce05 <+21>:    mov    0x78(%rsp),%rsi
   0xffffffff8151ce0a <+26>:    movq   $0xffffffffffffffff,0x78(%rsp)
   0xffffffff8151ce13 <+35>:    callq  0xffffffff81052470 <do_async_page_fault>
   0xffffffff8151ce18 <+40>:    jmpq   0xffffffff8151d030 <error_exit>
End of assembler dump.
crash> x/40ag 0xffff88013a17df88
0xffff88013a17df88:     0x136ccfe9a0274430      0x54419066666666e1
0xffff88013a17df98:     0xf3b3e8fb89485355      0x6875c085d231ffff
0xffff88013a17dfa8:     0x750000003af13d83      0x8b547900207b805a
0xffff88013a17dfb8:     0x13c90c0c7481053       0xd5148b48e4314500
0xffff88013a17dfc8:     0x102c8b48818e1a60      0xef8948e1171c8be8
0xffff88013a17dfd8:     0xdf8948e1172013e8      0x2043f6fffff691e8
0xffff88013a17dfe8:     0x8948ee8948167401      0x8948fffff3c5e8df
0xffff88013a17dff8:     0x8941e1171d52e8ef      0x0
0xffff88013a17e008:     0xffffffff8105247f <do_async_page_fault+15>     0x10
0xffff88013a17e018:     0x10087 0xffff88013a17e030
0xffff88013a17e028:     0x2b    0xffffffff8151ce18 <async_page_fault+40>
0xffff88013a17e038:     0xffff88013ee03c48      0x6fd2
0xffff88013a17e048:     0x1     0xffff88013ee03af0
0xffff88013a17e058:     0x6fd2  0x1
0xffff88013a17e068:     0x1000  0x200
0xffff88013a17e078:     0x0     0x0
0xffff88013a17e088:     0xd0c0  0xffffffff8151bac7 <native_irq_return_iret>
0xffff88013a17e098:     0xd0c0  0x0
0xffff88013a17e0a8:     0xffff88013a17e0e8      0xffffffffffffffff
0xffff88013a17e0b8:     0xffffffff8105247f <do_async_page_fault+15>     0x10
crash> vtop 0xffff88013a17df88
VIRTUAL           PHYSICAL        
ffff88013a17df88  13a17df88       

PML4 DIRECTORY: ffffffff81813000
PAGE DIRECTORY: 1af5067
   PUD: 1af5020 => 13961c063
   PMD: 13961ce80 => 13a073063
   PTE: 13a073be8 => 800000013a17d161
  PAGE: 13a17d000

      PTE         PHYSICAL   FLAGS
800000013a17d161  13a17d000  (PRESENT|ACCESSED|DIRTY|GLOBAL|NX)

      PAGE       PHYSICAL      MAPPING       INDEX CNT FLAGS
ffffea00044b5358 13a17d000 ffffffff8151ce18 ffff88013ee03c48  0 2b locked,error,uptodate,lru
```

We can find what instruction generated double-fault exception.
* info line: line number of source file
* disassemble: show instructions
```
crash> info line *0xffffffff8151cdfd
Line 1292 of "/build/linux-GU1w8g/linux-3.16.39/arch/x86/kernel/entry_64.S" starts at address 0xffffffff8151cdf3 <async_page_fault+3> and ends at 0xffffffff8151ce23 <machine_check+3>.
crash> disassemble async_page_fault+13
Dump of assembler code for function async_page_fault:
   0xffffffff8151cdf0 <+0>:     data32 xchg %ax,%ax
   0xffffffff8151cdf3 <+3>:     data32 xchg %ax,%ax
   0xffffffff8151cdf6 <+6>:     data32 xchg %ax,%ax
   0xffffffff8151cdf9 <+9>:     sub    $0x78,%rsp
   0xffffffff8151cdfd <+13>:    callq  0xffffffff8151cf80 <error_entry>
   0xffffffff8151ce02 <+18>:    mov    %rsp,%rdi
   0xffffffff8151ce05 <+21>:    mov    0x78(%rsp),%rsi
   0xffffffff8151ce0a <+26>:    movq   $0xffffffffffffffff,0x78(%rsp)
   0xffffffff8151ce13 <+35>:    callq  0xffffffff81052470 <do_async_page_fault>
   0xffffffff8151ce18 <+40>:    jmpq   0xffffffff8151d030 <error_exit>
End of assembler dump.
```

# other useful commands

* sys: system information
* bt -l: detail call-stack
* mach: machine information and registers (stack address of kernel, irq and etc)
* kmem -s: kernel memory usage

```
crash> sys
      KERNEL: ./vmlinux-4.4.131-1-pserver
    DUMPFILE: /srv/unpacking/ps303a-71-dump.201901151115  [PARTIAL DUMP]
        CPUS: 56
        DATE: Tue Jan 15 11:15:07 2019
...
crash> bt -l
PID: 7251   TASK: ffff883fcf5b6a00  CPU: 24  COMMAND: "stress-ng-vm"
 #0 [ffff881fff383980] machine_kexec at ffffffff81043fff
    /build/linux-pserver-future-8eZtJS/linux-pserver-future-4.4.131/include/linux/ftrace.h: 671
 #1 [ffff881fff3839d0] crash_kexec at ffffffff810df4b3
    /build/linux-pserver-future-8eZtJS/linux-pserver-future-4.4.131/kernel/kexec_core.c: 875
 #2 [ffff881fff383a98] oops_end at ffffffff81008628
 ...
crash> mach
          MACHINE TYPE: x86_64
           MEMORY SIZE: 255.9 GB
                  CPUS: 56
       PROCESSOR SPEED: 2394 Mhz
                    HZ: 100
             PAGE SIZE: 4096
   KERNEL VIRTUAL BASE: ffff880000000000
   KERNEL VMALLOC BASE: ffffc90000000000
   KERNEL VMEMMAP BASE: ffffea0000000000
      KERNEL START MAP: ffffffff80000000
   KERNEL MODULES BASE: ffffffffa0000000
     KERNEL STACK SIZE: 16384
        IRQ STACK SIZE: 16384
            IRQ STACKS:
                 CPU 0: ffff881fff200000
                 CPU 1: ffff883ffea00000
                 CPU 2: ffff881fff220000
                 CPU 3: ffff883ffea20000
                 CPU 4: ffff881fff240000
                 CPU 5: ffff883ffea40000
                 CPU 6: ffff881fff260000
                 CPU 7: ffff883ffea60000
                 CPU 8: ffff881fff280000
                 CPU 9: ffff883ffea80000
                CPU 10: ffff881fff2a0000
                CPU 11: ffff883ffeaa0000
                CPU 12: ffff881fff2c0000
                CPU 13: ffff883ffeac0000
                CPU 14: ffff881fff2e0000
                CPU 15: ffff883ffeae0000
                CPU 16: ffff881fff300000
                CPU 17: ffff883ffeb00000
                CPU 18: ffff881fff320000
                CPU 19: ffff883ffeb20000
                CPU 20: ffff881fff340000
                CPU 21: ffff883ffeb40000
                CPU 22: ffff881fff360000
                CPU 23: ffff883ffeb60000
crash> kmem -s
CACHE            NAME                 OBJSIZE  ALLOCATED     TOTAL  SLABS  SSIZE
ffff883fee4d8700 kvm_async_pf             136          0         0      0     8k
ffff883fee4d8600 kvm_vcpu               17088          0         0      0    32k
ffff883fee4d8500 kvm_mmu_page_header      168          0         0      0     8k
ffff883feed68000 nf_conntrack_1           312         11      1938     38    16k
ffff881ffec04400 SCTPv6                  1424          1        22      1    32k
ffff881ffec04500 SCTP                    1264          0         0      0    32k
ffff881ffec04600 UDPLITEv6               1080          0         0      0    32k
ffff881ffec04700 UDPv6                   1080          2       210      7    32k
ffff881ffec04800 tw_sock_TCPv6            272          0         0      0    16k
ffff881ffec04900 request_sock_TCPv6       320          0         0      0    16k
ffff881ffec04a00 TCPv6                   2056          4        90      6    32k
ffff880074934100 kcopyd_job              3312          0         0      0    32k
ffff880074934000 dm_uevent               2632          0         0      0    32k
ffff883ff04a0700 cfq_queue                232          0         0      0    16k
ffff883ff04a0600 bsg_cmd                  312          0         0      0    16k
ffff883ff04a0500 mqueue_inode_cache       864          1        36      1    32k
ffff883ff04a0400 nfsd4_stateids           176          0         0      0     8k
ffff883ff04a0300 nfsd4_files              264          0         0      0    16k
ffff883ff04a0200 nfsd4_openowners         440          0         0      0    16k
ffff883ff04a0100 nfs_direct_cache         328          0         0      0    16k
ffff883ff04a0000 nfs_inode_cache         1000          0         0      0    32k
ffff881ff1df0c00 hugetlbfs_inode_cache    576          2       112      2    32k
ffff881ff1df0b00 squashfs_inode_cache     616          0         0      0    32k
ffff881ff1df0a00 jbd2_journal_handle       48          0         0      0     4k
ffff881ff1df0900 jbd2_journal_head        112          0         0      0     8k
ffff881ff1df0800 jbd2_revoke_table_s       16          0         0      0     4k
ffff881ff1df0700 jbd2_revoke_record_s      32          0         0      0     4k
ffff881ff1df0600 ext2_inode_cache         792          0         0      0    32k
ffff881ff1df0500 ext4_inode_cache        1056          0         0      0    32k
ffff881ff1df0400 ext4_free_data            64          0         0      0     4k
```
