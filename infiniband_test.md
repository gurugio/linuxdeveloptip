# configuration

## server
```
root@ib2:~# ibstat
CA 'mlx4_0'
	CA type: MT26418
	Number of ports: 2
	Firmware version: 2.9.1000
	Hardware version: b0
	Node GUID: 0x0002c9030008ca16
	System image GUID: 0x0002c9030008ca19
	Port 1:
		State: Active
		Physical state: LinkUp
		Rate: 20
		Base lid: 1
		LMC: 0
		SM lid: 1
		Capability mask: 0x0251086a
		Port GUID: 0x0002c9030008ca17
		Link layer: InfiniBand
	Port 2:
		State: Active
		Physical state: LinkUp
		Rate: 20
		Base lid: 3
		LMC: 0
		SM lid: 3
		Capability mask: 0x0251086a
		Port GUID: 0x0002c9030008ca18
		Link layer: InfiniBand
```
## client
```
root@ib1:/sys/module/ib_ipoib# ibstat
CA 'mlx4_0'
	CA type: MT26428
	Number of ports: 2
	Firmware version: 2.9.1000
	Hardware version: b0
	Node GUID: 0x0002c903004ed0b2
	System image GUID: 0x0002c903004ed0b5
	Port 1:
		State: Active
		Physical state: LinkUp
		Rate: 20
		Base lid: 2
		LMC: 0
		SM lid: 1
		Capability mask: 0x02510868
		Port GUID: 0x0002c903004ed0b3
		Link layer: InfiniBand
	Port 2:
		State: Active
		Physical state: LinkUp
		Rate: 20
		Base lid: 4
		LMC: 0
		SM lid: 3
		Capability mask: 0x02510868
		Port GUID: 0x0002c903004ed0b4
		Link layer: InfiniBand
```

# ibping

server:
```
root@ib2:~# ibping -S
```

client
```
root@ib1:/sys/module/ib_ipoib# ibping 1
Pong from ib2.(none) (Lid 1): time 0.418 ms
Pong from ib2.(none) (Lid 1): time 0.560 ms
Pong from ib2.(none) (Lid 1): time 0.560 ms
Pong from ib2.(none) (Lid 1): time 0.558 ms
Pong from ib2.(none) (Lid 1): time 0.561 ms
```

# ib_send_bw

Data exchange between HCA

install packages
* rdmacm-utils
* ibutils
* ibverbs-utils
* ibverbs-driver-mlx4

server:
* ``# ib_send_bw -a -d mlx4_0 -i 1``
* -a: all sizes
* -c UD: connection type
* -d: IB device
* -i: IB port

client:
* ``# ib_send_bw -a -d mlx4_0 -i 1 -F --report_gbits 192.168.68.101``
* 192.168.68.101 is ip address of ib interface

