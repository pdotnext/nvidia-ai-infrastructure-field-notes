# InfiniBand - Transport Layer

Transport Layer is the most important layer among other layers.

This layer allows end to end communication of application which bypasses CPU completely ( it also referred as zero copy)

It is done by establishing virtual lanes among both the applications address space.

## Queue Pairs (QPs)
QPs are applications interface to the Infiniband fabric.

Application maps these QPs to their address space.

So when application wish to send a request e.g. RDMA_Write, it create Write Request

and send it to Write Queue. Write queue is converted into Work Queue Element (WEQ)

and HCA processes it and confirm back that it is completed using Completed Queue Element (CQE)

They are referred as pairs because for every instance of communication

there will be send and receive queue created.

Queue is created at the both end of the application.

## Transport Service Types
There are four transport service types:
* Reliable Connected
* UnReliable Connected
* Reliable Datagram
* UnReliable Datagram

### Reliable Connected
Reliable Connected are mostly used for the following reasons:

* High performance
* One to one connection, so more connection more memory
* Ensures Delivery, Integrity and Ordering. This is most critical requirement for allreduce operation
* Can send any message size as segmentation and reassembly automatically done by respective HCA
* Integrity and Ordering is achieved using Packet Sequence Number
* PSN tracks the packet order, which ensures there there is no duplication, packet loss or missing piece
* PSN also uses counter (in plain englich timeout). when the timeout is reached and ACK does not come, it retransmits
* PSN works on ACK and NACK. NACK leads to retransmission. Retransmission here might be fault in fabric esp in cable, dirty fiber

### Unreliable Datagram

Opposite is Unreliable Datagram:

* It is one to many
* Cannot guarantee Delivery, integrity and ordering
* Cannot cross MTU size

## Semantics

There are two types of semantics:
* Channel
* Memory

### Channel Semantics
Channel semantics has following important characteristic:

* Both send and receiver are active, in other word send/receive
* It is synchronous communication
* Sender has no information about receiver addresses/memory layout and receiver decides where data goes
* Receiver sends in advance called pre-post buffer

**Why we need it:**
* Infrequent communication with smaller data size
* Control plane communication - SM to SMA communication using unreliable datagram on VL15
* Storage protocol commands - where command header size differr, receiver decides where to place it
* Connection setup - where no data is shared earlier, like chicken and egg
* Where receiver has idea data is coming, but not much information is available


### Memory Semantics
Memory Semantics has following characteristics:
* RDMA Read/Write verbs
* Asynchronous
* It single side communication. Let me explain what is single side means:
* Receiver shares buffer with HCA
* Then receiver shares address space and permission keys directy with the sender
* And very important, no pre-posting of buffer

* *This is one time task:* Sender use the information send it directly to receiver, completely by passing CPU

**Use Cases:**
* Data plane
* AI and HPC workload because GPUs are aware of what data is expected e.g. gradient data is coming, its size and where to send it e.g. Tensor
When the data information is already there and no change is expected, then there is no need for receiver
keep sending the same address information and use CPUs. Imagine this operation happening frequently in a minute

---

## Partitions (PKeys)

As learned earlier, Partition ensures isolation, security a requirement for multi-tenant workload.

It is isolation of data within a subnet.

Partition key concept was discussed earlier, now we go one level deep here:

* Partition key is 16 bit in value and it part of data packet (in BTH)
* Default partition key is always present and has value 0x7FFF, you cannot modify or delete this P_KEY and every port is full member of this partition
* The left most bit of the p_key indicates if it is full or limited membership e.g. 1= Full and 0 = limited.

Partition defines which nodes are allowed to communicate with each other.

Partition keys are created by SM and assigned to each port. From here on, HCA ensures nodes communicate only based on the P_KEY.

Each port has a PKey table like LFT it is programmed by SM. Like LFT lookup happens, in the same sense Pkey table lookup happens at HCA level, for this is not a lookup but a check if the PKey is allowed.

When PKey is not present, it is not allowed and it is silently dropped but PKey violation is tracked in counters/trapped.

Next important concept in partition is Membership. There are two types of membership:

* Full membership
* Limited membership

> **Analogy to remember:** Think of Hotel staff. They can talk to each other.
> And Staff also needs to talk to their Guests and Guests as well need to talk to Staff.
> But Guests cannot barge into another guest's room. Staff is Full Membership and Guests are limited Membership.

To explain this in better way, lets walk through an example.

We have 4 node GPU cluster, with 2 node storage cluster and 2 SM nodes.

We know storage nodes (S1 and S2) always needs to communicate with each other.

Hence they can be configured in full membership. This we define as **partition-1**.

Now 4 node GPU cluster also needs to communicate with storage, hence we create

with storage a limited membership. This means S1,S2 has full membership and N1-N4 has limited membership.

So N1-N4 can talk to storage but N1-N4 cannot talk with each other. This we define as **partition-2**.

Finally, GPU Cluster must communicate each other for AllReduce operation.

For this we can create full membership for N1-N4 and create dedicate partition as **partition-3**.

And last and least, the default partition which is automatically created, can be used for management traffic.

Hence we assign full membership for management nodes and limited membership for all

other nodes i.e. N1-N4 and S1,S2.

This we name it as **partition-4**. In partition-4 management nodes can talk with each other and also with other nodes.

But the other nodes cannot talk with each other but only with management nodes.

Every time message is send, it attaches P_KEY with it, HCA ensures that P_KEY is allowed on HCA.

e.g. N1 wish to communicate with N2, it will attached P_KEY=partition-2. Only then HCA will allow communication.

If N1 wish to communicate with S1 or S2, it will attach P_KEY=partition-2.

### 16 Bit key

This is good to know information.

It is 16 bit number. It is split into two parts:

* **Bits 0-14:** Partition ID
* **Bit 15:** The last bit, known as Most Significant Bit (MSB) or high order bit is reserved for toggling membership

Lets assume Partition Number 5 i.e. `0000 0000 0000 0101` (binary)

| Membership Type | MSB | Partition Number (0-14) | Final 16-Bit Pkey | Hex Value |
| --- | --- | --- | --- | --- |
| Limited Membership | 0 | 000 0000 0000 0101 | 0000 0000 0000 0101 | 0x0005 |
| Full Membership | 1 | 000 0000 0000 0101 | 1000 0000 0000 0101 | 0x8005 |

Here Hex column is added because hex number are much easier to read compared to binary.

If the PKey starts are 0 (0x0005) it is limited membership and if it is starting with 1 (0x8005), then it is Full.

> **Note:** `0x` in front of the number says "Hallo, I'm hex".
> And, whenever you see an InfiniBand PKey start with the number 8 (like 0x8005), it is just a shortcut telling you: "The high-order bit is flipped to 1, which means this is a Full Member."
