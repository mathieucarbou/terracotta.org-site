---
---
#BigMemory Max Sizing Guide

{toc|2:3}

##Introduction

This guide covers general sizing considerations for planning a BigMemory installation. Each installation of BigMemory is unique, and there are many factors that affect sizing decisions. In addition, planning work is necessarily done by estimation. Ultimately, approaching the optimal sizing for the best performance is a matter of approximation and experimentation, however, this guide should help you take an educated approach.

This guide addresses the following topics:

* Capacity Planning
* How to size BigMemory to hold your data.
* Factors that affect the hardware and software resources needed to run BigMemory


###Sizing Goals

The two main goals of sizing might seem contradictory at first:

* Provide enough space for your application's data
* Constrain the space so that in-memory data management operates optimally

Providing enough space when sizing the BigMemory system involves the resources (heap, off-heap, disk) being used to store the data. Constraining the amount of available resources involves configuring them to prevent adverse consequences for the environment (JVM, Host OS or Guest OS) and the other software running within each of these layers.

The intent is to make each tier of BigMemory as large as possible, but no larger than necessary, which can mean:
  * not bigger than the maximal size of the data set.
  * not bigger than the “hot set” plus the "warm set" (a complex function of the hit distribution and the latency requirements).
  * no larger than the environment can bear:
    * heap is likely constrained by the GC pause time requirements.
    * off-heap is likely constrained by the available physical memory and/or the license.

Upper tiers are "closer" caches of what is the "hottest" data. When choosing tier sizes think of the caching tiers like a pyramid; make sure the pyramid is built in the right order. For clustered caches this means:

  * server disk is larger than server off-heap
  * server off-heap is larger than local off-heap
  * local off-heap is larger than local heap


###Sizing Scopes

Three areas can be sized:

* Cache: you can constrain the sizes of some things on a cache-by-cache basis.
* CacheManager: you can constrain the sizes of some things at the CacheManager level.  This means that the sum total of the resource used by the caches within the CacheManager will be constrained. For more information, see [Pooling Resources](http://www.terracotta.org/documentation/4.1/bigmemorymax/configuration/cache-size#pooling-resources-versus-sizing-individual-data-sets).

  The above two areas are covered in [Sizing Client-side BigMemory Max](#sizing-client-side-bigMemory-max).

* Terracotta Server: you can constrain the sizes of some things at the Terracotta Server level. This means the data stored within the server will be constrained. Note that this is not directly constraining sizes at the level of the Terracotta Server Array, but only the sizes of its component servers. Ultimately the sum total of all server constraints also constrains the array as well, but the actual enforcement is more tightly scoped.

  Terracotta server sizing is covered in [Sizing Terracotta Server Arrays](#sizing-terracotta-server-arrays).

**Important Note**: All Terracotta sizing controls are constraints on maximum values. For example, this means if you configure maxEntriesLocalHeap to be 10000 for a given cache then we target keeping the number of entries below 10000 not at 10000.



## Sizing Procedure Overview
The following is an overview of the steps for sizing and planning your BigMemory Max installation. It can be used in conjunction with the worksheets that follow.

1. Estimate the total size of your application's data. You can use [Worksheet 1](#worksheet-1-estimating-total-data-size) below.


2. Assess the other application requirements that affect the total amount of space to configure. Refer to [Worksheet 2](#worksheet-2-calculating-overhead).

  Some BigMemory features require additional space for managing the dataset. This is called overhead. The features that require quantifiable overhead are:
  * Persistence with [Fast Restartability](#fast-restart)
  * Disk usage with [BigMemory Hybrid](#hybrid)
  * [Search](#)
  * Data life configurations such as TTI, TTL, and pinnning covered under [Automatic Resource Control (ARC)](#)

  **Note**: Trying to estimate overheads per cache entry or per cache operation is not an effective approach to BigMemory sizing. There are too many factors in play during operation to be able to provide any real numbers to use for estimation.


3. Determine the total size for the Terracotta Server Array (TSA). Refer to [Worksheet 3](#worksheet-3-calculating-total-tsa-size).

  Total TSA size is the amount of BigMemory that will be necessary to hold and manage the data. This number will be the total data size (estimated in #1 above), plus the overhead for the other application requirements (assessed in #2 above).


4. Determine how many active servers will be in the TSA. Refer to [Worksheet 4](#worksheet-4-terracotta-servers-deployment).

  Now that you know the total amount of BigMemory, you need to decide how this will be distributed across the active servers in the TSA. You will need to know:
  * How much RAM is available for each server
  * If using Hybrid, how much SSD space is available for each server?
  * If not using Hybrid, how much HDD space is available for each server?

  For more discussion, see [Sizing Terracotta Server Arrays](#sizing-terracotta-aerver-arrays).


5. Determine how many additional servers will be in the TSA. You can use [Worksheet 4](#worksheet-4-terracotta-servers-deployment) to account for all of the servers in your installation.

  Reasons for additional servers:
  * If you are deploying a high availability installation, you will want at least one mirror server for each active server, forming a mirror group (also known as stripe). A mirror server should reside on a different physical machine than the active server it is mirroring, and it should have the same configuration as the active server. For more information, see [High Availability](http://terracotta.org/documentation/4.1/terracotta-server-array/high-availability).  
  * If you are deploying WAN replication, you will need to decide the number of servers at each data center, plus the servers needed for the WAN Orchestrators. For more information, see [BigMemory WAN Replication Service](http://terracotta.org/documentation/4.1/wan/introduction).
  * Other requirments that might affect the number of servers in your installation include backup, data loading, etc.

  For more discussion, see [Factors in Determining the Number of Servers](#factors-in-determining-the-number-of-servers).


6.  Determine how much BigMemory will be configured in the application servers (Terracotta clients). Refer to [Worksheet 5](#worksheet-5-terracotta-clients-deployment).

  Data that is "hot" (frequently read or updated) can be stored in BigMemory on your application servers to reduce latency. Knowing the approximate amount of hot data will help you to determine how much BigMemory to configure on the application servers (which are Terracotta clients when part of a BigMemory installation).

  Hot data will be a percentage of the total data stored in the TSA. The rest of the dataset will still be quickly accessible ("warm"). The TSA holds the entire dataset, while the application servers (Terracotta clients) hold the subset of the total that should be kept hot. With its Automatic Resource Control (ARC) feature, BigMemory determines where to store cache entries for optimal performance, based upon actual real-time data usage. Because of the dynamic nature of BigMemory's data management, you do not have to indicate which data should be kept where.

  For sizing, it is generally helpful to configure as much BigMemory on the Terracotta clients as they have available RAM.

  Some questions to help you determine the amount of BigMemory for Terracotta clients:
  * How often will data be created, read, updated, deleted?
  * How many clients will need to share data?  
  * How much extra RAM do clients have to be used for BigMemory?

  For more discussion, see [Factors in Determining the Number of Clients](#factors-in-determining-the-number-of-clients).

##Sizing Worksheets

The following sizing worksheets can be used in conjunction with the [Sizing Procedure Overview](#sizing-procedure-overview).

###Worksheet 1: Estimating Total Data Size

Use this worksheet to calculate the total data size by cache.

####Procedure
Load the cache, and then call `cache.calculateInMemorySize()` and `cache.calculateOffHeapSize()`. These methods will return the serialized memory consumption in bytes.

For large caches, this exercise could be done with a percentage of the cache. Load Elements that are representative of the typical Element sizes for that cache, and then multiply by the factor needed to represent the entire cache.  


| Cache Name | In-memory Size<br>(Heap Memory) | Off-heap Size | Total Bytes |
|:-------|:-------|:-------|:------------|
|||||
|||||
|||||
|||||
|||||
|||||
|||||
|||||
|||||
|||||
|Totals||||

**Note**: This should only be done as a measuring exercise and not in production, as there is a time cost to running the methods.

####Alternate method for estimating total data size

Alternatively, total data size can be estimated by cache entry. Cache entry size is the number of bytes the Element object takes when it is serialized. (An Element is comprised of a key, a value, and miscellaneous attributes.) For this approach, you can take one of the following approaches:

* Add the Element sizes of all Elements in all caches.

* Multiply the number of Elements in a cache by the average Element size to get a total size for the cache, and then add all of the caches sizes.

* Use the Terracotta Management Console (TMC). Set up a test cluster with the expected dataset, and connect it to the TMC. Then navigate to Application Data > Sizing Panel, and review the Relative Cache Sizes by Tier section. Note that the average cache-entry size reported in the TMC is an estimate. For more information, refer to [Using the TMC](/documentation/4.1/tms/tmc-using).


###Worksheet 2: Calculating Overhead

Overhead is the additional space, on top of dataset size, needed for operating with certain BigMemory features.

| Factor | Where configured | Overhead Calculation | Total |
|:-------|:-------|:-------|:------------|
|Straight Overhead<br>(TSA with RAM only)|Configured per server<br>dataStorage and offheap elements|Total data size x 1.2||
|Data persistence<br>(Fast Restartability)|Configured per TSA<br>restartable element |Requires ___% additional RAM<br>Requires ___% disk space||
|SSD usage<br>(BigMemory Hybrid)|Configured per server<br>dataStorage and offheap elements| total data size x 3.2<br>(overhead on SSD only)||
|Search|Configured per cache<br>searchable element| total data size x 3<br>(overhead on disk only) ||
|Data life - TTI/TTL||Requires 20% additional space||
|Data life - pinnning|||


###Worksheet 3: Calculating Total TSA Size

Total TSA size is the BigMemory needed for the dataset plus the overhead space for data management.

Use this worksheet to consolidate the information from Worksheets 1 & 2, and to determine TSA size and amount of BigMemory to configure in application servers (Terracotta clients).


| Factor | Calculation | Total    | Notes |
|:-------|:-------|:-------|:------------|
|Dataset Size|Total size of all caches|||
|Total Overhead| Calculated on Worksheet 2|||
|Total TSA Size|Total dataset size + Total Overhead|||
|Data "hot set"|Percentage of data that is frequently<br>accessed + percentage overhead|| Amount of off-heap to configure<br>in Terracotta clients|
|Data "warm set"|Remaining percentage of total data<br>+ percentage overhead for this data|||
|Size check|Total size of hot set<br>+ total size of warm set||Should equal Total TSA size|


###Worksheet 4: Terracotta Servers Deployment

Use this worksheet to account for the Terracotta servers that will be used in your TSA and WAN replication deployments.

Note that Total TSA Size calculated in Worksheet 3 must equal the sum of the `dataStorage` allocations of the active servers in the TSA. `dataStorage` configured for additional servers in a mirror group is not counted when figuring the total TSA size.

|Server Name | Location | Mirror Group Name | dataStorage Allocation | offheap<br>Allocation | SSD space available | HDD space available |
|:-------|:-------|:-------|:-------|:-------|:------------|
|||||||
|||||||
|||||||
|||||||
|||||||
|||||||
|||||||
|||||||
|||||||
|||||||
|||||||
|||||||
|||||||

SSD = Solid-state drive (non-moving disk)

HDD = Hard-disk drive (rotating disk)

###Worksheet 5: Terracotta Clients Deployment

Terracotta clients run on the same server machines as your application. Terracotta clients configured for BigMemory can hold a "hot set" of the total dataset.

Use this worksheet to account for the application servers running BigMemory.

| Server ID | Location | Heap<br>Allocation | RAM<br>Allocation |Direct Memory<br>Allocation|
|:-------|:-------|:-------|:-------|:------------|
||||||
||||||
||||||
||||||
||||||
||||||
||||||



##Sizing Client-side BigMemory Max

In the BigMemory installed on an application server -- also known as the Terracotta client or L1 -- size is controlled via Cache and CacheManager configurations of the heap and off-heap memory of the application servers.

###Local Heap

Local heap refers to the heap memory to be used by BigMemory on the application server (Terracotta client). The local heap allocation needs to be large enough to avoid out-of-memory errors but small enough to avoid excessive garbage collection. You should set maxEntriesLocalHeap to at least 100 elements when using an off-heap store to avoid performance degradation.

Configuration concerns the maximum number of entries or bytes a data set can use of the local heap memory or, when set at the CacheManager level (`maxBytesLocalHeap` only), as a pool available to all data sets under that CacheManager.

|| Measure | Configuration Attribute/Parameter|
|:-------|:-------|:-------|:------------|
| Cache | Count | maxEntriesLocalHeap |
||Byte| maxBytesLocalHeap |
| CacheManager | Count | n/a |
||Byte| maxBytesLocalHeap |
|JVM Heap Size|Byte|-Xmx|

For more information, see [Memory Store Configuration](http://www.terracotta.org/documentation/4.1/bigmemorymax/configuration/storage-options#memory-store).

####Byte-based Sizing
* maxBytesLocalHeap on the CacheManager can be either an absolute value, or a percentage of the JVM's maximum heap size.
* maxBytesLocalHeap on the cache can be either an absolute value, or a percentage of the enclosing CacheManager constraint (assuming there is one).

Byte-based sizing in the heap relies on the functionality of the size-of engine to correctly calculate the heap size of the objects stored in the cache.  Sizing with the size-of engine is not a zero cost operation, therefore using byte based sizing with values that consist of very deep object graphs can cause measurably reduced performance when putting to the cache. For more information, refer to [Built-in Sizing Computation and Enforcement](http://www.terracotta.org/documentation/4.1/bigmemorymax/configuration/cache-size#built-in-sizing-computation-and-enforcement).

####Count-based Sizing
Count based sizing does not have to invoke the size-of engine, so it can be more performant. It does however require you to have a good handle on the size of the stored values so that the count-based constraint can be converted easily in to a size constraint to compare against the rest of application's heap usage.

<table>
<caption>TIP: Calculating the Size of a Cache</caption>
<tr><td> <a href="http://www.ehcache.org/apidocs/2.8.4/net/sf/ehcache/Cache.html#calculateInMemorySize%28%29">calculateInMemorySize()</code></a> is a convenient method
which can provide an estimate of the size (in bytes) of the memory store. It returns the serialized size of the cache, providing a rough estimate. Do not use this method in production as it is has a negative effect on performance.</td></tr>
</table>

####JVM heap size
Allocate at least enough heap using the -Xmx Java option to accomodate the on-heap storage specified in your configuration, plus enough extra heap to run the rest of your application. This property is set by adding the parameter to the java invocation. For example:

    -Xmx1g

For additional information about heap memory, see [Tuning Heap Memory Performance](http://terracotta.org/documentation/4.1/bigmemorymax/best-practices#tuning-heap-memory-performance).


###Local Off-heap
Local off-heap refers to the RAM to be used by BigMemory on the application server (Terracotta client). The local off-heap allocation can be as large as the available RAM on the machine. The minimum amount that can be allocated is 1MB.

Part of successful sizing is to find the right balance between heap and off-heap. Heap storage is faster but has the disadvantage of being subject to garbage collection. Once GC is factored into the average TPS, the best strategy is usually to maximize off-heap and minimize heap memory, in order to reduce the drag that heap GC imposes on performance.

Configuration concerns the maximum number of bytes a data set can use in off-heap memory, or when set at the CacheManager level, as a pool available to all data sets under that CacheManager. `maxBytesLocalOffHeap` sets the amount of off-heap memory available to the cache and is in effect only if overflowToOffHeap is true. The minimum amount that can be allocated is 1MB. There is no maximum.


|| Measure | Configuration Attribute/Parameter|
|:-------|:-------|:-------|:------------|
| Cache | Count | n/a |
||Byte| maxBytesLocalOffHeap |
| CacheManager | Count | n/a |
||Byte| maxBytesLocalOffHeap |
|JVM Direct Memory|Byte|MaxDirectMemorySize|


Off-heap sizing is byte-based only because values stored in the off-heap must be serialized in order to be stored.

* maxBytesLocalOffHeap on a CacheManager can be either an absolute value, or a percentage of the maximum possible allocation. The maximum possible allocation being the minimum of the licensed amount, and the JVM imposed direct memory cap (MaxDirectMemorySize).
* maxBytesLocalOffHeap on a Cache can be either an absolute value, or a percentage of the enclosing CacheManagers size.

For more information, see [Off-heap Store Configuration](http://www.terracotta.org/documentation/4.1/bigmemorymax/configuration/storage-options#off-heap-store).

Note: At cache startup time, absolute cache constraints are subtracted from the CacheManager constraint, and the remainder is allocated as a single pool. If after initialization a new absolute-sized Cache is created, then the CacheManager pool size cannot be adjusted. This means that adding further absolute-sized off-heap caches after CacheManager initialization will cause the CacheManager off-heap size constraint to be violated.

####Direct Memory Configuration
The off-heap store uses the direct-memory portion of the JVM. Since direct memory may be shared with other processes, allocate at least 256MB (or preferably 1GB) more to direct memory than will be allocated to the off-heap store, while not exceeding physical RAM. This property is set by adding the parameter to the java invocation, for example:

    -XX:MaxDirectMemorySize=7G


For more information, refer to [Allocating Direct Memory in the JVM](http://www.terracotta.org/documentation/4.1/bigmemorymax/configuration/storage-options#allocating-direct-memory).

### Automatic Resource Control

With its Automatic Resource Control (ARC) feature, BigMemory determines where to store hot and warm cache entries for optimal performance, based upon actual real-time data usage. In addtion, BigMemory gives you fine-grained controls for tuning performance and enabling trade-offs between throughput, latency and data access. Independently adjustable configuration parameters include differentiated tier-based sizing and pinning hot or eternal data in the most effective tier.

#### Dynamic Sizing
Tuning often involves sizing stores appropriately. There are a number of ways to size the different BigMemory Max data tiers using simple configuration sizing attributes. The [Sizing](/documentation/4.1/bigmemorymax/configuration/cache-size) page explains how to tune tier sizing by configuring dynamic allocation of memory and automatic balancing.

#### Pinning Data
One of the most important aspects of running an in-memory data store involves managing the life of the data in each BigMemory Max tier. See the [Data Life](/documentation/4.1/bigmemorymax/configuration/data-life) page for more information on the pinning, expiration, and eviction of data.


####Balancing Automatic and Manual Tuning
A cache that is likely to be hit more often, or whose contents are more expensive to replace, may need to be allocated more space. ARC makes this kind of cache utility judgment automatically when you use CacheManager scoped sizing, but it can also be done manually at either the Cache or CacheManager level.  Exactly how important this balancing procedure is, and whether or not ARC can be trusted to perform it automatically, will depend on both the application's workload and latency requirements.

####Calculating Object Sizes
When using Automatic Resource Control, BigMemory calculates object sizes using `net.sf.ehcache.pool.impl.DefaultSizeOfEngine`.

A new pluggable sizeof engine module is available at [http://terracotta-oss.github.io/ehcache-sizeofengine/](http://terracotta-oss.github.io/ehcache-sizeofengine/).



##Sizing Terracotta Servers

In the BigMemory installed on a server in a Terracotta Server Array -- also known as a Terracotta server or L2 -- size is controlled via cache and dataStorage configurations. This section also discusses server hardware and heap memory requirements.

###Server Hardware

It is recommended to run Terracotta servers on physically different hardware than the application servers (Terracotta clients).

CPU requirements depend upon factors such as the data sizes, frequency and complexity of operations, and performance requirements. It is recommended that every server have at least one core. Because servers are multi-threaded processes, running on more than one core can be beneficial.

In general, CPU utilization should not exceed 80%. If you find that the CPU utilization is greater than 80%, you should add more cores or server machines.

####Disk usage (HDD) with off-heap only servers

The Terracotta Server Array can be configured to be restartable in addition to including searchable caches, but both of these features require HDD space. When both are enabled, be sure that enough disk space is available. Depending upon the number of searchable attributes, the amount of disk space required might be 3 times the amount of in-memory data.

It is highly recommended to store the search index (`<index>`) and the Fast Restart data (`<data>`) on separate disks.

####Disk usage (SSD) with hybrid servers

BigMemory Hybrid allows you to extend the space for BigMemory by adding Terracotta server SSD to the off-heap RAM, so that the data size can exceed the available off-heap memory.

BigMemory Hybrid supports writing to a single mount per Terracotta server. It is recommended that the mount be used exclusively for the Terracotta server process.

To account for the overhead necessary for consistent performance, the formulas below are suggested as initial starting points for sizing the amount of space allocated for BigMemory Hybrid operation.

Minimum SSD flash memory requirement = planned total data size * 3.2

Minimum DRAM requirement = planned maximum number of elements * (168 + key size)

Note: It is strongly recommended to configure enough offheap to accommodate all cache keys in DRAM.

**Note**: The effective capacity of the TSA is not the sum of the offheap and disk. The offheap acts as a cache of the disk. The capacity is reflected in the `dataStorage` configuration which includes off-heap and SSD/Flash storage within it.


###Server heap memory allocation

The heap allocation needs to be large enough to avoid OutOfMemory errors but small enough to avoid excessive garbage collection.

Refer to the following table for store size guidelines for servers in the TSA.

| When Off-heap is set between | Configure at least this much Heap |
|:-------|:------------|
|4 - 10 GB|1 GB (Note: The default heap size is 2 GB.)|
|10 - 100 GB|2 GB|
|100 GB - 1 TB +|3 GB +|
|(Off-heap is configured in the tc-config.xml)|(Heap is configured using the -Xmx Java option)|


####JVM heap size
Allocate at least enough heap using the `-Xmx` Java option to accomodate the on-heap storage specified in your configuration, plus enough extra heap to run the rest of your application. This property is set by adding it to the command-line parameters for the server startup script.


###Server Off-heap

Each server requires an off-heap store. The maximum off-heap size is limited only by the amount of memory in your server. The minimum off-heap size is 4 GB. Be sure to allocate at least 20 percent more off-heap memory than the size of your data set--however, this percentage can vary overhead requirements.

Server off-heap size is configured in the tc-config.xml file.

    <tc-config>
      <servers>
        <server>
          <dataStorage>
            <offheap size="200g"/>



Additional hybrid storage is optional. Specify the `<dataStorage>` size and the `<offheap>` size. The `<offheap>` size can be set to the amount of memory available in your server for data. If you enable `<hybrid>`, then the `<dataStorage>` size can exceed the `<offheap>` size.

        <dataStorage size=”800g”>
           <offheap size=”200g”/>
           <hybrid/>


####Direct Memory and Off-heap Memory Allocations
To accommodate server communications layer requirements, the value of `MaxDirectMemorySize` must be greater than the value of `offheap`. The exact amount greater depends upon the size of `offheap`. The minimum is 256MB, but if you allocate 1GB more to the `MaxDirectMemorySize`, it should be sufficient. The server will only use what it needs and the rest will remain available. This property is set by adding it to the command-line parameters for the server startup script.

    -XX:MaxDirectMemorySize=201G

For additional information about server off-heap, refer to [General Memory Allocation](http://terracotta.org/documentation/4.1/bigmemorymax/best-practices#general-memory-allocation).




##Sizing Terracotta Server Arrays

Sizing the TSA involves taking into account all of the resources to be employed for BigMemory operation. Typically, the TSA has more than one active server (which may be part of a stripe or mirror group) that manages off-heap RAM and SSD or HDD space. All of the active servers with all of the their RAM and disk space comprise the capacity of the TSA.

###Data Sizing Configurations

The attribute `maxEntriesInCache` configures the maximum number of entries in a distributed cache. The `dataStorage` and `offheap` elements are used to constrain the amount of data stored in a given Terracotta Server.

| Tier | Measure | Attribute/Element | Configuration |
|:-------|:-------|:-------|:------------|
| Cache | Count | `maxEntriesInCache` | ehcache.xml |
| Server RAM | Byte |  `offheap` | tc-config.xml |
| Server RAM plus SSD | Byte |  `dataStorage` | tc-config.xml |

Note that individual caches are not assigned to single servers. Generally the distribution of data within the BigMemory installation is handled by Automatic Resource Control (ARC).

###TSA Sizing Considerations
The Terracotta Server Array size should be large enough to hold the total data size, plus the required additional space for data management (overhead). The formula is:

TSA size = total data size + required additional space

where:

TSA size = the sum of the `dataStorage` size of all active servers in the server array

and

Total data size = the sum of all of the distributed caches (all `maxEntriesInCache` configurations)

The required additional space depends upon your configuration. The following are general guidelines:

* A TSA with RAM only (no Hybrid or Fast Restart) should be configured to hold at least 1.2 times the total data size.

* A TSA with Fast Restart enabled should be configured with disks that can hold at least 2.5 times the size of the data. For more information, see the Fast Restart section below.

* A TSA with Hybrid enabled should be configured with SSD/Flash that can hold at least 3.2 times the size of the data. For more information, see the Hybrid section below.

### Sizing and ARC
With Automatic Resource Control (ARC), the TSA keeps as much data in memory without eviction and still protects against out-of-memory errors...

The resource-based evictor monitors disk usage as well as dataStorage and offheap sizes. If a resource goes over its critical threshold, the evictor will work contiually until the resource falls below the critical threshold.

With larger configured dataStorage sizes (relative to the total dataset size) the eviction threshold could be higher than 80%, with smaller sizes it could be less than 80%.

When planning to achieve a certain TPS window, you should take evictions into account. Evictions happen when elements expire, but there are also evictions of unexpired elements if BigMemory usage reaches critical thresholds. You can avoid unexpired evictions which could affect your average TPS by allocating extra space for data management. If you can keep resource usage under 80% of the dataStorage size, this will help to maintain a steady TPS.

As part of TPS you are expecting you need to account for the evictions.


###Factors in Determining the Number of Servers
The following factors should be considered when determining the number of servers or server stripes (mirror groups):

* Data TPS - cache transactions per second. A transaction is any CRUD operation that requires access to Terracotta. However, in cases where there are multiple reads within the same JVM, they do not count as separate read transactions.

* Read/Write Ratios and sticky sessions - Read-heavy scenarios with sticky sessions would have locality of data. This would require less stripping to handle traditional loads. Write-heavy scenarios or scenarios with read/write ratio as 50:50 would need to stripe out faster than a 90-10 read write ratio.

* Average Object Sizes - Is the size of an average object 1KB, or 1MB or 10MB. Smaller objects can be easily transmitted across the wire.

* Search - Search indexes are partitioned across stripes and performs better with additional stripes. While search performance is use-case dependent, there can be application level tuning done to address this as well.

* Amount of total data and hardware limitations - Hardware RAM constraints and total dataset sizes also account for sizing. For example, if you have 100GB of data and your physical boxes have only 30GB, you need 4 stripes. Size of BigMemory is roughly 20% more added to the raw object size of the data.

* Is there a hot set of data - L1 BigMemory can be used to host a hot set if there is one. This needs to be factored in the calculations as well.

* CPU constraints - The CPU that TSA runs on should not be maxed out. If there is a CPU constraint, then you probably need to scale out as well.

* Data access patterns, consistency requirements, usage of Compare and Swap (CAS) operations or other usage of locks  


###Factors in Determining the Number of Clients

While the number of L1s that can exist in a Terracotta cluster is theoretically unbounded (and cannot be configured), effectively planning for resource limitations and the size of the shared data set should yield an optimum number. Typically, the most important factors that will impact that number are the requirements for performance and availability. Typical questions when sizing a cluster:

* What is the desired transactions-per-second?
* What are the failover scenarios?
* How fast and reliable is the network and hardware? How much memory and disk space will each machine have?
* How much shared data is going to be stored in the cluster? How much of that data should be on the L1s? Will BigMemory be used?
* How many stripes (active Terracotta servers) does the cluster have?
* How much load will there be on the L1s? On the L2s?

A TSA node follows SEDA architecture, therefore sockets and disk io are asynchronous, and the number of threads in the system does not grow linearly with the number of clients. In addition, the `concurrency` attribute may be tuned. This means that a TSA could theoretically handle tens of thousands of clients, depending on how far SEDA stages can be stretched. But while there is no theoretical limit of L1s connected to a TSA, the client count depends very much on what the clients does. If fewer clients are keeping TSA busy, then there is no benefit to adding more clients.




##Approach to Experimentation

The most important method for determining the optimum size of a cluster is to test various cluster configurations under load and observe how well each setup meets overall requirements.


Sizing for the number of stripes of TSA Servers is not an exact science. Its simillar to sizing the number of Application Servers that can handle a given ecommerce load.
We evaluate a number of different categories. It is an art and is determined by finding the bottleneck. Find the "weakest link" and eliminate it, go on to the "next weakest link," eliminate it, and so on.

Consider the number of factors at play. To start, the form of the data in a text file is going to be very different from the form of the data in the JVM (as Java data types). It also depends on how you aggregate those fields (e.g. onto the same java object, in a collection, etc.) -- many different things will affect the size of just your raw data. And then the form you choose to represent the data will affect how it serializes into off-heap. And which fields you decide to have searchable will affect how much space indexing needs...

The best way to be *really sure* you are sizing properly is to make a small program that actually creates cache entries that look exactly as they will in the actual application, load some many thousands of them into a cache (configured the same way the actual program will configure it) and then look at how much space that uses in order to project how much space the full data set will need.



##Case Studies in Sizing [See Cases for sizing.docx attached to BF-2397]

Case 1: Element overhead

Case 2: Upper bounds of usage to still have consistent performance

Case 3: Reducing the number of servers, increasing memory/disk size per server

Case 4: Tuning to maximize performance

Case 5: Effect of request size on TPS

Case 6: Comparison of read/write of large files in memory versus on disk
