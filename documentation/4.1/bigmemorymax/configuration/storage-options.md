---
---
# Storage Tiers {#Storage-Options}
{toc|2:3}

## Introduction
BigMemory Max has three storage tiers, summarized here:

* Memory store &ndash; Heap memory that holds a copy of the hottest subset of data from the off-heap store. Subject to Java GC.
* Off-heap store &ndash; Limited in size only by available RAM. Not subject to Java GC. Can store serialized data only. Provides overflow capacity to the memory store.
* Disk store &ndash; Backs up in-memory data and provides overflow capacity to the other tiers. Can store serialized data only. **Note**: The disk tier is available for local (standalone) instances of BigMemory Max. For distributed BigMemory Max, a Terracotta server or server array supplies the third tier. (For more information, refer to the [Distributed BigMemory Max Configuration Guide](/documentation/4.1/bigmemorymax/configuration/distributed-configuration).)

This document defines the standalone storage tiers and their suitable element types and then details the configuration for each storage tier.

Before running in production, it is strongly recommended that you test the BigMemory Max tiers with the actual amount of data you expect to use in production. For information about sizing the tiers, refer to [Configuration Sizing Attributes](/documentation/4.1/bigmemorymax/configuration/cache-size#sizing-attributes).

## Memory Store
The memory store is always enabled and exists in heap memory. For the best performance, allot as much heap memory as possible without triggering GC pauses, and use the [off-heap store](#off-heap-store) to hold the data that cannot fit in heap (without causing GC pauses).

The memory store has the following characteristics:

* Accepts all data, whether serializable or not
* Fastest storage option
* Thread safe for use by multiple concurrent threads

## Configuring the Memory Store
The memory store is the top tier and is automatically used by BigMemory Max to store the data hotset because it is the fastest store. It requires no special configuration to enable, and its overall size is taken from the Java heap size. Since it exists in the heap, it is limited by Java GC constraints.

### Memory Use, Spooling, and Expiry Strategy in the Memory Store

All caches specify their maximum in-memory size, in terms of the number of elements, at configuration time.

When an element is added to a cache and it goes beyond its maximum
memory size, an existing element is either deleted, if overflow
is not enabled, or evaluated for spooling to another tier, if overflow is enabled. The overflow options are `overflowToOffHeap` and `<persistence>` (disk store).

If overflow is enabled, a check for expiry is carried out. If it is expired
it is deleted; if not it is spooled. The eviction of an item from
the memory store is based on the optional `MemoryStoreEvictionPolicy` attribute
specified in the configuration file. Legal values are LRU (default), LFU and FIFO:

* **Least Recently Used (LRU)**&mdash;LRU is the default setting. The last-used timestamp is updated when an element is put into the cache or an element is retrieved from the cache with a get call.
* **Least Frequently Used (LFU)**&mdash;For each get call on the element the number of hits is updated. When a put call is made for a new element (and assuming that the max limit is reached for the memory store) the element with least number of hits, the Less Frequently Used element, is evicted.
* **First In First Out (FIFO)**&mdash;Elements are evicted in the same order as they come in. When a put call is made for a new element (and assuming that the max limit is reached for the memory store) the element that was placed first (First-In) in the store is the candidate for eviction (First-Out).

For all the eviction policies there are also `putQuiet` and `getQuiet` methods which do not update the last used timestamp.

When there is a `get` or a `getQuiet` on an element, it is checked for expiry. If expired, it is removed and null is returned. Note that at any point in time there will usually be some expired elements in the cache. Memory sizing of an application must always take into
account the maximum size of each cache.

<table>
<caption>TIP: Calculating the Size of a Cache</caption>
<tr><td> <a href="http://www.ehcache.org/apidocs/2.8.4/net/sf/ehcache/Cache.html#calculateInMemorySize%28%29">calculateInMemorySize()</code></a> is a convenient method
which can provide an estimate of the size (in bytes) of the memory store. It returns the serialized size of the cache, providing a rough estimate. Do not use this method in production as it is has a negative effect on performance.</td></tr>
</table>

An alternative would be to have an expiry thread. This is a trade-off between lower memory use and short locking periods and CPU utilization. The design is in favour of the latter. For those concerned with memory use, simply reduce the tier size. For more information, refer to [Sizing Storage Tiers](/documentation/4.1/bigmemorymax/configuration/cache-size).


## Off-Heap Store

The off-heap store extends the in-memory store to memory outside the of the object heap. This store, which is not subject to Java GC, is limited only by the amount of RAM available.

Because off-heap data is stored in bytes, only data that is `Serializable` is suitable for the off-heap store. Any non serializable data overflowing to the `OffHeapMemoryStore` is simply removed, and a WARNING level log message emitted.

Since serialization and deserialization take place on putting and getting from the off-heap store, it is theoretically slower than the memory store. This difference, however, is mitigated when GC involved with larger heaps is taken into account.

### Allocating Direct Memory in the JVM {#allocating-direct-memory}
The off-heap store uses the direct-memory portion of the JVM. You must allocate sufficient direct memory for the off-heap store by using the JVM property `MaxDirectMemorySize`.

For example, to allocate 2GB of direct memory in the JVM:

    java -XX:MaxDirectMemorySize=2G ...

Since direct memory may be shared with other processes, allocate at least 256MB (or preferably 1GB) more to direct memory than will be allocated to the off-heap store.

Note the following about allocating direct memory:

* If you configure off-heap memory but do not allocate direct memory with -XX:MaxDirectMemorySize, the default value for direct memory depends on your version of your JVM. Oracle HotSpot has a default equal to maximum heap size (-Xmx value), although some early versions may default to a particular value.
* MaxDirectMemorySize must be added to the local node's startup environment.
* Direct memory, which is part of the Java process heap, is separate from the object heap allocated by -Xmx. The value allocated by MaxDirectMemorySize must not exceed physical RAM, and is likely to be less than total available RAM due to other memory requirements.
* The amount of direct memory allocated must be within the constraints of available system memory and configured off-heap memory.
* The maximum amount of direct memory space you can use depends on the process data model (32-bit or 64-bit) and the associated operating system limitations, the amount of virtual memory available on the system, and the amount of physical memory available on the system.

#### Using Off-Heap Store with 32-bit JVMs
The amount of heap-offload you can achieve is limited by addressable memory. 64-bit systems can allow as much memory as the hardware and operating system can handle, while 32-bit systems have strict limitations on the amount of memory that can be effectively managed.

For a 32-bit process model, the maximum virtual address size of the process is typically 4GB, though most 32-bit operating systems have a 2GB limit. The maximum heap size available to Java is lower still due to particular operating-system (OS) limitations, other operations that may run on the machine (such as mmap operations used by certain APIs), and various JVM requirements for loading shared libraries and other code. A useful rule to observe is to allocate no more to off-heap memory than what is left over after -Xmx is set. For example, if you set -Xmx3G, then off-heap should be no more than 1GB. Breaking this rule may not cause an OutOfMemoryError on startup, but one is likely to occur at some point during the JVM's life.

If Java GC issues are afflicting a 32-bit JVM, then off-heap store can help. However, note the following:

* Everything has to fit in 4GB of addressable space. If 2GB of heap is allocated (with `-Xmx2g`) then at most are are 2GB left for off-heap data.
* The JVM process requires some of the 4GB of addressable space for its code and shared libraries plus any extra Operating System overhead.
* Allocating a 3GB heap with -Xmx, as well as 2047MB of off-heap memory, will not cause an error at startup, but when it's time to grow the heap an OutOfMemoryError is likely.
* If both -Xms3G and -Xmx3G are used with 2047MB of off-heap memory, the virtual machine will start but then complain as soon as the off-heap store tries to allocate the off-heap buffers.
* Some APIs, such as `java.util.zip.ZipFile` on a 1.5 JVM, may &lt;mmap> files in memory. This will also use up process space and may trigger an OutOfMemoryError.

## Configuring the Off-Heap Store {#off-heap-store-config}
If an off-heap store is configured, the corresponding memory store overflows to that off-heap store. Configuring an off-heap store can be done either through XML or programmatically. With either method, the off-heap store is configured on a per-cache basis.

### Declarative Configuration

The following XML configuration creates an off-heap cache with an in-heap store (maxEntriesLocalHeap)
of 10,000 elements which overflow to a 1-gigabyte off-heap area.

    <ehcache updateCheck="false" monitoring="off" dynamicConfig="false">
      <defaultCache maxEntriesLocalHeap="10000"
                  eternal="true"
                  memoryStoreEvictionPolicy="LRU"
                  statistics="false" />

      <cache name="sample-offheap-cache"
           maxEntriesLocalHeap="10000"
           eternal="true"
           memoryStoreEvictionPolicy="LRU"
           overflowToOffHeap="true"
           maxBytesLocalOffHeap="1G"/>
    </ehcache>

The configuration options are:

#### overflowToOffHeap

Values may be true or false. When set to true, enables the cache to utilize off-heap memory
storage to improve performance. Off-heap memory is not subject to Java
GC cycles and has a size limit set by the Java property MaxDirectMemorySize.
The default value is false.

#### maxBytesLocalOffHeap
Sets the amount of off-heap memory available to the cache and is in effect only if `overflowToOffHeap` is tr
ue. The minimum amount that can be allocated is 1MB. There is no maximum.

For more information on sizing data stores, refer to [Sizing Storage Tiers](/documentation/4.1/bigmemorymax/configuration/cache-size).

<table>
<caption>NOTE: Heap Configuration When Using an Off-heap Store</caption>
<tr><td>
You should set maxEntriesLocalHeap to at least 100 elements
when using an off-heap store to avoid performance degradation. Lower values for maxEntriesLocalHeap trigger a warning to be logged.
</td></tr>
</table>

### Programmatic Configuration {#programmatic-configuration}

The equivalent cache can be created using the following programmatic configuration:

    public Cache createOffHeapCache()
    {
      CacheConfiguration config = new CacheConfiguration("sample-offheap-cache", 10000)
      .overflowToOffHeap(true).maxBytesLocalOffHeap("1G");
      Cache cache = new Cache(config);
      manager.addCache(cache);
      return cache;
    }




## Disk Store {#DiskStore}
The disk store provides a thread-safe disk-spooling facility that can be used for either additional storage or persisting data through system restarts.

**Note**: The disk tier is available for local (standalone) instances of BigMemory Max. For distributed BigMemory Max, a Terracotta server or server array is used instead of a disk tier. (For more information, refer to [Terracotta Server Arrays](/documentation/4.1/terracotta-server-array/server-arrays#disk-usage).)

This section covers local disk usage. Additional information may also be found on the [Fast Restartability](/documentation/4.1/bigmemorymax/configuration/fast-restart) page.

### Serialization
Only data that is `Serializable` can be placed in the disk store. Writes to and from the disk use `ObjectInputStream` and the Java serialization mechanism. Any non-serializable data overflowing to the disk store is removed and a NotSerializableException is thrown.

Serialization speed is affected by the size of the objects being
serialized and their type. It has been found that:

  * The serialization time for a Java object consisting of a large Map of String arrays was 126ms, where the serialized size was 349,225 bytes.
  * The serialization time for a byte[] was 7ms, where the serialized size was 310,232 bytes.

Byte arrays are 20 times faster to serialize, making them a better choice for increasing disk-store performance.


## Configuring the Disk Store

Disk stores are configured on a per CacheManager basis. If one or more caches requires a disk store but none is configured, a default directory is used and
a warning message is logged to encourage explicit configuration of the disk store path.

Configuring a disk store is optional. If all caches use only memory and off-heap stores, then there is no need to configure a disk store. This simplifies configuration, and uses fewer threads. This also makes it unnecessary to configure multiple disk store paths when multiple CacheManagers are being used.

### Disk Storage Options

Two disk store options are available:

*  Temporary store ("localTempSwap")
*  Persistent store ("localRestartable")

See the following sections for more information on these options.

#### localTempSwap

The "localTempSwap" persistence strategy allows local disk usage during BigMemory operation, providing an extra tier for storage. This disk storage is temporary and is cleared after a restart.

If the disk store path is not specified, a default path is used, and the default will be auto-resolved in the case of a conflict with another CacheManager.

The localTempSwap disk store creates a data file for each cache on startup called "`<cache_name>`.data".

#### localRestartable

This option implements a restartable store for all in-memory data. After any restart, the data set is automatically reloaded from disk to the in-memory stores.

The path to the directory where any required disk files will be created is configured with the `<diskStore>`  sub-element of the Ehcache configuration. In order to use the restartable store, a unique and explicitly specified path is required.

### Disk Store Configuration Element

Files are created in the directory specified by the `<diskStore>`
configuration element. The `<diskStore>` element has one attribute called `path`.

    <diskStore path="/path/to/store/data"/>

Legal values for the path attibute are legal file system paths. For example, for Unix:

<pre>
/home/application/cache
</pre>

The following system properties are also legal, in which case they are translated:

* user.home - User's home directory
* user.dir - User's current working directory
* java.io.tmpdir - Default temp file path
* ehcache.disk.store.dir - A system property you would normally specify on the command line&mdash;for example, `java -Dehcache.disk.store.dir=/u01/myapp/diskdir`.

Subdirectories can be specified below the system property, for example:

<pre>
user.dir/one
</pre>


To programmatically set a disk store path:

    DiskStoreConfiguration diskStoreConfiguration = new DiskStoreConfiguration();

    diskStoreConfiguration.setPath("/my/path/dir");

    // Already created a configuration object ...
    configuration.addDiskStore(diskStoreConfiguration);

    CacheManager mgr = new CacheManager(configuration);

**Note**: A CacheManager's disk store path cannot be changed once it is set in configuration. If the disk store path is changed, the CacheManager must be recycled for the new path to take effect.


### Disk Store Expiry and Eviction
Expired elements are eventually evicted to free up disk space. The element is also removed from the in-memory index of elements.

One thread per cache is used to remove expired elements. The optional attribute `diskExpiryThreadIntervalSeconds`
sets the interval between runs of the expiry thread.

**Warning**: Setting `diskExpiryThreadIntervalSeconds` to a low value can cause excessive disk-store locking and high CPU utilization. The default value is 120 seconds.

If a cache's disk store has a limited size, Elements will be evicted from the disk store when it exceeds this limit. The LFU algorithm is used for these evictions. It is not configurable or changeable.

**Note**: With the "localTempSwap" strategy, you can use `maxEntriesLocalDisk` or `maxBytesLocalDisk` at either the Cache or CacheManager level to control the size of the disk tier.


### Turning off Disk Stores

To turn off disk store path creation, comment out the `diskStore` element in ehcache.xml.

The default Ehcache configuration, ehcache-failsafe.xml, uses a disk store. To avoid use of a disk store, specify a custom ehcache.xml with the `diskStore` element commented out.


## Configuration Examples

These examples show how to allocate 8GB of machine memory to different stores. It assumes a
data set of 7GB - say for a cache of 7M items (each 1kb in size).

Those who want minimal application response-time variance (or minimizing GC pause times), will likely
want all the cache to be off-heap.
Assuming that 1GB of heap is needed for the rest of the app, they will set their Java config as follows:

<pre>
java -Xms1G -Xmx1G -XX:maxDirectMemorySize=7G
</pre>

And their Ehcache config as:

<pre>
&lt;cache
 maxEntriesLocalHeap=100
 overflowToOffHeap="true"
 maxBytesLocalOffHeap="6G"
... /&gt;
</pre>

<table>
<caption>NOTE: Direct Memory and Off-heap Memory Allocations</caption>
<tr><td>
To accommodate server communications layer requirements, the value of maxDirectMemorySize must be greater than the value of maxBytesLocalOffHeap. The exact amount greater depends upon the size of maxBytesLocalOffHeap. The minimum is 256MB, but if you allocate 1GB more to the maxDirectMemorySize, it will certainly be sufficient. The server will only use what it needs and the rest will remain available.
</td></tr>
</table>

Those who want best possible performance for a hot set of data, while still reducing overall application repsonse time variance, will likely want a combination of on-heap and off-heap. The heap will be used for the hot set, the offheap for the rest. So, for example
if the hot set is 1M items (or 1GB) of the 7GB data. They will set their Java config as follows

<pre>
java -Xms2G -Xmx2G -XX:maxDirectMemorySize=6G
</pre>

And their Ehcache config as:

<pre>
&lt;cache
  maxEntriesLocalHeap=1M
  overflowToOffHeap="true"
  maxBytesLocalOffHeap="5G"
  ...&gt;
</pre>

This configuration will compare VERY favorably against the alternative of keeping the less-hot set in a
database (100x slower) or caching on local disk (20x slower).

Where the data set is small and pauses are not a problem, the whole data set can be kept on heap:

<pre>
&lt;cache
  maxEntriesLocalHeap=1M
  overflowToOffHeap="false"
  ...&gt;
</pre>

Where latency isn't an issue, the disk can be used:

<pre>
&lt;cache
  maxEntriesLocalHeap=1M
  &lt;persistence strategy="localTempSwap"/>
  ...&gt;
</pre>
