---
---
# Working With the Terracotta Toolkit

{toc|2:3}


## Introduction

The Terracotta Toolkit provides access to a number of useful classes, or tools, such as distributed collections. All provided Toolkit types are automatically distributed and all changes (except as noted below) are shared across all nodes by the Terracotta Server Array. This gives all nodes the same view of state.

While the Toolkit types differ in their usage and functionality based on characteristics inherited from their parent types, there are [certain shared features](#shared-characteristics).

This document provides information on getting started using the Terracotta Toolkit. For more detailed information on Toolkit types, see the Terracotta Toolkit Javadoc.

<a id="initialize"></a>
## Initializing the Toolkit
To access the tools in the Toolkit, your application must first initialize the Terracotta Toolkit. Initializing the Terracotta Toolkit always begins with starting a Terracotta client:

    ...
    // These classes must be imported:
    import org.terracotta.toolkit.*;
    ...

    // A configuration source must be specified in the argument.
    // In this case the source is a server and its configured port.
    Toolkit toolkit = ToolkitFactory.createToolkit("toolkit:terracotta://localhost:9510");
    ...

When a Terracotta client is started, it must load a Terracotta configuration. Programmatically, `createToolkit()` takes as argument the source for the client's configuration. In the example above, the configuration source is a Terracotta server running on the local host, with port set to 9510. In general, a filename, an URL, or a resolvable hostname or IP address and port number can be used. The specified server instance must be running and accessible before the code that starts the client executes.

The data structures and other tools provided by the Toolkit are automatically distributed (clustered) when your application is run in a Terracotta cluster. Since the Toolkit is obtained from an instantiated client, all Toolkit tools must be used clustered. Unclustered use is not supported in this version.

### Adding Rejoin Behavior
Clients can be initialized with the ability to [rejoin](/documentation/4.1/bigmemorymax/terracotta-toolkit/toolkit-reference#rejoin) the cluster after being ejected. Clients rejoin as new clients.

Clients are initialized with rejoin using a Properties object:

    Properties properties = new Properties();
    properties.put("rejoin","true");
    Toolkit toolkit = ToolkitFactory
                        .createToolkit("toolkit:terracotta://localhost:9510", properties);


## Toolkit Data Structures
All data structures must have serializable keys and values. In addition, data structures may have other requirements on keys as noted.

### ToolkitStore
The `ToolkitStore` is a key-value data store that is automatically distributed in a Terracotta cluster. The `ToolkitStore` is used similarly to the `ToolkitCache` except that it lacks eviction. It is limited to keys of type String.

To create a `ToolkitStore`, use the following:

    // Enforce type checking -- can store only Value.class.
    ToolkitStore<Value> store1 = toolkit.getStore("myStore1", Value.class);

    // Do not enforce type checking -- any type of value can be stored.
    ToolkitStore store2 = toolkit.getStore("myStore2",  null);

    // Use the provided configuration instead of default configuration.
    ToolkitStore<Value> store3 = toolkit.getStore("myStore3", configuration, Value.class);


#### Store Configuration
Configuration properties available for the toolkit are shown in the following table.

<a id="store-config-table"></a>
<table>
  <tr><th>Property</th><th>Default Value</th><th>Definition</th><th>Edit Dynamically?</th></tr>
  <tr><td>CONSISTENCY</td><td>EVENTUAL</td><td>EVENTUAL sets no cache-level locks for better performance and allows reads without locks, which means the cache may temporarily return stale data in exchange for substantially improved performance. STRONG uses cache-level locks for immediate cache consistency at a potentially high cost to performance, guaranteeing that after any update is completed no local read can return a stale value. If using strong consistency with off-heap, a large number of locks may need to be stored in client and server heaps. In this case, be sure to test the cluster with the expected data set to detect situations where OutOfMemory errors are likely to occur. SYNCHRONOUS_STRONG adds synch-write locks to STRONG consistency, which otherwise uses asynchronous writes. SYNCHRONOUS_STRONG provides extreme data safety <emphasis>at a very high performance cost</emphasis> by requiring that a client receive server acknowledgement of a transaction before that client can proceed.</td><td>No</td>
  </tr>
  <tr><td>CONCURRENCY</td><td>256</td><td>Sets the number of segments for the map backing the underlying server store managed by the Terracotta Server Array. If concurrency is not explicitly set (or set to "0"), the system selects an optimized value.</td><td>No</td>
  </tr>
  <tr><td>MAX_COUNT_LOCAL_HEAP</td><td>0 (no limit)</td><td>The maximum number of data entries that the store can have in local heap memory.</td><td>Yes (locally)</td>
  </tr>
  <tr><td>MAX_BYTES_LOCAL_HEAP<td>0 (no limit)</td><td>The maximum amount of data in bytes that the store can have in local heap memory.</td><td>Yes (locally)</td>
  </tr>
  <tr><td>MAX_BYTES_LOCAL_OFF_HEAP</td><td>0 (no limit)</td><td>The maximum amount of data in bytes that the store can have in local off-heap memory.</td><td>No</td>
  </tr>
  <tr><td>OFF_HEAP_ENABLED</td><td>false</td><td>Enables ("true") or disables ("false") the store's use of off-heap memory.</td><td>No</td>
  </tr>
  <tr><td>LOCAL_CACHE_ENABLED</td><td>true</td><td>Enables ("true") or disables ("false") local caching of distributed store data, forcing all of that cached data to reside on the Terracotta Server Array. Disabling local caching may improve performance for distributed stores in write-heavy use cases.</td><td>Yes (locally)</td>
  </tr>
</table>

Stores created without explicit configuration are provided a default configuration. To customize configuration, either pass configuration to `ToolkitStore` or edit the store's configuration.

You can also create a default store configuration with `ToolkitCacheConfigBuilder()`:

    Configuration initiallyDefaultConfig = new ToolkitStoreConfigBuilder().build();

This configuration can be passed to a store and further edited. Store configuration is governed by the same rules as cache configuration. To learn more about how to configure a store and edit its configuration, see the [cache configuration section](#configuring-toolkitcache), remembering to substitute the "store" types and methods.

### ToolkitCache
The `ToolkitCache` is a fully featured cache that is automatically distributed in a Terracotta cluster. It has configurable eviction and also inherits all of the functionality of the [Toolkit store](#toolkitstore). It is limited to keys of type String.

To create a `ToolkitCache`, use the following:

    // Enforce type checking -- can store only Value.class.
    ToolkitCache<Value> cache1 = toolkit.getCache("myCache1", Value.class);

    // Do not enforce type checking -- any type of value can be stored.
    ToolkitCache cache2 = toolkit.getCache("myCache2",  null);

    // Use the provided configuration instead of default configuration.
    ToolkitCache<Value> cache3 = toolkit.getCache("myCache3", configuration, Value.class);

    // You can also create a default cache configuration with ToolkitCacheConfigBuilder():

    Configuration initiallyDefaultConfig = new ToolkitCacheConfigBuilder().build();

This configuration can be passed to a cache and further edited based on [certain rules](#editing-cache-configuration).

#### Configuring ToolkitCache
Caches created without explicit configuration are provided a default configuration. To customize configuration, either pass configuration to `ToolkitCache` or [edit the cache's configuration](#editing-configuration).

In addition to the configuration properties inherited from `ToolkitStore`, the following configuration properties are available to the `ToolkitCache`:

<a id="cache-config-table"></a>
<table>
  <tr><th>Property</th><th>Default Value</th><th>Definition</th><th>Edit Dynamically</th></tr>
  <tr><td>MAX_TTI_SECONDS</td><td>0</td>
  <td>The time-to-idle setting for all cache entries. Default (0) is unlimited.</td><td>Yes (cluster-wide)</td>
  </tr>
  <tr><td>MAX_TTL_SECONDS</td><td>0</td>
  <td>The time-to-live setting for all cache entries. Default (0) is unlimited.</td><td>Yes (cluster-wide)</td>
  </tr>
  <tr><td>MAX_TOTAL_COUNT</td><td>0</td>
  <td>The total number of entries allowed for the cache across all client nodes.</td><td>Yes (cluster-wide)</td>
  </tr>
  <tr><td>PINNING_STORE</td><td>NONE</td>
  <td>Cache data is forced to remain in the designated storage tier, if any, and cannot be evicted. Valid values are INCACHE (any tier where the cache's data is located), LOCALHEAP, LOCALMEMORY (heap and off-heap), NONE. Issues due to lack of memory could arise with pinned caches that grow in size.</td><td>No</td>
  </tr>
</table>

To create a cache with a custom configuration use the following:

    // Create the configuration with the fluent builder interface.
    Configuration configuration = new ToolkitCacheConfigBuilder()
                  .maxBytesLocalHeap(2g)
                  .offheapEnabled(true)
                  .maxBytesLocalOffheap(10g)
                  .maxTTLseconds(3600)
                  .maxTTIseconds(600)
                  .maxTotalCount(40000)
                  .pinningStore(LOCALMEMORY)
                  .build();

    ToolkitCache<String> cache3 = toolkit.getCache("myCache3", configuration, String.class);

    // Create the configuration using a builder instance.
        ToolkitCacheConfigBuilder builder = new ToolkitCacheConfigBuilder();
        builder.maxCountLocalHeap(10000);
        builder.offheapEnabled(true);
        builder.maxBytesLocalOffheap(10 * GIGABYTE);

        ToolkitCache cache4 = toolkit.getCache("myCache4", null);

        builder.apply(cache4);

To find the values of specific configuration properties, use the corresponding get methods. For example:

		Long maxCountLocalHeap = builder.getMaxCountLocalHeap()

#### Editing Cache Configuration
Some configuration properties can be edited dynamically. For example:

        // Change the time-to-live value for cache4.

        builder.maxTTLseconds(3000);

        builder.apply(cache4);

In the example above, `builder` reapplies its other configuration properties with the same values without effect. Applying configuration properties with values that match the cache's configuration values leaves those values unchanged.

To learn which configuration properties can be edited dynamically, see the [table showing basic configuration](#store-config-table) and the [table showing cache-specific configuration](#cache-config-table).

Note that default configurations have MAX_COUNT_LOCAL_HEAP in effect, and this cannot be edited:

    // Create the configuration using a builder instance.
        ToolkitCacheConfigBuilder builder5 = new ToolkitCacheConfigBuilder();
        builder.maxBytesLocalHeap(2g);
        builder.offheapEnabled(true);
        builder.maxBytesLocalOffheap(10 * GIGABYTE);

        // cache5 will have a default configuration.
        ToolkitCache cache5 = toolkit.getCache("myCache5", null);

        // The following will throw an IllegalStateException.
        builder5.apply(cache5);


### Clustered Collections

Clustered collections, all based on standard Java clustered collections, are found in the package `org.terracotta.collections`. Operations on these collections are locked.


#### ToolkitBlockingQueue
Since this is a bounded `BlockingQueue`, use `ToolkitBlockingQeue.getCapacity()` to return an integer representing the queue's maximum capacity (maximum number of entries), which is not changeable once the queue has been created. If no capacity is specified, the effective capacity is `Integer.MAX_VALUE`.

To create a `ToolkitBlockingQueue`, use the following:

    // A blocking queue with no capacity restriction. Restricted to the type Value.class.
    ToolkitBlockingQueue<Value> queue1 =
        toolkit.getBlockingQueue("myBlockingQueue1", Value.class);

    // A blocking queue with no capacity or type restrictions.
    ToolkitBlockingQueue queue2 = toolkit.getBlockingQueue("myBlockingQueue2", null);

    // A blocking queue with specified capacity. Restricted to the type Value.class.
    ToolkitBlockingQueue queue3 =
        toolkit.getBlockingQueue("myBlockingQueue3", 1024, Value.class);

If the blocking queue specified by `getBlockingQueue()` exists, it is returned instead (no new blocking queue is created). Note that if queue3 exists and has a capacity value different than the one specified in `getBlockingQueue()`, an IllegalArgumentException is thrown.


#### ToolkitMap and ToolkitSortedMap
These types behave as expected. Note that `ToolkitSortedMap` values must allow ordering (implement Comparable).

To create a `ToolkitMap`, use the following:

    // Enforce type checking -- can store only Value.class using keys of type Key.class.
    ToolkitMap<Key, Value> map = toolkit.getMap("myMap", Key.class, Value.class);

    // Do not enforce type checking -- any type of key-value can be stored.
    ToolkitMap<Key, Value> map = toolkit.getMap("myMap", null, null);

`ToolkitSortedMap` is created similarly.


#### ToolkitSet, ToolkitSortedSet, and ToolkitList

These types behave as expected, but note the following:

* ToolkitList does not allow null elements.
* SortedSet values must allow ordering (implement Comparable).

All are created similarly. To create a `ToolkitSet`, for example, use the following:

    // Enforce type checking -- can store only Value.class.
    ToolkitSet<Value> set1 = toolkit.getSet("mySet1", Value.class);

    // Do not enforce type checking -- any type of value can be stored.
    ToolkitSet set2 = toolkit.getSet("mySet2", null);



## Cluster Information and Messaging

To implement messaging in a Terracotta cluster, you can use the built-in cluster events and custom messages with a clustered notifier.

### Cluster Events
You can access cluster information for monitoring the nodes in a Terracotta cluster, as well as obtaining specific information about those nodes.

For example, you can set up a cluster listener to receive events about the status of client nodes:

    // Start a client and access cluster events and meta data such as topology
    // for the cluster that the client belongs to:

    ClusterInfo clusterInfo = toolkit.getClusterInfo();

    // Register a cluster listener:
    clusterInfo.addClusterListener(new ClusterListener());

#### Custom Listener Classes
You can write your own listener classes that implement the event methods, and add or remove your own listeners:

    MyClusterListener myClusterListener = new MyClusterListener();
    clusterInfo.addClusterListener(new myClusterListener());

    // To remove a listener:
    clusterInfo.removeClusterListener(myClusterListener);

#### Handling Cluster Events
To handle events, use `onClusterEvent()`, which is called whenever a ClusterEvent occurs. See the suggested approaches below.

##### Custom Implemented Methods
Implement methods to handle each type of event, then use them in a test for the event type.

    public void onClusterEvent(ClusterEvent event) {

       if (event.getType() == NODE_LEFT) {
          handleNodeLeft();
       }

       if (event.getType() == NODE_JOINED) {
          handleNodeJoined();
       }

       if (event.getType() == OPERATIONS_DISABLED) {
          handleOperationsDisabled();
       }

       if (event.getType() == OPERATIONS_ENABLED) {
          handleOperationsEnabled();
       }

    }

##### Switch Case
Implement a switch-case block to handle each element type.

    public void onClusterEvent(ClusterEvent event) {
        switch (event.getType()) {
            case NODE_JOINED:
                  // Handle the fact that a node joined.
                 break;
            case NODE_LEFT:
                  // Handle node leaving.
                 break;
            case OPERATIONS_ENABLED:
                  // Handle operations disabled.
                 break;
            case OPERATIONS_ENABLED:
                  // Handle operations disabled.
                 break;
            default:
                  // handle other events generically
        }
    }


### Toolkit Notifier
`ToolkitNotifier` provides simple-to-use, clustered, and customizable messaging.

To create a `ToolkitNotifier`, use the following:

    // Enforce type checking -- can send/receive only Value.class.
    ToolkitNotifier<Value> notifier1 = toolkit.getNotifier("my-notifier1", Value.class);

    // Do not enforce type checking -- any type of value can be sent/received.
    ToolkitNotifier notifier2 = toolkit.getNotifier("my-notifier2", null);

    // Add a notification listener.
    notifier1.addNotificiationListener(new MyToolkitNotificationListener());

The notification listener is a custom implementation of the `ToolkitNotificationListener` interface. Specifically, implement `ToolkitNotificationListener.onNotification()`.

To send a message, use the following:

    notifier1.notifyListeners(new Value());

Only remote listeners receive the message.


## Locks {#37461}

Clustered locks allow you to perform safe operations on clustered data. The following types of locks are available:

* **READ-WRITE** &ndash;  This provides an exclusive read lock and a shared read lock.
* **WRITE** &ndash;  This is an exclusive  write lock that blocks all reads and writes. To improve performance, this lock flushes changes to the Terracotta Server Array asynchronously.

To get and use clustered READ-WRITE locks, use the following:

    ToolkitReadWriteLock rwLock = toolkit.getReadWriteLock("myReadWriteLock");
    rwlock.readLock().lock(); // shared read lock
    try {
        // some read operation under the lock
    } finally {
        rwlock.readLock().unlock();
    }

    rwlock.writeLock().lock(); // exclusive write lock
    try {
        // some write operation under the lock
    } finally {
        rwlock.writeLock().unlock();
    }

To get and use an exclusive WRITE lock, use the following:

    ToolkitLock wLock = toolkit.getLock("myWriteLock");
    wLock.lock();
    try {
        // some operation under the lock
    } finally {
        wLock.unlock();
    }

If you are using BigMemory Max, you can use explicit locking methods on specific keys. See the BigMemory Max documentation for more information.


## Barriers

Coordinating independent nodes is useful in many aspects of development, from running more accurate performance and capacity tests to more effective management of workers across a cluster. A clustered barrier is a simple and effective way of coordinating client nodes.

To get a clustered barrier, use the following:


    // Get a clustered barrier with 4 parties.
    ToolkitBarrier clusteredBarrier = toolkit.getBarrier("myBarrier", 4);

Note the following:

* `getBarrier()` as implemented in Terracotta Toolkit returns an object that behaves like  CyclicBarrier but lacks `barrierAction` support.
* The number of parties must be an integer equal to or greater than 1.
* If `getBarrier()` returns an existing barrier, and the specified number of parties does not match that of the returned barrier, an IllegalArgumentException is thrown.

## Utilities

Utilities such as a clustered `AtomicLong` help track counts across a cluster. You can get (or create) a `ToolkitAtomicLong` using `toolkit.getAtomicLong(String name)`.

An events utility, available with `Toolkit.fireOperatorEvent(OperatorEventLevel level, String applicationName, String eventMessage)`, can be used to log specific application-related events.


<a id="shared-characteristics"></a>
## Shared Characteristics
Certain characteristics are shared by many of the Terracotta Toolkit types.

<a id="nonstop"></a>
### Nonstop Behavior for Data Structures
The nonstop feature is enabled by specifying it at the time a toolkit is initialized:

    Toolkit toolkit =
        ToolkitFactory.createToolkit("nonstop-toolkit:terracotta://localhost:9510");

Use the following if you are passing a properties object (to [add rejoin behavior](#initialize) for example):

    Toolkit toolkit = ToolkitFactory
                .createToolkit("nonstop-toolkit:terracotta://localhost:9510", properties);

Nonstop behavior allows the following limited functionality whenever a Terracotta client disconnects from the TSA:

* Throw exception for read and write operations
* Silently ignore read and write operations (no-op)
* Allow local reads

In addition, nonstop data structures running in a client that is unable to locate the TSA at startup will initiate nonstop behavior as if the client had disconnected.

#### Toolkit Types Supporting Nonstop
ToolkitCaches and ToolkitStores can be configured with the full range of nonstop behaviors. The following Toolkit types support only throwing a NonStopException: ToolkitAtomicLong, ToolkitList<E>, ToolkitMap<K,V>, ToolkitSortedMap<K,V>, ToolkitLock, ToolkitReadWriteLock, ToolkitNotifier<T>, ToolkitSet<E>, ToolkitSortedSet<E>. Other Toolkit types cannot be configured with nonstop.

#### Adding Nonstop Config to a Cache
You can add nonstop configuration to Toolkit caches.

    // Get the cache:
    ToolkitCache<String> cache = toolkit.getCache("myCache", configuration, String.class);

    // Build the nonstop configuration:
    NonStopConfiguration nsconfig = new NonStopConfigurationBuilder()
                                      .nonStopReadTimeoutBehavior(LOCAL_READS)
                                      .timeOutMillis(90000)
                                      .build();

    // Register the nonstop configuration with the cache:
    NonStopFeature nonStop = toolkit.getFeature(ToolkitFeatureType.NONSTOP);
    NonStopConfigurationRegistry registry = nonStop.getNonStopConfigurationRegistry();
    registry.registerForInstance(nsconfig, myCache, ToolkitCache)


### Disposing of Toolkit Objects
Many types are destroyable and can be disposed of using `destroy()`. If the object is distributed, it is destroyed in all nodes if `destroy()` is called in one node. For example, `ToolkitCache.destroy()` disposes of the cache and its data in all nodes where it exists.

Calling `destroy()` on a nonexistent object has no effect (no-op). However, attempting to use a destroyed object throws an IllegalStateException.

The following Toolkit types are destroyable: ToolkitAtomicLong, ToolkitBarrier, ToolkitBlockingQueue<E>, ToolkitCache<K,V>, ToolkitList<E>, ToolkitMap<K,V>, ToolkitNotifier<T>, ToolkitSet<E>, ToolkitSortedMap<K,V>, ToolkitSortedSet<E>, ToolkitStore<K,V>.

### Finding the Assigned Name
Most Toolkit types have assigned String names. You can find an object's assigned name using its `getName()` method, even after that object has been destroyed.

The following Toolkit types support `getName()`: ToolkitAtomicLong, ToolkitBarrier, ToolkitBlockingQueue<E>, ToolkitCache<K,V>, ToolkitList<E>, ToolkitLock, ToolkitMap<K,V>, ToolkitNotifier<T>, ToolkitReadWriteLock, ToolkitSet<E>, ToolkitSortedMap<K,V>, ToolkitSortedSet<E>, ToolkitStore<K,V>.

### Returning Existing Toolkit Objects
With certain Toolkit types, the `Toolkit` get methods will either return the existing named object or create it. For example, running `Toolkit.getMap("myMap", null, null)` returns myMap if it already exists, otherwise a new map is created with the name "myMap".

The following Toolkit types support this functionality: ToolkitBlockingQueue<E>, ToolkitCache<K,V>, ToolkitList<E>, ToolkitMap<K,V>, ToolkitNotifier<T>, ToolkitSet<E>, ToolkitSortedMap<K,V>, ToolkitSortedSet<E>, ToolkitStore<K,V>.

Names can be reused. If the specified name belonged to a destroyed object, a new object is created using that name.
