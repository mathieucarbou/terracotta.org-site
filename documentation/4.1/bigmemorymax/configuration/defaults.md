---
---
# Default Settings for Terracotta Distributed BigMemory

{toc|2:3}

## Introduction
A number of properties control the way the Terracotta Server Array and its clients perform in a Terracotta cluster. Some of these settings are found in the Terracotta configuration file (`tc-config.xml`), while others are found in the BigMemory configuration file (`ehcache.xml`). A few must be set programmatically.

The following sections detail the most important of these properties and shows their default values. To confirm the latest default values for your version of Terracotta software, see the XSD included with your Terracotta kit.

## Terracotta Server Array

A Terracotta cluster is composed of clients and servers. Terracotta properties often use a shorthand notation where a client is referred to as "l1" and a server as "l2".

These properties are set at the top of `tc-config.xml` using a configuration block similar to the following:

    <tc-properties>
       <property name="l2.nha.tcgroupcomm.reconnect.enabled" value="true" />
    <!-- More properties here. -->
    </tc-properties>

See the [Terracotta Server Array documentation](/documentation/4.1/terracotta-server-array/introduction) for more information on the Terracotta Server Array.

### Reconnection and Logging Properties
The following reconnection properties are shown with default values. These properties can be set to custom values using Terracotta configuration properties (`<tc-properties>`/`<property>` elements in `tc-config.xml`).

<table>
<tr><th>Property</th><th>Default Value</th><th>Notes</th></tr>
<tr><td> l2.nha.tcgroupcomm.reconnect.enabled</td><td>true

</td><td>Enables server-to-server reconnections.</td></tr>

<tr><td>l2.nha.tcgroupcomm.reconnect.timeout</td><td>5000ms

</td><td>l2-l2 reconnection timeout.</td></tr>

<tr><td>l2.l1reconnect.enabled</td><td>true

</td><td>Enables an l1 to reconnect to servers.</td></tr>

<tr><td> l2.l1reconnect.timeout.millis</td><td>5000ms

</td><td>The reconnection time out, after which an l1 disconnects.</td></tr>

<tr><td> tc.config.getFromSource.timeout</td><td>30000ms

</td><td>Timeout for getting configuration from a source. For example, this controls how long a client can try to access configuration from a server. If the client fails to do so, it will fail to connect to the cluster.</td></tr>

<tr><td> logging.maxBackups</td><td>20

</td><td>Upper limit for number of backups of Terracotta log files.</td></tr>

<tr><td> logging.maxLogFileSize</td><td>512MB

</td><td>Maximum size of Terracotta log files before rolling logging starts.</td></tr>
</table>

### HealthChecker Tolerances

The following properties control disconnection tolerances between Terracotta servers (l2 <-> l2), Terracotta servers and Terracotta clients (l2 -> l1), and Terracotta clients and Terracotta servers (l1 -> l2).

#### l2<->l2 GC tolerance : 40 secs, cable pull/network down tolerance : 10secs

    l2.healthcheck.l2.ping.enabled = true
    l2.healthcheck.l2.ping.idletime = 5000
    l2.healthcheck.l2.ping.interval = 1000
    l2.healthcheck.l2.ping.probes = 3
    l2.healthcheck.l2.socketConnect = true
    l2.healthcheck.l2.socketConnectTimeout = 5
    l2.healthcheck.l2.socketConnectCount = 10

#### l2->l1 GC tolerance : 40 secs, cable pull/network down tolerance : 10secs

    l2.healthcheck.l1.ping.enabled = true
    l2.healthcheck.l1.ping.idletime = 5000
    l2.healthcheck.l1.ping.interval = 1000
    l2.healthcheck.l1.ping.probes = 3
    l2.healthcheck.l1.socketConnect = true
    l2.healthcheck.l1.socketConnectTimeout = 5
    l2.healthcheck.l1.socketConnectCount = 10

#### l1->l1 GC tolerance : 50 secs, cable pull/network down tolerance : 10secs

    l1.healthcheck.l2.ping.enabled = true
    l1.healthcheck.l2.ping.idletime = 5000
    l1.healthcheck.l2.ping.interval = 1000
    l1.healthcheck.l2.ping.probes = 3
    l1.healthcheck.l2.socketConnect = true
    l1.healthcheck.l2.socketConnectTimeout = 5
    l1.healthcheck.l2.socketConnectCount = 13


## Terracotta Clients
Client configuration properties typically address the behavior, size, and functionality of in-memory data stores. Others affect certain types of cache-related bulk operations.

See also [How Server Settings Can Override Client Settings](/documentation/4.1/bigmemorymax/configuration/reference-guide#how-server-settings-affect-client-settings).

Properties are set in `ehcache.xml` except as noted.

### General Settings
The following default settings affect in-memory data. For more information on these settings, see the [BigMemory Max documentation](/documentation/4.1/bigmemorymax/configuration/configuration).

<table>
<tr><th>Property</th><th>Default Value</th><th>Notes</th></tr>
<tr><td>value mode</td><td>SERIALIZATION</td><td></td></tr>
<tr><td>consistency</td><td>EVENTUAL</td><td></td></tr>
<tr><td>XA</td><td>false</td><td></td></tr>
<tr><td>orphan eviction</td><td>true</td><td></td></tr>
<tr><td>local key cache</td><td>false</td><td></td></tr>
<tr><td>synchronous writes</td><td>false</td><td></td></tr>
<tr><td>ttl</td><td>0</td><td>0 means never expire.</td></tr>
<tr><td>tti</td><td>0</td><td>0 means never expire.</td></tr>
<tr><td>transactional mode</td><td>off</td><td></td></tr>
<tr><td>persistence strategy</td><td>none</td><td></td></tr>
<tr><td>maxEntriesInCache</td><td>0</td><td>0 means that the cache will not undergo capacity eviction (but periodic and resource evictions are still allowed)</td></tr>
<tr><td>maxBytesLocalHeap</td><td>0</td><td></td></tr>
<tr><td>maxBytesLocalOffHeap</td><td>0</td><td></td></tr>
<tr><td>maxEntriesLocalHeap</td><td>0</td><td>0 means infinite.</td></tr>
</table>

### NonStop Cache
The following default settings affect the behavior of the cache when while the client is disconnected from the cluster. For more information on these settings, see the [nonstop-cache documentation](/documentation/4.1/bigmemorymax/configuration/non-stop-cache).

<table>
<tr><th>Property</th><th>Default Value</th><th>Notes</th></tr>
<tr><td>enable</td><td>false</td><td></td></tr>
<tr><td>timeout behavior</td><td>exception</td><td></td></tr>
<tr><td>timeout</td><td>30000ms</td><td></td></tr>
<tr><td><code>net.sf.ehcache.nonstop<br>.bulkOpsTimeoutMultiplyFactor</code></td><td>10</td><td>
This value is a timeout multiplication factor affecting bulk operations such as <code>removeAll()</code> and <code>getAll()</code>. Since the default nonstop timeout is 30 seconds, it sets a timeout of 300 seconds for those operations. The default can be changed programmatically:

   <code>
cache.getTerracottaConfiguration()
     .getNonstopConfiguration()
     .setBulkOpsTimeoutMultiplyFactor(10)
   </code>
    </td></tr>
</table>

### Bulk Operations
The following properties are shown with default values. These properties can be set to custom values using [Terracotta configuration properties](#terracotta-server-array).

Increasing batch sizes may improve throughput, but could raise latency due to the load on resources from processing larger blocks of data.

<table>
<tr><th>Property</th><th>Default Value</th><th>Notes</th></tr>
<tr><td>ehcache.bulkOps.maxKBSize</td><td>1MB</td><td>Batch size for bulk operations such as putAll and removeAll.</td></tr>

<tr><td>ehcache.getAll.batchSize</td><td>1000</td><td>The number of elements per batch in a getAll operation.</td></tr>

<tr><td>ehcache.incoherent.putsBatchByteSize</td><td>5MB</td><td>For bulk-loading mode. The minimum size of a batch in a bulk-load operation. Increasing batch sizes may improve throughput, but could raise latency due to the load on resources from processing larger blocks of data.</td></tr>

<tr><td>ehcache.incoherent.putsBatchTimeInMillis</td><td>600 ms</td><td>
For bulk-loading mode. The maximum time the bulk-load operation takes to batch puts before flushing to the Terracotta Server Array.</td></tr>
</table>
