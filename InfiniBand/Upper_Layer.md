# Describe the Upper Layer
The upper layer is layer where applications start using the InfiniBand Fabric.
All layer below bring the data to this layer

- Physical and Link Layer - Roads and lanes e.g. packets are moving on the wire
- Network - routing between the cities
- transport layer - Reliable delivery trucks
- Upper layer - Actual Business services using the roads


# The main responsibilities of the Upper Layer

It has three main responsibilites

- Upper Layer Protocols (ULP)
- Management Service Protocols (MSP)
- Software transport verbs (STV)

## Common Upper Layer Protocols

Primary idea is to provide bridge to allow applications, storage protocols, TCP to use InfiniBand fabric.

### MPI 
MPI stands for Messaging Passing Interface, it is not protocol but application communication library used by
CUDA, NCCL and Pytorch.
The actual flow looks like 
--> CUDA/PyTorch/HPC - MPI - RDMA - InfiniBand Hardware

### IPoIB
Even though we are using InfinBand as Fabric, logging to node, communicating with nodes still need TCP/IP
These are established communication protocols and no need to rewrite for IB.
With IPoIB, IP packets are encapsulated in IB to make TCP/IP work on IB.

### Socket Direct Protocol
Like IP, some application uses socket i.e. application - socket - network stack - os stack - nic
With SDP - application - SDP - RDMA - HCA

In simple words, Socket based application can use RDMA to improve performance.


### Storage Protocols
The most industry proven protocols to access storage are

- Fiber Channel
- iSCSI
- NFS

#### SCSI RDMA Protocol (SRP)
So to make FC work on IB, we have SCSI RDMA Protocol (SRP).
In SRP, SCSI commands are carried by RDMA and providing benefits as

- high throughput
- low latency
- low CPU
- Zero Copy

Simple flow - when initiator wants to write on the storage, the target simply see this requests and reads directly from initiator
In other words, instead of initiator pushing the packets, target pulls the packet.
It is same concept of Memory Semantics where it is one-side communication eventually resulting into direct write into memory

#### iSCSI extensions for RDMA (iSER)

Here the CPU is offloaded for the CRC operations to the hardware and data transfer happens over RDMA

#### NFS over RDMA
Traditional NFS use TCP/IP stack, with RDMA TCP/IP stack is replaced by RDMA.
AI, HPC Computing and GPU Cluster heavily uses NFS over RDMA

## Summarize management service protocols

It is about how management traffic is using Infiniband fabric

There are two types of traffic

SMP - Subnet Management Packet
GMP - General Management Packet

SMP is like emergency lane on highway and is critical for SM to access nodes to carry out critical functions like
fabric initialization, node discovery, LID assignment, routing table calculation and programming switches with it and Topology changes
To achieve this SMP uses VL15 and QP0
Both are reserved for SMP and no flow control e.g. credit-based flow control is applied

On other hand, GMP is traffic from management but less critical e.g. device management, performance management
This traffic uses VL0 and QP1 and subject to flow control.

## explain software transport verb functionality

We have already learned about Channel and Memory Semantic.
These are application passing ACTIONs (send, receive, RDMA_Read/Write) instruction to RDMA
There is nothing new from that perspective here.
Channel semantic is send and receive and Memory semantic is RDMA_Read and RDMA_Write
All one should note is, InfiniBand does not define API code, that is defined by OFA under OFED

OFA = Open Fabric Association
OFED = Open Fabrics Enterprise Distribution







