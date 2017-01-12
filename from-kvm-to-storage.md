(current format is moniwiki)



= overview =


Refer to https://gurugio.kldp.net/wiki/wiki.php/linux_kernel_test_md_driver to analysis from syscall to md driver.

{{{
qemu -> md0 -> RAID1(dm-1 & dm-2) -> MULTIPATH(sd[abcd]) -> scsi -> srp -> infiniband HCA <--------> infiniband HCA -> srpt -> SCST & target_handler -> dm-3 -> RAID1(sda & sdb)
}}}

 * md & dm: /drivers/md
 * multipath: /drivers/md/dm-mpath.c
 * srp: drivers/infiniband/ulp/srp
 * infiniband HCA: drivers/infinibahd/core, drivers/infiniband/hw/mlx4
 * srpt: drivers/infiniband/ulp/srpt, scst/srpt/
 * scst target-handler: scst/dev_handlers



= mdadm =

create dev file: 9->major, 111->minor because name is md111
{{{
mknod("/dev/.tmp.md.20601:9:111", S_IFBLK|0600, makedev(9, 111)) = 0
open("/dev/.tmp.md.20601:9:111", O_RDWR|O_EXCL|O_DIRECT) = 4
}}}

ioctl sequence:
 * SET_ARRAY_INFO
 * ADD_NEW_DISK
 * RUN_ARRAY


= md =

md device management & sysfs entries
 * https://www.kernel.org/doc/Documentation/md.txt

start with raid0_personality
 * http://stackoverflow.com/questions/34588184/how-does-linux-md-driver-write-data-to-sata-disk

http://www.linuxjournal.com/article/2391?page=0,0


md_init:
 * create work-queue: "md", "md_misc", "md_reload"
 * blk_register_region() register probe callback: md_probe
 * md_probe() -> md_alloc() is called when /dev/mdX device file is created


md_alloc:
 * create "struct mddev" object for each mdX device file
  * disk->private_data = mddev
  * md_ioctl(struct block_device *bdev, ...) -> bdev->bd_disk->private_data == mddev
 * create queue: {{{mddev->queue = blk_alloc_queue(GFP_KERNEL);}}}
 * make request: {{{blk_queue_make_request(mddev->queue, md_make_request);}}}
 * register "block_device_operations": {{{disk->fops = &md_fops;}}}
  * md_open: do almost nothing
  * DO NOT have read/write because read/write are handled by VFS layer def_blk_fops.
{{{
static const struct block_device_operations md_fops =
{
	.owner		= THIS_MODULE,
	.open		= md_open,
	.release	= md_release,
	.ioctl		= md_ioctl,
#ifdef CONFIG_COMPAT
	.compat_ioctl	= md_compat_ioctl,
#endif
	.getgeo		= md_getgeo,
	.media_changed  = md_media_changed,
	.revalidate_disk= md_revalidate,
};
}}}


struct mddev:
 * struct md_personality *pers = raid1.c:struct md_personality raid1_personality
 * 

md_ioctl:
 * SET_ARRAY_INFO
  * initialize mddev
  * create bitmap at mddev->bitmap
 * ADD_NEW_DISK
  * get info from user
  * add_new_disk()
   * create "struct md_rdev" for child devices: md100 = dm-0 + dm-1 -> md_rdev stores infor for dm-0/1
   * bind dm-0/1 into md100
 * RUN_ARRAY
  * do_md_run()
  * mddev->pers->run() = raid1_personality.run
   * set mddev->private = conf
   * conf->raid_disks = mddev->raid_disks: the number of component disks 
   * conf->thread == raid1d() -> [md111_raid1]: each mdX device has its own thread
    * raid1d thread
     * 
  * set MD_RECOVERY_NEEDED at mddev->recovery -> it might need to start a resync/recover


IO
 * md_make_request
  * calls mddev->pers->make_request = raid1.c:make_request()


struct r1bio
 * struct bio *master_bio: original bio going to /dev/mdX
  * This is re-used for READ
 * {{{struct bio *bios[0]}}}: For WRITE, store multiple bios
  * if raid1 and mdX is composed with two disks, bios can be 4
  * 2 for two disks, 2 for replacements


raid1.c:make_request()
 * make_request() receives the original bio
 * set r1_bio->master_bio = bio
 * READ:
  * read_balance() decides disk to which bio should go -> '''do not read every component disks, read from only one disk among them'''
  * struct bio *read_bio = bio_clone_mddev(original-bio, GFP_NOIO, mddev)
  * read_bio->bi_bdev = mirror->rdev->bdev
   * conf->mirrors: setup_conf(), current working rdev of component disks
  * read_bio->bi_end_io = raid1_end_read_request
   * call bio_endio with bios
   * decrease the number of pending bio of the disk
  * read_bio->bi_rw = READ | do_sync
  * generic_make_request(read_bio) -> '''pass bio to target disk'''
 * WRITE:
  * find target disks
  * make clone-bio with bio_clone_mddev()
  * r1_bio->bios[0] = clone-bio, r1_bio->bios[1] = clone-bio
  * clonebio->bi_end_io = raid1_end_write_request
  * add cloned-bio into conf->pending_bio_list
  * wake up mddev->thread -> '''= raid1d(), md111_raid1 kernel-thread can be found with "px aux | grep raid1"'''
  * raid1()
   * flush_pending_writes() is core
   * lock conf->device_lock
   * get bio from conf->pending_bio_list
   * call generic_make_request with bio


= dm & dm_multipath =

dm: device mapper
 * What is the "device mapper"?: https://en.wikipedia.org/wiki/Device_mapper
  * The device mapper is a framework provided by the Linux kernel for mapping physical block devices onto higher-level virtual block devices. It forms the foundation of LVM2, software RAIDs and dm-crypt disk encryption, and offers additional features such as file system snapshots.
  * pass data from the virtual device to another block device
  * Data can be also modified in transition, which is performed, for example, in the case of device mapper providing disk encryption or simulation of unreliable hardware behavior.
 * How to use
  * issue ioctl to /dev/mapper/control device node via libdevmapper.so
 * features
  * software raid is a feature of device mapper
  * multipath: provide "load balancing" and "Path failover and recover"
   * https://en.wikipedia.org/wiki/Linux_DM_Multipath
   * including multipath, multipathd, devmap-name and kpartx
  * delay, zero and etc

dm device is registered as misc device, so MAJOR number is 252.

And minor number is device number. For example, dm-0 is (252,0) and dm-1 is (252,1).
{{{
root@storage1:/sys/block/md111# ls -l /dev/dm-0
brw-rw---- 1 root disk 252, 0 Sep 22 04:51 /dev/dm-0
root@storage1:/sys/block/md111# ls -l /dev/dm-1
brw-rw---- 1 root disk 252, 1 Sep 22 04:51 /dev/dm-1
root@storage1:/sys/block/md111# ls -l /dev/dm-2
brw-rw---- 1 root disk 252, 2 Sep 23 09:36 /dev/dm-2
}}}

driver/md/Makefile: CONFIG_DM_MULTIPATH build from dm-mpath.c & dm-path-selector.c
{{{
dm-multipath-y	+= dm-path-selector.o dm-mpath.o
obj-$(CONFIG_DM_MULTIPATH)	+= dm-multipath.o dm-round-robin.o
}}}

'''multipath.c is not CONFIG_DM_MULTIPATH, but CONFIG_MD_MULTIPATH. We use CONFIG_DM_MULTIPATH.'''

struct mapped_device
 * 

Initialize bio handling
 * {{{strict file_operations _ctl_fops.unlocked_ioctl = dm_ctl_ioctl() -> ctl_ioctl() -> dev_create() -> dm_create() -> alloc_dev() -> dm_init_md_queue()}}}

dm.c:alloc_dev(): create dm device
 * get minor number and create object of "struct mapped_device" type.
 * make queue: {{{md->queue = blk_alloc_queue(GFP_KERNEL);}}}
 * dm_init_md_queue(): {{{blk_queue_make_request(md->queue, dm_request);}}}
 * make gendisk: {{{md->disk = alloc_disk(1)}}}
 * create work-queue "kdmflush": {{{md->wq = alloc_workqueue("kdmflush", WQ_MEM_RECLAIM, 0);}}}
  * each dm device has its own kdmflush: If system has dm-0 and dm-1, there are two kdmflush threads
 * register "struct block_device_operations dm_blk_dop"
  * {{{.ioctl = dm_blk_ioctl,}}}

dm-mpath.c:dm_multipath_init()
 * registers target: {{{dm_register_target(&multipath_target);}}}
  * callbacks: map_rq, rq_end_io, ioctl and etc
  * multipath_target.ctr = multipath_ctr()
   * alloc_multipath()
    * create "struct multipath" object
    * target->private = "struct multipath"
 * workqueue: {{{[dm-0_kmpathd]}}}

dm device ioctl: via "struct target_type"
 * call "struct dm_target" -> "struct target_type" -> ioctl
  * {{{r = tgt->type->ioctl(tgt, cmd, arg);}}}
 * target is registered by dm-mpath.c:dm_multipath_init()
  * {{{dm_register_target(&multipath_target);}}}


'''CONCLUSION'''
 * IOCTL: dm.c:dm_blk_dops.ioctl = dm_blk_ioctl() -> {{{tgt->type->ioctl()}}} -> dm-mpath.c:multipath_ioctl()
  * "struct multipath" has the current path "struct pgpath" that has "struct block_device" for target path
  * call __blkdev_driver_ioctl(): "struct block_device"->bdev->bd_disk->fops->ioctl()
   * just pass ioctl command to the target path
 * bio: dm.c:dm_request()
  * insert bio into md->deferred list
  * activate dm->wq work-queue with md->work = dm_wq_work()
   * dm_wq_work(): call generic_make_request() with bio in dm->deferred list


Finally dm passes bio into scsi driver


= scsi =



= srp & srpt =

'''InfiniBand SCSI RDMA Protocol'''

https://en.wikipedia.org/wiki/SCSI_RDMA_Protocol
 * allows one computer to access SCSI devices attached to another computer via remote direct memory access (RDMA).
 * RDMA is only possible with network adapters that support RDMA in hardware. -> Infiniband HCAs
 * an SRP initiator implementation, an SRP target implementation and networking hardware supported by the initiator and target are needed. 
 * SRP initiator: drivers/infiniband/ulp/srp/
 * SRP target
  * drivers/infiniband/ulp/srpt/
  * Mellanox OFFED target(http://www.mellanox.com/pdf/products/pdk_dev/Gen2_SRPT_README.txt)
  * SCST srpt target
  * all are the same but which has the latest patches??

how to use: https://gurugio.kldp.net/wiki/wiki.php/infiniband


=== srpt ===

struct scst_tgt_template: registered in srpt_init_module()
{{{
	/*
	 * This function should detect the target adapters that
	 * are present in the system. The function should return a value
	 * >= 0 to signify the number of detected target adapters.
	 * A negative value should be returned whenever there is
	 * an error.
	 *
	 * MUST HAVE
	 */
	int (*detect)(struct scst_tgt_template *tgt_template);
	/*
	 * This function should free up the resources allocated to the device.
	 * The function should return 0 to indicate successful release
	 * or a negative value if there are some issues with the release.
	 * In the current version the return value is ignored.
	 *
	 * MUST HAVE
	 */
	int (*release)(struct scst_tgt *tgt);
	/*
	 * Forcibly close a session. Note: this function may operate
	 * asynchronously - there is no guarantee the session will actually
	 * have been closed at the time this function returns. May be called
	 * with scst_mutex held. Activity may be suspended while this function
	 * is invoked. May sleep but must not wait until session
	 * unregistration finished. Must return 0 upon success and -EINTR if
	 * the session has not been closed because a signal has been received.
	 *
	 * OPTIONAL
	 */
	int (*close_session)(struct scst_session *sess);
	/*
	 * Called to notify target driver that the command is being aborted.
	 * If target driver wants to redirect processing to some outside
	 * processing, it should get it using scst_cmd_get(). Since this
	 * function is invoked from the context of a task management function
	 * it runs asynchronously with the regular command processing finite
	 * state machine. In other words, it is only safe to invoke functions
	 * like scst_tgt_cmd_done() or scst_rx_data() from this callback
	 * function if the target driver owns the command and if appropriate
	 * measures have been taken to avoid that the target driver would
	 * invoke one of these functions from the regular command processing
	 * path. Note: if scst_tgt_cmd_done() or scst_rx_data() is invoked
	 * from this callback function these must be invoked with the
	 * SCST_CONTEXT_THREAD argument.
	 *
	 * OPTIONAL
	 */
	void (*on_abort_cmd)(struct scst_cmd *cmd);
	/*
	 * This function informs the driver that the corresponding task
	 * management function has been completed, i.e. all the corresponding
	 * commands completed and freed. No return value expected.
	 *
	 * This function is expected to be NON-BLOCKING.
	 *
	 * Called without any locks held from a thread context.
	 *
	 * MUST HAVE if the target supports task management.
	 */
	void (*task_mgmt_fn_done)(struct scst_mgmt_cmd *mgmt_cmd);
	/*
	 * Called to execute CDB. Useful, for instance, to implement
	 * data caching. The result of CDB execution is reported via
	 * cmd->scst_cmd_done() callback.
	 * Returns:
	 *  - SCST_EXEC_COMPLETED - the cmd is done, go to other ones
	 *  - SCST_EXEC_NOT_COMPLETED - the cmd should be sent to SCSI
	 *	mid-level.
	 *
	 * If this function provides sync execution, you should set
	 * exec_sync flag and consider to setup dedicated threads by
	 * setting threads_num > 0.
	 *
	 * Dev handlers implementing internal queuing in their exec() callback
	 * should call scst_check_local_events() just before the actual
	 * command's execution (i.e. after it's taken from the internal queue).
	 *
	 * OPTIONAL, if not set, the commands will be sent directly to SCSI
	 * device.
	 */
	int (*exec)(struct scst_cmd *cmd);
}}}
{{{
/* SCST target template for the SRP target implementation. */
static struct scst_tgt_template srpt_template = {
	.name				 = DRV_NAME,
	.sg_tablesize			 = SRPT_DEF_SG_TABLESIZE,
	.max_hw_pending_time		 = RDMA_COMPL_TIMEOUT_S,
#if !defined(CONFIG_SCST_PROC)
	.enable_target			 = srpt_enable_target,
	.is_target_enabled		 = srpt_is_target_enabled,
	.tgt_attrs			 = srpt_tgt_attrs,
	.sess_attrs			 = srpt_sess_attrs,
#endif
#if defined(CONFIG_SCST_DEBUG) || defined(CONFIG_SCST_TRACING)
	.default_trace_flags		 = DEFAULT_SRPT_TRACE_FLAGS,
	.trace_flags			 = &trace_flag,
#endif
	.detect				 = srpt_detect,
	.release			 = srpt_release,
	.close_session			 = srpt_close_session,
	.xmit_response			 = srpt_xmit_response,
	.rdy_to_xfer			 = srpt_rdy_to_xfer,
	.on_abort_cmd			 = srpt_on_abort_cmd,
	.on_hw_pending_cmd_timeout	 = srpt_pending_cmd_timeout,
	.on_free_cmd			 = srpt_on_free_cmd,
	.task_mgmt_fn_done		 = srpt_tsk_mgmt_done,
	.get_initiator_port_transport_id = srpt_get_initiator_port_transport_id,
	.get_scsi_transport_version	 = srpt_get_scsi_transport_version,
};
}}}


srpt_init_module
 * scst_register_target_template(): register SCST target template for SRP target implementation
  * exec callback is not set in srpt_template, so commands will be sent directly to SCSI device
 * ib_register_client(): set up IB connection
 * rdma_bind_addr() & rdma_listen(): listen on RDMA device


static int srpt_compl_thread(void *arg)
 * srpt_cm_handler() (IB connection manager) calls srpt_ib_cm_req_recv()->srpt_cm_req_recv() for IB_CM_REQ_RECEIVED(=SRP_LOGIN_REQ) event
  * srpt_cm_req_recv() 
   * create "struct srpt_rdma_ch"
   * creates kthread {{{[srpt_mlx4_0-1]}}} with "struct srpt_rdma_ch" object
 * 


{{{
static int srpt_compl_thread(void *arg)
{
	enum { poll_budget = 65536 };
	struct srpt_rdma_ch *ch;

	/* Hibernation / freezing of the SRPT kernel thread is not supported. */
	current->flags |= PF_NOFREEZE;

	ch = arg;
	BUG_ON(!ch);

	while (ch->state < CH_LIVE) {

if ch->state is CH_CONNECTING, kthread goes to sleeps until ch->state become CH_LIVE.
/**
 * enum rdma_ch_state - SRP channel state.
 * @CH_CONNECTING:    QP is in RTR state; waiting for RTU.
 * @CH_LIVE:	      QP is in RTS state.
 * @CH_DISCONNECTING: DREQ has been sent and waiting for DREP or DREQ has
 *                    been received.
 * @CH_DRAINING:      DREP has been received or waiting for DREP timed out
 *                    and last work request has been queued.
 * @CH_DISCONNECTED:  Last completion has been received.
 */
enum rdma_ch_state {
	CH_CONNECTING,
	CH_LIVE,
	CH_DISCONNECTING,
	CH_DRAINING,
	CH_DISCONNECTED,
};

		set_current_state(TASK_INTERRUPTIBLE);
		if (srpt_process_completion(ch, poll_budget) >= poll_budget)
			cond_resched();
		else
			schedule();
	}

	srpt_process_wait_list(ch);


NOW ch->state is CH_LIVE. So kthread do srpt_process_completion() until ch->state become CH_DISCONNECTED.

WHY state is TASK_INTERRUPTIBLE???


	while (ch->state < CH_DISCONNECTED) {
		set_current_state(TASK_INTERRUPTIBLE);
		if (srpt_process_completion(ch, poll_budget) >= poll_budget)
			cond_resched();
		else
			schedule();
	}
	set_current_state(TASK_RUNNING);

thread state is TASK_RUNNING.


	TRACE_DBG("%s-%d: about to invoke scst_unregister_session()",
		  ch->sess_name, ch->qp->qp_num);
	scst_unregister_session(ch->scst_sess, false, srpt_unreg_sess);



Sleep until be stopped with TASK_RUNNING state.


	while (!kthread_should_stop())
		schedule_timeout(DIV_ROUND_UP(HZ, 10));

	return 0;
}
}}}


1. What does srpt_process_completion() do?
 * srpt_process_completion()->ib_poll_cq()->mlx4_ib_poll_cq()
  * poll a CQ for completion
  * check how many commands in CQ are finished?
 * srpt_process_one_compl()->srpt_process_rcv_completion() & sprt_process_send_completion()
  * process an IB receive/send completion: I DON'T KNOW the detail
 * '''WHY those completion is done with TASK_INTERRUPTIBLE state?'''


2. Where does srpt kthread be stopped?
 * ch->thread = srpt_process_completion()
 * srpt_unreg_sess() calls kthread_stop(ch->thread)
 * srpt_unregister_session() register srpt_unreg_sess() callback into sess->unreg_done_fn
  * {{{sess->unreg_done_fn = unreg_done_fn;}}}
  * '''scst_rx_mgmt_fn_lun()'''
   * '''Abort all outstanding commands and clear reservation, if necessary'''
   * '''This can be stuck!!!'''
  * {{{scst_sess_put(sess);}}}: this should make sess->unreg_done_fn be called. -> scst_global_mgmt_thread()-> scst_free_session_callback()->scst_free_session() calls sess->unreg_done_fn()
  * wait is false, so wait_for_completion() is not called
 * So there are two suspects
  * 1. scst_rx_mgmt_fn_lun() did not finish
  * 2. scst_sess_put() could not call sess->unreg_done_fn callback.

sess->unreg_done_fn is called in scst_global_mgmt_thread()
 * scst_sess_put()->scst_sched_session_free()->wakt_up(&scst_mgmt_waitQ) ----> scst_global_mgmt_thread(): wait_event_locked(scst_mgmt_waitQ,...)
 * scst_global_mgmt_thread() is the global thread for all sessions..
 * If there were about 100 sessions, could it be bottle-neck?




= scst =


srpt is the target of srp connection

SCST is the target of SCSI

srp protocol is the link layer of SCSI & SCST connection

scst read/write "scsi middle level"

how to use: https://gurugio.kldp.net/wiki/wiki.php/linux_kernel_scst

how to use iscsi(another scsi target driver): https://gurugio.kldp.net/wiki/wiki.php/iscsi

=== dev_handler: scst_vdisk ===

{{{
static struct scst_dev_type vdisk_blk_devtype = {
	.name =			"vdisk_blockio",
	.type =			TYPE_DISK,
	.threads_num =		1,
	.parse_atomic =		1,
	.dev_done_atomic =	1,
#ifdef CONFIG_SCST_PROC
	.no_proc =		1,
#endif
	.attach =		vdisk_attach,
	.detach =		vdisk_detach,
	.attach_tgt =		vdisk_attach_tgt,
	.detach_tgt =		vdisk_detach_tgt,
	.parse =		non_fileio_parse,
	.exec =			non_fileio_exec,
	.task_mgmt_fn_done =	vdisk_task_mgmt_fn_done,
	.get_supported_opcodes = vdisk_get_supported_opcodes,
	.devt_priv =		(void *)blockio_ops,
#ifndef CONFIG_SCST_PROC
	.add_device =		vdisk_add_blockio_device,
	.del_device =		vdisk_del_device,
	.dev_attrs =		vdisk_blockio_attrs,
	.add_device_parameters =
		"blocksize, "
		"filename, "
		"nv_cache, "
		"read_only, "
		"removable, "
		"rotational, "
		"thin_provisioned, "
		"tst, "
		"write_through",
#endif
#if defined(CONFIG_SCST_DEBUG) || defined(CONFIG_SCST_TRACING)
	.default_trace_flags =	SCST_DEFAULT_DEV_LOG_FLAGS,
	.trace_flags =		&trace_flag,
	.trace_tbl =		vdisk_local_trace_tbl,
#ifndef CONFIG_SCST_PROC
	.trace_tbl_help =	VDISK_TRACE_TBL_HELP,
#endif
#endif
};
}}}

struct struct scst_dev_type vdisk_blk_devtype
 * http://scst.sourceforge.net/scst_pg.html: 5.1 Structure scst_dev_type
 * int (*exec) (struct scst_cmd *cmd)
  * called to execute CDB. Useful, for instance, to implement data caching. The result of CDB execution is reported via cmd->scst_cmd_done() callback. Returns:
  * SCST_EXEC_COMPLETED - the cmd is done, go to other ones
  * SCST_EXEC_NOT_COMPLETED - the cmd should be sent to SCSI mid-level.
  * If this function provides sync execution, you should set exec_sync flag and consider to setup dedicated threads by setting threads_num > 0. Optional, if not set, the commands will be sent directly to SCSI device. If this function is implemented, scst_check_local_events() shall be called inside it just before the actual command's execution.
  * non_fileio_exec()
   * {{{vdisk_op_fn *ops = virt_dev->vdev_devt->devt_priv}}}
   * vdisk_op_fn blockio_ops set callback for each command, for instance, READ_6, WRITE_6 and etc.
    * READ_* -> blockio_exec_read
    * WRITE_* -> blockio_exec_write
   * call vdev_do_job with ops
   * vdev_do_job() -> ops[opcode] = blockio_exec_read or blockio_exec_write -> blockio_exec_rw -> submit_bio
   * blockio_exec_rw()
    * create bio and call submit_bio
    * bio->bi_end_io = blockio_endio




= references =


block device overview
 * http://free-electrons.com/doc/block_drivers.pdf
 * https://yannik520.github.io/blkdevarch.html

storage stack diagram
 * https://www.thomas-krenn.com/en/wiki/Linux_Storage_Stack_Diagram

lwn articles
 * https://lwn.net/Kernel/Index/#Block_layer
 * https://lwn.net/Kernel/Index/#Device_drivers-Block_drivers
 * https://lwn.net/Kernel/Index/#Filesystems
 * https://lwn.net/Kernel/Index/#Multipath_IO
 * https://lwn.net/Kernel/Index/#SCSI
 * https://lwn.net/Kernel/Index/#Solid-state_storage_devices
 * md/dm layer: https://lwn.net/Kernel/Index/#RAID

