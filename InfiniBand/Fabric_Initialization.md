# Fabric initialization

Here we are consolidating what was learn from the previous week.

## List the stages of InfiniBand fabric initialization

- Cabling which is operator initiated
- SM is powered ON
- Node discovery starts
- LID is assigned
- short path to the node is configured using min-hop
- routing table is calculated and programmmed on switches
- ports are configured
- SL to VL mapping is populated
- Activate ports using both physical and Logical state
- Ready for production data traffic

Lets explore the main bullet points from the listed above

## Node Discovery

Node discovery is where control plane comes into play.
Subnet Management Packet (SMP) we learned earlier is send by SM using VL15 and QP0
Both VL15 and QP0 is reserved for SMP and has not flow control mechnanism, as they
are emergency lane and must be always free.

As Node still does not have LID assigned, SM uses directed routing addresses.
It is as simple as hopping to each port. Once switch is discovered, it treats it as gateway
to find another gateway i.e. switch. It continues discovery further till it finds HCAs which are leaf, where discovery stops

During the discovery at high level following takes places
- Which HCA exists
- Which switch exists
- Which ports are alive
- What is MTU configured
- What is Lane Speed and Lane Width
- How are these devices connected.

It also runs get Node Info and Get Port Info
Node Info - provides node type info, no of ports, GUIDs per port and description of node
Port Info - Lane Width, Lane Speed and MTU

Now we have all details, LIDs assigning is easier.

## LID Assignment and min-hop
LIDs are assigned to all nodes as discussed earlier

- HCA each port
- Single IC switch
- Modular Chassis IC, 1 per IC

Do not confuse System GUID. This is purely for Chassis identification.

Now once LID is assigned, SM checks how these nodes are reachable with minimum number of hops
It does this using min-hop alogorithm.
e.g. Minimum number of hops from the switch port to destination LID

Sometime it happens that destination LID is reachable from
the same number of hops from
more than one switch exit port.
In this case (which is called as tie-breaker)
switch chooses the one which has minimum number
of destination LIDs assigned.

e.g.

Switch Port 1 --> LID 1
Switch Port 2 --> LID 2
Switch Port 3 -->

LID 3 is reachable from both Switch Port 1 and Port 2
with same number of hops, then it will assign as follows

Switch Port 1 --> LID 1
Switch Port 2 --> LID 2
Switch Port 3 --> LID 3

## Linear Forwarding Table (LFT)
Now SM is ready to calculate LFT and program them
on all Switches.
As learned here is the flow one more time

- packet arrives at switch
- switch checks in LRH the  Destination LID
- switch checks LFT for destination LID for switch to choose the exit port for destination LID


## Port configuration

SM not only assigns LIDs but also checks
- MTU
- Lane Width and Speed

If either of these parameters are not matching,
performance drop e.g. MTU mismatch, Link Speed is less e.g. 400 Gbps for NDR link


## SL to VL mapping
As we learned Service Level and Virtual Lane table is created by SM
and controlled by HCA. When the packet arrives at Port, SL to VL table is checked
and right lane is assigned and based on the weight,
arbitrator schedules the packet on the lane

here is the flow with SL and VL in picture

- packet arrives at switch
- switch checks in LRH the  Destination LID
- switch checks LFT for destination LID for switch to choose the exit port for destination LID
- SL to VL map is checked, VL is assigned
- arbitrator checks weight and schedule the frequency of packet

Simple traffic analogy

- SL is car
- VL is a line of the Highway
- Weight is how many times traffic signal becomes green for specific Lane
- Arbitrator is traffic inspector

## State checking

We already learned the following physical states

- Polling
- Training
- LinkUp

But this only means physically link is ready, but if it can send data, is decided by the
combination of both physical and logical state

The logical states are

- init : only SMP/control plane traffic is allowed
- armed: configuration ready, pending verification
- Active: ready for production

Both these states goes hand in hand

Physical state: Polling --> Training --> LinkUp
Logical state: init --> armed --> Active

Armed state is special state where SM sends synthetic packet between two nodes
This synthetic packet has VCRC included. Because with VCRC we can only check
integrity between two ends of the nodes. For end to end integrity we use ICRC.
ICRC is not needed in this case.

At this stage, fabric is ready for production traffic but this is just first step
as HPL Burn in test which is hardware burn test must be followed for end to end checks

The troubleshooting flow looks like

- check physical state (expected is: LinkUp)
- check logical state (expected is: Active)
- check if LID is assigned
- check switches are visible using ibswitches
- check LFT is properly populated using ibroute switch-lid
- check VL

## Understanding in a Single Diagram

```shell

                 ┌────────────────────┐
                 │   Subnet Manager   │
                 │  Control Plane     │
                 └─────────┬──────────┘
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
        ▼                  ▼                  ▼
 Discover topology   Assign LIDs       Calculate paths
 via SMP / VL15      local addresses   min-hop / policy
        │                  │                  │
        └──────────────────┼──────────────────┘
                           ▼
                  Program switch LFTs
                  Destination LID → Port
                           │
                           ▼
                  Configure ports/QoS
                  MTU, width, speed,
                  SL-to-VL mapping
                           │
                           ▼
                  Activate ports
                  Physical LinkUp +
                  Logical Active
                           │
                           ▼
                  Data traffic works

```


