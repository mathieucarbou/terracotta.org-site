---
---
#  The Terracotta Server Array

{toc|2:3}

For BigMemory Max, Quartz Scheduler, and Terracotta Web Sessions

## Introduction
The Terracotta Server Array (TSA) provides the platform for Terracotta products and the backbone for Terracotta clusters. A Terracotta Server Array can vary from a basic two-node tandem to a multi-node array providing configurable scale, high performance, and deep failover coverage.

The main features of the Terracotta Server Array include:

* **Distributed In-memory Data Management** &ndash;  Manages 10-100x more data in memory than data grids
* **Scalability Without Complexity** &ndash;  Simple configuration to add server instances to meet growing demand and facilitate capacity planning
* **High Availability** &ndash;  Instant failover for continuous uptime and services
* **Configurable Health Monitoring** &ndash;  Terracotta <a href="high-availability#85916">HealthChecker</a> for inter-node monitoring
* **Persistent Application State** &ndash;  Automatic permanent storage of all current shared in-memory data
* **Automatic Node Reconnection** &ndash;  Temporarily disconnected server instances and clients rejoin the cluster without operator intervention

###New for BigMemory Max 4.x
The 4.x TSA is an in-memory data platform, providing faster, more consistent, and more predictable access to data. With resource management, if you have more data than memory available, the TSA protects itself from going over its limit through data eviction and throttling. In most cases, it will recover and come back to its normal working state automatically. In addition, four systems are available to protect data: the Fast Restart feature, BigMemory Hybrid's use of SSD/Flash, active-mirror server groups, and backups.

#####Fast Restartability for Data Persistence
BigMemory's Fast Restart feature is now integrated into the TSA, providing crash resilience with quick recovery, plus a consistent record of the entire in-memory data set, no matter how large. For more information, refer to [Fast Restartability](/documentation/4.1/terracotta-server-array/server-arrays#fast-restartability).

#####Hybrid Data Storage
[BigMemory Hybrid](/documentation/4.1/terracotta-server-array/hybrid) extends BigMemory distributed in a Terracotta Server Array so that data can be stored across a hybrid mixture of RAM and SSD/Flash. This additional storage is managed with the in-memory data as one TSA data set.

#####Resource Management
Resource management provides better control over the TSA's in-memory data through time, size, and count limitations. This enables automatic handling of, and recovery from, near-memory-full conditions. For more information, refer to [Automatic Resource Management](/documentation/4.1/terracotta-server-array/operations#automatic-resource-management).

#####Predictable Eviction Strategy
Based upon user-configured time, size, and count limitations, the TSA's 3-pronged eviction strategy works automatically to ensure predictable behavior when memory becomes full. For more information, refer to [Eviction](/documentation/4.1/terracotta-server-array/operations#eviction).

#####Continuous Uptime
Improvements to provide continuous availability of data include flexibility in server startup sequencing, better utilization of extra mirrors in mirror groups, multi-stripe backup capability, optimizations to bulk load, and performance improvements for data access on rejoin. In addition, the TSA no longer uses Oracle Berkeley DB, enabling in-memory data to be ready for use much more quickly after any planned or unplanned restart.

#####Terracotta Management Console (TMC)
The expanded TMC replaces the Developer Console and Operations Center as the integrated platform for monitoring, managing, and administering all Terracotta deployments. There is also support for additional REST APIs for management and monitoring. For more information, start with [Terracotta Management Console](/documentation/4.1/tms/tms).

#####Additional Security Features
Active Directory (AD) and Lightweight Directory Access Protocol (LDAP) support on Terracotta servers, and custom SecretProvider on Terracotta clients. For more information, refer to [Securing Terracotta Clusters](/documentation/4.1/terracotta-server-array/tsa-security) and [Setting up LDAP-based Authentication](/documentation/4.1/terracotta-server-array/tsa-ldap).

#####No More DSO, plus Simplified Configuration
DSO configuration has been deprecated, and the tc-config has a new format. Most of the elements are the same, but the structure is revised. For more information, refer to [Terracotta Configuration Reference](/documentation/4.1/terracotta-server-array/config-reference).

## Definitions and Functional Characteristics

The major components of a Terracotta installation are the following:

* **Cluster** &ndash;  All of the Terracotta server instances and clients that work together to share application state or a data set.
* **Terracotta client** &ndash;  Terracotta clients run on application servers along with the applications being clustered by Terracotta. Clients manage live shared-object graphs.
* **Terracotta server instance** &ndash;  A single Terracotta server. An *active* server instance manages Terracotta clients, coordinates shared objects, and persists data. Server instances have no awareness of the clustered applications running on Terracotta clients. A mirror (sometimes called "hot standby") is a live backup server instance which continuously replicates the shared data of an active server instance, instantaneously replacing the active if the active fails. **Mirror servers add failover coverage within each mirror group**.
* **Terracotta mirror group** &ndash;  A unit in the Terracotta Server Array. Sometimes also called a "stripe," a mirror group is composed of exactly one active Terracotta server instance and at least one mirror Terracotta server instance. The active server instance manages and persists the fraction of shared data allotted to its mirror group, while each mirror server in the mirror group replicates (or mirrors) the shared data managed by the active server. **Mirror groups add capacity to the cluster**. The mirror servers are optional but highly recommended for providing failover.
* **Terracotta Server Array** &ndash;  The platform, consisting of all of the Terracotta server instances in a single cluster. Clustered data, also called in-memory data, or shared data, is partitioned equally among active Terracotta server instances for management and persistence purposes.


<table>
<caption>TIP:  Nomenclature</caption>
<tr>
<td>
This documentation may refer to a Terracotta server instance as L2, and a Terracotta client (the node running your application) as L1. These are the shorthand references used in Terracotta configuration files.
</td>
</tr>
</table>

<a name="24373"></a>Figure 1 illustrates a Terracotta cluster with three mirror groups. Each mirror group has an active server and a mirror, and manages one third of the shared data in the cluster.

<img src="/images/documentation/MirrorGroupArray.png" alt="Terracotta cluster with 3 mirror groups." class="right" >

A Terracotta cluster has the following functional characteristics:

* Each mirror group automatically elects one active Terracotta server instance.
    There can never be more than one active server instance per mirror group, but there can be any number of mirrors. However, a performance overhead may become evident when adding more mirror servers due to the load placed on the active server by having to synchronize with each mirror.
* Every mirror group in the cluster must have a Terracotta server instance in active mode before the cluster is ready to do work.
* The shared data in the cluster is automatically partitioned and distributed to the mirror groups.
    The number of partitions equals the number of mirror groups. In Fig. 1, each mirror group has one third of the shared data in the cluster.
* Mirror groups **cannot** provide failover for each other.
    Failover is provided *within* each mirror group, not across mirror groups. This is because mirror groups provide scale by managing discrete portions of the shared data in the cluster -- they do not replicate each other. In Fig. 1, if Mirror Group 1 goes down, the cluster must pause (stop work) until Mirror Group 1 is back up with its portion of the shared data intact.
* Active servers are self-coordinating among themselves.
    No additional configuration is required to coordinate active server instances.
* Only mirror server instances can be hot-swapped in an array.
    In Fig. 1, the L2 MIRROR servers can be shut down and replaced with no affect on cluster functions. However, to add or remove an entire mirror group, the cluster must be brought down. Note also that in this case the original Terracotta configuration file is still in effect and no new servers can be added. Replaced mirror servers must have the same address (hostname or IP address). If you must swap in a mirror with a different configuration, see [Changing Cluster Topology in a Live Cluster](/documentation/4.1/terracotta-server-array/operations#45877).



##Where To Go Next


| For more about | Go to |
|:-------|:------------|
|Architecture|[Terracotta Server Array Architecture](/documentation/4.1/terracotta-server-array/server-arrays)|
|High Availability|[Configuring Terracotta Clusters for High Availability](/documentation/4.1/terracotta-server-array/high-availability)|
|Configuration|[Working with Terracotta Configuration Files](/documentation/4.1/terracotta-server-array/configuration-guide)|
|Resource Management|[TSA Operations &ndash; Automatic Resource Management](/documentation/4.1/terracotta-server-array/operations#automatic-resource-management)|
|Live Data Backup|[TSA Operations &ndash; Distributed In-memory Data Backup](/documentation/4.1/terracotta-server-array/operations#live-backup-of-distributed-in-memory-data)|
