---
---
# Cache Exception Handlers


{toc|2:3}

## Introduction
BigMemory Max's Ehcache implementation includes Cache exception handlers. By default, most cache operations will propagate a runtime CacheException on failure. An interceptor,
using a dynamic proxy, may be configured so that a CacheExceptionHandler can be configured to
intercept Exceptions. Errors are not intercepted.

Caches with ExceptionHandling configured are of type `Ehcache`. To get the exception handling behavior they must
be referenced using `CacheManager.getEhcache()`, not `CacheManager.getCache()`, which returns the underlying undecorated cache.

CacheExceptionHandlers may be set either declaratively in the ehcache.xml configuration file, or programmatically.


## Declarative Configuration

Cache event listeners are configured per cache. Each cache can have
at most one exception handler.
An exception handler is configured by adding a `cacheExceptionHandlerFactory` element as shown in the following example:

    <cache ...>
     <cacheExceptionHandlerFactory
       class="net.sf.ehcache.exceptionhandler.CountingExceptionHandlerFactory"
       properties="logLevel=FINE"/>
    </cache>


## Implementing a Cache Exception Handler Factory and Cache Exception Handler

A `CacheExceptionHandlerFactory` is an abstract factory for creating
cache exception handlers. Implementers should provide their own concrete
factory, extending this abstract factory. It can then be configured in
ehcache.xml.
The factory class needs to be a concrete subclass of the abstract
factory class `CacheExceptionHandlerFactory`, which is reproduced below:

    /**
    * An abstract factory for creating <code>CacheExceptionHandler</code>s at configuration
    * time, in ehcache.xml.
    * <p/>
    * Extend to create a concrete factory
    *
    */
    public abstract class CacheExceptionHandlerFactory {
    /**
    * Create an <code>CacheExceptionHandler</code>
    *
    * @param properties implementation specific properties. These are configured as comma
    *                   separated name value pairs in ehcache.xml
    * @return a constructed CacheExceptionHandler
    */
    public abstract CacheExceptionHandler createExceptionHandler(Properties properties);
    }

The factory creates a concrete implementation of the `CacheExceptionHandler`
interface, which is reproduced below:

    /**
    * A handler which may be registered with an Ehcache, to handle exception on Cache operations.
    *
    * Handlers may be registered at configuration time in ehcache.xml, using a
    * CacheExceptionHandlerFactory, or *  set at runtime (a strategy).
    *
    * If an exception handler is registered, the default behaviour of throwing the exception
    * will not occur. The handler * method on Exception will be called. Of course, if
    * the handler decides to throw the exception, it will * propagate up through the call stack.
    * If the handler does not, it won't.
    *
    * Some common Exceptions thrown, and which therefore should be considered when implementing
    * this class are listed below:
    * <ul>
    * <li>{@link IllegalStateException} if the cache is not
    * {@link net.sf.ehcache.Status#STATUS_ALIVE}
    * <li>{@link IllegalArgumentException} if an attempt is made to put a null
    * element into a cache
    * <li>{@link net.sf.ehcache.distribution.RemoteCacheException} if an issue occurs
    *  in remote synchronous replication
    * <li>
    * <li>
    * </ul>
    *
    */
    public interface CacheExceptionHandler {
    /**
    * Called if an Exception occurs in a Cache method. This method is not called
    * if an Error occurs.
    *
    * @param Ehcache   the cache in which the Exception occurred
    * @param key       the key used in the operation, or null if the operation does not use a
    * key or the key was null
    * @param exception the exception caught
    */
    void onException(Ehcache ehcache, Object key, Exception exception);
    }

The implementations need to be placed in the classpath accessible to Ehcache.
See the page on [Classloading](/documentation/4.1/bigmemorymax/api/class-loading) for details on how classloading
of these classes will be done.

## Programmatic Configuration
The following example shows how to add exception handling to a cache, and then adding the
cache back into cache manager so that all clients obtain the cache handling decoration.

    CacheManager cacheManager = ...
    Ehcache cache = cacheManger.getCache("exampleCache");
    ExceptionHandler handler = new ExampleExceptionHandler(...);
    cache.setCacheLoader(handler);
    Ehcache proxiedCache = ExceptionHandlingDynamicCacheProxy.createProxy(cache);
    cacheManager.replaceCacheWithDecoratedCache(cache, proxiedCache);
