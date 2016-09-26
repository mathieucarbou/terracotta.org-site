---
---
# Cache Extensions {#Cache-Extensions}

{toc|2:3}

## Introduction {#generic-extensions-to-a-Cache}
BigMemory Max's Ehcache implementation includes Cache extensions. Cache extensions are a general-purpose mechanism to allow generic extensions to a Cache. Cache extensions are tied into the Cache lifecycle. For that reason, this interface has the lifecycle
methods.

Cache extensions are created using the `CacheExtensionFactory`, which has a `createCacheCacheExtension()` method that takes as a parameter a
Cache and properties. It can thus call back into any public method on Cache, including, of
course, the load methods. Cache extensions are suitable for timing services, where you want to create a timer to perform cache operations. (Another way of adding Cache behavior is to decorate a cache. See [Blocking Cache](/documentation/4.1/bigmemorymax/api/constructs) for an example of how to do this.)

Because a `CacheExtension` holds a reference to a Cache, the `CacheExtension` can do things such as registering a `CacheEventListener` or even a `CacheManagerEventListener`, all from within a `CacheExtension`, creating more opportunities for customization.

## Declarative Configuration
Cache extensions are configured per cache. Each cache can have zero or more.
A `CacheExtension` is configured by adding a `cacheExceptionHandlerFactory` element as shown in the following example:

    <cache ...>
        <cacheExtensionFactory class="com.example.FileWatchingCacheRefresherExtensionFactory"
        properties="refreshIntervalMillis=18000, loaderTimeout=3000,
            flushPeriod=whatever, someOtherProperty=someValue ..."/>
    </cache>

## Implementing a Cache Extension Factory and Cache Extension {#CacheExtensionFactory}
A `CacheExtensionFactory` is an abstract factory for creating
cache extension. Implementers should provide their own concrete
factory, extending this abstract factory. It can then be configured in
ehcache.xml.
The factory class needs to be a concrete subclass of the abstract
factory class `CacheExtensionFactory`, which is reproduced below:

<pre><code>/**
* An abstract factory for creating CacheExtensions. Implementers should
* provide their own * concrete factory extending this factory. It can then be configured
* in ehcache.xml.
*
*/
public abstract class CacheExtensionFactory {
/**
* @param cache the cache this extension should hold a reference to, and to whose
* lifecycle it should be bound.
* @param properties implementation specific properties configured as delimiter separated
* name value pairs in ehcache.xml
*/
public abstract CacheExtension createCacheExtension(Ehcache cache, Properties properties);
}
</code></pre>

The factory creates a concrete implementation of the `CacheExtension`
interface, which is reproduced below:

<pre><code>/**
* This is a general purpose mechanism to allow generic extensions to a Cache.
*
* CacheExtensions are tied into the Cache lifecycle. For that reason this interface has the
*  lifecycle methods.
*
* CacheExtensions are created using the CacheExtensionFactory which has a
* <code>createCacheCacheExtension()</code> method which takes as a parameter a Cache and
* properties. It can thus call back into any public method on Cache, including, of course,
*  the load methods.
*
* CacheExtensions are suitable for timing services, where you want to create a timer to
* perform cache operations. The other way of adding Cache behaviour is to decorate a cache.
* See {@link net.sf.ehcache.constructs.blocking.BlockingCache} for an example of how to do
* this.
*
* Because a CacheExtension holds a reference to a Cache, the CacheExtension can do things
* such as registering a CacheEventListener or even a CacheManagerEventListener, all from
* within a CacheExtension, creating more opportunities for customisation.
*
*/
public interface CacheExtension {
/**
* Notifies providers to initialise themselves.
*
* This method is called during the Cache's initialise method after it has changed it's
* status to alive. Cache operations are legal in this method.
*
* @throws CacheException
*/
void init();
/**
* Providers may be doing all sorts of exotic things and need to be able to clean up on
* dispose.
*
* Cache operations are illegal when this method is called. The cache itself is partly
* disposed when this method is called.
*
* @throws CacheException
*/
void dispose() throws CacheException;
/**
* Creates a clone of this extension. This method will only be called by Ehcache before a
* cache is initialized.
*
* Implementations should throw CloneNotSupportedException if they do not support clone
* but that will stop them from being used with defaultCache.
*
* @return a clone
* @throws CloneNotSupportedException if the extension could not be cloned.
*/
public CacheExtension clone(Ehcache cache) throws CloneNotSupportedException;
/**
* @return the status of the extension
*/
public Status getStatus();
}
</code></pre>

The implementations need to be placed in the classpath accessible to ehcache.
See the page on [Classloading](/documentation/4.1/bigmemorymax/api/class-loading) for details on how class
loading of these classes will be done.

## Programmatic Configuration
Cache extensions may also be programmatically added to a Cache as shown.

<pre><code>
TestCacheExtension testCacheExtension = new TestCacheExtension(cache, ...);
testCacheExtension.init();
cache.registerCacheExtension(testCacheExtension);
</code></pre>
