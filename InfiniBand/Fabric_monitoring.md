# Fabric Monitoring

Now that Fabric is initialized, monitoring of the fabric start
In typical infrastructure this is similar to server is in production,
now start monitoring with goal of minimal or Zero downtime, planning
changes.

To ensure Fabric runs without major event, SM provide High availability
solution. This is the major consideration for production.
These notes reflects my understanding.

## Impact of Master SM unavailability
Before i discuss the main topic, let me
explain what is planned and unplanned failure

## Handover (planned failover)
Planned Failover is referred as Handover. When Master SM is active, Slave SM is also present
But for maintenance reason, we need to introduce or create a new SM, in this case SM with high priority is
started, it takes over the over of Master SM. This is planned failover and has
very limited disruption compare to unplanned failover

## Failover
Here Master SM in suddenly unavailable, Slave SM detects it using SMInfo.
SMInfo is similar to heartbeat exchange where SM exchanges Priority, State and GUID of the port.
Slave SM takes over the Master role but what happens between Master SM is unavailabe and Slave SM becomes master?

### When Master SM is unavailable
- no new sessions can be established. It means no new AI training can occur till SM is active again
- If node or switch fails during this window, SM cannot detect the failure or update the LFT.

### What works then?
Existing running AI jobs works. Because it works in data plane because SM works in control plane.
Even though unavailability of SM may not be seen as production event, it is visible in production because
of the reason mentioned above, if the SM remains unavailable for long time,
even AI job may stop which is the primary reason to have minimum two SM nodes in any production fabric.

| What survives failover |  what's disrupted during failover |
| - | - |
| Existing QP connection | New QP setups (SM needs path resolution to new SM) |
| In-flight RDMA traffic (data plane unaffected) | SM queries pause briefly |
| PKey memberships (persisted config)  |  Multicast group join/leave (MCMembers) |
| Existing Multicast forwarding | New node join (needs full discovery) |
| Active routes | Topology change response delayed by failover window |

## Master SM election
This topic has lot of similarities with vSphere HA
master election process. In production, it is recommended
to deploy minimum two SMs. One is Master and other is
Slave.

Master is elected based a priority value.
By default, zero priority value is assigned to all SMs
So whoever starts first becomes Master.
If they start at same time, based on GUID of the port master
is elected. Lowest GUID port wins.

Priority: 0 (default) is the lowest
Priority: 15 is the highest, reserve this for master_sm_priority value explained below

This non-deterministic behavior is a operational risk
for production for the following main reasons

- Master SM should be always identified to monitor it, to detect if the critical event has occurred.
  - e.g. if SM node, reboots and you are unaware if it is Master or Slave, it makes troubleshooting quite
  difficult
- Change management needs to know the impact of SM unavailability, depending upon
  where are you planning the change
- deterministic behavior i.e. manually configuring or deciding who becomes master
  esp if there is hardware difference between Master and Slave or stability reason is important.

### Set priority manually

Master SM gets Priority 14
Slave SM gets Priority 13
Additional Slave SM gets Priority 10

### Commands to check and set priorities

```shell
# login to switch -A, assume it is Master SM
ib sm sm-priority 14
# check if the priority set
show ib sm priority
# output
14

# login to switch -B, assume it is Slave SM
ib sm sm-priority 13
# check if the priority set
show ib sm priority
# output
13

# login to switch -C, assume it is Master SM
ib sm sm-priority 10
# check if the priority set
show ib sm priority
# output
10

```

## Double Failover

Now SM is elected, we configured deterministic behavior to minimize operation risk.
When Master SM (priority 14) reboots, Slave SM (priority 13) takes over the role.
Now Slave SM is Master SM, after sometime, Master SM is back online.
It has higher priority (14), it takes over the role again.
This transfer of role creates unnecessary noise in the fabric and could lead to
same production event listed above but it is not as much distruptive as unavailability of SM
However, there is always a risk if such event repeats, creating unnecessary role transfer.

To avoid such behavour, one should configure master_sm_priority=15
When this is configured, lets revisit the scenario
When Master SM (priority 14) reboots,
Slave SM (priority 13) takes over the role. Now Slave SM is Master SM, but with master_sm_priority,
its priority is elevated to 15. After sometime, Master SM is back online.
But since new Master SM has higher priority (15) than old Master SM (14), role change does not happen

## Tradeoff on where SM should run?

SM can run on switch in embedded mode or it can run on
dedicated hardware. Lets discuss what is the trade off

### Embedded

Pros:
- simplified Lifecycle management
  - drivers, firmware tested and verified by vendor
  - patched as part of switch lifecycle management

Cons:
  - Switch gets two role to play, switch down, SM node increasing operational complexities
  - Limited scalability (max 680 nodes), in terms Compute/memory on switch's ASCI

### Host-based SM
Deployed on dedicated hardware with opensm as service

Pros:
  - can be sized based on the requirement, and can scaled accordingly
  - better logging and monitoring
  - separate failure domain
  - some advanced routing engine demands dedicated host

Cons:
  - additional host to manage i.e. lifecycle management (patching, maintenance, update)
  - placement decision e.g. closed to Switch with reliable connection




## Information Syncing and Sweeping

Now we are sure which SM is elected as Master SM and when Master SM fails, it does not pull the role back when it is back online.
But then how does it keep themselves in sync. It keeps in sync using Ethernet management network (again same similarities with vSphere HA)
So, no infiniband network is used. There is also a VIP which always point to Master SM. So when make change to Master SM,
it will always replicate these changes to Slave SM. VIP concept is not elaborated in the training. Therefore i'm skipping it.

Lets talk about how SM monitors fabric.
SM monitors fabric using two sweeping modes. Light and Heavy

### Light Sweeping

This mode is like check if the fabric is configured as seen last time.
Light Sweeping happens every 10 seconds (it is default configuration).
During this sweep it checks

- all nodes and ports of all switches are live
- status change of port
- SM events e.g. SM priority changes or SM not reachable

### Heavy Sweeping
This is can be seen an production impact sweep.
It is triggered based on what is changed in the fabric e.g.
your spine switch is unavailable. Then this leads to
- Rediscover/Recalculation/Reprogramming of the fabric
This must happen if the spine switch is unavailable.
But does it need to happen singe node fails or single switch fails.
It can be argued that if switch fails, 24 ports are available
But all nodes are spread across leaf, so this is still limited production impact.
So this bit of design consideration esp if you have rail optimized network, then this limited production impact can be greater than thought about.

In short
1. Sweep (active)
  - Light sweep is check mode, cheap
  - heavy sweep can lead to rediscover/Recalculation/Reprogramming, and hence expensive



### Commands to check light sweep and heavy sweep

```shell
# defaul 10 seconds
show ib sm sweep-interval
# output
10 seconds

# heavy sweep is disabled by default
show ib sm use-heavy-sweep
# output
disable

```

## Target Topology change handling
As discussed above if single node is unreachable, SM should trigger a fabric reinitialization.
It is control-plane heavy operation esp. when one node is unavailable
Hence you can change this behaviour because by default if node send a InfiniBand trap
it will trigger heavy sweep. Just to repeat/add, this trap is basically send by Node (SMA) via MADs
learned earlier to SM.

TRAP is reactive, while sweep is proactive.
TRAP is something SM continuously watch for, does happen frequently as light sweep.
Sweeps are scheduled, trap are interrupts.

### Commands to change trap behavior

```shell
# by default it is enable, so to disable start with no
no ib sm sweep-on-trap
# output
enable

# check if the value is enabled
show ib sm sweep-on-trap
# output
disable

```

## LID preservation
When the slave SM takes over esp during failover event,
LIDs might get reassigned. LIDs re-assign should be
avoid at all cost because LIDs are used by services
and cached by applications. Think of it like, changing
IP of all servers. This is huge operational risk.

In InfiniBand, SM can re-use existing SMs using two method

First during discovery, SM creates a GUID <-> LID table
for all ports in the fabric and this is cached.
Purpose of this table is to persist LID assignment when SM reboots or failover happens.
This table is reused by new SM assuming it is present, valid and
no re-assign setting is enabled.

Second, LID is also cached by each node, SM can request this from each
node when first method does not work.

### Command to change the cache behavior

```shell
# by default it is disabled
ib sm guid2lid-cache

# check if it is now enabled
show ib sm guid2lid-cache

```

## Production perspective

- Always ensure there minimum two SMs in a fabric
- Master configures and monitors fabric and slave monitors Master
- set priority to elect the master of your choice to reduce operational complexity
- disable Trap for a single node unreachable i.e. disable heavy sweep for node unavailable event

