---
---
# Bulk Loading {#Bulk-Loading}

{toc|2:3}

## Introduction
BigMemory Max has a bulk-loading mode that dramatically speeds up bulk loading into caches using the Terracotta Server Array (TSA). Bulk loading is designed to be used for:

*   cache warming - where caches need to be filled before bringing an application online
*   periodic batch loading - for example, an overnight batch process that uploads data

The bulk-load API optimizes bulk-loading of data by removing the requirement for locks and by adding transaction batching. The bulk-load API also allows applications to discover whether a cache is in bulk-load mode and to block based on that mode.

## API
With bulk loading, the API for putting data into BigMemory Max stays the same. Just use `cache.put(...)`
`cache.load(...)` or `cache.loadAll(...)`.
What changes is that there is a special mode that suspends the normal distributed-cache consistency guarantees
and provides optimised flushing to the TSA (the L2 cache).

<table>
<caption>
NOTE: The Bulk-Load API and the Configured Consistency Mode</caption>
<tr><td>
  The initial consistency mode of a cache is set by configuration and cannot be changed programmatically (see the attribute "consistency" in <a href="/documentation/4.1/bigmemorymax/configuration/distributed-configuration#70873"><code>&lt;terracotta></code></a>. The bulk-load API should be used for temporarily suspending the configured consistency mode to allow for bulk-load operations.
</td></tr>
</table>

The following are the bulk-load API methods that are available in `org.terracotta.modules.ehcache.Cache`.

* `public boolean isClusterBulkLoadEnabled()`

    Returns true if a cache is in bulk-load mode (is not consistent) throughout the cluster. Returns false if the cache is not in bulk-load mode (is consistent) anywhere in the cluster.

* `public boolean isNodeBulkLoadEnabled()`

    Returns true if a cache is in bulk-load mode (is not consistent) on the current node. Returns false if the cache is not in bulk-load mode (is consistent) on the current node.

* `public void setNodeBulkLoadEnabled(boolean)`

    Sets a cache's consistency mode to the configured consistency mode (false) or to bulk load (true) on the local node. There is no operation if the cache is already in the mode specified by `setNodeBulkLoadEnabled()`. When using this method on a [nonstop cache](/documentation/4.1/configuration/non-stop-cache), a multiple of the nonstop cache's timeout value applies. The bulk-load operation must complete within that timeout multiple to prevent the configured nonstop behavior from taking effect. For more information on tuning nonstop timeouts, see [Tuning Nonstop Timeouts and Behaviors](/documentation/4.1/configuration/non-stop-cache#78696).

* `public void waitUntilBulkLoadComplete()`

    Waits until a cache is consistent before returning. Changes are automatically batched and the cache is updated throughout the cluster. Returns immediately if a cache is consistent throughout the cluster.

Notes on bulk-load mode:

* Consistency cannot be guaranteed because `isClusterBulkLoadEnabled()` can return false in one node just before another node calls `setNodeBulkLoadEnabled(true)` on the same cache. Understanding exactly how your application uses the bulk-load API is crucial to effectively managing the integrity of cached data.
* If a cache is not consistent, any ObjectNotFound exceptions that may occur are logged.
* `get()` methods that fail with ObjectNotFound return null.
* Eviction is independent of consistency mode. Any configured or manually executed eviction proceeds unaffected by a cache's consistency mode.

##Bulk-Load API Example Code
The following example code shows how a clustered application with BigMemory Max can use the bulk-load API to optimize a bulk-load operation:

<pre><code>import net.sf.ehcache.Cache;
public class MyBulkLoader {
CacheManager cacheManager = new CacheManager();  // Assumes local ehcache.xml.
  Cache cache = cacheManager.getEhcache(\"myCache\"); // myCache defined in ehcache.xml.
  cache.setNodeBulkLoadEnabled(true); // myCache is now in bulk mode.
// Load data into myCache.

// Done, now set myCache back to its configured consistency mode.
cache.setNodeBulkLoadEnabled(false);
}
</code></pre>

<table>
<caption>NOTE: Potentional Error With Non-Singleton CacheManager</caption>
<tr><td>Ehcache does not allow multiple CacheManagers with the same name to exist in the same JVM. <code>CacheManager()</code> constructors creating non-singleton CacheManagers can violate this rule, causing an error. If your code may create multiple CacheManagers of the same name in the same JVM, avoid this error by using the <a href="http://www.ehcache.org/apidocs/2.8.5/index.html"><code>static CacheManager.create() methods"()</code></a>, which always return the named (or default unnamed) CacheManager if it already exists in that JVM. If the named (or default unnamed) CacheManager does not exist, the <code>CacheManager.create()</code> methods create it.
</td></tr>
</table>

On another node, application code that intends to touch myCache can run or wait, based on whether myCache is consistent or not:

<pre><code>...
if (!cache.isClusterBulkLoadEnabled()) {
// Do some work.
}
else {
 cache.waitUntilBulkLoadComplete()
 // Do the work when waitUntilBulkLoadComplete() returns.
}
...
</code></pre>

Waiting may not be necessary if the code can handle potentially stale data:

<pre><code>...
if (!cache.isClusterBulkLoadEnabled()) {
// Do some work.
}
else {
// Do some work knowing that data in myCache may be stale.
}
...
</code></pre>


## Performance Improvement
The performance improvement is an order of magnitude faster.
[ehcacheperf](http://svn.terracotta.org/svn/forge/projects/ehcacheperf/trunk/) (Spring Pet Clinic) now has a bulk load test which shows the performance improvement for using
a Terracotta cluster. Consider also that multi-threading is likely to improve performance.

## FAQ {#faq}

#### How does bulk-loading affect pinned caches?
If a cache has been [pinned](/documentation/4.1/bigmemorymax/configuration/data-life), switching the cache into bulk-load mode removes the cached data. The data will then be faulted in from the TSA as it is needed.

#### Are there any alternatives to putting the cache into bulk-load mode?
 * Bulk-loading Cache methods `putAll()`, `getAll()`, and `removeAll()` provide high-performance and eventual consistency. These can also be used with strong consistency. If you can use them, bulk-load mode might not be necessary. For details, see the [API documentation](http://www.ehcache.org/apidocs/2.8.5/index.html).

#### Why does the bulk loading mode only apply to Terracotta clusters?
Ehcache, both standalone and replicated, is already fast.

#### How does bulk load with RMI distributed caching work?
The core updates are fast. RMI updates are batched by default once per second,
so bulk loading is efficiently replicated.

#### Bulk Loading and Nonstop
If a nonstop cache is bulk-loaded, a multiplier is applied to the configured nonstop timeout whenever the method `net.sf.ehcache.Ehcache.setNodeBulkLoadEnabled(boolean)` is used. The default value of the multiplier is 10. You can tune the multiplier using the `bulkOpsTimeoutMultiplyFactor` system property:

~~~
-Dnet.sf.ehcache.nonstop.bulkOpsTimeoutMultiplyFactor=10
~~~

For a bulk-loaded nonstop cache, the cache size displayed in the TMC is subject to the `bulkOpsTimeoutMultiplyFactor`. Increasing this multiplier on the clients can facilitate more accurate size reporting.

This multiplier also affects the methods `net.sf.ehcache.Ehcache.getAll()`, `net.sf.ehcache.Ehcache.removeAll()`, and `net.sf.ehcache.Ehcache.removeAll(boolean)`.



## Performance Tips {#performance-tips}

#### Bulk Loading on Multiple Nodes
The implementation scales well when the load is split up against multiple Ehcache CacheManagers on multiple machines. Adding nodes for bulk loading is likely to improve performance.

#### Why not run in bulk load mode all the time
Terracotta clustering provides consistency, scaling and durability. Some applications might require consistency. However, for reference data it might be acceptable to run a cache permanently in inconsistent mode.
