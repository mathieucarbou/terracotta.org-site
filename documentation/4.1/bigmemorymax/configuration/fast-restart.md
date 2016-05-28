---
---
# Fast Restartability {#Fast-Restart}
{toc|2:3}

## Introduction

BigMemory's Fast Restart feature provides enterprise-ready crash resilience by keeping a fully consistent, real-time record of your in-memory data. After any kind of shutdown &mdash; planned or unplanned &mdash; the next time your application starts up, all of the data that was in BigMemory is still available and very quickly accessible.

The advantages of the Fast Restart feature include:

* In-memory data survives crashes and enables fast restarts. Because your in-memory data does not need to be reloaded from a remote data source, applications can resume at full speed after a restart.

* A real-time record of your in-memory data provides true fault tolerance. Even with BigMemory, where terabytes of data can be held in memory, the Fast Restart feature provides the equivalent of a local "hot mirror," which guarantees full data consistency.

* A consistent record of your in-memory data opens many possibilities for business innovation, such as arranging data sets according to time-based needs or moving data sets around to different locations. The uses of the Fast Restart store can range from a simple key-value persistence mechanism with fast read performance, to an operational store with in-memory speeds during operation for both reads and writes.


## Data Persistence Implementation
The BigMemory Fast Restart feature works by creating a real-time record of the in-memory data, which it persists in a Fast Restart store on the local disk. After any restart, the data that was last in memory (both heap and off-heap stores) automatically loads from the Fast Restart store back into memory.

Data persistence is configured by adding the `<persistence>` sub-element to a cache configuration. The `<persistence>` sub-element includes two attributes: `strategy` and `synchronousWrites`.

    <cache>
       <persistence strategy="localRestartable|localTempSwap|none|distributed"
                    synchronousWrites="false|true"/>
    </cache>

###Strategy Options
The options for the `strategy` attribute are:

*  **"localRestartable"** &mdash;  Enables the Fast Restart feature which automatically logs all BigMemory data. This option provides fast restartability with fault tolerant data persistence.

*  **"localTempSwap"** &mdash; Enables temporary local disk usage. This option provides an extra tier for data storage during operation, but this store is not persisted. After a restart, the disk is cleared of any BigMemory data.

*  **"none"** &mdash; Does not offload data to disk. With this option, all of the working data is kept in memory only. This is the default mode.

*  **"distributed"** &mdash; Defers to the `<terracotta>` configuration for persistence settings. For more information, refer to [Terracotta Clustering Configuration Elements](/documentation/4.1/bigmemorymax/configuration/distributed-configuration#95592).

###Synchronous Writes Options
If the `strategy` attribute is set to "localRestartable", then the `synchronousWrites` attribute may be configured. The options for `synchronousWrites` are:

*  **synchronousWrites="false"** &mdash; This option specifies that an eventually consistent record of the data is kept on disk at all times. Writes to disk happen when efficient, and cache operations proceed without waiting for acknowledgement of writing to disk. After a restart, the data is recovered as it was when last synced. This option is faster than `synchronousWrites="true"`, but after a crash, the last 2-3 seconds of written data may be lost.

	If not specified, the default for `synchronousWrites` is "false".

*  **synchronousWrites="true"** &mdash; This option specifies that a fully consistent record of the data is kept on disk at all times. As changes are made to the data set, they are synchronously recorded on disk. The write to disk happens before a return to the caller. After a restart, the data is recovered exactly as it was before shutdown. This option is slower than `synchronousWrites="false"`, but after a crash, it provides full data consistency.

  For transaction caching with `synchronousWrites`, soft locks are used to protect access. If there is a crash in the middle of a transaction, then upon recovery the soft locks are cleared on next access.

**Note**: The `synchronousWrites` attribute is also available in the `<terracotta>` sub-element. If configured in both places, it must have the same value.  


###DiskStore Path
The path to the directory where any required disk files will be created is configured with the `<diskStore>` sub-element of the Ehcache configuration.

  * For "localRestartable", a unique and explicitly specified path is required for each node.

  *  For "localTempSwap", if the DiskStore path is not specified, a default path is used for the disk tier, and the default path will be auto-resolved in the case of a conflict with another CacheManager.

  **Note**: The Fast Restart feature does not use the disk tier in the same way that conventional disk persistence does. Therefore, when configured for "localRestartable", diskStore size measures such as Cache.getDiskStoreSize() or Cache.calculateOnDiskSize() are not applicable and will return zero. On the other hand, when configured for "localTempSwap", these measures will return size values.

## Configuration Examples
This section presents possible disk usage configurations for standalone Ehcache 2.6.

###Options for Crash Resilience

The following configuration provides fast restartability with fully consistent data persistence:

    <ehcache>
      <diskStore path="/path/to/store/data"/>
      <cache>
        <persistence strategy="localRestartable" synchronousWrites="true"/>
      </cache>
    </ehcache>  


The following configuration provides fast restartability with eventually consistent data persistence:  

    <ehcache>
      <diskStore path="/path/to/store/data"/>
      <cache>
        <persistence strategy="localRestartable" synchronousWrites="false"/>
      </cache>
    </ehcache>  

###Clustered Caches
For distributing BigMemory Max across a Terracotta Server Array (TSA), the persistence strategy in the ehcache.xml should be set to "distributed", and the `<terracotta>` sub-element must be present in the configuration.

    <cache>
      maxEntriesInCache="100000">
      <persistence strategy="distributed"/>
      <terracotta clustered="true" consistency="eventual" synchronousWrites="false"/>
    </cache>    

The attribute `maxEntriesInCache` configures the maximum number of entries in a distributed cache. (`maxEntriesInCache` is not required, but if it is not set, the default is unlimited.)

 **Note**: Restartability must be enabled in the TSA in order for clients to be restartable.

###Temporary Disk Storage

The "localTempSwap" persistence strategy create a local disk tier for in-memory data during BigMemory operation. The disk storage is temporary and is cleared after a restart.

    <ehcache>
      <diskStore path="/auto/default/path"/>
      <cache>
        <persistence strategy="localTempSwap"/>
      </cache>
    </ehcache>  

**Note**: With the "localTempSwap" strategy, you can use `maxEntriesLocalDisk` or `maxBytesLocalDisk` at either the Cache or CacheManager level to control the size of the disk tier.

###In-memory Only Cache
When the persistence strategy is "none", all cache stays in memory (with no overflow to disk nor persistence on disk).

    <cache>
      <persistence strategy="none"/>
    </cache>


###Programmatic Configuration Example
The following is an example of how to programmatically configure cache persistence on disk:

    Configuration cacheManagerConfig = new Configuration()
        .diskStore(new DiskStoreConfiguration()
        .path("/path/to/store/data"));
    CacheConfiguration cacheConfig = new CacheConfiguration()
        .name("my-cache")
        .maxBytesLocalHeap(16, MemoryUnit.MEGABYTES)
        .maxBytesLocalOffHeap(256, MemoryUnit.MEGABYTES)
        .persistence(new PersistenceConfiguration().strategy(Strategy.LOCALRESTARTABLE));

    cacheManagerConfig.addCache(cacheConfig);

    CacheManager cacheManager = new CacheManager(cacheManagerConfig);
    Ehcache myCache = cacheManager.getEhcache("my-cache");


##Fast Restart Performance
When configured for fast restartability ("localRestartable" persistence strategy), BigMemory becomes active on restart after all of the in-memory data is loaded. The amount of time until BigMemory is restarted is proportionate to the amount of in-memory data and the speed of the underlying infrastructure. Generally, recovery can occur as fast as the disk speed. With an SSD, for example, if you have a read throughput of 1 GB per second, you will see a similar loading speed during recovery.


##Fast Restart Limitations
The following recommendations should be observed when configuring BigMemory for fast restartability:

* The size of on-heap or off-heap stores should not be changed during a shutdown. If the amount of memory allocated is reduced, elements will be evicted upon restart.

* Restartable caches should not be removed from the CacheManager during a shutdown.

* If a restartable cache is disposed, the reference to the cache is deleted, but the cache contents remain in memory and on disk. After a restart, the cache contents are once again recovered into memory and on disk. The way to safely dispose of an unused restartable cache, so that it does not take any space in disk or memory, is to call clear on the cache and then dispose.
