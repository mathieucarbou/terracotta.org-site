---
---
# Shutting Down BigMemory {#Shutting-Down-BigMemory}

{toc|2:3}

## Introduction
BigMemory is shut down through the Ehcache API. Note that Hibernate automatically shuts down its Ehcache `CacheManager`.

The recommended way to shutdown the Ehcache is:

* to call `CacheManager.shutdown()`
* in a web app, register the Ehcache `ShutdownListener`

Note that when the CacheManager is shut down, existing cache data is removed locally but may still remain in the TSA (with distributed caches) or disk (if restartable). This is the same as occurs when only the cache is removed (`Cache.dispose()` or `CacheManager.removeCache()`). To ensure that unwanted data is not persisted, call `Cache.removeAll()` in all caches whose data is no longer wanted.

## Explicitly Removing Data
Specific entries can be removed from a cache using `Cache.remove()`. To empty the cache, `Cache.removeAll()`. If the cache itself is removed (`Cache.dispose()` or `CacheManager.removeCache()`), then any data still remaining in the cache is also removed locally. However, that remaining data is *not* removed from the TSA or disk (if restartable).


Though not recommended, Ehcache also lets you register a JVM shutdown hook.

## ServletContextListener
Ehcache provides a ServletContextListener that shuts down the CacheManager. Use this to shut down Ehcache automatically, when the web application is shut down.
To receive notification events, this class must be configured in the deployment
descriptor for the web application. To do so, add the following to web.xml in your web application:

    <listener>
      <listener-class>net.sf.ehcache.constructs.web.ShutdownListener</listener-class>
    </listener>

## The Shutdown Hook
The Ehcache CacheManager can optionally register a shutdown hook.
To do so, set the system property `net.sf.ehcache.enableShutdownHook=true`.
This will shut down the CacheManager when it detects the Virtual Machine shutting down and
it is not already shut down.

Use the shutdown hook when the CacheManager is not already being shutdown by a framework you are using, or by your application.

**Note**: Shutdown hooks are inherently problematic. The JVM is shutting down, so sometimes
things that can never be null are. Ehcache guards against as many of these as it can, but the shutdown
hook should be the last option to use.

The shutdown hook is on CacheManager. It simply calls the shutdown method.
The sequence of events is:

*   call dispose for each registered CacheManager event listener.
*   call dispose for each Cache. Each Cache will:
*   shutdown the MemoryStore. The MemoryStore will flush to the DiskStore.
*   shutdown the DiskStore. If the DiskStore is persistent ("localRestartable"), it will write the entries and index to disk.
*   shutdown each registered CacheEventListener.
*   set the Cache status to shutdown, preventing any further operations on it.
*   set the CacheManager status to shutdown, preventing any further operations on it.

The shutdown hook runs when:

* A program exists normally. For example,  `System.exit()` is called, or the last non-daemon thread exits.
* the Virtual Machine is terminated, e.g. CTRL-C. This corresponds to `kill -SIGTERM pid` or `kill -15 pid` on Unix systems.

The shutdown hook will not run when:

* the Virtual Machine aborts.
* A SIGKILL signal is sent to the Virtual Machine process on Unix
systems, e.g. `kill -SIGKILL pid` or `kill -9 pid`.
* A `TerminateProcess` call is sent to the process on Windows systems.

## Dirty Shutdown
If Ehcache is shutdown dirty, all in-memory data will be retained if BigMemory is configured for restartability. For more information, refer to [Fast Restartability](/documentation/4.1/bigmemorymax/configuration/fast-restart).
