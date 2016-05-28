---
---
# Terracotta Cluster Events

{toc|2:3}

## Introduction
The Terracotta distributed BigMemory Max cluster events API provides access to Terracotta cluster events and cluster topology. This event-notification mechanism reports events related to the nodes in the Terracotta cluster, not cache events.

## Cluster Topology
The interface `net.sf.ehcache.cluster.CacheCluster` provides methods for obtaining topology information for a Terracotta cluster.
The following methods are available:

* `String getScheme()`

    Returns a scheme name for the cluster information. Currently `TERRACOTTA` is the only scheme supported. The scheme name is used by CacheManager.getCluster() to return cluster information.
* `Collection<ClusterNode> getNodes()`

    Returns information on all the nodes in the cluster, including ID, hostname, and IP address.
* `boolean addTopologyListener(ClusterTopologyListener listener)`

    Adds a cluster-events listener. Returns true if the listener is already active.
* `boolean removeTopologyListener(ClusterTopologyListener)`

    Removes a cluster-events listener. Returns true if the listener is already inactive.

The interface `net.sf.ehcache.cluster.ClusterNode` provides methods for obtaining information on specific cluster nodes.

<pre><code>public interface ClusterNode {
/**
* Get a unique (per cluster) identifier for this node.
*
* @return Unique per cluster identifier
*/
String getId();
/**
* Get the host name of the node
*
* @return Host name of node
*/
String getHostname();
/**
* Get the IP address of the node
*
* @return IP address of node
*/
String getIp();
}
</code></pre>

## Listening For Cluster Events

The interface `net.sf.ehcache.cluster.ClusterTopologyListener` provides methods for detecting the following cluster events:

<pre><code>public interface ClusterTopologyListener {
/**
* A node has joined the cluster
*
* @param node The joining node
*/
void nodeJoined(ClusterNode node);
/**
* A node has left the cluster
*
* @param node The departing node
*/
void nodeLeft(ClusterNode node);
/**
* This node has established contact with the cluster and can execute clustered operations.
*
* @param node The current node
*/
void clusterOnline(ClusterNode node);
/**
* This node has lost contact (possibly temporarily) with the cluster and cannot execute
* clustered operations
*
* @param node The current node
*/
void clusterOffline(ClusterNode node);
}
/**
* This node lost contact and rejoined the cluster again.
* <p />
* This event is only fired in the node which rejoined and not to all the connected nodes
* @param oldNode The old node which got disconnected
* @param newNode The new node after rejoin
*/
void clusterRejoined(ClusterNode oldNode, ClusterNode newNode);
</code></pre>

### Example Code
This example prints out the cluster nodes and then registers a `ClusterTopologyListener`
which prints out events as they happen.

<pre><code>CacheManager mgr = ...
CacheCluster cluster = mgr.getCluster("TERRACOTTA");
  // Get current nodes
Collection&lt;ClusterNode> nodes = cluster.getNodes();
for(ClusterNode node : nodes) {
  System.out.println(node.getId() + " " + node.getHostname() + " " + node.getIp());
}
  // Register listener
cluster.addTopologyListener(new ClusterTopologyListener() {
  public void nodeJoined(ClusterNode node) { System.out.println(node + " joined"); }
  public void nodeLeft(ClusterNode node) { System.out.println(node + " left"); }
  public void clusterOnline(ClusterNode node) { System.out.println(node + " enabled"); }
  public void clusterOffline(ClusterNode node) { System.out.println(node + " disabled"); }
  public void clusterRejoined(ClusterNode node, ClusterNode newNode) {
    System.out.println(node + " rejoined the cluster as " + newNode);
  }
});
</code></pre>

## Troubleshooting
In most cases the Terracotta cluster-events API behaves as expected. Unexpected results can occur under the circumstances described below.

### getCluster Returns Null For Programmatically Created CacheManagers
If a CacheManager instance is created and configured programmatically (without an <code>ehcache.xml</code> or other external configuration resource), <code>getCluster("TERRACOTTA")</code> may return null even if a Terracotta cluster exists. To ensure that cluster information is returned in this case, get a cache that is clustered with Terracotta:

    // mgr created and configured programmatically.
    CacheManager mgr = new CacheManager();
    // myCache has Terracotta clustering.
    Cache cache = mgr.getEhcache("myCache");
    // A Terracotta client has started, making available cluster information.
    CacheCluster cluster = mgr.getCluster("TERRACOTTA");

### nodeJoined for the Current Node {#68163}

Since the current node joins the cluster before code adding the topology listener runs, the current node may never receive the nodeJoined event. You can detect if the current node is in the cluster by checking if the cluster is online:

    cluster.addTopologyListener(cacheListener);
    if(cluster.isClusterOnline()) {
      cacheListener.clusterOnline(cluster.getCurrentNode());
    }

### Multiple NodeJoined Events in the Same JVM
Since multiple Terracotta clients can exist in the same JVM, for example when using the Terracotta Toolkit and Ehcache, multiple NodeJoined events can be generated in that JVM. Remote clients will not be able to differentiate between the clients that generated the NodeJoined events.
