# TiDB Internal (III) - Scheduling

By xuliwen

This article introduces the mysterious module of PD (Placement Driver). This part is more complex and covers topics that are rarely discussed in other articles. We'll follow the same approach as the previous articles - first discussing what functionality we need, then how we implement it. Understanding the requirements first makes it easier to understand the considerations behind our design.

## Why Do We Need Scheduling?

Recall from [TiDB Internal (I) - Data Storage](https://cn.pingcap.com/blog/tidb-internal-1) that the TiKV cluster is TiDB's distributed KV storage engine. Data is replicated and managed in Regions, with each Region having multiple Replicas distributed across different TiKV nodes. The Leader is responsible for reading/writing, while Followers are responsible for synchronizing the Raft log sent by the Leader. With this in mind, consider the following questions:

- How do we ensure multiple Replicas of the same Region are distributed across different nodes? What happens if multiple TiKV instances are running on the same machine?
- When the TiKV cluster is deployed across multiple data centers for disaster recovery, how do we ensure that if one data center goes offline, we won't lose multiple Replicas of a Raft Group?
- After adding a new node to the TiKV cluster, how do we move data from other nodes in the cluster?
- What happens when a node goes offline? What needs to be done across the cluster? How do we handle temporary node failures (service restart) versus long-term failures (disk failure with complete data loss)?
- Assuming the cluster needs N copies for each Raft Group, what happens when a single Raft Group has too few Replicas (due to node failures) or too many (when previously offline nodes rejoin the cluster)? How do we adjust the number of Replicas?
- Since reading/writing is done through the Leader, what impact does it have if Leaders are concentrated on a small number of nodes?
- Not all Regions are frequently accessed. What should we do when hotspots are concentrated in just a few Regions?
- When performing load balancing, cluster data needs to be relocated. How do we prevent this data migration from consuming too much network bandwidth, disk I/O, and CPU, which could affect online services?

While each of these problems might have simple solutions in isolation, solving them all together is challenging. Some issues might seem to only require considering the internal state of a single Raft Group (like deciding whether to add a Replica based on having enough copies). However, deciding where to add that Replica requires considering the overall cluster state. The system is also constantly changing - Regions split, nodes join or fail, and access hotspots shift. The scheduling system needs to continuously work toward an optimal state under these dynamic conditions. Without centralized information and control over the overall situation, it would be difficult to meet these requirements. This is why we need the PD (Placement Driver) module.

## Scheduling Requirements

Let's categorize and organize the questions listed above. Overall, there are two major categories of problems:

**As a distributed high-availability storage system, we must meet these requirements:**

1. The number of Replicas must be neither too many nor too few
2. Replicas need to be distributed across different machines
3. After adding new nodes, we can migrate Replicas from other nodes
4. After node failure, we need to migrate data from that node

**As a well-designed distributed system, we need to optimize these areas:**

1. Leaders should be evenly distributed across the cluster
2. Maintain uniform storage capacity across nodes
3. Maintain uniform distribution of access hotspots
4. Control the speed of load balancing to avoid impacting online services
5. Manage node status, including manual online/offline operations and automatic handling of failed nodes

Meeting the first set of requirements gives the system multi-copy fault tolerance, dynamic scaling, node failure tolerance, and automatic error recovery. Meeting the second set of requirements helps balance the overall system load and makes it easier to manage.

To meet these requirements, we first need to collect sufficient information, such as:
- Status of each node
- Information about each Raft Group
- Business access operation statistics

Then we need strategies that use this information to formulate scheduling plans that meet the above requirements. Finally, we need basic operations to execute these scheduling plans.

## Basic Scheduling Operations

Let's start with the simplest aspect - the basic scheduling operations available to implement our strategies. Understanding what tools we have available helps determine how to use them effectively.

The scheduling requirements above may seem complex, but they can all be accomplished using just three operations:

1. Add a Replica
2. Delete a Replica
3. Transfer Leader role between different Replicas in a Raft Group

The Raft protocol supports these three needs through the AddReplica, RemoveReplica, and TransferLeader commands.

## Information Collection

Scheduling depends on collecting information about the entire cluster. In short, we need to know the status of each TiKV node and each Region. The TiKV cluster reports two types of information to PD:

**Each TiKV node regularly reports overall node information to PD**

There is a heartbeat connection between TiKV nodes (Stores) and PD. PD uses these heartbeats to:
1. Check if each Store is alive and detect new Stores
2. Collect [Store status information](https://github.com/pingcap/kvproto/blob/master/proto/pdpb.proto#L294), including:

- Total disk capacity
- Available disk capacity
- Number of Regions
- Data write speed
- Number of Snapshots sent/received (Snapshots may be used to sync data between Replicas)
- Whether it's overloaded
- Label information (labels are hierarchical Tags)

**Each Raft Group's Leader regularly reports to PD**

There is a heartbeat connection between each Raft Group's Leader and PD for reporting [Region state](https://github.com/pingcap/kvproto/blob/master/proto/pdpb.proto#L207), including:

- Leader location
- Followers' locations
- Number of Replicas
- Data read/write speed

PD continuously collects information about the entire cluster through these two types of heartbeat messages and uses this information as the basis for decision-making. Additionally, PD can receive supplementary information through its management interface to make more accurate decisions. For example, when a Store's heartbeat is interrupted, PD cannot determine if the node failure is temporary or permanent, so it waits for a period (default 30 minutes) before considering the Store offline and deciding to relocate all its Regions. However, when operations staff deliberately take a machine offline, they can notify PD through its management interface that the Store is unavailable, allowing PD to immediately begin relocating its Regions.

## Scheduling Strategies

After PD collects this information, it needs strategies to formulate specific scheduling plans.

**Correct Number of Replicas in a Region**

When PD detects through a Region Leader's heartbeat that the number of Replicas doesn't meet requirements, it adjusts the number using Add/Remove Replica operations. This can happen when:

- A node goes offline and all its data is lost, leaving some Regions with insufficient Replicas
- A previously offline node resumes service and automatically reconnects to the cluster, causing some Regions to have too many Replicas
- An administrator adjusts the replication strategy by changing the [max-replicas](https://github.com/tikv/pd/blob/master/conf/config.toml#L54) configuration

**Multiple Replicas in a Raft Group Must Not Share Location**

Note that "same location" is different from "same node". Generally, PD only ensures multiple Replicas don't exist on one node to avoid losing multiple Replicas if a single node fails. In actual deployments, there may be additional requirements:

- Multiple nodes deployed on the same physical machine
- TiKV nodes distributed across multiple racks, requiring system availability even if a single rack loses power
- TiKV nodes distributed across multiple data centers, requiring system availability even if a single data center goes down

These requirements essentially define nodes with common location attributes as the smallest fault-tolerant unit. We want to avoid having multiple Replicas of a Region within such a unit. Nodes can be configured with [labels](https://github.com/tikv/tikv/blob/master/etc/config-template.toml#L16), and PD's [location-labels](https://github.com/tikv/pd/blob/master/conf/config.toml#L59) configuration indicates which labels mark locations, ensuring Replicas aren't assigned to locations with matching location labels.

**Even Distribution of Replicas Across Stores**

Since each Replica has a fixed upper limit on stored data capacity, maintaining a balance in the number of Replicas across nodes helps balance the overall load.

**Even Distribution of Leaders Across Stores**

The Raft protocol requires all reads and writes to go through the Leader, so the computational load is mainly on Leaders. PD tries to distribute Leaders evenly across nodes.

**Even Distribution of Access Hotspots Across Stores**

Each Store and Region Leader reports current access load information (like Key read/write speeds) in their heartbeats. PD detects access hotspots and spreads them across nodes.

**Even Storage Space Across Stores**

Each Store specifies a Capacity parameter at startup indicating its storage space ceiling. PD considers remaining storage space when scheduling.

**Control Scheduling Speed to Avoid Impacting Online Services**

Scheduling operations consume CPU, memory, disk I/O, and network bandwidth. We need to avoid excessive impact on online services. PD controls the number of ongoing operations. The default speed control is conservative, but scheduling can be manually accelerated through pd-ctl (for example, when services are stopped for upgrades and new nodes need to be balanced quickly).

**Support Manual Node Offline Operations**

When manually taking a node offline through pd-ctl, PD relocates the node's data at a controlled rate. Once scheduling is complete, the node is marked offline.

## Scheduling Implementation

Now that we understand the above information, let's look at the entire scheduling process.

PD continuously collects information through Store and Leader heartbeats to obtain detailed cluster data. It generates scheduling operation sequences based on this information and scheduling strategies. Each time it receives a Region Leader heartbeat, PD checks if there are any operations pending for that Region. If so, it returns the operations to the Region Leader in the heartbeat response and monitors execution results in subsequent heartbeats. Note that these operations are only recommendations to the Region Leader - there's no guarantee they will be implemented. Whether and when they execute is determined by the Region Leader based on its current state.

## Summary

This article covers topics you rarely see in other articles. Every design has its underlying considerations. We hope you now understand what a distributed storage system needs to consider when implementing scheduling, and how to decouple strategies to support more flexible strategy expansion.

This concludes our three-part series. We hope you now understand TiDB's basic concepts and implementation principles. Future articles will introduce TiDB's internals from both architectural and code perspectives. If you have any questions, please email shenli@pingcap.com. 
