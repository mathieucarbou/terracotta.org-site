---
---
# BigMemory Max Configuration Reference {#53258}

BigMemory Max uses the standard Ehcache configuration file to set clustering and consistency behavior, optimize cached data, provide support for JTA and OSGi, and more.

{toc|2:3}



## Dynamically Changing Cache Configuration
While most of the BigMemory Max configuration is not changeable after startup, certain cache configuration parameters can be modified dynamically at runtime. These include the following:

* Expiration settings
    * timeToLive &ndash; The maximum number of seconds an element can exist in the cache regardless of access. The element expires at this limit and will no longer be returned from the cache. The default value is 0, which means no TTL eviction takes place (infinite lifetime).

    * timeToIdle &ndash; The maximum number of seconds an element can exist in the cache without being accessed. The element expires at this limit and will no longer be returned from the cache. The default value is 0, which means no TTI eviction takes place (infinite lifetime).

    Note that the `eternal` attribute, when set to "true", overrides `timeToLive` and `timeToIdle` so that no expiration can take place.

* Local sizing attributes
    * maxEntriesLocalHeap
    * maxBytesLocalHeap
    * maxEntriesLocalDisk
    * maxBytesLocalDisk.

* memory-store eviction policy
* CacheEventListeners can be added and removed dynamically

This example shows how to dynamically modify the cache configuration of a running cache:

    Cache cache = manager.getCache("sampleCache");
    CacheConfiguration config = cache.getCacheConfiguration();
    config.setTimeToIdleSeconds(60);
    config.setTimeToLiveSeconds(120);
    config.setmaxEntriesLocalHeap(10000);
    config.setmaxEntriesLocalDisk(1000000);

Dynamic cache configurations can also be disabled to prevent future changes:

    Cache cache = manager.getCache("sampleCache");
    cache.disableDynamicFeatures();

In `ehcache.xml`, you can disable dynamic configuration by setting the `<ehcache>` element's `dynamicConfig` attribute to "false".


### Dynamic Configuration Changes for Distributed BigMemory Max
Just as for standalone BigMemory, mutating the configuration of distributed BigMemory requires access to the set methods of `cache.getCacheConfiguration()`.

The following table provides information for dynamically changing common configuration options in a Terracotta cluster. The table's Scope column, which specifies where the configuration is in effect, can have one of the following values:

* Client &ndash; The Terracotta client where the CacheManager runs.
* TSA &ndash; The Terracotta Server Array for the cluster.
* BOTH &ndash; Both the client and the TSA.

Note that configuration options whose scope covers "BOTH" are distributed and therefore affect a cache on all clients.

<table>
<tr><th>Configuration Option</th><th>Dynamic</th><th>Scope</th><th>Notes</th></tr>
<tr>
<td>Cache name</td>
<td>NO</td>
<td>TSA</td>
<td></td>
</tr><tr>
<td>Nonstop</td>
<td>NO</td>
<td>Client</td>
<td>Enable High Availability</td>
</tr><tr>
<td>Timeout</td>
<td>YES</td>
<td>Client</td>
<td>For nonstop.</td>
</tr><tr>
<td>Timeout Behavior</td>
<td>YES</td>
<td>Client</td>
<td>For nonstop.</td>
</tr><tr>
<td>Immediate Timeout When Disconnected</td>
<td>YES</td>
<td>Client</td>
<td>For nonstop.</td>
</tr><tr>
<td>Time to Idle</td>
<td>YES</td>
<td>BOTH</td>
<td></td>
</tr><tr>
<td>Time to Live</td>
<td>YES</td>
<td>BOTH</td>
<td></td>
</tr><tr>
<td>Maximum Entries or Bytes in Local Stores</td>
<td>YES</td>
<td>Client</td>
<td>This and certain other sizing attributes may be pooled by the CacheManager, creating limitations on how they can be changed.</td>
</tr><tr>
<td>Maximum Entries in Cache</td>
<td>YES</td>
<td>TSA</td>
<td></td>
</tr><tr>
<td>Persistence Strategy</td>
<td>N/A</td>
<td>N/A</td>
<td></td>
</tr><tr>
<td>Disk Expiry Thread Interval</td>
<td>N/A</td>
<td>N/A</td>
<td></td>
</tr><tr>
<td>Disk Spool Buffer Size</td>
<td>N/A</td>
<td>N/A</td>
<td></td>
</tr><tr>
<td>Maximum Off-heap</td>
<td>N/A</td>
<td>N/A</td>
<td>Maximum off-heap memory allotted to the TSA.</td>
</tr><tr>
<td>Eternal</td>
<td>YES</td>
<td>BOTH</td>
<td></td>
</tr><tr>
<td>Pinning</td>
<td>NO</td>
<td>BOTH</td>
<td>See <a href="/documentation/4.1/bigmemorymax/configuration/data-life">Pinning, Expiration, and Eviction</a>.</td>
</tr><tr>
<td>Clear on Flush</td>
<td>NO</td>
<td>Client</td>
<td></td>
</tr><tr>
<td>Copy on Read</td>
<td>NO</td>
<td>Client</td>
<td></td>
</tr><tr>
<td>Copy on Write</td>
<td>NO</td>
<td>Client</td>
<td></td>
</tr><tr>
<td>Statistics</td>
<td>YES</td>
<td>Client</td>
<td>Cache statistics. Change dynamically with <code>cache.setStatistics(boolean)</code> method.</td>
</tr><tr>
<td>Logging</td>
<td>NO</td>
<td>Client</td>
<td>Ehcache and Terracotta logging is specified in configuration. However, <a href="/documentation/4.1/bigmemorymax/api/cluster-events">cluster events</a> can be set dynamically.</td>
</tr><tr>
<td>Consistency</td>
<td>NO</td>
<td>Client</td>
<td>It is possible to switch to and from <a href="/documentation/4.1/bigmemorymax/api/bulk-loading">bulk mode</a>.</td>
</tr><tr>
<td>Synchronous Writes</td>
<td>NO</td>
<td>Client</td>
<td></td>
</tr>
</table>

To apply non-dynamic L1 changes, remove the existing cache and then add (to the same CacheManager) a new cache with the same name as the removed cache, and which has the new configuration. Restarting the CacheManager with an updated configuration, where all cache names are the same as in the previous configuration, will also apply non-dynamic L1 changes.

## How Server Settings Can Override Client Settings

The tc.property `ehcache.clustered.config.override.mode` allows you to determine if and how the cache settings on the server override the cache settings on the client. You can set this tc.property in the `tc-config.xml` or define it as a system property with the `com.tc.` prefix.

The possible values are:

 * NONE - Default behavior: any discrepancy between the client and server versions of a cache configuration option causes the client to throw an exception.
 * GLOBAL - Allows any cache-wide settings from the server to override those from the client. For example, if a client were to join with a `TTI=10` while the server has a `TTI=15`, then
   * The server TTI value overrides the client TTI value
   * Local settings, such as `maxEntriesLocalHeap`, are not overriden.
 * ALL - Causes the client to accept all values from the server’s configuration. This includes the cache-wide settings of GLOBAL as well as the local settings, such as `maxBytesLocalHeap`.

## Passing Copies Instead of References
By default, a `get()` operation on store data returns a reference to that data, and any changes to that data are immediately reflected in the memory store. In cases where an application requires a *copy* of data rather than a reference to it, you can configure the store to return a copy. This allows you to change a copy of the data without affecting the original data in the memory store.

This is configured using the `copyOnRead` and `copyOnWrite` attributes of the &lt;cache> and &lt;defaultCache elements in your configuration, or programmatically as follows:

    CacheConfiguration config = new CacheConfiguration("copyCache", 1000)
                                       .copyOnRead(true).copyOnWrite(true);
    Cache copyCache = new Cache(config);

The default configuration is "false" for both options.

To copy elements on `put()`-like and/or `get()`-like operations, a copy strategy is used. The default implementation uses serialization to copy elements. You can provide your own implementation of `net.sf.ehcache.store.compound.CopyStrategy` using the &lt;copyStrategy> element:

    <cache name="copyCache"
        maxEntriesLocalHeap="10"
        eternal="false"
        timeToIdleSeconds="5"
        timeToLiveSeconds="10"
        copyOnRead="true"
        copyOnWrite="true">
      <copyStrategy class="com.company.ehcache.MyCopyStrategy"/>
    </cache>

A single instance of your `CopyStrategy` is used per cache. Therefore, in your implementation of `CopyStrategy.copy(T)`, T has to be thread-safe.

A copy strategy can be added programmatically in the following way:

    CacheConfiguration cacheConfiguration = new CacheConfiguration("copyCache", 10);

    CopyStrategyConfiguration copyStrategyConfiguration = new CopyStrategyConfiguration();
    copyStrategyConfiguration.setClass("com.company.ehcache.MyCopyStrategy");

    cacheConfiguration.addCopyStrategy(copyStrategyConfiguration);


## Special System Properties

<a id="net.sf.ehcache.disabled"></a>
### net.sf.ehcache.disabled
Setting this system property to `true` (using `java -Dnet.sf.ehcache.disabled=true` in the Java command line) disables caching in ehcache. If disabled, no elements can be added to a cache (puts are silently discarded).

<a id="net.sf.ehcache.use.classic.lru"></a>
### net.sf.ehcache.use.classic.lru
When LRU is selected as the eviction policy, set this system property to `true` (using `java -Dnet.sf.ehcache.use.classic.lru=true` in the Java command line) to use the older LruMemoryStore implementation. This is provided for ease of migration.

<a id="96087"></a>
## Non-Blocking Disconnected (Nonstop) Cache {#nonstop}

A nonstop cache allows certain cache operations to proceed on clients that have become disconnected from the cluster or if a cache operation cannot complete by the nonstop timeout value. One way clients go into nonstop mode is when they receive a "cluster offline" event. Note that a nonstop cache can go into nonstop mode even if the node is not disconnected, such as when a cache operation is unable to complete within the timeout interval allotted by the nonstop configuration.

### Configuring Nonstop

Nonstop is configured in a `<cache>` block under the `<terracotta>` sub-element. In the following example, myCache has nonstop configuration:

    <cache name="myCache" maxEntriesLocalHeap="10000" eternal="false">
       <persistence strategy="distributed"/>
       <terracotta>
         <nonstop immediateTimeout="false" timeoutMillis="30000">
           <timeoutBehavior type="noop" />
         </nonstop>
       </terracotta>
    </cache>

Nonstop is enabled by default or if `<nonstop>` appears in a cache's `<terracotta>` block.


### Nonstop Timeouts and Behaviors

Nonstop caches can be configured with the following attributes:

* `enabled` &ndash;  Enables ("true" DEFAULT) or disables ("false") the ability of a cache to execute certain actions after a Terracotta client disconnects. This attribute is optional for enabling nonstop.
* `immediateTimeout` &ndash;  Enables ("true") or disables ("false" DEFAULT) an immediate timeout response if the Terracotta client detects a network interruption (the node is disconnected from the cluster). If enabled, the first request made by a client can take up to the time specified by `timeoutMillis` and subsequent requests are timed out immediately.
* `timeoutMillis` &ndash;  Specifies the number of milliseconds an application waits for any cache operation to return before timing out. The default value is 30000 (thirty seconds). The behavior after the timeout occurs is determined by `timeoutBehavior`.

`<nonstop>` has one self-closing sub-element, &lt;timeoutBehavior>. This subelement determines the response after a timeout occurs (`timeoutMillis` expires or an immediate timeout occurs). The response can be set by the &lt;timeoutBehavior> attribute `type`. This attribute can have one of the values listed in the following table:

<table>
<tr>
<th>Value</th>
<th>Behavior</th>
</tr>
<tr>
<td>
<code>exception</code>
</td>
<td>
(DEFAULT) Throw <code>NonStopCacheException</code>. See <a href="#97568">When is NonStopCacheException Thrown?</a> for more information on this exception.
</td>
</tr>
<tr>
<td>
<code>noop</code>
</td>
<td>
Return null for gets. Ignore all other cache operations. Hibernate users may want to use this option to allow their application to continue with an alternative data source.
</td>
</tr>
<tr>
<td>
<code>localReads</code>
</td>
<td>
For caches with Terracotta clustering, allow inconsistent reads of cache data. Ignore all other cache operations. For caches without Terracotta clustering, throw an exception.
</td>
</tr>
</table>


#### Tuning Nonstop Timeouts and Behaviors {#78696}

You can tune the default timeout values and behaviors of nonstop caches to fit your environment.

##### Network Interruptions
For example, in an environment with regular network interruptions, consider disabling `immediateTimeout` and increasing `timeoutMillis` to prevent timeouts for most of the interruptions.

For a cluster that experiences regular but short network interruptions, and in which caches clustered with Terracotta carry read-mostly data or there is tolerance of potentially stale data, you might want to set `timeoutBehavior` to `localReads`.

##### Slow Cache Operations
In an environment where cache operations can be slow to return and data is required to always be in sync, increase `timeoutMillis` to prevent frequent timeouts. Set `timeoutBehavior` to `noop` to force the application to get data from another source or `exception` if the application has stopped.

For example, a `cache.acquireWriteLockOnKey(key)` operation might exceed the nonstop timeout while waiting for a lock. This would trigger nonstop mode only because the lock could not be acquired in time. To avoid this problem,use `cache.tryWriteLockOnKey(key, timeout)` with the method's timeout set to less than the nonstop timeout.

##### Bulk Loading
If a nonstop cache is bulk-loaded using the <a href="../api/bulk-loading">Bulk-Load API</a>, a multiplier is applied to the configured nonstop timeout whenever the method `net.sf.ehcache.Ehcache.setNodeBulkLoadEnabled(boolean)` is used. The default value of the multiplier is 10. You can tune the multiplier using the `bulkOpsTimeoutMultiplyFactor` system property:

~~~
-Dnet.sf.ehcache.nonstop.bulkOpsTimeoutMultiplyFactor=10
~~~


Note that when nonstop is enabled, the cache size displayed in the TMC is subject to the `bulkOpsTimeoutMultiplyFactor`. Increasing this multiplier on the clients can facilitate more accurate size reporting.

This multiplier also affects the methods `net.sf.ehcache.Ehcache.getAll()`, `net.sf.ehcache.Ehcache.removeAll()`, and `net.sf.ehcache.Ehcache.removeAll(boolean)`.


#### When is NonStopCacheException Thrown? {#97568}

NonStopCacheException is usually thrown when it is the configured behavior for a nonstop cache in a client that disconnects from the cluster. In the following example, the exception would be thrown 30 seconds after the disconnection (or the "cluster offline" event is received):

    <nonstop immediateTimeout="false" timeoutMillis="30000">
    <timeoutBehavior type="exception" />
    </nonstop>

However, under certain circumstances the NonStopCache exception can be thrown even if a nonstop cache's timeout behavior is *not* set to throw the exception. This can happen when the cache goes into nonstop mode during an attempt to acquire or release a lock. These lock operations are associated with certain lock APIs and special cache types such as <a href="../api/explicitlocking">Explicit Locking</a>, BlockingCache, SelfPopulatingCache, and UpdatingSelfPopulatingCache.

A NonStopCacheException can also be thrown if the cache must fault in an element to satisfy a `get()` operation. If the Terracotta Server Array cannot respond within the configured nonstop timeout, the exception is thrown.

A related exception, InvalidLockAfterRejoinException, can be thrown during or after client rejoin (see <a href="#71266">Using Rejoin to Automatically Reconnect Terracotta Clients</a>). This exception occurs when an unlock operation takes place on a lock obtained *before* the rejoin attempt completed.

<table>
<caption>TIP: Use try-finally Blocks</caption>
<tr>
<td>
To ensure that locks are released properly, application code using Ehcache lock APIs should encapsulate lock-unlock operations with try-finally blocks:

    myLock.acquireLock();
    try {
      // Do some work.
    } finally {
      myLock.unlock();
    }
</td>
</tr>
</table>


## How Configuration Affects Element Eviction {#30343}

Element eviction is a crucial part of keeping cluster resources operating efficiently. Element eviction and expiration are related, but an expired element is not necessarily evicted immediately and an evicted element is not necessarily an expired element. Cache elements might be evicted due to resource and configuration constraints, while expired elements are evicted from the Terracotta client when a *get* or *put* operation occurs on that element (sometimes called *inline* eviction).

The Terracotta server array contains the full key set (as well as all values), while clients contain a subset of keys and values based on elements they have faulted in from the server array.

Typically, an expired cache element is evicted, or more accurately flushed, from a client tier to a lower tier when a `get()` or `put()` operation occurs on that element. However, a client may also flush expired, and then unexpired elements, whenever a cache's sizing limit for a specific tier is reached or it is under memory pressure. This type of eviction is intended to meet configured and real memory constraints.

To learn about eviction and controlling the size of the cache, see [data life](/documentation/4.1/bigmemorymax/configuration/data-life) and [sizing tiers](/documentation/4.1/bigmemorymax/configuration/cache-size).

Flushing from clients does not mean eviction from the server array. Servers will evict expired elements and elements can become candidates for eviction from the server array when servers run low on allocated BigMemory. Unexpired elements can also be evicted if they meet the following criteria:

* They are in a cache with infinite TTI/TTL (Time To Idle and Time To Live), or no explicit settings for TTI/TTL.
    Enabling a cache's `eternal` flag overrides any finite TTI/TTL values that have been set.
* They are not resident on any Terracotta client.
    These elements can be said to have been "orphaned". Once evicted, they will have to be faulted back in from a system of record if requested by a client.

For more information about Terracotta Server Array data eviction, refer to [Automatic Resource Management](/documentation/4.1/terracotta-server-array/operations#automatic-resource-management).


## Understanding Performance and Cache Consistency {#30971}

Cache consistency modes are configuration settings and API methods that control the behavior of clustered caches with respect to balancing data consistency and application performance. A cache can be in one of the following consistency modes:

* **Strong** &ndash; This mode ensures that data in the cache remains consistent across the cluster at all times. It guarantees that a read gets an updated value only after all write operations to that value are completed, and that each put operation is in a separate transaction. The use of locking and transaction acknowledgments maximizes consistency at **a potentially substantial cost in performance**. This mode is set using the Ehcache configuration file and cannot be changed programmatically (see the attribute "consistency" in <a href="distributed-configuration#70873">`<terracotta>`</a>).

* **Eventual** &ndash;  This mode guarantees that data in the cache will eventually be consistent. Read/write performance is substantially boosted at the cost of potentially having an inconsistent cache for brief periods of time. This mode is set using the Ehcache configuration file and cannot be changed programmatically (see the attribute "consistency" in <a href="distributed-configuration#70873">`<terracotta>`</a>).

 To optimize consistency and performance, consider using eventually consistent caches while selectively using appropriate locking in your application where cluster-wide consistency is critical at all times. Eventual cache operations that are explicitly locked become strongly consistent with respect to each other (but not with respect to non locked operations). For example, a reservation system could be designed with an eventual cache that is only explicitly locked when a customer starts to make a reservation; with this design, eventually consistent data would be available during a customer's search, but during the reservation process, reads and writes would be strongly consistent.

* **Bulk Load** &ndash;  This mode is optimized for bulk-loading data into the cache without the slowness introduced by locks or regular eviction. It is similar to the eventual mode, but has batching, higher write speeds, and weaker consistency guarantees. This mode is set using the bulk-load API only (see <a href="../api/bulk-loading">Bulk-Load API</a>). When turned off, allows the configured consistency mode (either strong or eventual) to take effect again.

Use configuration to set the permanent consistency mode for a cache as required for your application, and the bulk-load mode only during the time when populating (warming) or refreshing the cache.

The following APIs and settings also affect consistency:

* **Explicit Locking** &ndash;  This API provides methods for cluster-wide (application-level) locking on specific elements in a cache. There is guaranteed consistency across the cluster at all times for operations on elements covered by a lock. When used with the strong consistency mode in a cache, *each cache operation* is committed in a single transaction. When used with the eventual consistency mode in a cache, *all cache operations covered by an explicit lock* are committed in a single transaction. While explicit locking of elements provides fine-grained locking, there is still the potential for contention, blocked threads, and increased performance overhead from managing clustered locks. See <a href="../api/explicitlocking">Explicit Locking</a> for more information.
* **UnlockedReadsView** &ndash;  A cache decorator that allows dirty reads of the cache. This decorator can be used only with caches in the strong consistency mode. UnlockedReadsView raises performance for this mode by bypassing the requirement for a read lock. See <a href="../api/unlocked-reads-view">Unlocked Reads for Consistent Caches (UnlockedReadsView)</a> for more information.
* **Bulk-loading methods** &ndash; Bulk-loading Cache methods putAll(), getAll(), and removeAll() provide high-performance and eventual consistency. These can also be used with strong consistency. If you can use them, it's unnecessary to use bulk-load mode.
* **Atomic methods** &ndash;  To guarantee write consistency at all times and avoid potential race conditions for put operations, use atomic methods. The following Compare and Swap (CAS) operations are available:

  *   `cache.replace(Element old, Element new)`  &mdash; Conditionally replaces the specified key's old value with the new value, if the currently existing value matches the old value.
  *   `cache.replace(Element)` &mdash; Maps the specified key to the specified value, if the key is currently mapped to some value.
  *   `cache.putIfAbsent(Element)` &mdash; Puts the specified key/value pair into the cache only if the key has no currently assigned value. Unlike `put`, which can replace an existing key/value pair, `putIfAbsent` creates the key/value pair only if it is not present in the cache.
  *   `cache.removeElement(Element)` &mdash; Conditionally removes the specified key, if it is mapped to the specified value.

  Normally, these methods are used with strong consistency. Unless the property `org.terracotta.clusteredStore.eventual.cas.enabled` is set to "true", these methods throw an UnsupportedOperationException if used with eventual consistency since a race condition cannot be prevented. Other ways to guarantee the return value in eventual consistency are to use the [cache decorator](/documentation/4.1/bigmemorymax/api/cache-decorators) StronglyConsistentCacheAccessor, or to use locks (see <a href="../api/explicitlocking">Explicit Locking</a>). The StronglyConsistentCacheAccessor will use locks with its special substituted versions of the atomic methods. Note that using locks may impact performance.


## Cache Events in a Terracotta Cluster {#82378}

Cache events are fired for certain cache operations:

* **Evictions** &ndash;  An eviction on a client generates an eviction event on that client. An eviction on a Terracotta server fires an event on a random client.
* **Puts** &ndash;  A `put()` on a client generates a put event on that client.
* **Updates** &ndash;  If a cache uses fast restart, then an update on a client generates a put event on that client.
* **Orphan eviction** &ndash;  An orphan is an element that exists only on the Terracotta Server Array. If an orphan is evicted, an eviction event is fired on a random client.

See <a href="/documentation/4.1/bigmemorymax/configuration/distributed-configuration#20075">Cache Events Configuration</a> for more information on configuring the scope of cache events.


### Handling Cache Update Events

Caches generate put events whenever elements are put or updated. If it is important for your application to distinguish between puts and updates, check for the existence of the element during `put()` operations:

    if (cache.containsKey(key)) {
      cache.put(element);
      // Action in the event handler on replace.
    } else {
      cache.put(element);
      // Action in the event handler on new puts.
    }

To protect against races, wrap the if block with explicit locks (see <a href="../api/explicitlocking">Explicit Locking</a>). You can also use the atomic cache methods putIfAbsent() or to check for the existence of an element:

    // Returns null if successful or returns the existing (old) element.
    if((olde = cache.putIfAbsent(element)) == null) {

      // Action in the event handler on new puts.

    } else {
      cache.replace(old, newElement); // Returns true if successful.
      // Action in the event handler on replace.
    }

If your code cannot use these approaches (or a similar workaround), you can force update events for cache updates by setting the Terracotta property `ehcache.clusteredStore.checkContainsKeyOnPut` at the top of the Terracotta configuration file (`tc-config.xml` by default) before starting the Terracotta Server Array:

    <tc-properties>
     <property name="ehcache.clusteredStore.checkContainsKeyOnPut" value="true" />
    </tc-properties>

**Enabling this property can substantially degrade performance.**


## Configuring Caches for High Availability

Enterprise Ehcache caches provide the following High Availability (HA) settings:

* **Non-blocking cache** &ndash;  Also called nonstop cache. When enabled, this attribute gives the cache the ability to take a configurable action after the Terracotta client receives a cluster-offline event. See <a href="#96087">Non-Blocking Disconnected (Nonstop) Cache</a> for more information.
* **Rejoin** &ndash;  The rejoin attribute allows a Terracotta client to reconnect to the cluster after it receives a cluster-online event. See <a href="#71266">Using Rejoin to Automatically Reconnect Terracotta Clients</a> for more information.

To learn about configuring HA in a Terracotta cluster, see <a href="/documentation/4.1/terracotta-server-array/high-availability">Configuring Terracotta Clusters For High Availability</a>.


### Using Rejoin to Automatically Reconnect Terracotta Clients {#71266}

A Terracotta client might disconnect and be timed out (ejected) from the cluster. Typically, this occurs because of network communication interruptions lasting longer than the configured HA settings for the cluster. Other causes include long GC pauses and slowdowns introduced by other processes running on the client hardware.

You can configure clients to automatically rejoin a cluster after they are ejected. If the ejected client continues to run under nonstop cache settings, and then senses that it has reconnected to the cluster (receives a clusterOnline event), it can begin the rejoin process.

Note the following about using the rejoin feature:

* Rejoin is for CacheManagers with only nonstop caches. If one or more of a CacheManager's caches is not set to be nonstop, and rejoin is enabled, an exception is thrown at initialization. An exception is also thrown in this case if a cache is created programmatically without nonstop.
* Clients rejoin as new members and will wipe all cached data to ensure that no pauses or inconsistencies are introduced into the cluster.
* Any nonstop-related operations that begin (and do not complete) before the rejoin operation completes may be unsuccessful and may generate a NonStopCacheException.
* If a client with rejoin enabled is running in a JVM with Terracotta clients that do not have rejoin, then only that client will rejoin after a disconnection. The remaining clients cannot rejoin and may cause the application to behave unpredictably.
* Once a client rejoins, the clusterRejoined event is fired on that client only.


#### Configuring Rejoin

The rejoin feature is disabled by default. To enable the rejoin feature in an Enterprise Ehcache client, follow these steps:


1.  Ensure that all of the caches in the Ehcache configuration file where rejoin is enabled have nonstop enabled.
2.  Ensure that your application does not create caches on the client without nonstop enabled.
3.  Enable the rejoin attribute in the client's `<terracottaConfig>` element:

        <terracottaConfig url="myHost:9510" rejoin="true" />

For more options on configuring `<terracottaConfig>`, see the [configuration reference](/documentation/4.1/bigmemorymax/configuration/distributed-configuration#35085).


## Working With Transactional Caches {#93038}

Transactional caches add a level of safety to cached data and ensure that the cached data and external data stores are in sync. Distributed caches can support Java Transaction API (JTA) transactions as an XA resource. This is useful in JTA applications requiring caching, or where cached data is critical and must be persisted and remain consistent with System of Record data.

However, transactional caches are slower than non-transactional caches due to the overhead from having to write transactionally. Transactional caches also have the following restrictions:

* Data can be accessed only transactionally, even for read-only purposes.
    You must encapsulate data access with `begin()` and `commit()` statements. This may not be necessary under certain circumstances (see, for example, the discussion on Spring in [Transactions](/documentation/4.1/bigmemorymax/api/jta#transactional-caches-with-spring)).
* `copyOnRead` and `copyOnWrite` must be enabled.
    These `<cache>` attributes are "false" by default and must set to "true".
* Caches must be strongly consistent.
    A transactional cache's `consistency` attribute must be set to "strong".
* Nonstop caches cannot be made transactional except in strict mode (xa_strict).
    Transactional caches in other modes must *not* contain the `<nonstop>` subelement.
* Decorating a transactional cache with UnlockedReadsView can return inconsistent results for data obtained through UnlockedReadsView.
    Puts, and gets not through UnlockedReadsView, are not affected.
* Objects stored in a transactional cache must override `equals()` and `hashCode()`.
    If overriding `equals()` and `hashCode()` is not possible, see <a href="#48013">Implementing an Element Comparator</a>.
* Caches can be dynamically changed to  bulk-load mode, but any attempt to perform a transaction when this is the case will throw a `CacheException`.

For more information about transactional caches, see [this API page](/documentation/4.1/bigmemorymax/api/jta).

You can choose one of three different modes for transactional caches:

* Strict XA &ndash;  Has full support for XA transactions. May not be compatible with transaction managers that do not fully support JTA.
* XA &ndash;  Has support for the most common JTA components, so likely to be compatible with most transaction managers. But unlike strict XA, may fall out of sync with a database after a failure (has no recovery). Integrity of cache data, however, is preserved.
* Local &ndash;  Local transactions written to a local store and likely to be faster than the other transaction modes. This mode does not require a transaction manager and does not synchronize with remote data sources. Integrity of cache data is preserved in case of failure.

<table>
<caption>NOTE: Deadlocks</caption>
<tr>
<td>
Both the XA and local mode write to the underlying store synchronously and using pessimistic locking. Under certain circumstances, this can result in a deadlock, which generates a DeadLockException after a transaction times out and a commit fails. Your application should catch DeadLockException (or TransactionException) and call <code>rollback()</code>.

Deadlocks can have a severe impact on performance. A high number of deadlocks indicates a need to refactor application code to prevent races between concurrent threads attempting to update the same data.
</td>
</tr>
</table>

These modes are explained in the following sections.


### Strict XA (Support for All JTA Components)

Note that Ehcache as an XA resource:

* Has an isolation level of ReadCommitted.
* Updates the underlying store asynchronously, potentially creating update conflicts.
    With this optimistic locking approach, Ehcache might force the transaction manager to roll back the entire transaction if a `commit()` generates a RollbackException (indicating a conflict).
* Can work alongside other resources such as JDBC or JMS resources.
* Guarantees that its data is always synchronized with other XA resources.
* Can be configured on a per-cache basis (transactional and non-transactional caches can exist in the same configuration).
* Automatically performs enlistment.
* Can be used standalone or integrated with frameworks such as [Hibernate](#45557).
* Is tested with the most common transaction managers by Atomikos, Bitronix, JBoss, WebLogic, and others.


#### Configuration

To configure a cache as an XA resource able to participate in JTA transactions, the following `<cache>` attributes must be set as shown:

* transactionalMode="xa_strict"
* copyOnRead="true"
* copyOnWrite="true"

In addition, the `<cache>` subelement `<terracotta>` must not have clustering disabled.

For example, the following cache is configured for JTA transactions with strict XA:

    <cache name="com.my.package.Foo"
         maxEntriesLocalHeap="500"
         eternal="false"
         copyOnRead="true"
         copyOnWrite="true"
         consistency="strong"
         transactionalMode="xa_strict">
       <persistence strategy="distributed"/>
       <terracotta />
    </cache>

Any other XA resource that could be involved in the transaction, such as a database, must also be configured to be XA compliant.


#### Usage

Your application can directly use a transactional cache in transactions. This usage must occur after the transaction manager has been set to start a new transaction and before it has ended the transaction.

For example:

    ...
    myTransactionMan.begin();
    Cache fooCache = cacheManager.getCache("Foo");
    fooCache.put("1", "Bar");
    myTransactionMan.commit();
    ...

If more than one transaction writes to a cache, it is possible for an XA transaction to fail. See <a href="#67713">Avoiding XA Commit Failures With Atomic Methods</a> for more information.

#### Setting Up Transactional Caches in Hibernate {#45557}

If your application is using JTA, you can set up transactional caches in a second-level cache with Ehcache for Hibernate. To do so, ensure the following:


##### Ehcache

* You are using Ehcache 2.1.0 or higher.
* The attribute `transactionalMode` is set to &quot;xa&quot; or "xa-strict".
* The cache is clustered (the &lt;cache&gt; element has the subelement &lt;terracotta clustered=&quot;true&quot;&gt;).
For example, the following cache is configured to be transactional:

    <cache name="com.my.package.Foo"
         ...
         transactionalMode="xa">
       <terracotta />
    </cache>

* The cache UpdateTimestampsCache is not configured to be transactional.
Hibernate updates to `org.hibernate.cache.UpdateTimestampsCache` prevent it from being able to participate in XA transactions.


##### Hibernate

* You are using Hibernate 3.3.
* The factory class used for the second-level cache is `net.sf.ehcache.hibernate.EhCacheRegionFactory`.
* Query cache is turned off.
* The value of `current_session_context_class` is `jta`.
* The value of `transaction.manager_lookup_class` is the name of a TransactionManagerLookup class (see your Transaction Manager).
* The value of `transaction.factory_class` is the name of a TransactionFactory class to use with the Hibernate Transaction API.
* The cache concurrency strategy is set to TRANSACTIONAL.
For example, to set the cache concurrency strategy for `com.my.package.Foo` in `hibernate.cfg.xml`:

    <class-cache class="com.my.package.Foo" usage="transactional"/>

Or in a Hibernate mapping file (hbm file):

    <cache usage="transactional"/>

Or using annotations:

    @Cache(usage=CacheConcurrencyStrategy.TRANSACTIONAL)
    public class Foo {...}

**WARNING: Use the TRANSACTIONAL concurrency strategy with transactional caches only. Using with other types of caches will cause errors.**

### XA (Basic JTA Support)

Transactional caches set to "xa" provide support for basic JTA operations. Configuring and using XA does not differ from using local transactions (see <a href="#20697">Local Transactions</a>), except that "xa" mode requires a transaction manager and allows the cache to participate in JTA transactions.

<table>
<caption>NOTE: Atomikos Transaction Manager</caption>
<tr>
<td>
When using XA with an Atomikos transaction Manager, be sure to set <code>com.atomikos.icatch.threaded_2pc=false</code> in the Atomikos configuration. This helps prevent unintended rollbacks due to a bug in the way Atomikos behaves under certain conditions.
</td>
</tr>
</table>

For example, the following cache is configured for JTA transactions with XA:

    <cache name="com.my.package.Foo"
         maxEntriesLocalHeap="500"
         eternal="false"
         copyOnRead="true"
         copyOnWrite="true"
         consistency="strong"
         transactionalMode="xa">
       <persistence strategy="distributed"/>
       <terracotta />
    </cache>

Any other XA resource that could be involved in the transaction, such as a database, must also be configured to be XA compliant.


### Local Transactions {#20697}

Local transactional caches (with the `transactionalMode` attribute set to "local") write to a local store using an API that is part of the Ehcache core API. Local transactions have the following characteristics:

* Recovery occurs at the time an element is accessed.
* Updates are written to the underlying store immediately.
* Get operations on the underlying store may block during commit operations.

To use local transactions, instantiate a TransactionController instance instead of a transaction manager instance:

    TransactionController txCtrl = cacheManager.getTransactionController();
    ...
    txCtrl.begin();
    Cache fooCache = cacheManager.getCache("Foo");
    fooCache.put("1", "Bar");
    txCtrl.commit();
    ...

You can use `rollback()` to roll back the transaction bound to the current thread.

<table>
<caption>TIP: Finding the Status of a Transaction on the Current Thread</caption>
<tr>
<td>
You can find out if a transaction is in process on the current thread by calling

`TransactionController.getCurrentTransactionContext()` and checking its return value. If the value isn't null, a transaction has started on the current thread.
</td>
</tr>
</table>


#### Commit Failures and Timeouts

Commit operations can fail if the transaction times out. If the default timeout requires tuning, you can get and set its current value:

    int currentDefaultTransactionTimeout = txCtrl.getDefaultTransactionTimeout();
    ...
    txCtrl.setDefaultTransactionTimeout(30); // in seconds -- must be greater than zero.

You can also bypass the commit timeout using the following version of `commit()`:

    txCtrl.commit(true); // "true" forces the commit to ignore the timeout.


### Avoiding XA Commit Failures With Atomic Methods {#67713}

If more than one transaction writes to a cache, it is possible for an XA transaction to fail. In the following example, if a second transaction writes to the same key ("1") and completes its commit first, the commit in the example may fail:

    ...
    myTransactionMan.begin();
    Cache fooCache = cacheManager.getCache("Foo");
    fooCache.put("1", "Bar");
    myTransactionMan.commit();
    ...

One approach to prevent this type of commit failure is to use one of the atomic put methods, such as `Cache.replace()`:

    myTransactionMan.begin();
    int val = cache.get(key).getValue();  // "cache" is configured to be transactional.
    Element olde = new Element (key, val);

    // True only if the element was successfully replaced.
    if (cache.replace(olde, new Element(key, val + 1)) {
     myTransactionMan.commit();
    }
    else { myTransactionMan.rollback(); }

Another useful atomic put method is `Cache.putIfAbsent(Element element)`, which returns null on success (no previous element exists with the new element's key) or returns the existing element (the put is not executed). Atomic methods cannot be used with null elements, or elements with null keys.


### Implementing an Element Comparator {#48013}

For all transactional caches, the atomic methods `Cache.removeElement(Element element)` and `Cache.replace(Element old, Element element)` must compare elements for the atomic operation to complete. This requires all objects stored in the cache to override `equals()` and `hashCode()`.

If overriding these methods is not desirable for your application, a default comparator is used (`net.sf.echache.store.DefaultElementValueComparator`). You can also implement a custom comparator and specify it in the cache configuration with &lt;elementValueComparator>:

    <cache name="com.my.package.Foo"
         maxEntriesLocalHeap="500"
         eternal="false"
         copyOnRead="true"
         copyOnWrite="true"
         consistency="strong"
         transactionalMode="xa">
       <elementValueComparator class="com.company.xyz.MyElementComparator" />
       <persistence strategy="distributed"/>
       <terracotta />
    </cache>

Custom comparators must implement `net.sf.ehcache.store.ElementValueComparator`.

A comparator can also be specified programmatically.


## Working With OSGi

To allow Enterprise Ehcache to behave as an OSGi component, the following attributes should be set as shown:

    <cache ... copyOnRead="true" ... >
    ...
      <terracotta ... clustered="true" ... />
    ...
    </cache>


Your OSGi bundle will require the following JAR files (showing versions from a BigMemory Max 4.0.0):

* `ehcache-ee-2.7.0.jar`
* `terracotta-toolkit-runtime-ee-4.0.0.jar`
* `slf4j-api-1.6.6.jar`
* `slf4j-nop-1.6.1.jar`

    Or use another appropriate logger binding.

Use the following directory structure:

     -- net.sf.ehcache
              |
              | - ehcache.xml
              |- ehcache-ee-2.7.0.jar
	      |
	      |- terracotta-toolkit-runtime-ee-4.0.0.jar
              |
              | - slf4j-api-1.6.6.jar
              |
              | - slf4j-nop-1.6.6.jar
              |
              | - META-INF/
                  | - MANIFEST.MF


The following is an example manifest file:

    Manifest-Version: 1.0
     Export-Package: net.sf.ehcache;version="2.7.0"
     Bundle-Vendor: Terracotta
     Bundle-ClassPath: .,ehcache-ee-2.7.0.jar,terracotta-toolkit-runtime-ee-4.0.0.jar
         ,slf4j-api-1.6.6.jar,slf4j-nop-1.6.6.jar
     Bundle-Version: 2.7.0
     Bundle-Name: EHCache bundle
     Created-By: 1.6.0_15 (Apple Inc.)
     Bundle-ManifestVersion: 2
     Import-Package: org.osgi.framework;version="1.3.0"
     Bundle-SymbolicName: net.sf.ehcache
     Bundle-RequiredExecutionEnvironment: J2SE-1.5

Use versions appropriate to your setup.

To create the bundle, execute the following command in the `net.sf.ehcache` directory:

    jar cvfm net.sf.ehcache.jar MANIFEST.MF *
