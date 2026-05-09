# Network Layer and Routing — InfiniBand Unit 5

## What is the Network Layer

Network layer sits above Link layer.
Network layer as in TCP/IP is L3 and it is where routing happens
Routing in plain text is traffic passing across the Subnet
Fabric in InfiniBand consists of devices e.g. switches, links and routers
Subnet Manager initializes this fabric and assign unique Subnet Identifier.

## How Cross-Subnet Routing Works
When packet needs to pass cross subnet it needs GID and it included in Global Route Header
GID is combination 64 bit subnet prefix (destination) + 64 GUID (destination port) = 128 bit address
This GID is wrapped under GRH and send to the router.

### Default Link-Local GID — FE80 Prefix
Though I have learned that every device e.g. HCA Port has GUID but
it has also GID with FE80, a prefix same as IPV6 to denote local address.

## Why Multiple Subnets
There are many consideration when more than one subnet makes sense

### Blast Radius
First and primary reason we look for multiple subnet is to reduce blast radius e.g. changes planned or unplanned do not impact other subnets
Under unplanned change could switch or link failure this will result in entire fabric re-routing and topology.
Keep fabric smaller will reduce this risk


### Independent Subnet Manager
This means you can independently modify Subnet e.g. QoS, update driver or firmware instead doing this for entire subnet
Subnets can divided to reduce operational risk for modifying parameters inside subnet.

### Scale Beyond 48,000 Nodes
Another reason is when we cross 48,0000 nodes in a subnet


## The Three Address Types in InfiniBand

### GUID — Hardware Identity
GUID is burned on the device, it is permanent and similar concept as MAC
GUID are assigned by Manufacture e.g. NVIDIA

### LID — Local Subnet Routing
Local IDs are assigned by SM during initialization and are temporary.
They can change during topology change and are not unique globally

### GID — Global Cross-Subnet Routing
GID is only required when Packet needs to cross subnet boundary i.e inter-subnet routined
GID are unique globally.

## LIDs vs GIDs — Two Routing Scopes
LID is local in scope and always present in LRH i.e. within Subnet or intra-subnet.
Linear Forward Table (LFT) uses destination LID to send the traffic via right port.
While GIDs only present when packets needs to travel inter-subnet and present in GRH
and GID is used by router.

## Multicast in InfiniBand
Another important topic which was covered is Multicast.
Multicast is one to many transmission, instead of sending packet to single device, 
it is send to multiple devices in one go, there by reducing the overall load in the fabric.
In multicast, a group is created and these packets are send to this group in one go.


