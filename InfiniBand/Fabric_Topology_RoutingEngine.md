# InfiniBand Topologies and Routing Engine

So far, I have learned all layers (Physical, data, network, transport, and upper layer)  and their functions.
Last week, I learned about fabric initialization and fabric monitoring, SM functions, SM elections, and its high availability considerations. Now, the focus shifts to packet routing that the SM should use based on the physical topology.

Fabric topology directly determines which routing engine the SM should use to calculate routes.

## Fabric Topology Concepts

Common Terms - Network Topology, Spine/core, Leaf/edge
Internal links = links connecting from a leaf to the spines
External links = links connecting the leaf to the HCA/servers
Super-spine = added when needed - spine to super spine

## Leaf-Spine advantages

Deterministic latency - every host can reach another host with the same number of hops, irrespective of source and destination

3 hops = leaf - spine - leaf (2-tier)
5 hops = leaf - spine - super-spine -spine -leaf (3-tier)

* Deterministic latency because predictable Hop Count
* scalability
* redundancy
* better congestion distribution

## InfiniBand Fabric Topologies

Physical topology = how devices are physically connected to each other, and provides various paths to reach the destination.
Logical topology = how data moves between these devices by utilizing the path provided by physical topology. It could be an optimized path, a shortest path, or a safe path to the destination.

### Topology selection criteria

When deciding which topology to use, the question should be asked.

Which topology best matches the workload communication pattern and business constraints?

Memory Trick BFRAP - Budget, Future Growth, Reliability, Availability, and Performance

## Fat-tree

As links go up from host to leaf and leaf to spine, bandwidth gets thicker. I have read about Zero oversubscription in the past weeks. With the oversubscription ratio formula, it now fills the gap.
Oversubscription Ratio = downlink bandwidth / uplink bandwidth

Here, downlink bandwidth = bandwidth x number of links divided by uplink bandwidth x number of uplinks
When the downlinks and uplinks are the same, then the oversubscription ratio is 1
i.e., 1:1 in Fat tree, and also referred to as non-blocking fabric.

Think of this water flowing from smaller pipes into fat pipes. So, as the water flows,
There are no blocks; rather, bigger pipes make it easier to make water flow.

But, the blocking fabric is exactly opposite, i.e., water moves from the fat pipes into the smaller pipes.
There might be some blocking, i.e., less throughput.

Server workload or non-AI workload is always designed with an oversubscription ratio, i.e., 2:1 or more.
But AI workloads are almost never, in fact, for DGX SuperPod it is 1:1 i.e. Zero oversubscriptions, non-blocking full fat-tree by design. But for storage fabric, it runs at 4:3 oversubscription (mixed workload mentioned below).

Non-Blocking use case

* AI Training
* HPC
* GPU to GPU communication.
* Any workload which needs low-latency (microsecond) , high-throughput,  and bandwidth

Blocking use cases

* Inference
* for management, mixed workload, or fine-tuning existing model

---

## Dragonfly+

* A group of nodes connected with each other using leaf-spine
* Create multiple groups and connect one group to another, and full mesh connectivity is established.
* Adaptive Routing is must/mandatory.
* Scale much better than Fat Tree, as cable management overhead is less.
* If you want to scale, just create a group and then connect that group to existing Groups.

When to use

* very large fabric, i.e., very large number of nodes
* cable management cost matters
* Future growth is planned.
* Traffic benefits from Adaptive routing

---

## Torus 3D

This routing topology is not used in AI Training/HPC, hence I’m keeping it very brief
Each node is connected to six neighbours. It ends up in 6:1 subscription ratio.
The real use case is supercomputers, where localization is important.

---

## Routing Engine

Now that the routing topology is clear, we choose how the packets move through this topology using the routing engine. Each routing topology has a corresponding routine engine, and they must match. If you do not choose a routing engine, min-hop is the default routing engine.

### Routing Topology and Routing Engine Mapping

- min-hop --> not tied to any routing topology.
  It optimizes path length but cannot prevent credit loops.
- up/down --> commonly used for Fat-Tree topology which prevents credit-loops deadlocks
- Fat Tree routing --> used for fat-tree
- Torus routing --> used and designed for Torus routing
- Dragonfly+ routing --> used and designed for Dragonfly+ and needs Adaptive routing

In summary, Fat-Tree is the only topology where two routing engine options (Up/Down, Fat-Tree) are available.

## Adaptive Routing

I would like to say, AR makes min-hop smart.
When min-hops finds that it can reach destination port from two different exit ports with the same
number of hops, it uses the exit port with the least assigned LIDs

But AR uses this information to its advantage. It updates LFT with this information. Then, for every new connection, it checks which exit port is least busy, and uses that port to send a packet to the destination port. So AR is also referred to as congestion-aware routing and it is dynamically changes the packet direction. It is always active.

```
Switch A - LID 9

Switch A - Port 1 - 2 Hops
Switch A - Port 2 - 2 Hops
AR uses Port 1 or Port 2, based on
which one is less congested.
It is a dynamic and congestion-aware protocol.
```

## Credit Loops

I already know how credit-based flow works. In brief, packets are only sent when the receiver
has enough credits. If there are no credits, no packet flow. Loop is basically when the packet is
waiting to be sent, but has no credits because all devices in the path are waiting for credits
from one another.

Example

Here, each switch is trying to send to another.
in a circular pattern. Buffers are filled on switches.
Then none of the switches can send or receive.
Deadlock. One workaround is to reboot at least one switch in the fabric to break the loop.


## Up-Down - deadlock solver

It assigns a rank to each leaf and spine.

Spine gets Rank0
Leaf gets Rank1

```
So the allowed pattern is

Rank0 (spine-A)  to Rank1 (leaf-A)
Rank1 (leaf-A) to Rank(spine-A)

Not allowed is

Rank1 to Rank0, Rank0 to Rank1 this up - down - up
Leaf-A --> Spine-A --> Leaf-B --> Spine-B
```

## Fat-tree

The Fat-Tree routing engine depends heavily on how the fabric is cabled. This matters because
This engine is designed to optimize traffic based on a symmetric or almost symmetric topology
This means

* proper cabling
* port mapping
* documented topology
* validated switch connections
* clean expansion

This routing engine also builds upon min-hop. When it learns that it can reach the destination
from different ports with an equal number of hops, It spreads the traffic by using different spine switches
which ensures that not a single spine switch is overloaded.

```
Example

LID 10 needs to reach LID 11.
Here are the possible combinations
optimized by min-hop

Leaf -A - Port 1  --> Spine -A --> LID11
Leaf -B - Port 1  --> Spine -A --> LID11
Leaf -C - Port 1  --> Spine -A --> LID11
Leaf -D - Port 1  --> Spine -A --> LID11

In the above packet flow, Spine A is overloaded.
Now, with Fat-Tree, it will ensure the exit port of each switch.
is using a different Spine.

Leaf -A - Port 1  --> Spine -A --> LID11
Leaf -B - Port 1  --> Spine -B --> LID11
Leaf -C - Port 1  --> Spine -A --> LID11
Leaf -D - Port 2  --> Spine -B --> LID11
```

Since Fat-Tree is aware that it can reach LID 11 from different ports and spine, It has freedom and choose wisely to spread the load. But it is static and not dynamic like AR. This LFT calculation is done by SM during initialization and during Topology change.

## Topology selection critiera

* Check what the workload characteristics are
  * AI, HPC, Mixed, Inference
* Choose the topology based on the workload.
* Decide on oversubscription based on the workload.
* Decide on the routing engine and Adaptive routing.


## Summary
Topology decides possible paths.
The routing engine decides which is the safest and most optimized path
Subnet manager programs them on the switch

## Source

- NVIDIA InfiniBand Networking course
- DGX Design Guide based on H200
