---
---
# Terracotta Toolkit Reference

This section describes functional aspects of the Terracotta Toolkit.

{toc|2:3}

<a id="rejoin"></a>
## Reconnected Client Rejoin
A Terracotta client may disconnect and be timed out (ejected) from the cluster. Typically, this occurs because of network communication interruptions lasting longer than the configured HA settings for the cluster. Other causes include long GC pauses and slowdowns introduced by other processes running on the client hardware.

With the rejoin feature, clients will follow a predictable pattern during temporary disconnections:

1. The client receives a clusterOffline event and attempts to reconnect.
2. The [reconnection timeout](/documentation/4.1/terracotta-server-array/high-availability) is reached and the TSA will no longer wait for the client (the client is ejected from the cluster).
3. The client no longer blocks the application, instead throwing RejoinException if any application threads attempt to use its data structures.
4. Upon receiving a clusterOnline event, the client reinitializes its state and joins the cluster as a new client.
5. If the rejoin fails, the client continues as before (throwing RejoinException as expected) until a new clusterOnline event is received.

Note the following about the rejoin process:

* Clients rejoin as new members and will wipe all local cached data to ensure that no pauses or inconsistencies are introduced into the cluster. Data that was pending on the rejoined client (data NOT yet acknowledged and persisted by the server) MAY BE LOST.
* Clients cannot rejoin a new cluster; if the TSA has been restarted and its data has not been persisted, the client can never rejoin and must be restarted.
* Any [nonstop-related operations](/documentation/4.1/bigmemorymax/terracotta-toolkit/toolkit-usage#nonstop) that begin (and do not complete) before the rejoin operation completes may be unsuccessful and may generate a NonStopCacheException. Other operations that begin and do not complete before the rejoin operation completes may throw other exceptions.
* Nonstop and rejoin are independent aspects of the behavior of disconnected clients, but if nonstop is in effect and set to throw an exception, only the NonStopException is thrown. RejoinException is thrown if nonstop is not in effect, or if it is set to no-op or local reads.
* If a Terracotta client with rejoin enabled is running in a JVM with clients that do not have rejoin, then only that client will rejoin after a disconnection. The remaining clients cannot rejoin and may cause the application to behave unpredictably.
* Once a client rejoins, the clusterRejoined event is fired on that client only.


## Connection Issues

Client creation can block on resolving URL at this point:

    TerracottaClient client = new TerracottaClient("myHost:9510");

If it is known that resolving "myHost" may take too long or hang, your application can should wrap client instantiation with code that provides a reasonable timeout.

A separate connection issue can occur after the server URL is resolved but while the client is attempting to connect to the server. The timeout for this type of connection can be set using the Terracotta property `l1.socket.connect.timeout` (see <a href="/documentation/4.1/terracotta-server-array/high-availability#introduction"> First-Time Client Connection</a>).


## Multiple Terracotta Clients in a Single JVM

When using the Terracotta Toolkit, you may notice that there are more Terracotta clients in the cluster than expected.


### Multiple Clients With a Single Web Application

This situation can arise whenever multiple classloaders are involved with multiple copies of the Toolkit JAR.

For example, to run a web application in Tomcat, one copy of the Toolkit JAR may need to be in the application's `WEB-INF/lib` directory while another may need to be in Tomcat's common `lib` directory to support loading of the context-level <Valve>. In this case, two Terracotta clients will be running with every Tomcat instance.


### Clients Sharing a Node ID

Clients instantiated using the same constructor (a constructor with matching parameters) in the same JVM will share the same node ID. For example, the following clients will have the same node ID:

    TerracottaClient client1 = new TerracottaClient("myHost:9510");
    TerracottaClient client2 = new TerracottaClient("myHost:9511");

Cluster events generated from client1 and client2 will appear to come from the same node. In addition, cluster topology methods may return ambiguous or useless results.

Web applications, however, can get a unique node ID even in the same JVM as long as the Terracotta Toolkit JAR is loaded by a classloader specific to the web application instead of a common classloader.
