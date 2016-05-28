---
---
# Transactions in Ehcache


{toc|2:2}


## Introduction

BigMemory Max suports Global Transactions, with "xa_strict" and "xa" modes, and Local Transactions with "local" mode.

For more discussion on these modes, and related topics, see [Working With Transactional Caches](/documentation/4.1/bigmemorymax/configuration/reference-guide#93038).


### All or nothing

If a cache is enabled for transactions, all operations on it must happen within a transaction context
otherwise a `TransactionException` will be thrown.

### Transactional Methods
The following methods require a transactional context to run:

* put()
* get()
* getQuiet()
* remove()
* getKeys()
* getSize()
* containsKey()
* removeAll()
* putWithWriter()
* removeWithWriter()
* putIfAbsent()
* removeElement()
* replace()

This list is applies to all [transactional modes](#when-to-use-transactional-modes).

All other methods work non-transactionally but can be called on a transactional cache, either within or outside of a transactional context.

### Change Visibility

The isolation level offered in BigMemory's Ehcache API is `READ_COMMITTED`. Ehcache can work as an XAResource, in which case,
full two-phase commit is supported.
Specifically:

* All mutating changes to the cache are transactional including `put`, `remove`, `putWithWriter`, `removeWithWriter` and `removeAll`.
* Mutating changes are not visible to other transactions in the local JVM or across the cluster until `COMMIT` has been called.
* Until then, reads such as by `cache.get(...)` by other transactions return the old copy. Reads do not block.


## When to use Transactional Modes

Transactional modes are a powerful extension of Ehcache allowing you to perform atomic operations on your caches and other data stores.

* "local" &mdash; When you want your changes across multiple caches to be performed atomically.
    Use this mode when you need to update your caches atomically. That is, you can have all your changes be committed or rolled back
using a straightforward API. This mode is most useful when a cache contains data calculated from other cached data.
* "xa" &mdash; Use this mode when you cache data from other data stores, such as a DBMS orJMS, and want to do it in an atomic way under the
control of the JTA API ("Java Transaction API") but without the overhead of full two-phase commit. In this mode, your cached data can
get out of sync with the other resources participating in the transactions in case of a crash. Therefore, only use this mode if you
can afford to live with stale data for a brief period of time.
* "xa_strict" &mdash; Similar to "xa" but use it only if you need strict XA disaster recovery guarantees. In this mode, the cached data can never
get out of sync with the other resources participating in the transactions, even in case of a crash. However, to get that extra safety the performance decreases significantly.



## Requirements
The objects you are going to store in your transactional cache must:

* implement `java.io.Serializable` &mdash; This is required to store cached objects when the cache is distributed in a Terracotta cluster, and it is also required by the
copy-on-read / copy-on-write mechanism used to implement isolation.
* override `equals` and `hashcode` &mdash; Those must be overridden because the transactional stores rely on element value comparison. See: `ElementValueComparator`
and the `elementValueComparator` configuration setting.

Additional restrictions:

* `transactionalMode` can only be used with `terracotta consistency="strong"`.
* Caches can be dynamically changed to bulk-load mode, but any attempt to perform a transaction when this is the case causes a `CacheException` to be thrown.
* If using "xa_strict", set synchronous writes (in `ehcache.xml`) to prevent potential loss of unwritten data on the client:

        <terracotta synchronousWrites="true" />

## Configuration
Transactions are enabled on a cache-by-cache basis with the `transactionalMode` cache attribute.
The allowed values are:

* "xa_strict"
* "xa"
* "local"
* 'off"

The default value is "off".
Enabling a cache for "xa_strict" transactions is shown in the following example:

    <cache name="xaCache"
       maxEntriesLocalHeap="500"
       eternal="false"
       timeToIdleSeconds="300"
       timeToLiveSeconds="600"
       diskExpiryThreadIntervalSeconds="1"
       transactionalMode="xa_strict">
     </cache>

### Transactional Caches with Spring
Note the following when using Spring:

* If you access the cache from an @Transactional Spring-annotated method, begin/commit/rollback statements are not required in application code because they are emitted by Spring.
* Both Spring and Ehcache need to access the transaction manager internally, and therefore you must inject your chosen transaction manager into Spring's PlatformTransactionManager as well as use an appropriate lookup strategy for Ehcache.
* The Ehcache default lookup strategy might not be able to detect your chosen transaction manager. For example, it cannot detect the WebSphere transaction manager (see [Transactions Managers](#transaction-managers)).
* Configuring a `<tx:method>` with read-only=true could be problematic with certain transaction managers such as WebSphere.

## Global Transactions
Global Transactions are supported by Ehcache. Ehcache can act as an {XAResouce} to participate in
JTA transactions under the control of a Transaction Manager. This is typically provided by your
application server. However you can also use a third party transaction manager such as Bitronix.
To use Global Transactions, select either "xa_strict" or "xa" mode. The differences are explained in the sections below.

### Implementation
Global transactions support is implemented at the Store level, through XATransactionStore and JtaLocalTransactionStore. The former decorates the underlying MemoryStore implementation, augmenting it with transaction isolation and two-phase commit support through
an &lt;XAResouce> implementation. The latter decorates a LocalTransactionStore-decorated cache to make it controllable by the standard JTA API
instead of the proprietary TransactionController API.
During its initialization, the Cache does a lookup of the TransactionManager using the provided TransactionManagerLookup implementation.
Then, using the `TransactionManagerLookup.register(XAResouce)`, the newly created XAResource is registered.
The store is automatically configured to copy every Element read from the cache or written to it. Cache is copy-on-read and copy-on-write.

## Failure Recovery
In support of the JTA specification, only *prepared* transaction data is recoverable.
Prepared data is persisted onto the cluster and locks on the memory are held. This means that non-clustered caches cannot persist
transaction data. Therefore, recovery errors after a crash might be reported by the transaction manager.

### Recovery
At any time after something went wrong, an `XAResource` might be asked to recover. Data that has been prepared might either be committed or rolled back
during recovery. XA data that has not yet been `prepared` is discarded.
The recovery guarantee differs depending on the XA mode.

#### xa Mode
With "xa", the cache doesn't get registered as an {XAResource} with the transaction manager but merely can follow the flow of a JTA
transaction by registering a JTA {Synchronization}. The cache can end up inconsistent with the other resources if there is a JVM crash
in the mutating node.
In this mode, some inconsistency might occur between a cache and other XA resources (such as databases) after a crash. However,
the cache data remains consistent because the transaction is still fully atomic on the cache itself.

#### xa_strict Mode
With "xa_strict", the cache always responds to the TransactionManager's recover calls with the list of
prepared XIDs of failed transactions. Those transaction branches can then be committed or rolled back by the transaction manager.
This mode supports the basic XA mechanism of the JTA standard.

## Sample Apps {#Sample-Apps}
We have three sample applications showing how to use XA with a variety of technologies.

### XA Sample App
This sample application uses JBoss application server. It shows an example using User managed transactions. Although most people use
JTA from within a Spring or EJB container rather than managing it themselves, this sample application is useful as a demonstration.
The following snippet from our SimpleTX servlet shows a complete transaction.

    Ehcache cache = cacheManager.getEhcache("xaCache");
    UserTransaction ut = getUserTransaction();
    printLine(servletResponse, "Hello...");
    try {
       ut.begin();
       int index = serviceWithinTx(servletResponse, cache);
       printLine(servletResponse, "Bye #" + index);
       ut.commit();
    } catch(Exception e) {
       printLine(servletResponse,
           "Caught a " + e.getClass() + "! Rolling Tx back");
       if(!printStackTrace) {
           PrintWriter s = servletResponse.getWriter();
           e.printStackTrace(s);
           s.flush();
       }
       rollbackTransaction(ut);
    }

The source code for the demo can be checked out from the [Terracotta Forge](http://svn.terracotta.org/svn/forge/projects/ehcache-jta-sample/trunk). A README.txt explains how to get the sample app going.

### XA Banking Application
This application shows a real world scenario. A Web app reads &lt;account transfer> messages from a queue
and tries to execute these account transfers.
With JTA turned on, failures are rolled back so that the cached account balance is always the same as the true balance
summed from the database.
This app is a Spring-based Java web app running in a Jetty container. It has (embedded) the following components:

* A message broker (ActiveMQ)
* 2 databases (embedded Derby XA instances)
* 2 caches (transactional Ehcache)

All XA Resources are managed by Atomikos TransactionManager. Transaction demarcation is done using Spring AOP's `@Transactional` annotation. You can run it with: `mvn clean jetty:run`. Then point your browser at: [http://localhost:9080](http://localhost:9080).
To see what happens without XA transactions: `mvn clean jetty:run -Dxa=no`

The source code for the demo can be checked out from the [Terracotta Forge](http://svn.terracotta.org/svn/forge/projects/ehcache-jta-banking/trunk). A README.txt explains how to get the sample app going.

## Transaction Managers

### Automatically Detected Transaction Managers
Ehcache automatically detects and uses the following transaction managers in the following order:

* GenericJNDI (e.g. Glassfish, JBoss, JTOM and any others that register themselves in JNDI at the standard location of java:/TransactionManager
* Weblogic (since 2.4.0)
* Bitronix
* Atomikos

No configuration is required; they work out-of-the-box.
The first found is used.

### Configuring a Transaction Manager
If your Transaction Manager is not in the list above or you want to change the priority, provide your own lookup class based on an implementation of `net.sf.ehcache.transaction.manager.TransactionManagerLookup`
and specify it in place of
the `DefaultTransactionManagerLookup` in `ehcache.xml`:

    <transactionManagerLookup
      class= "com.mycompany.transaction.manager.MyTransactionManagerLookupClass"
      properties="" propertySeparator=":"/>

Another option is to provide a different location for the JNDI lookup by passing the jndiName property to the DefaultTransactionManagerLookup.
The example below provides the proper location for the TransactionManager in GlassFish v3:

    <transactionManagerLookup
      class="net.sf.ehcache.transaction.manager.DefaultTransactionManagerLookup"
      properties="jndiName=java:appserver/TransactionManager" propertySeparator=";"/>

## Local Transactions
Local Transactions allow single phase commit across multiple cache operations, across one or more caches,
and in the same CacheManager.
This lets you apply multiple changes to a CacheManager all in your own transaction. If you also want to apply changes
to other resources, such as a database, open a transaction to them and manually handle commit and rollback
to ensure consistency.
Local transactions are not controlled by a Transaction Manager. Instead there is an explicit API where a reference is obtained
to a `TransactionController` for the CacheManager using `cacheManager.getTransactionController()` and the steps in the
transaction are called explicitly.
The steps in a local transaction are:

* `transactionController.begin()` - This marks the beginning of the local transaction on the current thread. The changes are not visible to other threads or to
 other transactions.
* `transactionController.commit()` - Commits work done in the current transaction on the calling thread.
* `transactionController.rollback()` - Rolls back work done in the current transaction on the calling thread. The changes done since begin are not applied to
 the cache.
These steps should be placed in a try-catch block which catches `TransactionException`. If any exceptions are thrown, rollback() should be
called.
Local Transactions has its own exceptions that can be thrown, which are all subclasses of `CacheException`. They are:
* `TransactionException` - a general exception
* `TransactionInterruptedException` - if Thread.interrupt() was called while the cache was processing a transaction.
* `TransactionTimeoutException` - if a cache operation or commit is called after the transaction timeout has elapsed.

### Introduction Video
Ludovic Orban, the primary author of Local Transactions, presents an [introductory video](http://vimeo.com/21299785) on Local Transactions.

### Configuration
Local transactions are configured as follows:

    <cache name="sampleCache"
       ...
       transactionalMode="local"
     </cache>

### Isolation Level
As with the other transaction modes, the isolation level is READ_COMMITTED.

### Transaction Timeouts
If a transaction cannot complete within the timeout period, a `TransactionTimeoutException` is thrown. To return the
cache to a consistent state, call `transactionController.rollback()`.
Because `TransactionController` is at the level of the CacheManager, a default timeout can be set which applies to all transactions
across all caches in a CacheManager. The default is 15 seconds.
To change the defaultTimeout:

    transactionController.setDefaultTransactionTimeout(int defaultTransactionTimeoutSeconds)

The countdown starts when `begin()` is called. You might have another local transaction on a JDBC connection and you might
be making multiple changes. If you think it might take longer than 15 seconds for an individual transaction, you can override the
default when you begin the transaction with:

    transactionController.begin(int transactionTimeoutSeconds) {

### Sample Code
This example shows a transaction which performs multiple operations across two caches.

    CacheManager cacheManager = CacheManager.getInstance();
    try {
       cacheManager.getTransactionController().begin();
        cache1.put(new Element(1, "one"));
       cache2.put(new Element(2, "two"));
       cache1.remove(4);
        cacheManager.getTransactionController().commit();
    } catch (CacheException e) {
       cacheManager.getTransactionController().rollback()
    }


## Performance

### Managing Contention
If two transactions, either standalone or across the cluster, attempt to perform a cache operation on the same element, the
following rules apply:

* The first transaction gets access
* The following transactions block on the cache operation until either the first transaction completes or the transaction timeout occurs.

Note: When an element is involved in a transaction, it is replaced with a new element with a marker that is locked, along
with the transaction ID. The normal cluster semantics are used.
Because transactions only work with consistency=strong caches, the first transaction is the thread that manages to atomically
place a soft lock on the Element. (This is done with the CAS based
putIfAbsent and replace methods.)

### What granularity of locking is used?
Ehcache uses soft locks stored in the Element itself and is on a key basis.

### Performance Comparisons
Any transactional cache adds an overhead which is significant for writes and nearly negligible for reads. Compared to `transactionalMode="off"`, the time it takes to perform writes will be noticeably slower with either "xa" or "local" specified, and "xa_strict" will be the slowest.

Accordingly, "xa_strict" should only be used where full guarantees are required, otherwise one of the other modes should be used.

## FAQ

### Why do some threads regularly time out and throw an exception?
In transactional caches, write locks are in force whenever an element is updated, deleted, or added. With concurrent access, these locks cause some threads to block and appear to deadlock. Eventually the deadlocked threads time out (and throw an exception) to avoid being stuck in a deadlock condition.

### Is IBM Websphere Transaction Manager supported?
Mostly. "xa_strict" is not supported due to each version of Websphere being a custom implementation, that is, there is no stable interface to
implement against. However, "xa", which uses TransactionManager callbacks and "local" are supported.

When using Spring, make sure your configuration is set up correctly with respect to the PlatformTransactionManager and the Websphere TM.

To confirm that Ehcache will succeed, try to manually register a `com.ibm.websphere.jtaextensions.SynchronizationCallback` in the `com.ibm.websphere.jtaextensions.ExtendedJTATransaction`. Get `java:comp/websphere/ExtendedJTATransaction` from JNDI, cast that to `com.ibm.websphere.jtaextensions.ExtendedJTATransaction` and call the `registerSynchronizationCallbackForCurrentTran` method. If you succeed, Ehcache should too.


### How do transactions interact with Write-behind and Write-through caches?
If your transactional enabled cache is being used with a writer, write operations are queued until transaction commit time. Solely
a Write-through approach would have its potential XAResource participate in the same transaction. Write-behind is supported, however it
should probably not be used with an XA transactional Cache because the operations would never be part of the same transaction. Your writer
would also be responsible for obtaining a new transaction.
Using Write-through with a non XA resource would also work, but there is no guarantee the transaction will succeed after the write
operations have been executed. On the other hand, any exception thrown during these write operations would cause the
transaction to be rolled back by having UserTransaction.commit() throw a RollbackException.

### Are Hibernate Transactions supported?
Ehcache is a "transactional" cache for Hibernate purposes. The `net.sf.ehcache.hibernate.EhCacheRegionFactory`
supports Hibernate entities configured with &lt;cache usage="transactional"/>.

### How do I make WebLogic 10 work with transactional Ehcache?
WebLogic uses an optimization that is not supported by our implementation. By default WebLogic 10 spawns threads to
start the Transaction on each XAResource in parallel. Because we need transaction work to be performed on the same Thread, you must
turn off this optimization by setting the `parallel-xa-enabled` option to `false` in your domain configuration:

    <jta>
      ...
      <checkpoint-interval-seconds>300</checkpoint-interval-seconds>
      <parallel-xa-enabled>false</parallel-xa-enabled>
      <unregister-resource-grace-period>30</unregister-resource-grace-period>
      ...
    </jta>

### How do I make Atomikos work with the Ehcache "xa" mode?
Atomikos has [a bug](http://fogbugz.atomikos.com/default.asp?community.6.802.3), which makes the "xa" mode's normal transaction termination mechanism unreliable,
There is an alternative termination mechanism built in that transaction mode that is automatically enabled when `net.sf.ehcache.transaction.xa.alternativeTerminationMode` is set to true or when Atomikos is detected as the controlling transaction manager.
This alternative termination mode has strict requirement on the way threads are used by the transaction manager and Atomikos's default settings will not work unless you configure the following property as follows:

    com.atomikos.icatch.threaded_2pc=false
