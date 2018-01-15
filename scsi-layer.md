note for scsi/sd layer


scsi_register_driver:


initialize: sd_probe_async -> 

IO handling: scsi_request_fn -> scsi_dispatch_cmd -> ata_scsi_queuecmd
