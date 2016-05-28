---
---
# Configuring Terracotta Clusters For High Availability {#19590}

{toc|2:3}


## Introduction
High Availability (HA) is an implementation designed to maintain uptime and access to services even during component overloads and failures. Terracotta clusters offer simple and scalable HA implementations based on the Terracotta Server Array (see <a href="server-arrays#67714">Terracotta Server Array Architecture</a> for more information).

The main features of a Terracotta HA architecture include:

* Instant failover using a hot standby or multiple active servers &ndash;  provides continuous uptime and services
* Configurable automatic internode monitoring &ndash;  Terracotta <a href="#85916">HealthChecker</a>
* Automatic permanent storage of all current shared (in-memory) data &ndash;  available to all server instances (no loss of application state)
* Automatic reconnection of temporarily disconnected server instances and clients &ndash;  restores hot standbys without operator intervention, allows "lost" clients to reconnect

    Client reconnection refers to reconnecting clients that have not yet been disconnected from the cluster by the Terracotta Server Array. To learn about reconnecting BigMemory Max clients that have been disconnected from their cluster, see <a href="#71266">Using Rejoin to Automatically Reconnect Terracotta Clients</a>.

<table>
<caption>TIP:  Nomenclature</caption>
<tr>
<td>
This document may refer to a Terracotta server instance as L2, and a Terracotta client (the node running your application) as L1. These are the shorthand references used in Terracotta configuration files.
</td>
</tr>
</table>


It is important to thoroughly test any High Availability setup before going to production. Suggestions for testing High Availability configurations are provided in <a href="#testing-high-availability-deployments">this section</a>.




## Basic High-Availability Configuration

A basic high-availability configuration has the following components:

* Two or More Terracotta Server Instances

    You may set up High Availability using either `<server>` or `<mirror-group>` configurations. Note that `<server>` instances do work together as a mirror group, but to create more than one stripe, or to configure the election-time, use `<mirror-group>` instances. See <a href="server-arrays#67714">Terracotta Server Arrays Architecture</a> on how to set up a cluster with multiple Terracotta server instances.

* Server-Server Reconnection

    A reconnection mechanism can be enabled to restore lost connections between active and mirror Terracotta server instances. See <a href="#41216">Automatic Server Instance Reconnect</a> for more information.

* Server-Client Reconnection

    A reconnection mechanism can be enabled to restore lost connections between Terracotta clients and server instances. See <a href="#77332">Automatic Client Reconnect</a> for more information.


## High-Availability Features

The following high-availability features can be used to extend the reliability of a Terracotta cluster. These features are controlled using properties set with the &lt;tc-properties&gt; section at the *beginning* of the Terracotta configuration file:

    <tc:tc-config xmlns:tc="http://www.terracotta.org/config"  
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"  
    xsi:schemaLocation="http://www.terracotta.org/schema/terracotta-9.xsd">

     <tc-properties>
       <property name="some.property.name" value="true"/>
       <property name="some.other.property.name" value="true"/>
       <property name="still.another.property.name" value="1024"/>
     </tc-properties>

    <!-- The rest of the Terracotta configuration goes here. -->

    </tc:tc-config>

See the section *Overriding tc.properties* in [Configuration Guide and Reference](/documentation/4.1/terracotta-server-array/config-reference#overriding-tcproperties) for more information.


### HealthChecker {#85916}

HealthChecker is a connection monitor similar to TCP keep-alive. HealthChecker functions between Terracotta server instances (in High Availability environments), and between Terracotta sever instances and clients. Using HealthChecker, Terracotta nodes can determine if peer nodes are reachable, up, or in a GC operation. If a peer node is unreachable or down, a Terracotta node using HealthChecker can take corrective action. HealthChecker is on by default.

You configure HealthChecker using certain Terracotta properties, which are grouped into three different categories:

* Terracotta server instance -> Terracotta client
* Terracotta Server -> Terracotta Server (HA setup only)
* Terracotta Client -> Terracotta Server

Property category is indicated by the prefix:

* l2.healthcheck.l1 indicates L2 -> L1
* l2.healthcheck.l2 indicates L2 -> L2
* l1.healthcheck.l2 indicates L1 -> L2

For example, the `l2.healthcheck.l2.ping.enabled` property applies to L2 -> L2.

The following HealthChecker properties can be set in the &lt;tc-properties> section of the Terracotta configuration file:

<table>
<tr>
<th>Property</th>
<th>Definition</th>
</tr>
<tr>
<td>
<code>l2.healthcheck.l1.ping.enabled</code>

<code>l2.healthcheck.l2.ping.enabled</code>

<code>l1.healthcheck.l2.ping.enabled</code>
</td>
<td>
Enables (True) or disables (False) ping probes (tests). Ping probes are high-level attempts to gauge the ability of a remote node to respond to requests and is useful for determining if temporary inactivity or problems are responsible for the node's silence. Ping probes may fail due to long GC cycles on the remote node.
</td>
</tr>
<tr>
<td>
<code>l2.healthcheck.l1.ping.idletime</code>

<code>l2.healthcheck.l2.ping.idletime</code>

<code>l1.healthcheck.l2.ping.idletime</code>
</td>
<td>
The maximum time (in milliseconds) that a node can be silent (have no network traffic) before HealthChecker begins a ping probe to determine if the node is alive.
</td>
</tr>
<tr>
<td>
<code>l2.healthcheck.l1.ping.interval</code>

<code>l2.healthcheck.l2.ping.interval</code>

<code>l1.healthcheck.l2.ping.interval</code>
</td>
<td>
If no response is received to a ping probe, the time (in milliseconds) that HealthChecker waits between retries.
</td>
</tr>
<tr>
<td>
<code>l2.healthcheck.l1.ping.probes</code>

<code>l2.healthcheck.l2.ping.probes</code>

<code>l1.healthcheck.l2.ping.probes</code>
</td>
<td>
If no response is received to a ping probe, the maximum number (integer) of retries HealthChecker can attempt.
</td>
</tr>
<tr>
<td>
<code>l2.healthcheck.l1.socketConnect</code>

<code>l2.healthcheck.l2.socketConnect</code>

<code>l1.healthcheck.l2.socketConnect</code>
</td>
<td>
Enables (True) or disables (False) socket-connection tests. This is a low-level connection that determines if the remote node is reachable and can access the network. Socket connections are not affected by GC cycles.
</td>
</tr>
<tr>
<td>
<code>l2.healthcheck.l1.socketConnectTimeout</code>

<code>l2.healthcheck.l2.socketConnectTimeout</code>

<code>l1.healthcheck.l2.socketConnectTimeout</code>
</td>
<td>
A multiplier (integer) to determine the maximum amount of time that a remote node has to respond before HealthChecker concludes that the node is dead (regardless of previous successful socket connections). The time is determined by multiplying the value in ping.interval by this value.
</td>
</tr>
<tr>
<td>
<code>l2.healthcheck.l1.socketConnectCount</code>

<code>l2.healthcheck.l2.socketConnectCount</code>

<code>l1.healthcheck.l2.socketConnectCount</code>
</td>
<td>
The maximum number (integer) of successful socket connections that can be made without a successful ping probe. If this limit is exceeded, HealthChecker concludes that the target node is dead.
</td>
</tr>
<tr>
<td>
<code>l1.healthcheck.l2.bindAddress</code>
</td>
<td>
Binds the client to the configured IP address. This is useful where a host has more than one IP address available for a client to use. The default value of "0.0.0.0" allows the system to assign an IP address.
</td>
</tr>
<tr>
<td>
<code>l1.healthcheck.l2.bindPort</code>
</td>
<td>
Set the client's callback port. Terracotta configuration does not assign clients a port for listening to cluster communications such as that required by HealthChecker. The default value of "0" allows the system to assign a port. To specify a port number or a range of ports, you can provide a port number as the value, or a comma separated list of ports where each element of the list can either be a single number or range. A value of "-1" disables a client's callback port.
</td>
</tr>
</table>

The following diagram illustrates how HealthChecker functions.

<img src="/images/documentation/HealthCheckerFlow.png" alt="Terracotta HealthChecker flow diagram." />


#### Calculating HealthChecker Maximum

The following formula can help you compute the maximum time it will take HealthChecker to discover failed or disconnected remote nodes:

    Max Time = (ping.idletime) + socketConnectCount * [(ping.interval * ping.probes)
               + (socketConnectTimeout * ping.interval)]

Note the following about the formula:

* The response time to a socket-connection attempt is less than or equal to `(socketConnectTimeout * ping.interval)`. For calculating the worst-case scenario (absolute maximum time), the equality is used. In most real-world situations the socket-connect response time is likely to be close to 0 and the formula can be simplified to the following:


        Max Time = (ping.idletime) + [socketConnectCount * (ping.interval * ping.probes)]

* `ping.idletime`, the trigger for the full HealthChecker process, is counted once since it is in effect only once each time the process is triggered.
* `socketConnectCount` is a multiplier that is in incremented as long as a positive response is received for each socket connection attempt.
* The formula yields an ideal value, since slight variations in actual times can occur.
* To prevent clients from disconnecting too quickly in a situation where an active server is temporarily disconnected from both the backup server and those clients, ensure that the Max Time for L1->L2 is approximately 8-12 seconds longer than for L2->L2. If the values are too close together, then in certain situations the active server may kill the backup and refuse to allow clients to reconnect.


#### Configuration Examples

The configuration examples in this section show settings for L1 -> L2 HealthChecker. However, they apply in the similarly to L2 -> L2 and L2 -> L1, which means that the server is using HealthChecker on the client.


##### Aggressive

The following settings create an aggressive HealthChecker with low tolerance for short network outages or long GC cycles:

    <property name="l1.healthcheck.l2.ping.enabled" value="true" />
    <property name="l1.healthcheck.l2.ping.idletime" value="2000" />
    <property name="l1.healthcheck.l2.ping.interval" value="1000" />
    <property name="l1.healthcheck.l2.ping.probes" value="3" />
    <property name="l1.healthcheck.l2.socketConnect" value="true" />
    <property name="l1.healthcheck.l2.socketConnectTimeout" value="2" />
    <property name="l1.healthcheck.l2.socketConnectCount" value="5" />

According to the HealthChecker "Max Time" formula, the maximum time before a remote node is considered to be lost is computed in the following way:

    2000 + 5 [( 3 * 1000 ) + ( 2 * 1000)] = 27000

In this case, after the initial idletime of 2 seconds, the remote failed to respond to ping probes but responded to every socket connection attempt, indicating that the node is reachable but not functional (within the allowed time frame) or in a long GC cycle. This aggressive HealthChecker configuration declares a node dead in no more than 27 seconds.

If at some point the node had been completely unreachable (a socket connection attempt failed), HealthChecker would have declared it dead sooner. Where, for example, the problem is a disconnected network cable, the "Max Time" is likely to be even shorter:

    2000 + 1[3 * 1000) + ( 2 * 1000 ) = 7000

In this case, HealthChecker declares a node dead in no more than 7 seconds.


##### Tolerant

The following settings create a HealthChecker with a higher tolerance for interruptions in network communications and long GC cycles:

    <property name="l1.healthcheck.l2.ping.enabled" value="true" />
    <property name="l1.healthcheck.l2.ping.idletime" value="5000" />
    <property name="l1.healthcheck.l2.ping.interval" value="1000" />
    <property name="l1.healthcheck.l2.ping.probes" value="3" />
    <property name="l1.healthcheck.l2.socketConnect" value="true" />
    <property name="l1.healthcheck.l2.socketConnectTimeout" value="5" />
    <property name="l1.healthcheck.l2.socketConnectCount" value="10" />

According to the HealthChecker "Max Time" formula, the maximum time before a remote node is considered to be lost is computed in the following way:

    5000 + 10 [( 3 x 1000 ) + ( 5 x 1000)] = 85000

In this case, after the initial idletime of 5 seconds, the remote failed to respond to ping probes but responded to every socket connection attempt, indicating that the node is reachable but not functional (within the allowed time frame) or excessively long GC cycle. This tolerant HealthChecker configuration declares a node dead in no more than 85 seconds.

If at some point the node had been completely unreachable (a socket connection attempt failed), HealthChecker would have declared it dead sooner. Where, for example, the problem is a disconnected network cable, the "Max Time" is likely to be even shorter:

    5000 + 1[3 * 1000) + ( 5 * 1000 )] = 13000

In this case, HealthChecker declares a node dead in no more than 13 seconds.

#### Tuning HealthChecker to Allow for GC or Network Interruptions

GC cycles do not affect a node's ability to respond to socket-connection requests, while network interruptions do. This difference can be used to tune HealthChecker to work more efficiently in a cluster where one or the other of these issues is more likely to occur:

* To favor detection of network interruptions, adjust the socketConnectCount down (since socket connections will fail). This will allow HealthChecker to disconnect a client sooner due to network issues.

* To favor detection of GC pauses, adjust the socketConnectCount up (since socket connections will succeed). This will allow clients to remain in the cluster longer when no network disconnection has occurred.

The ping interval increases the time before socket-connection attempts kick in to verify health of a remote node. The ping interval can be adjusted up or down to add more or less tolerance in either of the situations listed above.

### Automatic Server Instance Reconnect {#41216}

An automatic reconnect mechanism can prevent short network disruptions from forcing a restart for any Terracotta server instances in a server array with hot standbys. If not disabled, this mechanism is by default in effect in clusters set to networked-based HA mode.

<table>
<caption>NOTE:  Increased Time-to-Failover</caption>
<tr>
<td>
This feature increases time-to-failover by the timeout value set for the automatic reconnect mechanism.
</td>
</tr>
</table>

This event-based reconnection mechanism works independently and exclusively of HealthChecker. If HealthChecker has already been triggered, this mechanism cannot be triggered for the same node. If this mechanism is triggered first by an internal Terracotta event, HealthChecker is prevented from being triggered for the same node. The events that can trigger this mechanism are not exposed by API but are logged.

Configure the following properties for the reconnect mechanism:

* `l2.nha.tcgroupcomm.reconnect.enabled` &ndash;  (DEFAULT: false) When set to "true" enables a server instance to attempt reconnection with its peer server instance after a disconnection is detected. Most use cases should benefit from enabling this setting.
* `l2.nha.tcgroupcomm.reconnect.timeout` &ndash;  Enabled if `l2.nha.tcgroupcomm.reconnect.enabled` is set to true. Specifies the timeout (in milliseconds) for reconnection. Default: 2000. This parameter can be tuned to handle longer network disruptions.


### Automatic Client Reconnect {#77332}

Clients disconnected from a Terracotta cluster normally require a restart to reconnect to the cluster. An automatic reconnect mechanism can prevent short network disruptions from forcing a restart for Terracotta clients disconnected from a Terracotta cluster.

<table>
<caption>NOTE:  Performance Impact of Using Automatic Client Reconnect</caption>
<tr>
<td>
With this feature, clients waiting to reconnect continue to hold locks. Some application threads may block while waiting to for the client to reconnect.
</td>
</tr>
</table>

This event-based reconnection mechanism works independently and exclusively of HealthChecker. If HealthChecker has already been triggered, this mechanism cannot be triggered for the same node. If this mechanism is triggered first by an internal Terracotta event, HealthChecker is prevented from being triggered for the same node. The events that can trigger this mechanism are not exposed by API but are logged.

Configure the following properties for the reconnect mechanism:

* `l2.l1reconnect.enabled` &ndash; (DEFAULT: false) When set to "true" enables a client to reconnect to a cluster after a disconnection is detected. This property controls a server instance's reaction to such an attempt. It is set on the server instance and is passed to clients by the server instance. A client cannot override the server instance's setting. If a mismatch exists between the client setting and a server instance's setting, and the client attempts to reconnect to the cluster, the client emits a mismatch error and exits. Most use cases should benefit from enabling this setting.
* `l2.l1reconnect.timeout.millis` &ndash;  Enabled if `l2.l1reconnect.enabled` is set to true. Specifies the timeout (in milliseconds) for reconnection. This property controls a server instance's timeout during such an attempt. It is set on the server instance and is passed to clients by the server instance. A client cannot override the server instance's setting. Default: 2000. This parameter can be tuned to handle longer network disruptions.


### Special Client Connection Properties

Client connections can also be tuned for the following special cases:

* Client failover after server failure
* First-time client connection

The connection properties associated with these cases are already optimized for most typical environments. If you attempt to tune these properties, be sure to thoroughly test the new settings.


#### Client Failover After Server Failure

When an active Terracotta server instance fails, and a mirror Terracotta server is available, the mirror server becomes active. Terracotta clients connected to the previous active server automatically switch to the new active server. However, these clients have a limited window of time to complete the failover.

<table>
<caption>TIP: Clusters with a Single Server</caption>
<tr>
<td>
This reconnection window also applies in a cluster with a single Terracotta server that is restarted. However, a single-server cluster must have &lt;restartable&gt; enabled for the reconnection window to take effect.
</td>
</tr>
</table>

This window is configured in the Terracotta configuration file using the &lt;client-reconnect-window&gt; element:

    <servers>
      ...
      <client-reconnect-window>120</client-reconnect-window>
      <!-- The reconnect window is configured in seconds, with a default value of 120.
           The default value is "built in," so the element does not have to be explicitly
           added unless a different value is required. -->
      ...
    </servers>

Clients that fail to connect to the new active server must be restarted if they are to successfully reconnect to the cluster.


#### First-Time Client Connection {#98653}

When a Terracotta client is first started (or restarted), it attempts to connect to a Terracotta server instance based on the following properties:

    # Must the client and server be running the same version of Terracotta?
    l1.connect.versionMatchCheck.enabled = true
    # Time (in milliseconds) before a socket connection attempt is timed out.
    l1.socket.connect.timeout=10000
    # Time (in milliseconds; minimum 10) between attempts to connect to a server.
    l1.socket.reconnect.waitInterval=1000

The timing properties above control connection attempts only *after* the client has obtained and resolved its configuration from the server. To control connection attempts *before* configuration is resolved, set the following property on the client:

~~~
-Dcom.tc.tc.config.total.timeout=5000
~~~

This property limits the time (in milliseconds) that a client spends attempting to make an initial connection.

### Using Rejoin to Automatically Reconnect Terracotta Clients {#71266}

A Terracotta client may disconnect and be timed out (ejected) from the cluster. Typically, this occurs because of network communication interruptions lasting longer than the configured HA settings for the cluster. Other causes include long GC pauses and slowdowns introduced by other processes running on the client hardware.

You can configure clients to automatically rejoin a cluster after they are ejected. If the ejected client continues to run under nonstop cache settings, and then senses that it has reconnected to the cluster (receives a clusterOnline event), it can begin the rejoin process.

Note the following about using the rejoin feature:

* Disconnected clients can only rejoin clusters to which they were previously connected.
* Clients rejoin as new members and will wipe all cached data to ensure that no pauses or inconsistencies are introduced into the cluster.
* Clients cannot rejoin a new cluster; if the TSA has been restarted and its data has not been persisted, clients can never rejoin and must be restarted.
* If the TSA has been restarted and its data has been persisted, clients are allowed to rejoin.
* Any nonstop-related operations that begin (and do not complete) before the rejoin operation completes may be unsuccessful and may generate a NonStopCacheException.
* If a Terracotta client with rejoin enabled is running in a JVM with clients that do not have rejoin, then only that client will rejoin after a disconnection. The remaining clients cannot rejoin and may cause the application to behave unpredictably.
* Once a client rejoins, the clusterRejoined event is fired on that client only.


#### Configuring Rejoin

The rejoin feature is disabled by default. To enable the rejoin feature in an Terracotta client, follow these steps:

1. Ensure that all of the caches in the Ehcache configuration file where rejoin is enabled have nonstop enabled.
2. Ensure that your application does not create caches on the client without nonstop enabled.
3. Enable the rejoin attribute in the client's `<terracottaConfig>` element:

        <terracottaConfig url="myHost:9510" rejoin="true" />

For more options on configuring `<terracottaConfig>`, see the [configuration reference](/documentation/4.1/bigmemorymax/configuration/distributed-configuration#35085).


#### Exception During Rejoin

Under certain circumstances, if a lock is being used by your application, an InvalidLockAfterRejoinException could be thrown during or after client rejoin. This exception occurs when an unlock operation takes place on a lock obtained *before* the rejoin attempt completed.

To ensure that locks are released properly, application code should encapsulate lock-unlock operations with try-finally blocks:

    myLock.acquireLock();
    try {
      // Do some work.
    } finally {
      myLock.unlock();
    }


## Effective Client-Server Reconnection Settings: An Example

To prevent unwanted disconnections, it is important to understand the potentially complex interaction between HA settings and the environment in which your cluster runs. Settings that are not appropriate for a particular environment can lead to unwanted disconnections under certain circumstances.

In general, it is advisable to maintain an L1-L2 HealthChecker timeout that falls between the L2-L2 HealthChecker timeout as modified in the following ineqaulity:

~~~
L2-L2 HealthCheck + Election Time
        <  L1-L2 HealthCheck
                  <  L2-L2 HealthCheck + Election Time + Client Reconnect Window
~~~

This allows a cluster's L1s to avoid disconnecting before a client reconnection window is opened (a backup L2 takes over), or to not disconnect if that window is never opened (the original active L2 is still functional). The Election Time and Client Reconnect Window settings, which are found in the Terracotta configuration file, are respectively 5 seconds and 120 seconds by default.

For example, in a cluster where the L2-L2 HealthChecker triggers at 55 seconds, a backup L2 can take over the cluster after 180 seconds (55 + 5 + 120). If the L1-L2 HealthChecker triggers after a time that is greater than 180 seconds, clients may not attempt to reconnect until the reconnect window is closed and it's too late.

If the L1-L2 HealthChecker triggers after a time that is less than 60 seconds (L2-L2 HealthChecker + Election Time), then the clients may disconnect from the active L2 before its failure is determined. Should the active L2 win the election, the disconnected L1s would then be lost.

A check is performed at server startup to ensure that L1-L2 HealthChecker settings are within the effective range. If not, a warning with a prescription is printed.


## Testing High-Availability Deployments
This section presents recommendations for designing and testing a robust cluster architecture. While these recommendations have been tested and shown to be effective under certain circumstances, in your custom environment additional testing is still necessary to ensure an optimal setup, and to meet specific demands related to factors such as load.

### High-Availability Network Architecture And Testing

To take advantage of the Terracotta active-mirror server configuration, certain network configurations are necessary to prevent split-brain scenarios and ensure that Terracotta clients (L1s) and server instances (L2s) behave in a deterministic manner after a failure occurs. This is regardless of the nature of the failure, whether network, machine, or other type.

This section outlines two possible network configurations that are known to work with Terracotta failover.  While it is possible for other network configurations to work reliably, the configurations listed in this document have been well tested and are fully supported.

### _Deployment Configuration_: Simple (no network redundancy)
![IMAGE: Simple Network Setup](/images/documentation/TC.single.NIC.png)

#### Description
This is the simplest network configuration.  There is no network redundancy so when any failure occurs, there is a good chance that all or part of the cluster will stop functioning.  All failover activity is up to the Terracotta software.

In this diagram, the IP addresses are merely examples to demonstrate that the L1s (L1a & L1b) and L2s (TCserverA & TCserverB) can live on different subnets. The actual addressing scheme is specific to your environment. The single switch is a single point of failure.

#### Additional configuration

There is no additional network or operating-system configuration necessary in this configuration. Each machine needs a proper network configuration (IP address, subnet mask, gateway, DNS, NTP, hostname) and must be plugged into the network.

#### Test Plan - Network Failures Non-Redundant Network

To determine that your configuration is correct, use the following tests to confirm all failure scenarios behave as expected.

<table>
<tr><th>TestID</th><th> Failure </th><th>  Expected Outcome  </th></tr>
<tr><td>FS1</td><td> Loss of L1a (link or system)</td><td> Cluster continues as normal using only L1b </td></tr>
<tr><td>FS2</td><td> Loss of L1b (link or system)</td><td> Cluster continues as normal using only L1a </td></tr>
<tr><td>FS3</td><td> Loss of L1a & L1b </td><td> Non-functioning cluster </td></tr>
<tr><td>FS4</td><td> Loss of Switch    </td><td> Non-functioning cluster </td></tr>
<tr><td>FS5</td><td> Loss of Active L2 (link or system) </td><td> mirror L2 becomes new Active L2, L1s fail over to new Active L2 </td></tr>
<tr><td>FS6</td><td> Loss of mirror L2 </td><td> Cluster continues as normal without TC redundancy </td></tr>
<tr><td>FS7</td><td> Loss of TCservers A & B </td><td> Non-functioning cluster </td></tr>
</table>

#### Test Plan - Network Tests Non-redundant Network

After the network has been configured, you can test your configuration with simple ping tests.

<table>
<tr><th>TestID</th><th> Host </th><th> Action </th><th> Expected Outcome </th></tr>
<tr><td>NT1</td><td> all  </td><td> ping every other host </td><td> successful ping </td></tr>
<tr><td>NT2</td><td> all  </td><td> pull network cable during continuous ping </td><td> ping failure until link restored </td></tr>
<tr><td>NT3</td><td> switch </td><td> reload </td><td> all pings cease until reload complete and links restored </td></tr>
</table>

### _Deployment Configuration_: Fully Redundant

![IMAGE:Fully Redundant Deployment Configuration](/images/documentation/TC.redundant.NIC.png)

#### Description

This is the fully redundant network configuration. It relies on the failover capabilities of Terracotta, the switches, and the operating system.  In this scenario it is even possible to sustain certain double failures and still maintain a fully functioning cluster.

In this diagram, the IP addressing scheme is merely to demonstrate that the L1s (L1a & L1b) can be on a different subnet than the L2s (TCserverA & TCserverB).  The actual addressing scheme will be specific to your environment.  If you choose to implement with a single subnet, then there will be no need for VRRP/HSRP but you will still need to configure a single VLAN (can be VLAN 1) for all TC cluster machines.

In this diagram, there are two switches that are connected with trunked links for redundancy and which implement Virtual Router Redundancy Protocol (VRRP) or HSRP to provide redundant network paths to the cluster servers in the event of a switch failure.  Additionally, all servers are configured with both a primary and secondary network link which is controlled by the operating system.  In the event of a NIC or link failure on any single link, the operating system should fail over to the backup link without disturbing (e.g. restarting) the Java processes (L1 or L2) on the systems.

The Terracotta fail over is identical to that in the simple case above, however both NIC cards on a single host would need to fail in this scenario before the TC software initiates any fail over of its own.

#### Additional configuration

* Switch - Switches need to implement VRRP or HSRP to provide redundant gateways for each subnet.  Switches also need to have a trunked connection of two or more lines  in order to prevent any single link failure from splitting the virtual router in two.

* Operating System - Hosts need to be configured with bonded network interfaces connected to the two different switches.  For Linux, choose mode 1.  More information about Linux channel bonding can be found in the [RedHat Linux Reference Guide](https://www.redhat.com/docs/manuals/enterprise/RHEL-4-Manual/en-US/Reference_Guide/s2-modules-bonding.html). Pay special attention to the amount of time it takes for your VRRP or HSRP implementation to reconverge after a recovery.  You don't want your NICs to change to a switch that is not ready to pass traffic.  This should be tunable in your bonding configuration.


#### Test Plan - Network Failures Redundant Network

The following tests continue the tests listed in Network Failures (Pt. 1).  Use these tests to confirm that your network is configured properly.

<table>
<tr><th>TestID</th><th>Failure </th><th> Expected Outcome </th></tr>
<tr><td>FS8</td><td> Loss of any primary network link </td><td> Failover to standby link </td></tr>
<tr><td>FS9</td><td> Loss of all primary links </td><td> All nodes fail to their secondary link</td></tr>
<tr><td>FS10</td><td> Loss of any switch </td><td> Remaining switch assumes VRRP address and switches fail over NICs if necessary </td></tr>
<tr><td>FS11</td><td> Loss of any L1 (both links or system) </td><td> Cluster continues as normal using only other L1 </td></tr>
<tr><td>FS12</td><td> Loss of Active L2 </td><td> mirror L2 becomes the new Active L2, All L1s fail over to the new Active L2 </td></tr>
<tr><td>FS13</td><td> Loss of mirror L2 </td><td> Cluster continues as normal without TC redundancy </td></tr>
<tr><td>FS14</td><td> Loss of both switches </td><td> non-functioning cluster </td></tr>
<tr><td>FS15</td><td> Loss of single link in switch trunk </td><td> Cluster continues as normal without trunk redundancy </td></tr>
<tr><td>FS16</td><td> Loss of both trunk links </td><td> possible non-functioning cluster depending on VRRP or HSRP implementation </td></tr>
<tr><td>FS17</td><td> Loss of both L1s </td><td> non-functioning cluster </td></tr>
<tr><td>FS18</td><td> Loss of both L2s </td><td> non-functioning cluster </td></tr>
</table>

#### Test Plan - Network Testing Redundant Network

After the network has been configured, you can test your configuration with simple ping tests and various failure scenarios.

The test plan for Network Testing consists of the following tests:

<table>
<tr><th>TestID</th><th> Host </th><th> Action </th><th> Expected Outcome </th></tr>
<tr><td>NT4</td><td> any </td><td> ping every other host </td><td> successful ping </td></tr>
<tr><td>NT5</td><td> any </td><td> pull primary link during continuous ping to any other host </td><td> failover to secondary link, no noticable network interruption </td></tr>
<tr><td>NT6</td><td> any </td><td> pull standby link during continuous ping to any other host </td><td> no effect </td></tr>
<tr><td>NT7</td><td> Active L2 </td><td> pull both network links </td><td> mirror L2 becomes Active, L1s fail over to new Active L2 </td></tr>
<tr><td>NT8</td><td> Mirror L2 </td><td> pull both network links </td><td> no effect </td></tr>
<tr><td>NT9</td><td> switchA </td><td> reload </td><td> nodes detect link down and fail to standby link, brief network outage if VRRP transition occurs </td></tr>
<tr><td>NT10</td><td> switchB </td><td> reload </td><td>  brief network outage if VRRP transition occurs </td></tr>
<tr><td>NT11</td><td> switch </td><td> pull single trunk link </td><td> no effect </td></tr>
</table>

### Terracotta Cluster Tests

All tests in this section should be run after the Network Tests succeed.

#### Test Plan - Active L2 System Loss Tests - verify Mirror Takeover

The test plan for mirror takeover consists of the following tests:

<table>
<tr><th>TestID</th><th> Test </th><th> Setup </th><th> Steps </th><th> Expected Result </th></tr>
<tr><td>TAL1</td><td>Active L2 Loss - Kill </td><td>L2-A is active, L2-B is mirror. All systems are running and available to take traffic.</td><td>1. Run app<br>2. Kill -9 Terracotta PID on L2-A (Active)</td><td>L2-B(mirror) becomes active.  Takes the load. No drop in TPS on Failover.</td></tr>
<tr><td>TAL2</td><td>Active L2 Loss - clean shutdown</td><td> L2-A is active, L2-B is mirror. All systems are running and available to take traffic.</td><td>1. Run app   2.Run ~/bin/stop-tc-server.sh on L2-A (Active)</td><td>L2-B(mirror) becomes active.  Takes the load. No drop in TPS on Failover. </td></tr>
<tr><td>TAL3</td><td>Active L2 Loss - Power Down         </td><td> L2-A is Active, L2-B is mirror. All systems are running and available to take traffic</td><td>1. Run app     2. Power down L2-A (Active)</td><td>L2-B(mirror) becomes active.  Takes the load. No drop in TPS on Failover. </td></tr>
<tr><td>TAL4</td><td>Active L2 Loss - Reboot               </td><td>L2-A is Active, L2-B is mirror. All systems are running and available to take traffic</td><td>1. Run app    2. Reboot L2-A (Active)</td><td>L2-B(mirror) becomes active.  Takes the load. No drop in TPS on Failover. </td></tr>
<tr><td>TAL5</td><td>Active L2 Loss - Pull Plug               </td><td> L2-A is Active, L2-B is mirror. All systems are running and available to take traffic</td><td>1. Run app    2. Pull the power cable on L2-A (Active)</td><td>L2-B(mirror) becomes active.  Takes the load. No drop in TPS on Failover. </td></tr>
</table>

#### Test Plan - Mirror L2 System Loss Tests

System loss tests confirms High Availability in the event of loss of a single system. This section outlines tests for testing failure of the Terracotta mirror server.

The test plan for testing Terracotta mirror Failures consist of the following tests:

<table>
<tr><th>TestID</th><th> Test </th><th> Setup </th><th> Steps </th><th> Expected Result </th></tr>
<tr><td>TPL1</td><td>Mirror L2 loss   - kill                 </td><td> L2-A is active, L2-B is mirror. All systems are running and available to take traffic.</td><td>1. Run app  2. Kill -9 L2-B (mirror)</td><td>data directory needs to be cleaned up, then when L2-B is restarted,  it re-synchs state from Active Server.</td></tr>
<tr><td>TPL2</td><td>Mirror L2 loss   -clean                      </td><td> L2-A is active, L2-B is mirror. All systems are running and available to take traffic</td><td>1. Run app  2. Run ~/bin/stop-tc-server.sh on L2-B (mirror)</td><td>data directory needs to be cleaned up, then when L2-B is restarted,  it re-synchs state from Active Server.</td></tr>
<tr><td>TPL3</td><td>Mirror L2 loss   -power down                     </td><td> L2-A is active, L2-B is mirror. All systems are running and available to take traffic</td><td>1. Run app     2. Power down L2-B (mirror)</td><td>data directory needs to be cleaned up, then when L2-B is restarted,  it re-synchs state from Active Server.</td></tr>
<tr><td>TPL4</td><td>Mirror L2 loss   -reboot                     </td><td> L2-A is active, L2-B is mirror. All systems are running and available to take traffic</td><td>1. Run app     2. Reboot L2-B (mirror)</td><td>data directory needs to be cleaned up, then when L2-B is restarted,  it re-synchs state from Active Server.</td></tr>
<tr><td>TPL5</td><td>Mirror L2 loss   -Pull Plug            </td><td> L2-A is active, L2-B is mirror. All systems are running and available to take traffic</td><td>1. Run app     2. Pull plug on  L2-B (mirror)</td><td>data directory needs to be cleaned up, then when L2-B is restarted,  it re-synchs state from Active Server.</td></tr>
</table>

#### Test Plan - Failover/Failback Tests

This section outlines tests to confirm the cluster ability to fail-over to the mirror Terracotta server, and fail back.

The test plan for testing fail over and fail back consists of the following tests:

<table>
<tr><th>TestID</th><th> Test </th><th> Setup </th><th> Steps </th><th> Expected Result </th></tr>
<tr><td>TFO1</td><td>Failover/Failback</td><td> L2-A is active, L2-B is mirror. All systems are running and available to take traffic</td><td>1. Run application     2. Kill -9 (or run stop-tc-server) on L2-A (Active)                                                  3. After L2-B takes over as Active, start-tc-server on L2-A.   (L2-A is now mirror) 4. Kill -9 (or run stop-tc-server) on L2-B. (L2-A is now Active)</td><td>After first failover L2-A->L2-B, txns should continue. L2-A should come up cleanly in mirror mode when tc-server is run.  When second failover occurs L2-B->L2-A, L2-A should process txns.</td></tr>
</table>

#### Test Plan - Loss of Switch Tests

{tip}
This test can only be run on a redundant network
{tip}

This section outlines testing the loss of a switch in a redundant network, and confirming that no interrupt of service occurs.

The test plan for testing failure of a single switch consists of the following tests:

<table>
<tr><th>TestID</th><th> Test </th><th> Setup </th><th> Steps </th><th> Expected Result </th></tr>
<tr><td>TSL1</td><td>Loss of 1 Switch </td><td> 2 Switches in redundant configuration. L2-A is active, L2-B is mirror. All systems are running and available to take traffic.</td><td>1. Run application   2. Power down/pull plug on Switch</td><td>All traffic transparently moves to switch 2 with no interruptions</td></tr>
</table>

#### Test Plan - Loss of Network Connectivity

This section outlines testing the loss of network connectivity.

The test plan for testing failure of the network consists of the following tests:

<table>
<tr><th>TestID</th><th> Test </th><th> Setup </th><th> Steps </th><th> Expected Result </th></tr>
<tr><td>TNL1</td><td>Loss of NIC wiring (Active)             </td><td> L2-A is active, L2-B is mirror. All systems are runnng and available to traffic</td><td>1. Run application     2. Remove Network Cable on L2-A</td><td>All traffic transparently moves to L2-B with no interruptions</td></tr>
<tr><td>TNL2</td><td>Loss of NIC wiring (mirror)             </td><td> L2-A is active, L2-B is mirror. All systems are runnng and available to traffic</td><td>1. Run application     2. Remove Network Cable on L2-B</td><td>No user impact on cluster</td></tr>
</table>

#### Test Plan - Terracotta Cluster Failure

This section outlines the tests to confirm successful continued operations in the face Terracotta Cluster failures.

The test plan for testing Terracotta Cluster failures consists of the following tests:

<table>
<tr><th>TestID </th><th> Test </th><th> Setup </th><th> Steps </th><th> Expected Result </th></tr>
<tr><td>TF1</td><td>Process Failure Recovery             </td><td> L2-A is active, L2-B is mirror. All systems are runnng and available to traffic</td><td>1. Run application     2. Bring down all L1s and L2s                 3. Start L2s then L1s</td><td>Cluster should come up and begin taking txns again</td></tr>
<tr><td>TF2</td><td>Server Failure Recovery             </td><td> L2-A is active, L2-B is mirror. All systems are runnng and available to traffic</td><td>1. Run application     2. Power down all machines                           3. Start L2s and then L1s</td><td>Should be able to run application once all servers are up.</td></tr>
</table>

#### Client Failure Tests

This section outlines tests to confirm successful continued operations in the face of Terracotta client failures.

The test plan for testing Terracotta Client failures consists of the following tests:

<table>
<tr><th>TestID</th><th> Test </th><th> Setup </th><th> Steps </th><th> Expected Result </th></tr>
<tr><td>TCF1</td><td>L1 Failure -         </td><td> L2-A is active, L2-B is mirror.                                        2 L1s L1-A and L1-B                     All systems are running and available to traffic</td><td>1. Run application     2. kill -9 L1-A.                                        </td><td>L1-B should take all incoming traffic.                Some timeouts may occur due to txns in process when L1 fails over. </td></tr>
</table>
