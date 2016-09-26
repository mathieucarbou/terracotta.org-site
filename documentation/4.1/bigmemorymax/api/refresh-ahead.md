---
---
# Refresh Ahead

{toc|2:3}

## Introduction
This pattern is intended to proactively update cached data to avoid serving stale data. It is also a solution for the "thundering herd" problem of read-through caching.

## Inline Refresh Ahead
Inline refresh allows caches to automatically refresh entries based on a timer. Entries whose age reaches the configured time limit, and are accessed, are reloaded by [CacheLoader methods](#cacheloader).


### Configuring Inline Refresh
Inline refresh ahead is configured per cache using a [cache decorator](http://terracotta.org/documentation/bigmemorymax/api/cache-decorators):

    <cache ...

      <cacheLoaderFactory class="com.company.my.ConcreteCacheLoaderFactory"
          properties="some_property=1, some_other_property=2" />

      <cacheDecorator class="net.sf.ehcache.constructs.refreshahead.RefreshAheadCacheFactory"
          properties="name=myCacheRefresher,
          timeToRefreshSeconds=200,
          batchSize=10,
          numberOfThreads=4,
          maximumBacklogItems=100,
          evictOnLoadMiss=true" />
      </cacheDecorator>
      ...
    </cache>

The cache-decorator class is required for implementing the refresh-ahead mechanism. Note that inline-refresh configuration properties are optional unless marked REQUIRED. The following table describes these properties.

<table>
<tr><th>Property</th><th>Definition</th><th>Default Value</th></tr>
<tr><td>name</td><td>The name used to identify the cache decorator. If left null, the cache decorator is accessed instead of the underlying cache when the cache's name is referenced.</td><td>N/A</td></tr>
<tr><td>timeToRefreshSeconds</td><td>REQUIRED. The number of seconds an entry can exist in the cache before it is refreshed (reloaded) on access. Expired items that have yet to be evicted cannot be refreshed.</td><td>N/A</td></tr>
<tr><td>maximumBacklogItems</td><td>REQUIRED. The maximum number of refresh requests allowed in the refresh work queue. Once this limit is exceeded, items are dropped from the queue to prevent potentially overtaxing resources. If refresh requests are queued faster than they are being cleared, this limit can be exceeded.</td><td>N/A</td></tr>
<tr><td>batchSize</td><td>The maximum number of refresh requests that can be batched for processing by a thread. For more frequent processing of requests&mdash;at a cost to performance&mdash;lower the value.</td><td>100</td></tr>
<tr><td>numberOfThreads</td><td>The number of threads to use for background refresh operations on the decorator. If multiple cache loaders are configured, refresh operations occur in turn.</td><td>1</td></tr>
<tr><td>evictOnLoadMiss</td><td>A boolean to force an immediate eviction on a reload miss (true) or to not evict before normal eviction takes effect (false). If true, entries that do not refresh due to a failure of the reload operation (all cacheloaders fail) are evicted even if they have not expired based on time-to-live or time-to-idle settings.</td><td>false</td></tr>
</table>

### How timeToRefreshSeconds Works With Expiration

The `timeToRefreshSeconds` value is at least how old an entry must be before a get operation triggers an automatic refresh. The refresh is a reload operation that resets an entry's time-to-live (TTL) countdown. A Time-to-idle (TTI) setting is reset upon access and so has no effect if it is greater than the refresh time limit (TTR), unless no access occurs.

For example, for a cache with a TTR of 10 seconds, a TTL of 30 seconds, and a TTI of 0 (infinite idle time), any access to an entry that occurs between 10 and 30 seconds triggers an automatic refresh of that entry. This assumes that the entry is reloaded ahead of the TTL limit so that the TTL timer is reset.

## Scheduled Refresh Ahead

You can configure refresh-ahead operations to occur on a regular schedule using [Quartz scheduled jobs](http://quartz-scheduler.org/documentation). These jobs can asynchronously load new values using configured cache loaders on a schedule defined by a cron expression. This type of refresh operation is useful when all or a large portion of a cache's data must be updated regularly.

Note that at least one [CacheLoader](#cacheloader) must be configured for caches using scheduled refresh.

### Configuring Scheduled Refresh
_Scheduled_ refresh ahead is configured using a [cache extension](http://terracotta.org/documentation/bigmemorymax/api/cache-extensions):

     <cache ...

       <cacheLoaderFactory class="com.company.my.ConcreteCacheLoaderFactory"
         properties="some_property=1, some_other_property=2" />

       <cacheExtensionFactory
         class="net.sf.ehcache.constructs.scheduledrefresh.ScheduledRefreshCacheExtensionFactory"
         properties="batchSize=100;
                     quartzJobCount=2;
                     cronExpression=0 0 12 * * ?"
         propertySeparator=";"/>
    ...
    </cache>

Because a cron expression can contain commas (","), which is the default delimiter for properties, the `propertySeparator` attribute can be used to specify a different delimiter.

The cache-extension class is required for implementing the refresh-ahead mechanism. This cache extension is responsible for instantiating a dedicated Quartz scheduler with its own [RamJobStore](http://quartz-scheduler.org/documentation/quartz-2.x/tutorials/tutorial-lesson-09) for each non-clustered cache configured for scheduled refresh ahead.

Note that scheduled-refresh configuration properties are optional unless marked REQUIRED. The following table describes these properties. Classes are found in `net.sf.ehcache.constructs.scheduledrefresh`.

| Property | Definition | Default Value |
|---------|------------|---------------|
| cronExpression | REQUIRED. This is a string expression, in cron format, dictating when the refresh job should be run. See the [Quartz documentation](http://quartz-scheduler.org/documentation/quartz-1.x/tutorials/crontrigger) for more information. | N/A |
| batchSize | (Integer) This is the number of keys which a single job will refresh at a time as jobs are scheduled. | 100 |
| keyGenerator | If null, `SimpleScheduledRefreshKeyGenerator` is used to enumerate all keys in the cache. Or, to filter the keys to be refreshed, provide a class that implements the `ScheduledRefreshKeyGenerator` interface, overring its `generateKeys()` method. | Null |
| useBulkLoad | If this is true, the node will be put in <a href="http://terracotta.org/documentation/enterprise-ehcache/api-guide#75664">bulkload mode</a> as keys are loaded into the cache. | false |
| quartzJobCount | Controls the number of threads in the Quartz job store created by this instance of ScheduleRefreshCacheExtension. At least 2 threads are needed, 1 for the controlling job and 1 to run refresh batch jobs. There is always an additional thread created with respect to the value set. Therefore the default value of "2" creates the controlling job (always created) and 2 worker threads to process incoming jobs. | 2 |
| scheduledRefreshName | By default, the underlying Quartz job store is named based on a combination of CacheManager and cache names. If two or more ScheduledRefreshExtensions are going to be attached to the same cache, this property provides a necessary way to name separate instances. | Null |
| evictOnLoadMiss | A boolean to force an immediate eviction on a reload miss (true) or to not evict before normal eviction takes effect (false). If true, entries that do not refresh due to a failure of the reload operation (all cacheloaders fail) are evicted even if they have not expired based on time-to-live or time-to-idle settings. | true |
| jobStoreFactory | By default, a RAMJobStore is used by the Quartz instance. To define a custom job store, implement the `ScheduledRefreshJobStorePropertiesFactory` interface. | [See below.](#ScheduledRefreshRAMJobStoreFactory) |
| tcConfigUrl | The URL to a Terracotta server in the form &lt;server-address>:&lt;tsa-port>, same as the Quartz property `org.quartz.jobStore.tcConfigUrl`. Setting this allows the Quartz instances supporting scheduled refresh to run distributed in a Terracotta cluster. | N/A |
| parallelJobCount | Maximum number of simultaneous refresh batch jobs that can run when Quartz instances supporting scheduled refresh run distributed in a Terracotta cluster. Tune this value to improve performance by beginning with a value equal to the number of (identical) distributed scheduled-refresh nodes multiplied by the total number of Quartz nodes. For example, if there are 4 instances and 4 Quartz nodes, set this value to 16 and confirm that resources are not being overly utilized. If resources are still underutilized, increase this value and gauge the impact on resources and performance. | Equals the value of quartzJobCount. |

<a id="ScheduledRefreshRAMJobStoreFactory"></a>
The default jobStoreFactory is `ScheduledRefreshRAMJobStoreFactory`. However, if the property `tcConfigUrl` is specified, the jobStoreFactory used is `ScheduledRefreshTerracottaJobStoreFactory`.

<a id="cacheloader"></a>
## Implementing the CacheLoader
Implementing CacheLoaderFactory through the BigMemory Max API is required to effect reloading of entries for refresh-ahead operations. In configuration, specify the concrete class that extends `net.sf.ehcache.loader.CacheLoaderFactory` and call `createCacheLoader(myCache, properties)` to create the cache's cacheloader. For example, if the configured concrete class is the following:

    <cache name="myCache" ...

      <cacheLoaderFactory class="com.company.my.ConcreteCacheLoaderFactory"
          Properties="some_property=1, some_other_property=2" />

      <!-- Additional cacheLoaderFactory elements can be added.
           These form a chain so that if a CacheLoader returns null,
           the next in line is tried. -->

      ...
    </cache>

then it can be used programmatically:

    CacheLoader cacheLoader = ConcreteCacheLoaderFactory.createCacheLoader(myCache, properties);
    // Custom properties can be passed to the implemented CacheLoader.

`cacheLoader` must implement the `CacheLoader.loadAll()` method to load the refreshed entries.
