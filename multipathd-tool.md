multipathd is combination of multipathd daemon and multipathd command tool.

# multipathd daemon

activated multipathd/main.c: child()

create pidfile: location is ``#define DEFAULT_PIDFILE		"/" RUN_DIR "/multipathd.pid"``

print message: ``"--------start up--------")``

load configuration file /etc/multipath.conf

dm_init(): initialize libdevmapper

init_checkers(): libmultipath/checkers/ has many checker libraries which will be started by checkerloop()

create 4 threads
* ueventloop: receive event
  * call some udev_monitor_* functions, which are libudev APIs, to get events from udev
  * print "Forwarding 1 uevents" and add received events to uevq queue -> uevqloop-thread will handle the events
* uevqloop: process event
  * extract event from uevq queue
  * call uev_trigger() for each event
  * uev_add_map, uev_remove_map: if event is from kernel and dm-* and uev->action can be "change" or "remove".
  * uev_add_path, uev_remove_path, uev_update_path: uev->action can be "add, remove, change"
* uxlsnrloop: receive command from client
  * register handler of each command: "ADD+PATH"->cli_add_path()
  * make socket listener and receive commands via socket: uxsock_listen()
  * call uxsock_trigger() to handle command: parse client's command and call registered handler: eg) cli_add_path() for "multipathd add path sdX"
   * eg) cli_add_path(): calls ev_add_path()
* checkerloop:
  * loop repeatly
  * check every map
  * if a map waits an event but no event arrived, re-load the map with update_map()

Each map has its own thread to check state of the map periodically
* ev_add_path() creates waitevent()-thread
* ev_remove_map() and ev_remove_path(): stop the thread

wait for termination

Some event generates another event. "multipathd add path sda" do two things:
1. add sda and create map "26332393037343737" with scsi-id of sda 
1. send event to create dm-0
1. link the map and dm-0

```
root@pserver:~/test_multipathd# multipathd add path sdg
ok

root@pserver:~/multipath-tools# udevadm monitor
monitor will print the received events for:
UDEV - the event which udev sends out after rule processing
KERNEL - the kernel uevent


KERNEL[363417.255006] add      /devices/virtual/bdi/254:0 (bdi)
KERNEL[363417.255145] add      /devices/virtual/block/dm-0 (block)
UDEV  [363417.255305] add      /devices/virtual/bdi/254:0 (bdi)
UDEV  [363417.255495] add      /devices/virtual/bdi/254:0 (bdi)
KERNEL[363417.256297] change   /devices/virtual/block/dm-0 (block)
UDEV  [363417.261861] add      /devices/virtual/block/dm-0 (block)
UDEV  [363417.262077] add      /devices/virtual/block/dm-0 (block)
UDEV  [363417.287427] change   /devices/virtual/block/dm-0 (block)
UDEV  [363417.287524] change   /devices/virtual/block/dm-0 (block)
```

```
multipathd -d -v3

Oct 14 16:46:29 localhost multipathd: sdg: add path (operator)
Oct 14 16:46:29 localhost multipathd: sdg: mask = 0x1f
Oct 14 16:46:29 localhost multipathd: sdg: dev_t = 8:96
Oct 14 16:46:29 localhost multipathd: sdg: size = 204800
Oct 14 16:46:29 localhost multipathd: sdg: vendor = SCST_BIO
Oct 14 16:46:29 localhost multipathd: sdg: product = disk6
Oct 14 16:46:29 localhost multipathd: sdg: rev =  320
Oct 14 16:46:29 localhost multipathd: sdg: h:b:t:l = 4:0:0:6
Oct 14 16:46:29 localhost multipathd: sdg: tgt_node_name = iqn.2006-10.net.vlnb:tgt1
Oct 14 16:46:29 localhost multipathd: sdg: path state = running
Oct 14 16:46:29 localhost multipathd: sdg: 1024 cyl, 4 heads, 50 sectors/track, start at 0
Oct 14 16:46:29 localhost multipathd: 4:0:0:6: attribute vpd_pg80 not found in sysfs
Oct 14 16:46:29 localhost multipathd: failed to read sysfs vpd pg80
Oct 14 16:46:29 localhost multipathd: sdg: serial = c2907477
Oct 14 16:46:29 localhost multipathd: sdg: get_state
Oct 14 16:46:29 localhost multipathd: sdg: path_checker = tur (config file setting)
Oct 14 16:46:29 localhost multipathd: sdg: checker timeout = 30 ms (internal default)
Oct 14 16:46:29 localhost multipathd: sdg: state = up
Oct 14 16:46:29 localhost multipathd: sdg: getuid = "/lib/udev/scsi_id --whitelisted --device=/dev/%n" (config file default)
Oct 14 16:46:29 localhost multipathd: sdg: using deprecated getuid callout
Oct 14 16:46:29 localhost multipathd: formatted callout = /lib/udev/scsi_id --whitelisted --device=/dev/sdg
Oct 14 16:46:29 localhost multipathd: sdg: uid = 26332393037343737 (callout)
Oct 14 16:46:29 localhost multipathd: sdg: detect_prio = yes (config file default)
Oct 14 16:46:29 localhost multipathd: sdg: prio = const (internal default)
Oct 14 16:46:29 localhost multipathd: sdg: prio args = "" (internal default)
Oct 14 16:46:29 localhost multipathd: sdg: const prio = 1
Oct 14 16:46:29 localhost multipathd: 26332393037343737: user_friendly_names = no (internal default)
Oct 14 16:46:29 localhost multipathd: 26332393037343737: alias = 26332393037343737 (default to wwid)
Oct 14 16:46:29 localhost multipathd: sdg: ownership set to 26332393037343737
Oct 14 16:46:29 localhost multipathd: sdg: mask = 0xc
Oct 14 16:46:29 localhost multipathd: sdg: path state = running
Oct 14 16:46:29 localhost multipathd: sdg: get_state
Oct 14 16:46:29 localhost multipathd: sdg: state = up
Oct 14 16:46:29 localhost multipathd: sdg: const prio = 1
Oct 14 16:46:29 localhost multipathd: 26332393037343737: failback = "manual" (config file default)
Oct 14 16:46:29 localhost multipathd: 26332393037343737: path_grouping_policy = multibus (config file default)
Oct 14 16:46:29 localhost multipathd: 26332393037343737: path_selector = "round-robin 0" (config file default)
Oct 14 16:46:29 localhost multipathd: 26332393037343737: features = "0" (config file default)
Oct 14 16:46:29 localhost multipathd: 26332393037343737: hardware_handler = "0" (internal default)
Oct 14 16:46:29 localhost multipathd: 26332393037343737: rr_weight = "uniform" (internal default)
Oct 14 16:46:29 localhost multipathd: 26332393037343737: minio = 1 (config file setting)
Oct 14 16:46:29 localhost multipathd: 26332393037343737: no_path_retry = "fail" (config file default)
Oct 14 16:46:29 localhost multipathd: 26332393037343737: fast_io_fail_tmo = 5 (config file default)
Oct 14 16:46:29 localhost multipathd: 26332393037343737: retain_attached_hw_handler = yes (config file default)
Oct 14 16:46:29 localhost multipathd: 26332393037343737: deferred_remove = no (config file default)
Oct 14 16:46:29 localhost multipathd: 26332393037343737: delay_watch_checks = "off" (internal default)
Oct 14 16:46:29 localhost multipathd: 26332393037343737: delay_wait_checks = "off" (internal default)
Oct 14 16:46:29 localhost multipathd: 26332393037343737: remove queue_if_no_path from '0'
Oct 14 16:46:29 localhost kernel: [363417.255428] bio: create slab <bio-1> at 1
Oct 14 16:46:29 localhost multipathd: 26332393037343737: assembled map [1 retain_attached_hw_handler 0 1 1 round-robin 0 1 1 8:96 1]
Oct 14 16:46:29 localhost multipathd: 26332393037343737: load table [0 204800 multipath 1 retain_attached_hw_handler 0 1 1 round-robin 0 1 1 8:96 1]
Oct 14 16:46:29 localhost multipathd: 26332393037343737: disassemble map [1 retain_attached_hw_handler 0 1 1 round-robin 0 1 1 8:96 1 ]
Oct 14 16:46:29 localhost multipathd: 26332393037343737: disassemble status [2 0 0 0 1 1 E 0 1 0 8:96 A 0 ]
Oct 14 16:46:29 localhost multipathd: 26332393037343737: discover
Oct 14 16:46:29 localhost multipathd: 26332393037343737: searching paths for valid hwe
Oct 14 16:46:29 localhost multipathd: sdg: vendor = SCST_BIO
Oct 14 16:46:29 localhost multipathd: sdg: product = disk6
Oct 14 16:46:29 localhost multipathd: sdg: rev =  320
Oct 14 16:46:29 localhost multipathd: searching hwtable
Oct 14 16:46:29 localhost multipathd: 26332393037343737: no hardware entry found, using defaults
Oct 14 16:46:29 localhost multipathd: 26332393037343737: rr_weight = "uniform" (internal default)
Oct 14 16:46:29 localhost multipathd: 26332393037343737: failback = "manual" (config file default)
Oct 14 16:46:29 localhost multipathd: 26332393037343737: no_path_retry = "fail" (config file default)
Oct 14 16:46:29 localhost multipathd: 26332393037343737: flush_on_last_del = yes (config file default)
Oct 14 16:46:29 localhost multipathd: 26332393037343737: event checker started
Oct 14 16:46:29 localhost multipathd: sdg [8:96]: path added to devmap 26332393037343737
Oct 14 16:46:29 localhost multipathd: uevent 'add' from '/devices/virtual/block/dm-0'
Oct 14 16:46:29 localhost multipathd: Forwarding 1 uevents
Oct 14 16:46:29 localhost multipathd: uevent 'add' from '/devices/virtual/block/dm-0'
Oct 14 16:46:29 localhost multipathd: Forwarding 1 uevents
Oct 14 16:46:29 localhost multipathd: uevent 'change' from '/devices/virtual/block/dm-0'
Oct 14 16:46:29 localhost multipathd: Forwarding 1 uevents
Oct 14 16:46:29 localhost multipathd: dm-0: add map (uevent)
Oct 14 16:46:29 localhost multipathd: uevent 'change' from '/devices/virtual/block/dm-0'
Oct 14 16:46:29 localhost multipathd: Forwarding 1 uevents
Oct 14 16:46:29 localhost multipathd: dm-0: add map (uevent)
```

## uev_add/remove_map



## uev_add/remove/update_path



# multipathd command tool <> client for daemon

uxclnt()
* connect unix domain socket: {{{#define DEFAULT_SOCKET		"/org/kernel/linux/storage/multipathd"}}}
  * uxsock_timeout configuration
* send command
  * For example, if "multipathd show topology", send "show topology" to daemon
* receive result
  * ok or topology string
  * Or error message
* If it failed, use stract tool to find error

That's it. Very simple ;-)
