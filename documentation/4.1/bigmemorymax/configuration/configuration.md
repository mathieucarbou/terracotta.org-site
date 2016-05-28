---
---
# Configuring BigMemory Max


{toc|2:3}

## Introduction
BigMemory Max supports declarative configuration via an XML configuration file, as well as programmatic configuration via class-constructor APIs. Choosing one approach over the other can be a matter of preference or a requirement, such as when an application requires a certain runtime context to determine appropriate configuration settings.

If your project permits the separation of configuration from runtime use, there are advantages to the declarative approach:

* Cache configuration can be changed more easily at deployment time.
* Configuration can be centrally organized for greater visibility.
* Configuration lifecycle can be separated from application-code lifecycle.
* Configuration errors are checked at startup rather than causing an unexpected runtime error.
* If the configuration file is not provided, a default configuration is always loaded at runtime.

This documentation focuses on XML declarative configuration. Programmatic
configuration is explored in certain examples and is documented in
<a href="http://www.ehcache.org/apidocs/2.8.5/index.html" target="_blank">Javadocs</a>.

## XML Configuration

BigMemory Max uses Ehcache as its user-facing interface and is configured
using the Ehcache configuration system.  By default, Ehcache looks for an ASCII
or UTF8 encoded XML configuration file called `ehcache.xml` at the top level of the Java
classpath. You may specify alternate paths and filenames for the XML configuration file by
using <a href="http://www.ehcache.org/apidocs/2.8.5/index.html"
target="_blank">the various CacheManager constructors</a>.

To avoid resource conflicts, one XML configuration is required for each CacheManager that is created. For example, directory paths and listener ports require unique values. BigMemory Max will attempt to resolve conflicts, and, if one is found, it will emit a warning reminding the user to use separate configurations for multiple CacheManagers.

The sample `ehcache.xml` is included in the BigMemory Max distribution. It contains full commentary on how to configure each element. This file can also be downloaded from [http://www.ehcache.org/ehcache.xml](http://ehcache.org/ehcache.xml).

### ehcache.xsd
Ehcache configuration files must be comply with the Ehcache XML schema, `ehcache.xsd`,
which can be downloaded from [http://ehcache.org/ehcache.xsd](http://ehcache.org/ehcache.xsd).

Each BigMemory Max distribution also contains a copy of `ehcache.xsd`.

### ehcache-failsafe.xml
If the CacheManager default constructor or factory method is called, Ehcache looks for a
  file called `ehcache.xml` in the top level of the classpath. Failing that it looks for
  `ehcache-failsafe.xml` in the classpath. `ehcache-failsafe.xml` is packaged in the Ehcache JAR
  and should always be found.

`ehcache-failsafe.xml` provides an extremely simple default configuration to enable users to get started before they create their own `ehcache.xml`.

If it is used, Ehcache will emit a warning, reminding the user to set up a proper configuration.
The meaning of the elements and attributes are explained in the section on `ehcache.xml`.

    <ehcache>
      <diskStore path="java.io.tmpdir"/>
      <defaultCache
         maxEntriesLocalHeap="10000"
         eternal="false"
         timeToIdleSeconds="120"
         timeToLiveSeconds="120"
         maxEntriesLocalDisk="10000000"
         diskExpiryThreadIntervalSeconds="120"
         memoryStoreEvictionPolicy="LRU">
         <persistence strategy="localTempSwap"/>
      </defaultCache>
    </ehcache>

### About Default Cache
The `defaultCache` configuration is applied to any cache that is *not* explicitly configured. The `defaultCache` appears in `ehcache-failsafe.xml` by default, and can also be added to any BigMemory Max configuration file.

While the `defaultCache` configuration is not required, an error is generated if caches are created by name (programmatically) with no `defaultCache` loaded.


## More Information on Configuration

| Topic | Description |
|:-------|:------------|
|[Terracotta Clustering](/documentation/4.1/bigmemorymax/configuration/distributed-configuration)|BigMemory Max can manage in-memory data in a single, standalone installation or with a Terracotta server. This page provides configuration essentials for distributing BigMemory across a Terracotta cluster or server array.|
|[Storage Tiers](/documentation/4.1/bigmemorymax/configuration/storage-options)|BigMemory Max includes storage options for your in-memory data. This page discusses the storage tier options and shows how to configure them in a standalone installation. Refer to [Terracotta Server Array Architecture](/documentation/4.1/terracotta-server-array/server-arrays) for distributed configuration information.|
|[Sizing Storage Tiers](/documentation/4.1/bigmemorymax/configuration/cache-size)|Tuning BigMemory Max often involves sizing data storage tiers appropriately. BigMemory Max provides a number of ways to size tiers using simple cache-configuration sizing attributes. This page explains tuning of tier size by configuring dynamic allocation of memory and automatic load balancing.|
|[Expiration, Pinning, and Eviction](/documentation/4.1/bigmemorymax/configuration/data-life)|One of the most important aspects of managing in-memory data involves managing the life of the data in each tier. This page covers managing data life in BigMemory Max and the Terracotta Server Array, including the pinning features of Automatic Resource Control (ARC).|
|[Fast Restartability](/documentation/4.1/bigmemorymax/configuration/fast-restart)|This page covers persistence, fast restartability, and using the local disk as a storage tier. The Fast Restart feature provides enterprise-ready crash resilience, which can serve as a fast recovery system after failures, a hot mirror of the data set on the disk at the application node, and an operational store with in-memory speed for reads and writes.|
