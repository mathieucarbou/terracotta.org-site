---
---
# BigMemory Max FAQ

This FAQ answers questions about how to use BigMemory Max and Terracotta products, integration with other products, and solving issues. If your question doesn't appear here, consider posting it on the [Terracotta forums](http://forums.terracotta.org).

Other resources for resolving issues can be found on the [Release and Compatibility Information](http://www.terracotta.org/confluence/display/release/Home) page, including:

* Release Notes &ndash; Lists features and issues for specific versions of Terracotta products.
* Compatibility Information &ndash; Includes tables on compatible versions of Terracotta products, JVMs, and application servers.


The FAQ is divided into the following sections:

{toc|2:2}

## Getting Started

{toc-zone|3:3}

### What's the difference between BigMemory Go and BigMemory Max?

BigMemory Go is for in-memory data management on a single JVM (in-process). BigMemory Max is for distributed in-memory management across an array of servers. For more on Go vs. Max, see the [BigMemory Overview](http://terracotta.org/products/bigmemory).

### What platforms does Terracotta software run on? Which application stacks does Terracotta support?
Terracotta is designed to work with as broad an array of platforms, JVMs and application server versions as possible. Supported platforms are listed in the [Release and Platform Compatibility Information](http://www.terracotta.org/confluence/display/release/Home).

### What is the Terracotta Client?
The Terracotta Client is functionality in a Java library that operates inside your JVM that enables clustering. When your JVM starts and code is called that initializes Terracotta, the Terracotta Client automatically connects to the Terracotta Server Array to engage clustering services such as the lock manager, object manager, and memory manager.

### What is the Terracotta Server Array?
The Terracotta Server Array is a set of one or more processes that coordinate data sharing among all Terracotta Clients in the cluster. Each Terracotta Server Array process is a simple Java application that runs directly on the JVM (ie without an application server or container). The Terracotta Server Array is designed to provide maximal High Availability and Scalability.

{/toc-zone}


## Configuration, Development, and Operations

{toc-zone|3:3}

<a name="persistence-mode"></a>
### How do I enable restartable mode?

In the servers section of your tc-config.xml, include the following configurations:

    <servers>
       <server host="host1" name="Server1">
           <data-backup>path/to/my/backup/directory</data-backup>
       </server>
       <restartable enabled="true"/>
    </servers>

If persistence of shared data is not required across restarts, set `<restartable enabled>` to "false".

### How do I configure failover to work properly with two Terracotta servers?

Configure both servers in the `<servers>` section of the Terracotta configuration file. Start the two Terracotta server instances that use that configuration file, one server assumes control of the cluster (the ACTIVE) and the second becomes the mirror (the PASSIVE). See the [high-availability page](/documentation/4.1/terracotta-server-array/high-availability) for more information.


### How do I know that my application has started up with a Terracotta client and is sharing data?

The [Terracotta Management Console (TMC)](/documentation/4.1/tms/tms) displays cluster topology by listing Terracotta server groups and connected client nodes in a navigation tree.

In addition, check standard output for messages that the Terracotta client has started up without errors. Terracotta clients also log messages to a file specified in the `<clients>` section of the tc-config file.

### Is there a maximum number of objects that can be held by one Terracotta server instance?
The number of objects that can be held by a Terracotta server instance is two billion, a limit imposed by the design of Java collections. It is unlikely that a Terracotta cluster will need to approach even 50 percent of that maximum. However, if it does, other issues may arise that require the rearchitecting of how application data is handled in the cluster.

### How many Terracotta clients (L1s) can connect to the Terracotta Server Array (L2s) in a cluster?
While the number of L1s that can exist in a Terracotta cluster is theoretically unbounded (and cannot be configured), effectively planning for resource limitations and the size of the shared data set should yield an optimum number. Typically, the most important factors that will impact that number are the requirements for performance and availability. Typical questions when sizing a cluster:

* What is the desired transactions-per-second?
* What are the failover scenarios?
* How fast and reliable is the network and hardware? How much memory and disk space will each machine have?
* How much shared data is going to be stored in the cluster? How much of that data should be on the L1s? Will BigMemory be used?
* How many stripes (active Terracotta servers) does the cluster have?
* How much load will there be on the L1s? On the L2s?

The most important method for determining the optimum size of a cluster is to test various cluster configurations under load and observe how well each setup meets overall requirements.

### How can my application check that the Terracotta process is alive at runtime?
Your application can check to see if the system property `tc.active` is true. For example, the following line of code would return true if Terracotta is active at the time it is run:

~~~
Boolean.getBoolean("tc.active");
~~~

### How do I confirm that my Terracotta servers are up and running correctly?

Here are some ways to confirm that your Terracotta servers are running:

* Connect to the servers using the [Terracotta Management Console](/documentation/4.1/tms/tms).
* Check the standard output messages to see that each server started without errors.
* Check each server's logs to see that each server started without errors. The location of server logs is specified in `tc-config.xml`.
* Use the Terracotta script `server-stat.sh` or `server-stat.bat` to generate a short status report on one or more Terracotta servers.
* Use a tool such as [wget](http://www.gnu.org/software/wget/) to access the /config or /version servlet. For example, for a server running on localhost and port 9889, use the following wget command to connect to the version servlet:

~~~
[PROMPT] wget http://localhost:9889/version
~~~


<a name="monitor"></a>
### Are there ways I can monitor the cluster that don't involve using the Terracotta Management Console?
You can monitor the cluster using JMX.

Cluster events are available over JMX via the object name "org.terracotta:type=TC Operator Events,name=Terracotta Operator Events Bean". Use a tool such as JConsole to view other MBeans needed for monitoring.


<a name="log-level"></a>
### How can I control the logging level for Terracotta servers and clients?
Create a file called `.tc.custom.log4j.properties` and edit it as a standard `log4j.properties` file to configure logging, including level, for the Terracotta node that loads it. This file is searched for in the path specified by the environment variable TC_INSTALL_DIR as defined in the start-server script when running a server, or if user defined when running a client.


### Why is it a bad idea to change shared data in a shutdown hook?
If a node attempts to change shared data while exiting, and the shutdown thread blocks, the node may hang and be dropped from the cluster, failing to exit as planned. The thread may block for any number of reasons, such as the failure to obtain a lock. A better alternative is to use the cluster events API to have a second node (one that is not exiting) execute certain code when it detects that the first node is exiting. If you are using Ehcache, use the cluster-events Ehcache API.

You can call `org.terracotta.api.Terracotta.registerBeforeShutdownHook(Runnable beforeShutDownHook)` to perform various cleanup tasks before the Terracotta client disconnects and shuts down.

Note that a Terracotta client is not required to release locks before shutting down. The Terracotta server will reclaim those locks, although any outstanding transactions are not committed.

### Can I store collections inside collections that are stored as values in a cache?
This is not recommended unless your application logic takes careful account of how such values can be modified safely. Race conditions, undefined results, ConcurrentModificationException, and other problems can arise.

{/toc-zone}


## Environment and Interoperability

{toc-zone|3:3}

### Where is there information on platform compatibility for my version of Terracotta software?
Information on the latest releases of Terracotta products, including a link to the latest platform support, is found on the [Product Information](http://www.terracotta.org/confluence/display/release/Home) page. This page also contains a table with links to information on previous releases.

### Can I run the Terracotta process as a Microsoft Windows service?
Yes. See [Starting up TSA or CLC as Windows Service using the Service Wrapper](/documentation/4.1/terracotta-server-array/service).

### Do you have any advice for running Terracotta software on Ubuntu?

The known issues when trying to run Terracotta software on Ubuntu are:

* Default shell is *dash* *bash*. Terracotta scripts don't behave under dash. You might solve this issue by setting your default shell to bash or changing `/bin/sh` in our scripts to `/bin/bash`.

* The Ubuntu default JDK is from GNU. Terracotta software compatibility information is on the [Product Information](http://www.terracotta.org/confluence/display/release/Home) page.

* See the [UnknownHostException](#unknownhostexception) topic below.

### Which Garbage Collector should I use with the Terracotta Server (L2) process?

The Terracotta Server performs best with the default garbage collector. This is pre-configured in the startup scripts. If you believe that Java GC is causing performance degradation in the Terracotta Server, the simplest and best way to reduce latencies by reducing collection times is to store more data in BigMemory Max.

Generally, the use of the Concurrent Mark Sweep collector (CMS) is discouraged as it is known to cause heap fragmentation for certain application-data usage patterns. Expert developers considering use of CMS should consult the Oracle tuning and best-practice documentation.


### Does Terracotta clustering work with Hibernate?
Through Ehcache, you can enable and cluster [Hibernate second-level caches](http://www.ehcache.org/documentation/2.8/integrations/hibernate.html).

### What other technologies does Terracotta software work with?
Terracotta software integrates with most popular Java technologies being used today. For a full list, contact us at [{$contact_email}](mailto:{$contact_email}).

{/toc-zone}


## Troubleshooting

{toc-zone|3:3}

### After my application interrupted a thread (or threw InterruptedException), why did the Terracotta client die?
The Terracotta client library runs with your application and is often involved in operations which your application is not necessarily aware of. These operations may get interrupted, too, which is not something the Terracotta client can anticipate. Ensure that your application does not interrput clustered threads. This is a common error that can cause the Terracotta client to shut down or go into an error state, after which it will have to be restarted.

### Why does the cluster seem to be running more slowly?
There can be many reasons for a cluster that was performing well to slow down over time. The most common reason for slowdowns is Java Garbage Collection (GC) cycles. Another reason may be that near-memory-full conditions have been reached, and the TSA needs to clear space for continued operations by additional evictions. For more information, refer to [Terracotta Server Array Architecture](/documentation/4.1/terracotta-server-array/server-arrays).

Another possible cause is when an active server is syncing with a mirror server. If the active is under substantial  load, it may be slowed by syncing process. In addition, the syncing process itself may appear to slow down. This can happen when the mirror is waiting for specific sequenced data before it can proceed, indicated by log messages similar to the following:

~~~
WARN com.tc.l2.ha.L2HACoordinator - 10 messages in pending queue.
    Message with ID 2273677 is missing still
~~~

If the message ID in the log entries changes over time, no problems are indicated by these warnings.

Another indication that slowdowns are occurring on the server and that clients are throttling their transaction commits is the appearance of the following entry in client logs:

~~~
INFO com.tc.object.tx.RemoteTransactionManagerImpl - ClientID[2](: TransactionID=[65037) :
    Took more than 1000ms to add to sequencer  : 1497 ms
~~~

### Why do all of my objects disappear when I restart the server?
If you are not running the server in restartable mode, the server will remove the object data when it restarts. If you want object data to persist across server restarts, run the server in [restartable mode](#how-do-i-enable-persistent-mode).

### Why are old objects still there when I restart the server?
If you are running the server in restartable mode, the server keeps the object data across restarts. If you want objects to disappear when you restart the server, you can either disable restartable mode or remove the data files from disk before you restart the server. See [this question](#persistence-mode).

<a id="firewall-console"></a>
<a name="firewall-cluster"></a>
### Why can't certain nodes on my Terracotta cluster see each other on the network?
A firewall may be preventing different nodes on a cluster from seeing each other. If Terracotta clients attempt to connect to a Terracotta server, for example, but the server seems to not not have any knowledge of these attempts, the clients may be blocked by a firewall. Another example is a backup Terracotta server that comes up as the active server because it is separated from the active server by a firewall.

### Client and/or server nodes are exiting regularly without reason.
Client or server processes that quit ("L1 Exiting" or "L2 Exiting" in logs) for seemingly no visible reason may have been running in a terminal session that has been terminated. The parent process must be maintained for the life of the node process, or use another workaround such as the `nohup` option.

### I have a setup with one active Terracotta server instance and a number of standbys, so why am I getting errors because more than one active server comes up?

Due to network latency or load, the Terracotta server instances may not may be have adequate time to hold an election. Increase the `<election-time>` property in the Terracotta configuration file to the lowest value that solves this issue.

If you are running on Ubuntu, see the note at the end of the [UnknownHostException](#unknownhostexception) topic below.

### I have a cluster with more than one stripe (more than one active Terracotta server) but data is distributed very unevenly between the two stripes.

The Terracotta Server Array distributes data based on the hashcode of keys. To enhance performance, each server stripe should contain approximately the same amount of data. A grossly uneven distribution of data on Terracotta servers in a cluster with more than one active server can be an indication that keys are not being hashed well. If your application is creating keys of a type that does not hash well, this may be the cause of the uneven distribution.

### Why is a crashed Terracotta server instance failing to come up when I restart it?

If running in retartable mode, the ACTIVE Terracotta server instance should come up with all shared data intact. However, if the server's database has somehow become corrupt, you must clear the crashed server's data directory before restarting.

### I lost some data after my entire cluster lost power and went down. How can I ensure that all data persists through a failure?

If only some data was lost, then Terracotta servers were configured to persist data. The cause for losing a small amount of data could be disk "write" caching on the machines running the Terracotta server instances. If every Terracotta server instance lost power when the cluster went down, data remaining in the disk cache of each machine is lost.

Turning off disk caching is not an optimal solution because the machines running Terracotta server instances will suffer a substantial performance degradation. A better solution is to ensure that power is never interrupted at any one time to every Terracotta server instance in the cluster. This can be achieved through techniques such as using uninterruptible power supplies and geographically subdividing cluster members.

### Do I have to restart Terracotta clients after redeploying in a container?
Errors could occur if a client runs with a web application that has been redeployed, causing the client to not start properly or at all. If the web application is redeployed, be sure to restart the client.

<a name="sparc"></a>
### Why does the JVM on my SPARC machines crash regularly?
You may be encountering a known issue with the Hotspot JVM for SPARC. The problem is expected to occur with Hotspot 1.6.0_08 and higher, but may have been fixed in a later version. For more information, see this [bug report](http://bugs.java.com/bugdatabase/view_bug.do?bug_id=6849574).

{/toc-zone}

## Specific Errors and Warnings

{toc-zone|3:3}

### What does the warning "WARN com.tc.bytes.TCByteBufferFactory - Asking for a large amount of memory..." mean?
If you see this warning repeatedly, objects larger than the recommended maximum are being shared in the Terracotta cluster. These objects must be sent between clients and servers. In this case, related warnings containing text similar to **`Attempt to read a byte array of len: 12251178; threshold=8000000`**
and **`Attempting to send a message (com.tc.net.protocol.delivery.OOOProtocolMessageImpl) of size`** may also appear in the logs.

If there are a large number of over-sized objects being shared, low-memory issues and degradation of performance may result.

In addition, if elements too large to fit in a client are cached, the value will be stored on the server, thus degrading performance (reads will be slower). In this case, a warning is logged.


### When starting a Terracotta server, why does it throw a **DBVersionMismatchException**?

The server is expecting a Terracotta database with a compatible version, but is finding one with non-compatible version. This usually occurs when starting a Terracotta server with an older version of the database. Note that this can only occur with servers in the restartable mode.

### Why am I getting MethodNotFound and ClassNotFound exceptions?
If you've integrated a Terracotta product with a framework such as Spring or Hibernate and are getting one of these exceptions, make sure that an older version of that Terracotta product isn't on the classpath. With Maven involved, sometimes an older version of a Terracotta product is specified in a framework's POM and ends up ahead of the current version you've specified. You can use tools such as jconsole or jvisualvm to debug, or specify `-XX:+TraceClassLoading` on the command line.

If a ClassNotFound exception is thrown at startup, check that a [supported JDK](http://www.terracotta.org/confluence/display/release/Home) (not a JRE) is installed.


### When I start a Terracotta server, why does it fail with a schema error?

You may get an error similar to the following when a Terracotta server fails to start:

~~~
 Error Message:
Starting BootJarTool...
2008-10-08 10:29:29,278 INFO - Terracotta 2.7.0, as of 20081001-101049
(Revision 10251 by cruise@rh4mo0 from 2.7)
2008-10-08 10:29:30,459 FATAL -
*******************************************************************************
The configuration data in the file at "/opt/terracotta/conf/tc-config.xml"
does not obey the Terracotta schema:
[0]: Line 8, column 3: Element not allowed: server in element servers

*******************************************************************************
~~~

This error occurs when there's a schema violation in the Terracotta configuration file, at the line indicated by the error text. To confirm that your configuration file follows the required schema, see the schema file included with the Terracotta kit. The kit includes schema files (*.xsd) for Terracotta, Ehcache, and Quartz configurations.


### The Terracotta servers crash regularly and I see a ChecksumException.

If the logs reveal an error similar to `com.sleepycat.je.log.ChecksumException: Read invalid log entry type: 0 LOG_CHECKSUM`, there is likely a corrupted disk on at least one of the servers.

<a name="unknownhostexception"></a>
### Why is `java.net.UnknownHostException` thrown when I try to run Terracotta sample applications?

If an UnknownHostException occurs, and you experience trouble running the Terracotta Welcome application and the included sample applications on Linux (especially Ubuntu), you may need to edit the etc/hosts file.

The UnknownHostException may be followed by "unknown-ip-address".

For example, your etc/hosts file may contain settings similar to the following:

~~~
127.0.0.1       localhost
127.0.1.1    myUbuntu.usa myUbuntu
~~~

If myUbuntu is the host, you must change 127.0.1.1 to the host's true IP address.

**NOTE:**
You may be able to successfully start Terracotta server instances even with the "invalid" etc/hosts file, and receive no exceptions or errors, but other connectivity problems can occur. For example, when starting two Terracotta servers that should form a mirror group (one active and one standby), you may see behavior that indicates that the servers cannot communicate with each other.


### On a node with plenty of RAM and disk space, why is there a failure with errors stating that a "native thread" cannot be created?

You may be exceeding a limit at the system level. In *NIX, run the following command to see what the limits are:

~~~
ulimit -a
~~~

For example, a limit on the number of processes that can run in the shell may be responsible for the errors.

### Why does the Terracotta server crash regularly with *java.io.IOException: File exists*?

Early versions of JDK 1.6 had a [JVM bug](http://bugs.java.com/view_bug.do?bug_id=6693490) that caused this failure. Update JDK to avoid this issue.

###What does a warning about the ChangeApplicator or ServerMapApplicator mean?
If you see warning messages such as these:

~~~
WARN com.tc.object.applicator.ChangeApplicator.com.terracotta.toolkit.collections.map.ServerMapApplicator - ServerMap received delta changes for methods other than CLEAR
WARN com.tc.object.applicator.ChangeApplicator.com.terracotta.toolkit.collections.map.ServerMapApplicator - ServerMap shouldn't normally be broadcasting changes unless its a clear/destroy, but could be a resent txn after crash
~~~


The logging is telling you that there are changes being broadcasted to clients. Normally these are not broadcasted to clients, however in the event of a failover or active restart from disk, it is possible for transactions to be broadcasted. It is safe to ignore these warnings, however it would probably be a good idea to see if there really was an active server restart or failover, in case some other action is necessary.

{/toc-zone}
