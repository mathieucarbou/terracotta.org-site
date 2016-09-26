---
---
# Key Classes and Methods of the BigMemory API
{toc|2:3}


## Introduction
BigMemory currently uses Ehcache as its user-facing data access API. See the [Ehcache API documentation](http://www.ehcache.org/apidocs/2.8.5/index.html) for details.

The key Ehcache classes used are:

* CacheManager
* Cache
* Element

These classes form the core of the BigMemory API. This document
introduces these classes, along with other important components of the
BigMemory API.

When applications use BigMemory through the Ehcache API, a
CacheManager is instantiated to manage logical data sets (represented as Cache
objects in the Ehcache API&mdash;though, they may be used to store any kind of
data, not just cache data). These data sets contain name-value pairs called Elements.

The logical representations of these key components are actualized mostly
through the classes discussed below. These classes' methods provide the main
programmatic access to working with Ehcache.

## CacheManager

The class `CacheManager` is used to manage caches. Creation of, access to, and removal of caches is controlled by a named CacheManager.

### CacheManager Creation Modes {#cm-creation}

`CacheManager` supports two creation modes: singleton and instance. The two types can exist in the same JVM. However, multiple CacheManagers with the same name are not allowed to exist in the same JVM. `CacheManager()` constructors creating non-Singleton CacheManagers can violate this rule, causing a NullPointerException. If your code may create multiple CacheManagers of the same name in the same JVM, avoid this error by using the [static `CacheManager.create()` methods](http://www.ehcache.org/apidocs/2.8.5/index.html), which always return the named (or default unnamed) CacheManager if it already exists in that JVM. If the named (or default unnamed) CacheManager does not exist, the `CacheManager.create()` methods create it.

For singletons, calling `CacheManager.create(...)` returns the existing singleton CacheManager with the configured name (if it exists) or creates the singleton based on the passed-in configuration.

To work from configuration, use the `CacheManager.newInstance(...)` method, which parses the passed-in configuration to either get the existing named CacheManager or create that CacheManager if it doesn't exist.

To review, the behavior of the CacheManager creation methods is as follows:

* `CacheManager.newInstance(Configuration configuration)` &ndash; Create a new CacheManager or return the existing one named in the configuration.
* `CacheManager.create()` &ndash; Create a new singleton CacheManager with default configuration, or return the existing singleton. This is the same as `CacheManager.getInstance()`.
* `CacheManager.create(Configuration configuration)` &ndash; Create a singleton CacheManager with the passed-in configuration, or return the existing singleton.
* `new CacheManager(Configuration configuration)` &ndash; Create a new CacheManager, or throw an exception if the CacheManager named in the configuration already exists or if the parameter (configuration) is null.

Note that in instance-mode (non-singleton), where multiple CacheManagers can
be created and used concurrently in the same JVM, each CacheManager requires its own
configuration.

If the Caches under management use the disk store, the
disk-store path specified in each CacheManager configuration should be
unique. This is because when a new CacheManager is created, a check is made to ensure that no other CacheManagers are using the same disk-store path. Depending upon your persistence strategy, BigMemory Max will automatically resolve a disk-store path conflict, or it will let you know that you must explicitly configure the disk-store path.

If managed caches use only the memory store, there are no special considerations.

If a CacheManager is part of a cluster,
there will also be listener ports which must be unique.

See the [API documentation](http://ehcache.org/apidocs/2.8.4/net/sf/ehcache/CacheManager) for more information on these methods, including options for passing in configuration. For examples, see [Code Samples](/documentation/4.1/bigmemorymax/code-samples#declarative-configuration-via-xml).


## Cache
This is a thread-safe logical representation of a set of data elements, analogous to a cache region in many caching systems. Once a reference to a cache is obtained (through a CacheManager), logical actions can be performed. The physical implementation of these actions is relegated to the [stores](storage-options).

Caches are instantiated from configuration or programmatically using one of the `Cache()` constructors. Certain cache characteristics, such as ARC-related sizing, and pinning, must be set using configuration.

Cache methods can be used to get information about the cache (for example, `getCacheManager()`, `isNodeBulkLoadEnabled()`, `isSearchable()`, etc.), or perform certain cache-wide operations (for example, flush, load, initialize, dispose, etc.).

The methods provided in the `Cache` class also allow you to work with cache elements (for example, get, set, remove, replace, etc.) as well as get information about the them (for example, isExpired, isPinned, etc.).


## Element

An element is an atomic entry in a cache. It has a key, a value, and a record of
accesses. Elements are put into and removed from caches. They can also
expire and be removed by the cache, depending on the cache settings.

There is an API for Objects in addition to the one for Serializable. Non-serializable Objects can be stored only in heap. If an attempt is made to persist them,
they are discarded with a DEBUG-level log message but no error.

The APIs are identical except for the return methods from Element: `getKeyValue()` and `getObjectValue()` are used by the Object API in place of `getKey()` and `getValue()`.
