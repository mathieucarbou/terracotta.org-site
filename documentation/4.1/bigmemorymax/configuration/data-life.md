---
---
# Pinning, Expiration, and Eviction

{toc|2:3}

## Introduction
This page covers managing the life of the data in each of BigMemory's data storage tiers, including the pinning features of Automatic Resource Control (ARC).

The following are options for data life within the BigMemory tiers:

* Flush &ndash; To move an entry to a lower tier. Flushing is used to free up resources while still keeping data in BigMemory.
* Fault &ndash; To copy an entry from a lower tier to a higher tier. Faulting occurs when data is required at a higher tier but is not resident there. The entry is not deleted from the lower tiers after being faulted.
* Eviction &ndash; To remove an entry from BigMemory. The entry is deleted; it can only be reloaded from an outside source. Entries are evicted to free up resources.
* Expiration &ndash; A status based on Time To Live and Time To Idle settings. To maintain performance, expired entries may not be immediately flushed or evicted.
* Pinning &ndash; To keep data in memory indefinitely.  

![BigMemory Data Tiers and Data Life](/images/documentation/data-life-elements3.png)

## Setting Expiration {#24283}
BigMemory data entries expire based on parameters with configurable values. When eviction occurs, expired elements are the first to be removed. Having an effective expiration configuration is critical to optimizing the use of resources such as heap and maintaining overall performance.

To add expiration, edit the values of the following `<cache>` attributes, and tune these values based on results of performance tests:

* `timeToIdleSeconds` &ndash; The maximum number of seconds an element can exist in the BigMemory data store without being accessed. The element expires at this limit and will no longer be returned from BigMemory. The default value is 0, which means no TTI eviction takes place (infinite lifetime).
* `timeToLiveSeconds` &ndash; The maximum number of seconds an element can exist in the the BigMemory data store regardless of use. The element expires at this limit and will no longer be returned from BigMemory. The default value is 0, which means no TTL eviction takes place (infinite lifetime).
* `maxEntriesInCache` &ndash; The maximum sum total number of elements (cache entries) allowed for a distributed cache in all Terracotta clients. If this target is exceeded, eviction occurs to bring the count within the allowed  target. The default value is 0, which means that the cache will not undergo capacity eviction (but periodic and resource evictions are still allowed). Note that `maxEntriesInCache` reflects storage allocated on the TSA.
* `eternal` &ndash;  If the cache's `eternal` flag is set, it overrides any finite TTI/TTL values that have been set. Individual cache elements may also be set to eternal. Eternal elements and caches do not expire, however they may be evicted.

See [How Configuration Affects Element Eviction](#30343) for more information on how configuration can impact eviction.


## Pinning Data

Without pinning, expired cache entries can be flushed and eventually evicted, and even most non-expired elements can also be flushed and evicted as well, if resource limitations are reached. Pinning gives per-cache control over whether data can be evicted from BigMemory.

Data that should remain in memory can be pinned. You cannot pin individual entries, only an entire cache. There are two types of pinning, depending upon whether the pinning configuration should take precedence over resource constraints or the other way around. See the next sections for details.

### Configuring Pinning

Entire caches can be pinned using the `pinning` element in the Ehcache configuration. This element has a required attribute (`store`) to specify how the pinning will be accomplished.

The `store` attribute can have either of the following values:

* inCache &ndash; Data is pinned in the cache, in any tier in which cache data is stored. The tier is chosen based on performance-enhancing efficiency algorithms. Unexpired entries can never be evicted.
* localMemory &ndash; Data is pinned to the memory store or the off-heap store. Entries are evicted only in the event that the store's configured size is exceeded. Updated copies of invalidated elements are faulted in from the server, a behavior you can override by setting the [Terracotta property](/documentation/4.1/terracotta-server-array/config-reference#overriding-tcproperties) `l1.servermapmanager.faultInvalidatedPinnedEntries` to false.

For example, the following cache is configured to pin its entries:

    <cache name="Cache1" ... >
         <pinning store="inCache" />
    </cache>

The following cache is configured to pin its entries to heap or off-heap only:

    <cache name="Cache2" ... >
         <pinning store="localMemory" />
    </cache>

### Pinning and Cache Sizing
The interaction of the pinning configuration with the cache sizing configuration depends upon which pinning option is used.

For `inCache` pinning, the pinning setting takes priority over the configured cache size. Elements resident in a cache with this pinning option cannot be evicted if they haven't expired. This type of pinned cache is not eligible for eviction at all, and `maxEntriesInCache` should not be configured for this cache.

<table>
<caption>Use caution with `inCache` pinning</caption>
<tr><td>
Potentially, pinned caches could grow to an unlimited size. Caches should never be pinned unless they are designed to hold a limited amount of data (such as reference data) or their usage and expiration characteristics are understood well enough to conclude that they cannot cause errors.
</td></tr>
</table>

For `localMemory` pinning, the configured cache size takes priority over the pinning setting. `localMemory` pinning should be used for optimization, to keep data in heap or off-heap memory, unless or until the tier becomes too full. If the number of entries surpasses the configured size, entries will be evicted. For example, in the following cache the `maxEntriesOnHeap` and `maxBytesLocalOffHeap` settings override the pinning configuration:

    <cache name="myCache"
         maxEntriesOnHeap="10000"
         maxBytesLocalOffHeap="8g"
         ... >
        <pinning store="localMemory" />
    </cache>


### Scope of Pinning
Pinning as a setting exists in the local Ehcache client (L1) memory. It is never distributed in the cluster. In case the pinned cache is [bulk-loaded](/documentation/4.1/bigmemorymax/apis/bulk-loading), the pinned data is removed from the L1 and will be faulted back in from the TSA.

Pinning achieved programmatically will not be persisted &mdash; after a restart the pinned entries are no longer pinned. Pinning is also lost when an L1 [rejoins](/documentation/4.1/bigmemorymax/configuration/reference-guide#71266) a cluster. Cache pinning in configuration is reinstated with the configuration file.


## Explicitly Removing Data
To unpin all of a cache's pinned entries, clear the cache. Specific entries can be removed from a cache using `Cache.remove()`. To empty the cache, `Cache.removeAll()`. If the cache itself is removed (`Cache.dispose()` or `CacheManager.removeCache()`), then any data still remaining in the cache is also removed locally. However, that remaining data is *not* removed from the TSA or disk (if localRestartable).

## How Configuration Affects Element Flushing and Eviction {#30343}
The following example shows a cache with certain expiration settings:

    <cache name="myCache"
          eternal="false" timeToIdleSeconds="3600"
          timeToLiveSeconds="0" memoryStoreEvictionPolicy="LFU">
    </cache>

Note the following about the myCache configuration:

* If a client accesses an entry in myCache that has been idle for more than an hour (`timeToIdleSeconds`), that element is evicted.
* If an entry expires but is not accessed, and no resource constraints force eviction, then the expired entry remains in place until a periodic evictor removes it.
* Entries in myCache can live forever if accessed at least once per 60 minutes (`timeToLiveSeconds`). However, unexpired entries may still be flushed based on  other limitations (see [Sizing BigMemory Tiers](/documentation/4.1/bigmemorymax/configuration/cache-size)).


## Data Freshness and Expiration

Databases and other Systems of Record (SOR) that were not built to accommodate caching outside of the database do not normally come with any default mechanism for notifying external processes when data has been updated or modified.

When using BigMemory as a caching system, the following strategies can help to keep the data in the cache in sync:

*  **Data Expiration**: Use the eviction algorithms included with Ehcache, along with the `timeToIdleSeconds` and `timetoLiveSeconds` settings, to enforce a maximum time for elements to live in the cache (forcing a re-load from the database or SOR).

*  **Message Bus**: Use an application to make all updates to the database. When updates are made, post a message onto a message queue with a key to the item that was updated. All application instances can subscribe to the message bus and receive messages about data that is updated, and can synchronize their local copy of the data accordingly (for example by invalidating the cache entry for updated data)

*  **Triggers**: Using a database trigger can accomplish a similar task as the message bus approach. Use the database trigger to execute code that can publish a message to a message bus. The advantage to this approach is that updates to the database do not have to be made only through a special application. The downside is that not all database triggers support full execution environments and it is often unadvisable to execute heavy-weight processing such as publishing messages on a queue during a database trigger.

The Data Expiration method is the simplest and most straightforward. It gives you the most control over the data synchronization, and doesn't require cooperation from any external systems. You simply set a data expiration policy and let Ehcache expire data from the cache, thus allowing fresh reads to re-populate and re-synchronize the cache.

If you choose the Data Expiration method, you can read more about the cache configuration settings at [cache eviction algorithms](/documentation/4.1/bigmemorymax/api/cache-eviction-algorithms), and review the [timeToIdle and timeToLive configuration settings](/documentation/4.1/bigmemorymax/configuration/data-life#24283). The most important consideration when using the expiration method is balancing data freshness with database load. The shorter you make the expiration settings - meaning the more "fresh" you try to make the data - the more load you will incur on the database.

Try out some numbers and see what kind of load your application generates. Even modestly short values such as 5 or 10 minutes will afford significant load reductions.
