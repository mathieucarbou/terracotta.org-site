---
---
# Distributed BigMemory Max Configuration Guide {#94264}

{toc|2:3}

## Introduction

The BigMemory Max configuration file (`ehcache.xml` by default) contains the configuration for one instance of a CacheManager (the Ehcache class managing a set of defined caches). This configuration file must be in your application's classpath to be found. When using a WAR file, `ehcache.xml` should be copied to `WEB-INF/classes`.

Note the following about `ehcache.xml` in a Terracotta cluster:

* The copy on disk is loaded into memory from the first Terracotta client (also called application server or node) to join the cluster.
* Once loaded, the configuration is persisted in memory by the Terracotta servers in the cluster and survives client restarts.
* In-memory configuration can be edited in the Terracotta Management Console (TMC).
    Changes take effect immediately but are *not* written to the original on-disk copy of `ehcache.xml`.
* The in-memory cache configuration is removed with server restarts if the servers are not in persistent mode (`<restartable enabled="false">`), which is the default.
    The original (on-disk) `ehcache.xml` is loaded.
* The in-memory cache configuration survives server restarts if the servers are in persistent mode (`<restartable enabled="true">`).
    If you are using the Terracotta servers with persistence of shared data, and you want the cluster to load the original (on-disk) `ehcache.xml`, the servers' database must be wiped by removing the data files from the servers' `server-data` directory. This directory is specified in the Terracotta configuration file in effect (`tc-config.xml` by default). Wiping the database causes *all persisted shared data to be lost*.

A minimal distributed-cache configuration should have the following configured:

* A [CacheManager](#cachemanager-configuration)
* A [Clustering element](#70873) in individual cache configurations
* A source for [Terracotta client configuration](#35085)


## CacheManager Configuration
CacheManager configuration elements and attributes are fully described in the `ehcache.xml` reference file available in the kit.

### Via ehcache.xml

The attributes of `<ehcache>` are:

* name &ndash;
an optional name for the CacheManager.  The name is optional and primarily used
for documentation or to distinguish Terracotta clustered cache state.  With Terracotta
clustered caches, a combination of CacheManager name and cache name uniquely identify a
particular cache store in the Terracotta clustered memory.
The name will show up in the TMC.

<table>
<caption>TIP: Naming the CacheManager</caption>
<tr>
<td>
If you employ multiple Ehcache configuration files, use the <code>name</code> attribute in the &lt;ehcache> element to identify specific CacheManagers in the cluster. The TMC provides a menu listing these names, allowing you to choose the CacheManager you want to view.
</td>
</tr>
</table>

<a name="updatecheck"></a>

* updateCheck &ndash;
an optional boolean flag specifying whether this CacheManager should check
for new versions of Ehcache over the Internet.  If not specified, updateCheck="true".
* monitoring &ndash;
an optional setting that determines whether the CacheManager should
automatically register the SampledCacheMBean with the system MBean server.  Currently,
this monitoring is only useful when using Terracotta clustering. The "autodetect" value detects the presence of Terracotta clustering and registers the MBean.  Other allowed values
are "on" and "off".  The default is "autodetect".

        <Ehcache xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="ehcache.xsd"
        updateCheck="true" monitoring="autodetect">


### Programmatic Configuration

CacheManagers can be configured programmatically with a fluent API. The example below
creates a CacheManager with a Terracotta configuration specified in an URL, and creates a defaultCache
and a cache named "example".

    Configuration configuration = new Configuration()
    .terracotta(new TerracottaClientConfiguration().url("localhost:9510"))
    .defaultCache(new CacheConfiguration("defaultCache", 100))
    .cache(new CacheConfiguration("example", 100)
    .timeToIdleSeconds(5)
    .timeToLiveSeconds(120)
    .terracotta(new TerracottaConfiguration()));
    CacheManager manager = new CacheManager(configuration);




## Terracotta Clustering Configuration Elements {#95592}

Certain elements in the Ehcache configuration control the clustering of caches with Terracotta.

<a id"96514"></a>
### terracotta {#70873}

The `<terracotta>` element is an optional sub-element of `<cache>`. It can be set differently for each `<cache>` defined in `ehcache.xml`.

`<terracotta>` has one subelement, `<nonstop>` (see [Non-Blocking Disconnected (Nonstop) Operation](/documentation/4.1/bigmemorymax/configuration/non-stop-cache) for more information).

The following `<terracotta>` attribute allows you to control the type of data consistency for the distributed cache:

* consistency &ndash;  Uses no cache-level locks for better performance  ("eventual" DEFAULT) or uses cache-level locks for immediate cache consistency ("strong"). When set to "eventual", allows reads without locks, which means the cache may temporarily return stale data in exchange for substantially improved performance. When set to "strong", guarantees that after any update is completed no local read can return a stale value, but at a potentially high cost to performance, because a large number of locks may need to be stored in client and server heaps.  

    Once set, this consistency mode cannot be changed except by reconfiguring the cache using a configuration file and reloading the file. *This setting cannot be changed programmatically.* See <a href="http://terracotta.org/documentation/4.1/bigmemorymax/configuration/reference-guide#30971">Understanding Performance and Cache Consistency</a> for more information.

Except for special cases, the following `<terracotta>` attributes should operate with their default values:

* clustered &ndash;  Enables ("true" DEFAULT) or disables ("false") Terracotta clustering of a specific cache. Clustering is enabled if this attribute is not specified. Disabling clustering also disables the effects of all of the other attributes.
* localCacheEnabled &ndash; Enables ("true" DEFAULT) or disables ("false") local caching of distributed cache data, forcing all of that cached data to reside on the Terracotta Server Array. Disabling local caching may improve performance for distributed caches in write-heavy use cases.
* synchronousWrites &ndash;  Enables ("true") or disables ("false" DEFAULT) synchronous writes from Terracotta clients (application servers) to Terracotta servers. Asynchronous writes (synchronousWrites="false") maximize performance by allowing clients to proceed without waiting for a "transaction received" acknowledgement from the server. This acknowledgement is unnecessary in most use cases. Synchronous writes (synchronousWrites="true") provide extreme data safety but at a performance cost by requiring that a client receive server acknowledgement of a transaction before that client can proceed. If the cache's consistency mode is eventual, or while it is set to bulk load using the bulk-load API, only asynchronous writes can occur (synchronousWrites="true" is ignored).
* concurrency &ndash;  Sets the number of segments for the map backing the underlying server store managed by the Terracotta Server Array. If concurrency is not explicitly set (or set to "0"), the system selects an optimized value. See <a href="http://terracotta.org/documentation/4.1/bigmemorymax/configuration/reference-guide#86549">Tuning Concurrency</a> for more information on how to tune this value.

The following attributes are used with Hibernate:

* localKeyCache &ndash;  Enables ("true") or disables ("false" DEFAULT) a local key cache. BigMemory Max can cache a "hotset" of keys on clients to add locality-of-reference, a feature suitable for read-only cases. Note that the set of keys must be small enough for available memory.
* localKeyCacheSize &ndash;  Defines the size of the local key cache in number (positive integer) of elements. In effect if localKeyCache is enabled. The default value, 300000, should be tuned to meet application requirements and environmental limitations.
* orphanEviction &ndash;  Enables ("true" DEFAULT) or disables ("false") orphan eviction. *Orphans* are cache elements that are not resident in any Hibernate second-level cache but still present on the cluster's Terracotta server instances.
* orphanEvictionPeriod &ndash;  The number of local eviction cycles (that occur on Hibernate) that must be completed before an orphan eviction can take place. The default number of cycles is 4. Raise this value for less aggressive orphan eviction that can reduce faulting on the Terracotta server, or lower it if garbage on the Terracotta server is a concern.


#### Default Behavior

By default, adding `<terracotta/>` to a `<cache>` block is equivalent to adding the following:

    <cache name="sampleTerracottaCache"
        maxEntriesLocalHeap="1000"
        eternal="false"
        timeToIdleSeconds="3600"
        timeToLiveSeconds="1800"">
      <persistence strategy="distributed"/>
      <terracotta clustered="true" consistency="eventual" />
    </cache>

<a id"96514"></a>
### terracottaConfig {#35085}

The `<terracottaConfig>` element enables the distributed-cache client to identify a source of Terracotta configuration. It also allows a client to rejoin a cluster after disconnecting from that cluster and being timed out by a Terracotta server. For more information on the rejoin feature, see <a href="http://terracotta.org/documentation/4.1/bigmemorymax/configuration/reference-guide#71266">Using Rejoin to Automatically Reconnect Terracotta Clients</a>. In addition, the `<terracottaConfig>` element allows you to enable caches for WAN replication.

The client must load the configuration from a file or a Terracotta server. The value of the `url` attribute should contain a path to the file, a system property, or the address and TSA port (9510 by default) of a server.

<table>
<caption>TIP: Terracotta Clients and Servers</caption>
<tr>
<td>
In a Terracotta cluster, the application server is also known as the client.
</td>
</tr>
</table>

For more information on client configuration, see the *Clients Configuration Section* in the <a href="http://terracotta.org/documentation/4.1/terracotta-server-array/config-reference">Terracotta Configuration Reference</a>.


#### Adding an URL Attribute

Add the `url` attribute to the `<terracottaConfig>` element as follows:

    <terracottaConfig url="<source>" />

where `<source>` must be one of the following:

* A path (for example, `url="/path/to/tc-config.xml"`)
* An URL (for example, `url="http://www.mydomain.com/path/to/tc-config.xml`)
* A system property (for example, `url="${terracotta.config.location}"`), where the system property is defined like this:

~~~
System.setProperty("terracotta.config.location","10.x.x.x:9510"");
~~~

* A Terracotta host address in the form `<host>:<tsa-port>` (for example, `url="host1:9510"`). Note the following about using server addresses in the form `<host>:<tsa-port>`:
    * The default TSA port is 9510.
    * In a multi-server cluster, you can specify a comma-delimited list (for example, `url="host1:9510,host2:9510,host3:9510"`).
    * If the Terracotta configuration source changes at a later time, it must be updated in configuration.

#### Adding the WAN Attribute

For each cache to be replicated, its `ehcache.xml` configuration file must include the `wanEnabledTSA` attribute set to "true" within the `<terracottaConfig>` element. Add the `wanEnabledTSA` attribute to the `<terracottaConfig>` element as follows:

    <terracottaConfig wanEnabledTSA="true"/>

For more information, refer to [BigMemory WAN Replication](http://terracotta.org/documentation/4.1/wan/introduction).

### Embedding Terracotta Configuration

You can embed the contents of a Terracotta configuration file in `ehcache.xml` as follows:

    <terracottaConfig>
       <tc-config>
           <servers>
               <server host="server1" name="s1"/>
               <server host="server2" name="s2"/>
           </servers>
           <clients>
               <logs>app/logs-%i</logs>
           </clients>
       </tc-config>
    </terracottaConfig>



## Controlling Cache Size {#78357}

Certain Ehcache cache configuration attributes affect caches clustered with Terracotta.

See <a href="http://terracotta.org/documentation/4.1/bigmemorymax/configuration/reference-guide#30343">How Configuration Affects Element Eviction</a> for more information on how configuration affects eviction.

To learn about eviction and controlling the size of the cache, see the [data life](/documentation/4.1/bigmemorymax/configuration/data-life) and [sizing storage tiers](/documentation/4.1/bigmemorymax/configuration/cache-size) pages.

## Setting Cache Eviction {#24283}
Cache eviction removes elements from the cache based on parameters with configurable values. Having an optimal eviction configuration is critical to maintaining cache performance.

To learn about eviction and controlling the size of the cache, see the [data life](/documentation/4.1/bigmemorymax/configuration/data-life) and [sizing storage tiers](/documentation/4.1/bigmemorymax/configuration/cache-size) pages.

Ensure that the edited `ehcache.xml` is in your application's classpath. If you are using a WAR file,  `ehcache.xml` should be in `WEB-INF/classes`.

See <a href="http://terracotta.org/documentation/4.1/bigmemorymax/configuration/reference-guide#30343">How Configuration Affects Element Eviction</a> for more information on how configuration can impact eviction. See <a href="#95592">Terracotta Clustering Configuration Elements</a> for definitions of other available configuration properties.

## Cache-Configuration File Properties {#11000}

See <a href="#95592">Terracotta Clustering Configuration Elements</a> for more information.



## Cache Events Configuration {#20075}

The `<cache>` subelement &lt;cacheEventListenerFactory>, which registers listeners for cache events such as puts and updates, has a notification scope controlled by the attribute `listenFor`. This attribute can have one of the following values:

* local &ndash;  Listen for events on the local node. No remote events are detected.
* remote &ndash;  Listen for events on other nodes. No local events are detected.
* all &ndash;  (DEFAULT) Listen for events on both the local node and on remote nodes.

In order for cache events to be detected by remote nodes in a Terracotta cluster, event listeners must have a scope that includes remote events. For example, the following configuration allows listeners of type MyCacheListener to detect both local and remote events:

    <cache name="myCache" ... >
     <!-- Not defining the listenFor attribute for <cacheEventListenerFactory> is by default
          equivalent to listenFor="all". -->
     <cacheEventListenerFactory
        class="net.sf.ehcache.event.TerracottaCacheEventReplicationFactory" />
     <terracotta />
    </cache>

You must use `net.sf.ehcache.event.TerracottaCacheEventReplicationFactory` as the factory class to enable cluster-wide cache-event broadcasts in a Terracotta cluster.

See <a href="http://terracotta.org/documentation/4.1/bigmemorymax/configuration/reference-guide#82378">Cache Events in a Terracotta Cluster</a> for more information on cache events in a Terracotta cluster.


## Copy On Read

The `copyOnRead` setting is most easily explained by first examining what it does when not enabled and exploring the potential problems that can arise.
For a cache in which `copyOnRead` is NOT enabled, the following reference comparison will always be true:

    Object obj1 = c.get("key").getValue();
    // Assume no other thread changes the cache mapping between these get() operations ....
    Object obj2 = c.get("key").getValue();
    if (obj1 == obj2) {
     System.err.println("Same value objects!");
    }

The fact that the same object reference is returned across multiple `get()` operations implies that the cache is storing a direct reference to cache value. This default behavior (copyOnRead=false) is usually desired, although there are at least two scenarios in which it is problematic:

(1) Caches shared between classloaders

and

(2) Mutable value objects

Imagine two web applications that both have access to the same Cache instance (this implies that the core Ehcache classes are in a common classloader). Imagine further that the classes for value types in the cache are duplicated in the web application (so they are not present in the common loader).
In this scenario you would get ClassCastExceptions when one web application accessed a value previously
read by the other application.

One obvious solution to this problem is to move the value types to the
common loader, but another is to enable `copyOnRead`. When copyOnRead is in effect, the object references are unique with every `get()`. Having unique object references means that the thread context loader of the caller
will be used to materialize the cache values on each `get()`. This feature has utility in OSGi environments as well where a common cache service might be shared between bundles.

Another subtle issue concerns mutable value objects in a distributed cache. Consider this simple code with a Cache containing a mutable value type (Foo):

    class Foo {
     int field;
    }
    Foo foo = (Foo) c.get("key").getValue();
    foo.field++;
    // foo instance is never re-put() to the cache
    // ...

If the Foo instance is never put back into the Cache your local cache is no longer consistent with the cluster (it is locally modified only). Enabling `copyOnRead` eliminates this possibility since the only way to affect cache values is to call mutator methods on the Cache.

It is worth noting that there is a performance penalty to copyOnRead since values are deserialized on every `get()`.


## Configuring Robust Distributed In-memory Data Sets

Making the BigMemory Max in-memory data system robust is typically a combination of Ehcache configuration and Terracotta configuration and architecture. For more information, see the following documentation:

* [Nonstop caches](/documentation/4.1/bigmemorymax/configuration/non-stop-cache) &ndash; Configure caches to take a specified action after an Ehcache node appears to be disconnected from the cluster.
* [Rejoin the cluster](/documentation/4.1/bigmemorymax/configuration/reference-guide#71266) &ndash; Allow Ehcache nodes to rejoin the cluster as new clients after being disconnected from the cluster.
* [High Availability in a Terracotta cluster](/documentation/4.1/terracotta-server-array/high-availability) &ndash;  Configure nodes to ride out network interruptions and long Java GC cycles, connect to a backup Terracotta server, and more.
* [Architecture](/documentation/4.1/terracotta-server-array/server-arrays) &ndash; Design a cluster that provides failover.
* [BigMemory Hybrid](/documentation/4.1/terracotta-server-array/hybrid) &ndash; Extend BigMemory on Terracotta servers with hybrid storage that can include SSD and Flash technologies as well as DRAM.



## Incompatible Configuration

For any clustered cache, you must delete, disable, or edit configuration elements in `ehcache.xml` that are incompatible when clustering with Terracotta. Clustered caches have a `<terracotta/>` or `<terracotta clustered="true"/>` element.

The following Ehcache configuration attributes or elements should be deleted or disabled:

* Replication-related attributes such as `replicateAsynchronously` and `replicatePuts`.
* The attribute `MemoryStoreEvictionPolicy` is ignored (a clock eviction policy is used instead), however, if allowed to remain in a clustered cache configuration, the `MemoryStoreEvictionPolicy` may cause an exception.


## Exporting Configuration from the Terracotta Management Console {#68288}

To create or edit a cache configuration in a live cluster, see <a href="http://terracotta.org/documentation/4.1/tms/tms#42656">Editing Cache Configuration</a>.

To persist custom cache configuration values, create a cache configuration file by exporting customized configuration from the TMC or create a file that conforms to the required format. This file must take the place of any configuration file used when the cluster was last started.
