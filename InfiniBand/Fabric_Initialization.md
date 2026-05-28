# Fabric initialization

Here we are consolidating what was learn from the previous week.

## List the stages of InfiniBand fabric initialization

* Cabling which is operator initiated
* SM is powered ON
* Node discovery starts
* LID is assigned
* Short path to the node is configured using min-hop
* Routing table is calculated and programmmed on switches
* Ports are configured
* SL to VL mapping is populated
* Activate ports using both physical and Logical state
* Ready for production data traffic

Lets explore the main bullet points from the listed above.

## Node Discovery

* Node discovery is where control plane comes into play.
* Subnet Management Packet (SMP) we learned earlier is send by SM using VL15 and QP0.
* Both VL15 and QP0 is reserved for SMP and has not flow control mechnanism, as they are emergency lane and must be always free.

As Node still does not have LID assigned, SM uses directed routing addresses.

It is as simple as hopping to each port. Once switch is discovered, it treats it as gateway to find another gateway i.e. switch. It continues discovery further till it finds HCAs which are leaf, where discovery stops.

During the discovery at high level following takes places:

* Which HCA exists
* Which switch exists
* Which ports are alive
* What is MTU configured
* What is Lane Speed and Lane Width
* How are these devices connected.

It also runs get Node Info and Get Port Info:

* **Node Info** - provides node type info, no of ports, GUIDs per port and description of node
* **Port Info** - Lane Width, Lane Speed and MTU

Now, we have all details, LIDs assigning is easier.

## LID Assignment and min-hop

LIDs are assigned to all nodes as discussed earlier:

* HCA each port
* Single IC switch
* Modular Chassis IC, 1 per IC

> **Note:** Do not confuse System GUID. This is purely for Chassis identification.

Now once LID is assigned, SM checks how these nodes are reachable with minimum number of hops.

It does this using min-hop alogorithm.

e.g. Minimum number of hops from the switch port to destination LID

Sometime it happens that destination LID is reachable from the same number of hops from more than one switch exit port which is referred multiple equal paths. In this case, (also referred as tie-breaker) switch chooses the one which has minimum number of destination LIDs assigned.

```markdown
Switch Port 1 --> LID 1
Switch Port 2 --> LID 2
Switch Port 3 -->

LID 3 is reachable from both Switch Port 1 and Port 2
with same number of hops, then it will assign as follows

Switch Port 1 --> LID 1
Switch Port 2 --> LID 2
Switch Port 3 --> LID 3 # least assigned LID i.e. Zero

```

## Linear Forwarding Table (LFT)

Now SM is ready to calculate LFT and program them on all Switches.

As learned earlier, here is the flow:

* Packet arrives at switch
* Switch checks in LRH the Destination LID
* Switch does lookup in LFT to find the egress port to the LID

---

## Port configuration

SM not only assigns LIDs but also checks:

* MTU
* Lane Width and Speed

If either of these parameters are not matching, performance drop e.g. MTU mismatch, Link Speed is less e.g. 400 Gbps for NDR link.

---

## SL to VL mapping

As I learned Service Level and Virtual Lane table are created by SM and controlled by HCA. When the packet arrives at Port, SL to VL table is checked and right lane is assigned and based on the weight, arbitrator schedules the packet on the lane.

Here is the detailed flow with SL and VL in picture:

* Packet arrives at switch
* Switch checks in LRH the Destination LID
* Switch does lookup in LFT to find the egress port to the LID
* SL to VL map is checked, VL is assigned
* Arbitrator checks weight and schedule the frequency of packet

```markdown
Simple traffic analogy
- SL is car
- VL is a line of the Highway
- Weight is how many times traffic signal becomes green for specific Lane
- Arbitrator is traffic inspector

```

## State checking

We already learned the following physical states:

* Polling
* Training
* LinkUp

But this only means physically link is ready, but if it can send data, is decided by the combination of both physical and logical state.

The logical states are:

* **init:** only SMP/control plane traffic is allowed
* **armed:** configuration ready, pending verification
* **Active:** ready for production

Both these states goes hand in hand:

```markdown
Physical state: Polling --> Training --> LinkUp
Logical state: init --> armed --> Active

```

Armed state is special state where SM sends synthetic packet between two nodes.

This synthetic packet has VCRC included. Because with VCRC we can only check integrity between two ends of the nodes. For end to end integrity we use ICRC. ICRC is not needed in this case.

And very important these state are transitioned by SM i.e. SM sends PortInfo via MAD to the node.

At this stage, fabric is ready for production traffic but this is just first step as HPL Burn in test which is hardware burn test must be followed for end to end checks.

The troubleshooting flow looks like:

* Check physical state (expected is: LinkUp)
* Check logical state (expected is: Active)
* Check if LID is assigned (use `ibroute`)
* Check switches are visible using `ibswitches`
* Check LFT is properly populated using `ibroute switch-lid`
* Check VL
