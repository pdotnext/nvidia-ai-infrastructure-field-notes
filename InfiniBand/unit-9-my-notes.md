- Cables attached
- SM starts
- Topology discovery using SMPs. SMPs using VL15 and QP0 where no flow control happens.
  At this stage no LID is assigned, it use direct routed addressing.
    SM sends Get Node Info and Get Port Info
      Get Node Info - provides node type, no of ports, GUID and node description
      Get Port Info - MTU, Width, Speed and VL Information
    Production interpretation
    - Which HCAs exists ?
    - Which Switch exits ?. It uses switch has gateway and from this gateway it tries to reach another gateway. HCAs are than comparable to leaf where discovery stops
    - Which ports are live ?
      - What speed/width they are running
      - What MTU they support
      - What VLs are availabe ?
    - How is everything connected ?

- LID assignment
    - HCA Port 1 --> LID 1
    - HCA Port 2 --> LID 2
    - Switch A Port 1 --> LID 3
    - Switch B Port 1 --> LID 4
  Problem: LIDs is not assigned

  Cause:
  - SM Issue
  - Discovery Issue
  - parition/config Issue
  - fabric control plane issue

- Path calculation
  Min-Hop routing to calculate LFT which is uses fewest hop from the switch port to the destination port.
  If the number of hops available are same, then port with lowest destination LIDs

  e.g.
  Destination LID 8 can be reached via
  - Port 1: 2 hops
  - Port 2: 2 hops
  - Port 3: 4 hops

  To decide among Port 1 and Port 2, it checks which port (1 or 2) has lowest destination LIDs assigned.
  Port 2 has least destination LIDs assigned

Every time packet forward decisions needs to be done, LFT is referred. SM job has already finished when LFT is calculated.

- Switch forwarding table (LFT) programming into switches
Already learnt, here is one more time
Packet arrives at switch. Switch checks LRH for destination LID
Destination LID is then checked by the switch in LFT which has destination port assigned
e.g. from above LID 8 will exit via Port 2

ibroute can help you find what is programmed into switch.
ibswitches - what switches exist and what LIDs do they have


- Port configuration

Apart from LIDs, SM also configures
  - MTU
  - Lane Width
  - Lane Speed

e.g. NDR has 4 lane width x 100 lane speed = 400 Gbps

Troubleshooting example

Expected: NDR 4 x 100 Gbps
Actual: 1x
Impact: Lower Bandwidth

Expected: MTU 4096
Actual: MTU Mismatch
Impact: Degraded performance

Expected: Active
Actual: Polling
Impact: No data traffic possible



- SL to VL configuration table

packet arrives --> Switch Read LID --> looks in LFT Table for destination LID and port --> check SL to VL Mapping for the output port
--> Packet is placed into the selected VL --> Arbitrator decides which VL queue gets served next

SL = type of vehicle / service class
VL = physical queue/lane at the intersection
VL weight = how much green-light time that lane gets
Arbiter = traffic light controller

Physical State
Already learned they are
- LinkUp
- Link Error Recovery
- Polling
- Port Configuration Training

Logical State
- down --> physical layer not up
- init --> Physical link up but only SM/control traffic allowed
- Armed --> configuration completed, validation pending
- Active --> usable for data traffic


Physical state flow ---down --> polling --> training --> LinkUp
Logical state flow --down --> Init --> Armed --> Active

Armed State: It is special state where SM
sends a synthetic data packet through the link with a VCRC.
VCRC allows end to end integrity between two ports.


- Fabric becomes visible for data


Node is powered on
Cable is attached, link is established
SM Discovers Node and assign LIDs
SM calculates and programmes LFTs
SM activates Subnets


