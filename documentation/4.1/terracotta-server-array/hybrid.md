---
---
# BigMemory Hybrid
{toc|2:3}

## Introduction

BigMemory Hybrid is an optional extension to BigMemory Max that:

* Enables scaling up to economical solid-state drive (SSD) flash memory motionless “disks” in conjunction with conventional dynamic random-access memory (DRAM) memory. This means that the cache size can exceed the available off-heap memory.
* Provides predictable low latency at very large scale.
* Manages the data flow seamlessly and automatically from DRAM to flash “disk” according to the size of the cache.

    ![Hybrid Graphic](/images/documentation/hybrid1.png)

  *Figure 1. BigMemory Hybrid allows you to expand BigMemory in Terracotta servers, keeping more data closer to your application for increased transactions per second (TPS).*

* Performs much faster than conventional hard disks, although not quite as fast as a pure DRAM in-memory solution.
* Supports searching, Fast Restart backup and recovery, Web Sessions, Quartz, and WAN replication.
* Works with industry-standard SSD devices from popular vendors, such as Fusion IO and Intel SSD.

####How is BigMemory Hybrid different than Overflow to Disk?

| BigMemory Hybrid | Overflow to Disk |
|:-------|:------------|
|Available in version 4.1|Available in version 3.7|
|Leverages BigMemory's Fast Restart technology|Depends upon Berkeley DB to store data on disk|
|Optimizes management of data between off-heap and SSD for predictable performance|If data does not fit in off-heap, it is pushed to disk, hence performance is less predictable|
|Optimized for SSD usage|No optimization done for SSDs|


## System Requirements

BigMemory Hybrid supports writing to one single mount, so all of the BigMemory Hybrid capacity must be presented to the Terracotta process as one continuous region, which can be a single device or a RAID.

The mount should be used exclusively for the Terracotta server process.

**Note**: System utilization is higher when using BigMemory Hybrid, and it is not recommended to run multiple servers on the same machine. Doing so could result health checkers timing out, and killing or restarting servers. Therefore, it is important to provision sufficient hardware, and it is highly recommended to deploy servers on different machines.

## Hardware Capacity Guidelines
To account for the overhead necessary for consistent performance, the formulas below are suggested as initial starting points for sizing the amount of space allocated for BigMemory Hybrid operation.

Minimum SSD flash memory requirement = planned total data size * 3.2  

Minimum DRAM requirement = planned maximum number of elements * (168 + key size)

**Note**: It is strongly recommended to configure enough offheap to accommodate all cache keys in DRAM.


## Configuration

To configure BigMemory Hybrid, include the following elements in the `tc-config.xml` file:

* **dataStorage** &mdash; Specifies the maximum amount of data you plan to store on the server, using either DRAM alone or both DRAM and SSD flash memory.  
* **offheap**  &mdash; Specifies the maximum amount of data to hold in DRAM.
* **hybrid** &mdash; Enables the Hybrid option to use SSD flash memory in addition to off-heap DRAM.

For example:

    <servers>
        ....
        <server host="hostname" name="server1">
            ...
            <dataStorage size=”800g”>
               <offheap size=”200g”/>
               <hybrid/>
            </dataStorage>
        </server>
    </servers>

For Terracotta servers, a minimum of 4 GB is recommended for the size attribute of the `offheap` element.

If the `hybrid` element is present, then the BigMemory Hybrid functionality is enabled. With Hybrid enabled, the value of the size attribute for the `dataStorage` element can exceed that of the size attribute for the  `offheap` element. This enables SSD devices to supplement the DRAM and be many times larger than the DRAM.

If the `hybrid` element is absent, then BigMemory Hybrid functionality is off. With Hybrid off, the value of the size attribute for the `dataStorage` element must be less than or equal to the value of the size attribute for the `offheap` element. In this case, the `offheap` element is not required.

If the `dataStorage` element is absent, `dataStorage` size and `offheap` size default to 512 MB.

Although the `dataStorage` element is optional, if included, this element must have a value assigned to its size attribute.

**Note**: If you are migrating from BigMemory Max 4.0 to 4.1, the `dataStorage` element has replaced the `maxDataSize` element. The old element is still compatible for pure DRAM operation, but to enable Hybrid mode, you must use the new 4.1-compatible dataStorage element with the `hybrid` tag.

###Disk Storage Path
BigMemory Hybrid requires a unique and explicitly specified path. The default path is the Terracotta server's home directory. You can customize the path using the `<data>` element in the server's `tc-config.xml` configuration file.


###BigMemory Hybrid and Fast Restartability
If [Fast Restartabillity](/documentation/4.1/terracotta-server-array/server-arrays#fast-restartability) is enabled, then if you have a restart, data will be loaded into BigMemory Hybrid in the same way as for BigMemory, with no difference in behavior or time required to get the system running again.

If [Fast Restartabillity](/documentation/4.1/terracotta-server-array/server-arrays#fast-restartability) is not enabled, then on restart, you will have some artifacts from the previous run left on disk, and you may want to remove them. For more information, refer to [Clearing a Server’s Data](/documentation/4.1/terracotta-server-array/operations#clearing-a-servers-data).


## System Operations in the TMC

When you use the Terracotta Management Console (TMC), you can see the effect of the BigMemory Hybrid feature in the Monitoring > Runtime Stats panel as “Data Storage Usage”.
When the cache is operating at a steady state, the Data Used typically exceeds the OffHeap Max shown in the "OffHeap Usage" graph:

![Data Used  Graphic2](/images/documentation/monitoring-graphs.png)

## Operator Events

BigMemory Hybrid supports the existing operator events in the Terracotta Server Array (TSA), including

* a Throttle Mode when the amount of either RAM, Flash, or both is approaching its capacity limit.
* a Restricted Mode when the system has entered read-only mode.

For more information about TSA operations, see [Automatic Resource Management](/documentation/4.1/terracotta-server-array/operations#automatic-resource-management) and [Near-Memory-Full Conditions](/documentation/4.1/terracotta-server-array/operations#automatic-resource-management).
