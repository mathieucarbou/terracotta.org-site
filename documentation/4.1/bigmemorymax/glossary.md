---
---
# Key Terms and Acronyms 
{toc|2:2}

##Key Terms

#### BigMemory
Terracotta BigMemory is an in-memory data management solution for fast, reliable access to large amounts of data stored in a cache. Data management features include [Pinning, Expiration, and Eviction](/documentation/4.1/bigmemorymax/configuration/data-life), and replication across a wide-area network with [BigMemory WAN Replication Service](/documentation/4.1/wan/introduction). The client applications can be written in Java, C#.NET, or C++ - see [BigMemory Cross-Language Clients](http://terracotta.dev/documentation/4.1/cross-language/introduction).

#### BigMemory Hybrid
A feature of BigMemory Max that allows the cache to manage off-heap data across conventional memory (DRAM) and and "flash" memory solid-state drives (SDDs). See [BigMemory Hybrid](/documentation/4.1/terracotta-server-array/hybrid).

#### BigMemory SQL
[BigMemory SQL Queries](/documentation/4.1/bigmemorymax/search/bigmemory-sql) allows a user or application to fetch data form the cache with __SELECT__ statements.

#### Cache
A container of data that has been recently used and is likely to be reused soon. A cache contains elements, which are name/value pairs. Caches are physically implemented in-memory (`MemoryStore`) or on disk (`DiskStore`).

#### CacheManager
A `CacheManager`, such as Ehcache, manages the creation, access, and removal of caches.

#### CAS Operations
https://iwiki.eur.ad.sag/display/RNDTERRACOTTA/CAS+operations#CASoperations-CASOperations

#### Cluster
A set of Terracotta server instances and clients that work together to share application state or a data set. A cluster has a topology.


#### Distributed Garbage Collection (DGC)
BigMemoryMax includes the [Terracotta Distributed Garbage Collector](http://terracotta.org/documentation/terracotta-server-array/server-arrays#about-distributed-garbage-collection) (DGC), which clears objects across the entire Terracotta Server Array. DGC can be understood as "scaled out" GC.

#### Element
An cache element is key/value pair with an associated record of accesses. An element can be `put` into the cache or `removed` from the cache. See also time-to-live (TTL).

#### Garbage Collection (GC)
Java Virtual Machine garbage collection (GC) process frees up its memory by removing unneeded objects from the JVM heap, but, if the heap is large, the duration of the cleanup process interrupts the client application. These pauses can occur at unexpected times. Moving data "off-heap" into BigMemory is a way to keep the Java heap small and avoid long GC pauses.




## Garbage Collection
Java garbage collection (GC) is well-documented if sometimes poorly understood mechanism for clearing unneeded objects from JVM memory. The [Terracotta Distributed Garbage Collector](http://terracotta.org/documentation/terracotta-server-array/server-arrays#about-distributed-garbage-collection) (DGC), despite the similarity in names, is a completely different if somewhat related mechanism that clears objects from the Terracotta Server Array.

## Terracotta Active and Mirror Servers
The Terracotta Server Array, which backs Terracotta clients in a Terracotta cluster, must have at least one active server managing shared data in that cluster. Each active server can also have one or more mirror servers that act as "hot standbys" ready to take over as soon as an active fails. Together, the active and mirrors are called a *stripe*.

Typically, due to the overhead of continuous data synchronization between servers, only one or two mirrors are added to a stripe.

In older Terracotta documentation, the mirror was called the *passive* server.







#### Mirror group
A unit in the Terracotta Server Array (TSA). Sometimes also called a "stripe," a mirror group is composed of *one active* Terracotta server instance and *at least one mirror Terracotta server* instance.  The active server instance manages and persists the fraction of shared data allotted to its mirror group, while each mirror server in the mirror group replicates (or mirrors) the shared data managed by the active server. Mirror groups add capacity to the cluster. Each of the one or more mirrors is a "hot standby" that can take over the active role, if necessary.

#### Standalone cache
Either an open source Ehcache or the enterprise standalone Ehcache which has Offheap (BigMemory).

#### Stripe
See Mirror Group.

#### Terracotta Server Array
The platform, consisting of all of the Terracotta server instances in a single cluster. Clustered data, also called in-memory data, or shared data, is partitioned equally among active Terracotta server instances for management and persistence purposes.

#### Time-to-live (TTL)
The duration in which a cache element remains in the cache before expiring and being removed.

#### Terracotta Management Console (TMC)
A graphical user interface for monitoring caches.

#### Terracotta Management Server (TMS)
A single Terracotta server (L2) for BigMemory Go and BigMemory Max that is the central point of managing information from Ehcache.

#### Terracotta Server Array (TSA)
An array of L2 servers for BigMemory Max that is managed by the TMS. These servers are typically in clusters with an active node and at least one mirror node for failover.

#### Topology
The arrangement of Terracotta servers (L2s) and application clients (L1s) in the network. For details, see [Terracotta Server Array Architecture](http://terracotta.org/documentation/4.1/terracotta-server-array/server-arrays).




#### Management API
A RESTful interface defined for the monitoring and management of monitorable entities. USER DOCS: Admin Server API.

#### Managed Entity
A Terracotta application that implements a monitoring API; Ehcache CacheManager, Quartz Scheduler, Web Sessions, Terracotta Cluster.

#### Agent/Management Service
An embedded web service implementing the monitoring API that communicates directly with the monitorable entity. USER DOC: Admin REST Service.

#### Aggregator/Aggregated Management Service
A standalone web service implementation of the monitoring API that acts as to centralize interaction with a group of agents and aggregate agent responses when necessary. The aggregator will come bundled with the default web application client (RIA) in a Terracotta installation and also provided as a standalone product. USER DOC: Admin Server.

#### RIA/Management Console
A web application whose backend can speak to any monitorable entity's monitoring API. Any number of monitorable entities can be viewable at a time, much like the DevConsole allows you to see any number of clusters. USER DOC: Admin Console.

#### Admin Client API
The client version of the Admin Server API.




There exists the following management components:
Management agents, that are embedded in Ehcache and the TSA. These provide a REST interface which, while usable on their own, are meant to be accessed by the TMS.
TMS: a Java process that (1) provides a REST interface that bridges the cluster management agents and (2) serves up the TMC.
TMC: the Javascript UI that is served up by the TMS, which communicates soley with that TMS using its REST interface
Using the TMS REST interface is the recommended usage model, not only because it's required for a secured setup, but because the TMS can be located in the "DMZ" where it is externally accessible while the cluster is not.








#### Client Disconnect
An action taken by a Terracotta server to drop a Terracotta client from a cluster when the server has detected a severed connection to that client.

#### Cluster
See Terracotta Cluster.

#### Clustered Lock
A lock on a clustered object. A clustered lock allows access to data in a shared object.

#### Clustered Object
An object referenced by a Terracotta root directly or by another object referenced by the root. Clustered objects are shared across the Terracotta cluster and can be safely accessed and modified.

#### Concurrent Lock
A non-exclusive lock whose use is not communicated to the Terracotta server. A concurrent lock allows a Terracotta transaction to proceed, but there is no guarantee of correctness because two or more threads could write to an object concurrently.

#### DSO
Distributed Shared Objects (DSO) is the Terracotta component allowing an application to operate in a multi-JVM environment.


#### Greedy Lock
A lock issued to one Terracotta client based on that client's heavy usage of that lock. The Terracotta server prevents other clients from acquiring the lock. The Terracotta server can reacquire the lock if the owning client dies, or if other clients experience a negative impact from the inability to acquire the lock.


#### L1
An informal name for a Terracotta client. The terminology is borrowed from Level 1/Level 2 cache terminology in computer hardware.

#### L2
An informal name for a Terracotta server. The terminology is borrowed from Level 1/Level 2 cache terminology in computer hardware.

#### Literal
A field whose value corresponds to an actual value, as opposed to a fields whose value is a reference. Terracotta considers all non-reference fields to be literals.

#### Lock
Locks in Terracotta perform two duties: to coordinate access to critical sections of code between threads and to serve as boundaries for Terracotta transactions. Terracotta locks are analogous to synchronization in Java. >> Read more

#### Lock Upgrade
An attempt by a thread to exchange its read lock for a write lock.

#### Logically Managed Class
A class in which field changes are propagated by replaying method calls on the other members of the Terracotta cluster. These classes of objects are described as "logically managed" because Terracotta records and distributes the logical operations that were performed on them rather than changes to their internal structure. >> Read more


#### Named Lock
A lock specified in a "named lock" stanza in the Terracotta configuration A thread that attempts to execute a method that matches a named lock stanza must first acquire the lock of that name from the Terracotta server. Named locks are very coarse-grained and should only be used when autolocks are not possible. >> Read more

#### Non-Persistent Mode
A setting that forces Terracotta servers to use disk space only for swapping data. Found in the Terracotta configuration file under servers/server/dso/persistence. A Terracotta server in non-persistent mode writes data to disk on a temporary basis, and if restarted will not be restored to its pre-shutdown state.


#### On-load
A set of specific startup values and behaviors, specified in the Terracotta configuration file. For example, the on-load section can specify a method or code for a particular class included for instrumentation:
<include>
...
   <on-load>
    <execute><![CDATA[...]]></execute>
    <method>myMethod</method>
   </on-load>  
</include>

#### Persistent Mode
A data-preservation setting for Terracotta servers, found in the Terracotta configuration file under servers/server/dso/persistence. A Terracotta server in persistent mode immediately and permanently writes data to disk, and if restarted will reload that data to restore its pre-shutdown state0. If more than one Terracotta server is used in a cluster, persistent mode must be used.

#### Physically Managed Class
Physically Managed Classes are those in which Terracotta records and distributes changes to the physical structure of the object. When a field of a physically managed object changes, the new value of the field is sent to the Terracotta server and to other members of the Terracotta cluster that currently have the changed object in memory. >> Read more

#### Primitive
A primitive data type such as an int or boolean, and including primitive wrappers like Long and Integer. Terracotta roots defined as a primitive can be freely initialized and mutated. Compare to a root of reference type.

#### Read Lock
A lock set to read level, allowing access to data for read operations. A read lock is non-exclusive, so that multiple reads may occur on the same data. A read lock cannot grant read access to data which is under a write lock until the write lock is lifted.

#### Reconnect Window
A configurable interval of time during which Terracotta clients are allowed to reconnect to the Terracotta server after the server has restarted.

#### Root
A "root" is the top of a clustered object graph as declared in the Terracotta configuration. Object sharing in Terracotta starts with these declared roots. >> Read more

#### Shared Lock
A lock on a shared object, allowing safe access to that object.
Shared Object
An object shared across a Terracotta cluster.

#### Terracotta Client
A Terracotta client is a JVM that participates in a Terracotta cluster and that your application runs in. >> Read more

#### Terracotta Cluster
A Terracotta cluster consists of one or more Terracotta servers along with one or more Terracotta clients working together as if in a single JVM. The servers ensure failure recovery, maintain data coherence, and provide other services to clients.

#### Terracotta Server

The Terracotta server is the heart of a Terracotta cluster. It performs two basic functions:
1. Cluster-wide thread coordination and lock management
2. Clustered object data management and storage
>> Read more

#### Transactions
Terracotta transactions are sets of clustered object changes that must be applied atomically. Transactions are bounded by lock acquisition and release.
>> Read more

#### Transaction Boundary
The point at which a lock acquisition or release occurs. A Terracotta transaction begins at lock acquisition and ends at lock release.

#### Transient (Terracotta Transience)
Terracotta Transience is a mechanism to allow certain fields to be skipped during sharing, so that a few non-sharable objects do not prevent you from sharing other portions of an object graph. Terracotta also allows you to automatically run various methods or snippets of code when an object is loaded so that you can assign appropriate values to transient fields. >> Read more

#### Write Lock
A lock set to write level, allowing access to data for read/write operations. A write lock is exclusive, so that only one write operation can occur on the same data while the write lock is in effect. In the Terracotta configuration file, locks defined without a level are by default write locks. A write lock cannot grant write access to data which is under a read lock until the read lock is lifted.






https://confluence.terracotta.org/display/docs/Glossary








## Acronyms


TMS = the management server which connects to the cluster and servers the TMC to browsers,   TMC = the in-browser console, served-up by the TMS
