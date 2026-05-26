
# check if the RXE is loaded

```shell
❯ sudo lsmod | grep rdma_rxe
rdma_rxe              233472  0
ip6_udp_tunnel         16384  1 rdma_rxe
udp_tunnel             40960  1 rdma_rxe
ib_uverbs             225280  2 rdma_rxe,rdma_ucm
ib_core               598016  12 rdma_cm,ib_ipoib,rdma_rxe,rpcrdma,ib_srpt,iw_cm,ib_iser,ib_umad,ib_isert,rdma_ucm,ib_uverbs,ib_cm
```

- IB packets are encapsulated inside UDP IPv6 frames. RXE is short form for RCoE
- RCoEv2 is software implementation of RCoE (no need for separate NIC)

## Check per device summary

```shell
❯ ibstat
CA 'rxe0'
	CA type:
	Number of ports: 1
	Firmware version:
	Hardware version:
	Node GUID: 0x020000fffe000000
	System image GUID: 0x020000fffe000000
	Port 1:
		State: Active # logical state
		Physical state: LinkUp # physical stat
		Rate: 2.5 # link speed x link width. Here it is EDR
		Base lid: 0 # ideally should be valid lid
		LMC: 0 # Link mask control
		SM lid: 0 # identifies SM's LID
		Capability mask: 0x00090000
		Port GUID: 0x020000fffe000000 # GUID for the port
		Link layer: Ethernet # ideally it should be infiniband
```
lmc: how many addresses are assigned per port.

LMC:0 = 1
LMC:2 = 4
LMC:3 = 8

LMC = N, where N^2

Higher LMC values (1, 2, 3) allow multiple LIDs per port, used for LID-mask-control routing which enables path diversity.

## ibstatus

It is more human readable and than ibstat,
it is same info as ibstat but human readable

```shell
ibstatus
Infiniband device 'rxe0' port 1 status:
	default gid:	 fe80:0000:0000:0000:0200:00ff:fe00:0000 # 128 bit address, same as IPV6 for local communication only
	base lid:	 0x0
	sm lid:		 0x0
	state:		 4: ACTIVE # if do not remember what the number is, you can use this command
	phys state:	 5: LinkUp # same as above
	rate:		 2.5 Gb/sec (1X SDR) # also suggests what is the generation is
	link_layer:	 Ethernet
```

### GID
GID = subnet prefix + GUID
		= fe80:0000:0000:0000 + 0200:00ff:fe00:0000

`fe80:0000:0000:0000` is subnet prefix. Real IB fabrics use globally-routable GIDs when IB routers connect multiple subnets, where the subnet prefix is something other than fe80::.

## SMInfo

```shell

❯ sminfo

# ibwarn: [127648] get_smi_gsi_pair: Can't open SMI UMAD port (Input/output error) (rxe0:1)
# ibwarn: [127648] mad_rpc_open_port2: can't open UMAD port ((null):0)
# sminfo: iberror: failed: Failed to open '(null)' port '0'
```
### Info
UMAD stands for user mode MAD, since the program is loaded in user space.
SoftRCoE does not create /dev/infiniBand/umad0
But in real world if you see the above error, you need to check

- kernel module is loaded
- permissions to mount /dev/infiniBand/umad*
- Hardware supports MAD (which is the case here)

```shell
sudo lsmod | grep ib_umad # kernel module is loaded
tree /dev/infiniBand #- permissions to mount /dev/infiniBand/umad*

```


```shell

❯ ibv_devinfo -v | head -50
hca_id:	rxe0
	transport:			InfiniBand (0) # note this in production
	fw_ver:				0.0.0 # note this in production
	node_guid:			0200:00ff:fe00:0000
	sys_image_guid:			0200:00ff:fe00:0000
	vendor_id:			0xffffff # same as MAC, OUI allocated by NVIDIA
	vendor_part_id:			0
	hw_ver:				0x0
	phys_port_cnt:			1
	max_mr_size:			0xffffffffffffffff
	page_size_cap:			0xfffff000
	max_qp:				1048560 # note this in production
	max_qp_wr:			1048576
	device_cap_flags:		0x01223c76 # device capabilities. when you run ibperf, it queries what is available.
					BAD_PKEY_CNTR
					BAD_QKEY_CNTR
					AUTO_PATH_MIG
					CHANGE_PHY_PORT
					UD_AV_PORT_ENFORCE
					PORT_ACTIVE_EVENT
					SYS_IMAGE_GUID
					RC_RNR_NAK_GEN
					SRQ_RESIZE
					MEM_WINDOW
					MEM_MGT_EXTENSIONS
					MEM_WINDOW_TYPE_2B
	max_sge:			32
	max_sge_rd:			32
	max_cq:				1048576
	max_cqe:			32767
	max_mr:				524287
	max_pd:				1048576
	max_qp_rd_atom:			128
	max_ee_rd_atom:			0
	max_res_rd_atom:		258048
	max_qp_init_rd_atom:		128
	max_ee_init_rd_atom:		0
	atomic_cap:			ATOMIC_HCA (1) # supported atomic operations. compare-swap, fetch-and-add
	max_ee:				0
	max_rdd:			0
	max_mw:				524287
	max_raw_ipv6_qp:		0
	max_raw_ethy_qp:		0
	max_mcast_grp:			8192
	max_mcast_qp_attach:		56
	max_total_mcast_qp_attach:	458752
	max_ah:				32767
	max_fmr:			0
	max_srq:			917503
	max_srq_wr:			1048576
```


`max_qp: 1048560` is RXE's software-only ceiling. Real HCAs are typically limited to 16K–128K QPs by hardware resources.
It matters because workloads that create many QPs (e.g., GPUDirect at scale) need HCAs with high max_qp ceilings.
NCCL behavior changes when max_qp is exhausted.

```

❯ rdma link show
link rxe0/1 state ACTIVE physical_state LINK_UP netdev lo
```

```shell
❯ rdma resource show
0: rxe0: pd 2 cq 1 qp 1 cm_id 0 mr 0 ctx 0 srq 0

# pd = protection domain
# cq = completed queue
# qp = queue pair
# On an active workload, these numbers climb dramatically — which is itself a useful diagnostic for "is my RDMA application actually running."
```
