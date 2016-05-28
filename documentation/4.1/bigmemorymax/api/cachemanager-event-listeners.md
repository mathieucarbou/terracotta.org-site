---
---
# Event Listeners


{toc|2:3}


## CacheManager Event Listeners

BigMemory Max's Ehcache implementation includes CacheManager event listeners. These listeners allow implementers to register callback
methods that will be executed when a CacheManager event occurs. Cache listeners
implement the `CacheManagerEventListener` interface.
The events include:

* adding a `Cache`

* removing a `Cache`

Callbacks to these methods are synchronous and unsynchronized. It is
the responsibility of the implementer to safely handle the potential
performance and thread safety issues depending on what their listener
is doing.


### Configuration

One `CacheManagerEventListenerFactory` and hence one `CacheManagerEventListener` can be specified per CacheManager instance.
The factory is configured as below:

~~~
<cacheManagerEventListenerFactory class="" properties=""/>
~~~

The entry specifies a `CacheManagerEventListenerFactory` which will be used to
create a `CacheManagerEventListener`, which is notified when Caches are
added or removed from the CacheManager.
The attributes of a `CacheManagerEventListenerFactory` are:

* `class` &mdash; a fully qualified factory class name.

* `properties` &mdash; comma-separated properties having meaning only to the factory.

Callbacks to listener methods are synchronous and unsynchronized. It is
the responsibility of the implementer to safely handle the potential
performance and thread safety issues depending on what their listener
is doing.
If no class is specified, or there is no `cacheManagerEventListenerFactory` element, no listener is created. There
is no default.

### Implementing a CacheManager Event Listener Factory and CacheManager Event Listener

`CacheManagerEventListenerFactory` is an abstract factory for creating
cacheManager listeners. Implementers should provide their own concrete
factory extending this abstract factory. It can then be configured in
ehcache.xml.
The factory class needs to be a concrete subclass of the abstract
factory `CacheManagerEventListenerFactory`, which is reproduced below:

    /**
    * An abstract factory for creating {@link CacheManagerEventListener}s. Implementers should
    * provide their own concrete factory extending this factory. It can then be configured in
    * ehcache.xml
    *
    */
    public abstract class CacheManagerEventListenerFactory {
    /**
    * Create a CacheManagerEventListener
    *
    * @param properties implementation specific properties.
    * These are configured as comma-separated name value pairs in ehcache.xml.
    * Properties may be null.
    * @return a constructed CacheManagerEventListener
    */
    public abstract CacheManagerEventListener
           createCacheManagerEventListener(Properties properties);
    }

The factory creates a concrete implementation of `CacheManagerEventListener`, which is reproduced below:

    /**
    * Allows implementers to register callback methods that will be executed when a
    * CacheManager event occurs.
    * The events include:
    *
    * adding a Cache
    * removing a Cache
    *
    *
    * Callbacks to these methods are synchronous and unsynchronized. It is the responsibility of
    * the implementer to safely handle the potential performance and thread safety issues
    * depending on what their listener is doing.
    */
    public interface CacheManagerEventListener {
    /**
    * Called immediately after a cache has been added and activated.
    *
    * Note that the CacheManager calls this method from a synchronized method. Any attempt to
    * call a synchronized method on CacheManager from this method will cause a deadlock.
    *
    * Note that activation will also cause a CacheEventListener status change notification
    * from {@link net.sf.ehcache.Status#STATUS_UNINITIALISED} to
    * {@link net.sf.ehcache.Status#STATUS_ALIVE}. Care should be taken on processing that
    * notification because:
    * <ul>
    * <li>the cache will not yet be accessible from the CacheManager.
    * <li>the addCaches methods whih cause this notification are synchronized on the
    * CacheManager. An attempt to call {@link net.sf.ehcache.CacheManager#getCache(String)}
    * will cause a deadlock.
    * </ul>
    * The calling method will block until this method returns.
    *
    * @param cacheName the name of the Cache the operation relates to
    * @see CacheEventListener
    */
    void notifyCacheAdded(String cacheName);
    /**
    * Called immediately after a cache has been disposed and removed. The calling method will
    * block until this method returns.
    *
    * Note that the CacheManager calls this method from a synchronized method. Any attempt to
    * call a synchronized method on CacheManager from this method will cause a deadlock.
    *
    * Note that a {@link CacheEventListener} status changed will also be triggered. Any
    * attempt from that notification to access CacheManager will also result in a deadlock.
    * @param cacheName the name of the Cache the operation relates to
    */
    void notifyCacheRemoved(String cacheName);
    }

The implementations need to be placed in the classpath accessible to
Ehcache. Ehcache uses the ClassLoader returned by `Thread.currentThread().getContextClassLoader()`
to load classes.




## Cache Event Listeners
BigMemory Max's Ehcache implementation also includes Cache event listeners. Cache listeners allow implementers to register callback methods that
will be executed when a cache event occurs. Cache listeners
implement the `CacheEventListener` interface.
The events include:

* an Element has been put
* an Element has been updated. Updated means that an Element exists in the Cache with the same key as the Element being put.
* an Element has been removed
* an Element expires, either because `timeToLive` or `timeToIdle` have been reached.

Callbacks to these methods are synchronous and unsynchronized. It is
the responsibility of the implementer to safely handle the potential
performance and thread safety issues depending on what their listener
is doing.
Listeners are guaranteed to be notified of events in the order in which
they occurred.
Elements can be put or removed from a Cache without notifying listeners
by using the `putQuiet` and `removeQuiet` methods.


### Configuration

Cache event listeners are configured per cache. Each cache can have
multiple listeners.
Each listener is configured by adding a
`cacheEventListenerFactory` element as follows:

    <cache ...>
      <cacheEventListenerFactory class="" properties="" listenFor=""/>
      ...
    </cache>

The entry specifies a `CacheEventListenerFactory` which is used to
create a `CacheEventListener`, which then receives notifications.
The attributes of a `CacheEventListenerFactory` are:

* `class` &mdash; a fully qualified factory class name.
* `properties` &mdash; optional comma-separated properties having meaning only to the factory.
* `listenFor` &mdash; describes which events will be delivered in a clustered environment (defaults to "all").

    These are the possible values:

    * "all" &mdash; the default is to deliver all local and remote events
    * "local" &mdash; deliver only events originating in the current node
    * "remote" &mdash; deliver only events originating in other nodes (for BigMemory Max only)

Callbacks to listener methods are synchronous and unsynchronized. It is
the responsibility of the implementer to safely handle the potential
performance and thread safety issues depending on what their listener
is doing.

### Implementing a Cache Event Listener Factory and Cache Event Listener {#Implementing-a-CacheEventListenerFactory}
A `CacheEventListenerFactory` is an abstract factory for creating
cache event listeners. Implementers should provide their own concrete
factory, extending this abstract factory. It can then be configured in ehcache.xml.
The following example demonstrates how to create an abstract `CacheEventListenerFactory`:

    /**
    * An abstract factory for creating listeners. Implementers should provide their own
    * concrete factory extending this factory. It can then be configured in ehcache.xml
    *
    */
    public abstract class CacheEventListenerFactory {
    /**
    * Create a CacheEventListener
    *
    * @param properties implementation specific properties. These are configured as comma
    *                   separated name value pairs in ehcache.xml
    * @return a constructed CacheEventListener
    */
    public abstract CacheEventListener createCacheEventListener(Properties properties);
    }

The following example demonstrates how to create a concrete implementation of the `CacheEventListener`
interface:

    /**
    * Allows implementers to register callback methods that will be executed when a cache event
    *  occurs.
    * The events include:
    * <ol>
    * <li>put Element
    * <li>update Element
    * <li>remove Element
    * <li>an Element expires, either because timeToLive or timeToIdle has been reached.
    * </ol>
    *
    * Callbacks to these methods are synchronous and unsynchronized. It is the responsibility of
    * the implementer to safely handle the potential performance and thread safety issues
    * depending on what their listener is doing.
    *
    * Events are guaranteed to be notified in the order in which they occurred.
    *
    * Cache also has putQuiet and removeQuiet methods which do not notify listeners.
    *
    */
    public interface CacheEventListener extends Cloneable {
    /**
    * Called immediately after an element has been removed. The remove method will block until
    * this method returns.
    *
    * Ehcache does not check for
    *
    * As the {@link net.sf.ehcache.Element} has been removed, only what was the key of the
    * element is known.
    *
    *
    * @param cache   the cache emitting the notification
    * @param element just deleted
    */
    void notifyElementRemoved(final Ehcache cache, final Element element) throws CacheException;
    /**
    * Called immediately after an element has been put into the cache. The
    * {@link net.sf.ehcache.Cache#put(net.sf.ehcache.Element)} method
    * will block until this method returns.
    *
    * Implementers may wish to have access to the Element's fields, including value, so the
    * element is provided. Implementers should be careful not to modify the element. The
    * effect of any modifications is undefined.
    *
    * @param cache   the cache emitting the notification
    * @param element the element which was just put into the cache.
    */
    void notifyElementPut(final Ehcache cache, final Element element) throws CacheException;
    /**
    * Called immediately after an element has been put into the cache and the element already
    * existed in the cache. This is thus an update.
    *
    * The {@link net.sf.ehcache.Cache#put(net.sf.ehcache.Element)} method
    * will block until this method returns.
    *
    * Implementers may wish to have access to the Element's fields, including value, so the
    * element is provided. Implementers should be careful not to modify the element. The
    * effect of any modifications is undefined.
    *
    * @param cache   the cache emitting the notification
    * @param element the element which was just put into the cache.
    */
    void notifyElementUpdated(final Ehcache cache, final Element element) throws CacheException;
    /**
    * Called immediately after an element is found to be expired. The
    * {@link net.sf.ehcache.Cache#remove(Object)} method will block until this method returns.
    *
    * As the {@link Element} has been expired, only what was the key of the element is known.
    *
    * Elements are checked for expiry in Ehcache at the following times:
    * <ul>
    * <li>When a get request is made
    * <li>When an element is spooled to the diskStore in accordance with a MemoryStore
    * eviction policy
    * <li>In the DiskStore when the expiry thread runs, which by default is
    * {@link net.sf.ehcache.Cache#DEFAULT_EXPIRY_THREAD_INTERVAL_SECONDS}
    * </ul>
    * If an element is found to be expired, it is deleted and this method is notified.
    *
    * @param cache   the cache emitting the notification
    * @param element the element that has just expired
    *    
    *    Deadlock Warning: expiry will often come from the DiskStore
    *    expiry thread. It holds a lock to the DiskStore at the time the
    *    notification is sent. If the implementation of this method calls into a
    *    synchronized Cache method and that subsequently calls into
    *    DiskStore a deadlock will result. Accordingly implementers of this method
    *    should not call back into Cache.
    */
    void notifyElementExpired(final Ehcache cache, final Element element);
    /**
    * Give the replicator a chance to cleanup and free resources when no longer needed
    */
    void dispose();
    /**
    * Creates a clone of this listener. This method will only be called by Ehcache before a
    * cache is initialized.
    *
    * This may not be possible for listeners after they have been initialized. Implementations
    * should throw CloneNotSupportedException if they do not support clone.
    * @return a clone
    * @throws CloneNotSupportedException if the listener could not be cloned.
    */
    public Object clone() throws CloneNotSupportedException;
    }

Two other methods are also available:

* `void notifyElementEvicted(Ehcache cache, Element element)`

    Called immediately after an element is evicted from the cache. Eviction, which happens when a cache entry is deleted from a store, should not be confused with removal, which is a result of calling `Cache.removeElement(Element)`.

* `void notifyRemoveAll(Ehcache cache)`

    Called during `Ehcache.removeAll()` to indicate that all elements have been removed from the cache in a bulk operation. The usual `notifyElementRemoved(net.sf.ehcache.Ehcache, net.sf.ehcache.Element)` is not called. Only one notification is emitted because performance considerations do not allow for serially processing notifications where potentially millions of elements have been bulk deleted.

The implementations need to be placed in the classpath accessible to Ehcache. See the page on [Classloading](/documentation/4.1/bigmemorymax/api/class-loading) for details on how the loading
of these classes will be done.


### Adding a Listener Programmatically

To add a listener programmatically, follow this example:

    cache.getCacheEventNotificationService().registerListener(myListener);

<a id="multi-listeners"></a>
### Example: Running Multiple Event Listeners on Separate Nodes
If a listener B in one node is listening for an event generated by the action of listener A on another node, it will fail to receive an event unless listener A performs the action in a different thread.

For example, if listener A detects a put into cache A and in turn puts an element into cache B, then listener B should receive an event (if it is correctly registered to cache B). However, with the following code, listener B would fail to receive the event generated by the put:

    // This method is within listener A
    public void notifyElementPut(...) {
      ...
      ...
      cache.put(...);
      ...
      ...

    }

The following code allows listener B to receive the event:

    // This method is within listener A
    public void notifyElementPut(...) {
      executorService.execute(new Runnable() {
        public void run()     
        {
         ...
         ...
         cache.put(...);
         ...
         ...
         }
      ...
      ...
      }
    ...
