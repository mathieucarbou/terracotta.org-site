---
---
# BigMemory Max Best Practices

The following sections contain advice for optimizing BigMemory Max operations.

{toc|2:3}

## Tuning Off-Heap Store Performance
Memory-related or performance issues that arise during operations can be related to improper allocation of memory to the off-heap store. If performance or functional issues arise that can be traced back to the off-heap store, see the suggested tuning tips in this section.

### General Memory allocation

Committing too much of a system's physical memory is likely to result in paging of virtual memory to disk, quite likely during garbage-collection operations, leading to significant performance issues. On systems with multiple Java processes, or multiple processes in general, the sum of the Java heaps and off-heap stores for those processes should also not exceed the size of the physical RAM in the system. Besides memory allocated to the heap, Java processes require memory for other items, such as code (classes), stacks, and PermGen.

Note that MaxDirectMemorySize sets an upper limit for the JVM to enforce, but does not actually allocate the specified memory. Overallocation of direct memory (or buffer) space is therefore possible, and could lead to paging or even memory-related errors. The limit on direct buffer space set by MaxDirectMemorySize should take into account the total physical memory available, the amount of memory that is allotted to the JVM object heap, and the portion of direct buffer space that other Java processes may consume.

In addition, be sure to allocate at least 15 percent more off-heap memory than the size of your data set. To maximize performance, a portion of off-heap memory is reserved for meta-data and other purposes.

Note also that there could be other users of direct buffers (such as NIO and certain frameworks and containers). Consider allocating additional direct buffer memory to account for that additional usage.

### Compressed References

For 64-bit JVMs running Java 6 Update 14 or higher, consider enabling compressed references to improve overall performance. For heaps up to 32GB, this feature causes references to be stored at half the size, as if the JVM is running in 32-bit mode, freeing substantial amounts of heap for memory-intensive applications. The JVM, however, remains in 64-bit mode, retaining the advantages of that mode.

For the Oracle HotSpot, compressed references are enabled using the option `-XX:+UseCompressedOops`. For IBM JVMs, use `-Xcompressedrefs`.

### Slow Off-Heap Allocation {#slow-off-heap-allocation}
Based on configuration, usage, and memory requirements, BigMemory could allocate off-heap memory multiple times. If off-heap memory comes under pressure due to over-allocation, the host OS may begin paging to disk, thus slowing down allocation operations. As the situation worsens, an off-heap buffer too large to fit in memory can quickly deplete critical system resources such as RAM and swap space and crash the host OS.

To stop this situation from degrading, off-heap allocation time is measured to avoid allocating buffers too large to fit in memory. If it takes more than 1.5 seconds to allocate a buffer, a warning is issued. If it takes more than 15 seconds, the JVM is halted with `System.exit()` (or a different method if the Security Manager prevents this).

To prevent a JVM shutdown after a 15-second delay has occurred, set the `net.sf.ehcache.offheap.DoNotHaltOnCriticalAllocationDelay` system property to true. In this case, an error is logged instead.


### Maximum Serialized Size of an Element {#increasing-serialized-size}
This section applies when using BigMemory through the Ehcache API.

Unlike the memory and the disk stores, by default the off-heap store has a 4MB limit for classes with high quality hashcodes, and 256KB limit for those with pathologically bad hashcodes. The built-in classes such as `String` and the `java.lang.Number` subclasses Long and Integer have high quality hashcodes. This can issues when objects are expected to be larger than the default limits.

To override the default size limits, set the system property `net.sf.ehcache.offheap.cache_name.config.idealMaxSegmentSize` to the size you require.

For example,

    net.sf.ehcache.offheap.com.company.domain.State.config.idealMaxSegmentSize=30M

### Reducing Faulting
While the memory store holds a hotset (a subset) of the entire data set, the off-heap store should be large enough to hold the entire data set. The frequency of misses (get operations that fail to find the data in memory) begins to rise when the data is too large to fit into off-heap memory, forcing gets to fetch data from the disk store (called *faulting*). More misses in turn raise latency and lower performance.

For example, tests with a 4GB data set and a 5GB off-heap store recorded no misses. With the off-heap store reduced to 4GB, 1.7 percent of cache operations resulted in misses. With the off-heap store at 3GB, misses reached 15 percent.

### Swappiness and Huge Pages

An operating system (OS) that is swapping to disk can substantially slow down or even stop your application. If the OS is under pressure because Terracotta servers&mdash;along with other processes running on a host&mdash;are squeezing the available memory, then memory will start to be paged in and out. This type of operation, when too frequent, requires either tuning of the swap parameters or a permanent solution to a chronic lack of RAM.

An OS could swap data from memory to disk even if memory is not running low. For the purpose of optimization, data that appears to be unused may be a target for swapping. Because BigMemory can store substantial amounts of data in RAM, its data may be swapped by the OS. But swapping can degrade overall cluster performance by introducing thrashing, the condition where data is frequently moved forth and back between memory and disk.

To make heap memory use more efficient, Linux, Microsoft Windows, and Oracle Solaris users     should review their configuration and usage of swappiness as well as the size of the swapped   memory pages. In general, BigMemory benefits from lowered swappiness and the use of *huge pages* (also known as *big pages*, *large pages*, and *superpages*). Settings for these behaviors vary by OS and JVM. For Oracle HotSpot, `-XX:+UseLargePages` and  `-XX:LargePageSizeInBytes=<size>` (where &lt;size> is a value allowed by the OS for specific   CPUs) can be used to control page size. However, note that this setting does not affect how    off-heap memory is allocated. Over-allocating huge pages while also configuring substantial   off-heap memory *can starve off-heap allocation and lead to memory and performance problems*.

#### Reduce Swapping

Many tools are available to help you diagnose swapping. Popular options include using a built-in command-line utility. On Linux, for example:

* See available RAM with `free -m` (display memory statistics in megabtyes). Pay attention to swap utilization.
* `vmstat` displays swap-in ("si") and swap-out ("so") numbers. Non-zero values indicate swapping activity. Set `vmstat` to refresh on a short interval to detect trends.
* Process status can be used to get detailed information on all processes running on a node. For example, `ps -eo pid,ppid,rss,vsize,pcpu,pmem,cmd -ww --sort=pmem` displays processes ordered by memory use. You can also sort by virtual memory size ("vsize") and real memory size ("rss") to focus on both the most memory-consuming processes and their in-memory footprint.

**NOTE:** If the JVM is running in a guest virtual host, analyze swapping by both the virtual and underlying host.

If swappiness is being caused by memory pressure, offloading unnecessary or unrelated processes along with running smaller JVMs is often a successful cure. When computing the memory footprint of a JVM, be sure to include the off-heap being allocated.


## Tuning Heap Memory Performance
Long garbage collection (GC) cycles are one of the most common causes of issues in a Terracotta cluster because a full GC event pauses all threads in the JVM. Servers disconnecting clients, clients dropping servers, and timed-out processes are just some of the problems long GC cycles can cause. Having a clear understanding of how your application behaves with respect to creating garbage, and how that garbage is being collected, is necessary for avoiding or solving these issues.

### Printing and Analyzing GC Logs
The most effective way to gain that understanding is to create a profile of GC in your application by using tools made for that purpose. Consider using JVM options to generate logs of GC activity:

* `-verbose:gc`
* `-Xloggc:<filename>`
* `-XX:+PrintGCDetails`
* `-XX:+PrintGCTimeStamps`

Apply an appropriate parsing and visualization tool to GC log files to help analyze their contents.

### Observing GC Statistics With jstat
One way to observe GC statistics is by using the Java utility jstat. The following command will produce a log of GC statistics, updated every ten seconds:

    jstat -gcutil <pid> 10 1000000

An important statistic is the Full Garbage Collection Time. The difference between the total time for each reading is the amount of time the system was paused. A jump of more than a few seconds will not be acceptable in most application contexts.

### Solutions to Problematic GC {#gc-solutions}
Once your application's typical GC cycles are understood, consider one or more of the following solutions:

* Maximizing BigMemory to eliminate the drag GC imposes on performance in large heaps.

    BigMemory opens up off-heap memory for use by Java applications, and off-heap memory is not subject to GC.

* Configuring the [HealthChecker parameters](/documentation/4.1/terracotta-server-array/high-availability) in the Terracotta cluster to account for the observed GC cycles.

    Increase nodes' tolerance of inactivity in other nodes due to GC cycles.

* Tuning the GC parameters to change the way GC runs in the heap.

    If running multi-core machines and no collector is specifically configured, consider `-XX:+UseParallelGC` and `-XX:+UseParallelOldGC`.

    If running multiple JVMs or application processes on the same machine, tune the number of concurrent threads in the parallel collector with `-XX:ParallelGCThreads=<number>`.

    Another collector is called Concurrent Mark Sweep (CMS). This collector is normally not recommended (especially for Terracotta servers) due to certain performance and operational issues it raises. However, under certain circumstances related to the type of hosting platform and application data usage characteristics, it may boost performance and may be worth testing with.

* If running on a 64-bit JVM, and if your JDK supports it, use `-XX:+UseCompressedOops`.

    This setting can reduce substantially the memory footprint of object pointer used by the JVM.

* Refactoring clustered applications that unnecessarily create too much garbage.
* Ensuring that the problem node has enough memory allocated to the heap.


## Common Causes of Failures in a Cluster

The most common causes of failures in a cluster are interruptions in the network and long Java GC cycles on particular nodes. Tuning the HealthChecker and reconnect features can reduce or eliminate these two problems. However, additional actions should also be considered.

Sporadic disruptions in network connections between L2s and between L2s and L1s can be difficult to track down. Be sure to thoroughly test all network segments connecting the nodes in a cluster, and also test network hardware. Check for speed, noise, reliability, and other applications that grab bandwidth.

Other sources of failures in a cluster are disks that are nearly full or are running slowly, and running other applications that compete for a node's resources.

### Do Not Interrupt!

*Ensure that your application does not interrupt clustered threads*. This is a common error that can cause the Terracotta client to shut down or go into an error state, after which it will have to be restarted.

The Terracotta client library runs with your application and is often involved in operations which your application is not necessarily aware of. These operations can get interrupted, something the Terracotta client cannot anticipate. Interrupting clustered threads, in effect, puts the client into a state which it cannot handle.

### Diagnose Client Disconnections
If clients disconnect on a regular basis, try the following to diagnose the cause:

* Analyze the Terracotta client logs for potential issues, such as long GC cycles.
* Analyze the Terracotta server logs for disconnection information and any rejections of reconnection attempts by the client.
* See the operator events panel in the [Terracotta Management Console](/documentation/4.1/tms/tms) for disconnection events, and note the reason.

If the disconnections are due to long GC cycles or inconsistent network connections in the client, consider the remedies suggested in [this section](#gc-solutions). If disconnections continue to happen, consider configuring caches with [nonstop behavior](/documentation/4.1/bigmemorymax/configuration/non-stop-cache) and enabling [rejoin](/documentation/4.1/bigmemorymax/configuration/reference-guide#71266).


### Detect Memory Pressure Using the Terracotta Logs
Terracotta server and client logs contain messages that help you track memory usage. Locations of server and client logs are configured in the Terracotta configuration file, `tc-config.xml`.

You can view the state of memory usage in a node by finding messages similar to the following:

    2011-12-04 14:47:43,341 [Statistics Logger] ... memory free : 39.992699 MB
    2011-12-04 14:47:43,341 [Statistics Logger] ... memory used : 1560.007301 MB
    2011-12-04 14:47:43,341 [Statistics Logger] ... memory max : 1600.000000 MB

These messages can indicate that the node is running low on memory.

###Disk usage with both Search and Fast Restart enabled
The TSA may be configured to be restartable in addition to including searchable caches, but both of these features require disk storage. When both are enabled, be sure that enough disk space is available. Depending upon the number of searchable attributes, the amount of disk storage required may be up to 1.5 times the amount of in-memory data.


## Manage Sessions in a Cluster

* Make sure the configured time zone and system time is consistent between all application servers. If they are different a session may appear expired when accessed on different nodes.

* Set `-Dcom.tc.session.debug.sessions=true` and `-Dcom.tc.session.debug.invalidate=true` to generate more debugging information in the client logs.

* All clustered session implementations (including terracotta Sessions) require a mutated session object be put back into the session after it's mutated. If the call is missing, then the change isn't known to the cluster, only to the local node. For example:

         Session session = request.getSession();
         Map m  = session.getAttribute("foo");
         m.clear();
         session.setAttribute("foo", m); // Without this call, the clear() is not effective across the cluster.


Without a `setAttribute()` call, the session becomes inconsistent across the cluster. Sticky sessions can mask this issue, but as soon as the session is accessed on another node, its state does not match the expected one. To view the inconsistency on a single client node, add the Terracotta property `-Dcom.tc.session.clear.on.access=true` to force locally cached sessions to be cleared with every access.

If third-party code cannot be refactored to fix this problem, and you are running Terracotta 3.6.0 or higher, you can write a servlet filter that calls `setAttribute()` at the end of every request. Note that this solution may substantially degrade performance.

        package controller.filter;

        import java.io.IOException;
        import java.util.Enumeration;

        import javax.servlet.Filter;
        import javax.servlet.FilterChain;
        import javax.servlet.FilterConfig;
        import javax.servlet.ServletException;
        import javax.servlet.ServletRequest;
        import javax.servlet.ServletResponse;
        import javax.servlet.http.HttpServletRequest;
        import javax.servlet.http.HttpSession;

        public class IterateFilter implements Filter {

          public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
	      throws IOException, ServletException {
            HttpSession session = ((HttpServletRequest) request).getSession();
            if (session != null) {
              @SuppressWarnings("rawtypes")
              Enumeration e = session.getAttributeNames();
              while (e.hasMoreElements()) {
                String name = (String)e.nextElement();
                Object value = session.getAttribute(name);
                session.setAttribute(name, value);
              }
            }
          }

          public void init(FilterConfig filterConfig) throws ServletException {
            // TODO Auto-generated method stub
          }

          public void destroy() {
            // TODO Auto-generated method stub
          }
        }

## A Safe Failover Procedure {#21339}

To safely migrate clients to a standby server without stopping the cluster, follow these steps:


1.  If it is not already running, start the standby server using the start-tc-server script.
    The standby server must already be configured in the Terracotta configuration file.
2.  Ensure that the standby server is ready for failover (PASSIVE-STANDBY status). In the TMC, the status light will be cyan.
3.  Shut down the active server using the stop-tc-server script.

    NOTE: If the script detects that the mirror server in STANDBY state isn't reachable, it issues a warning and fails to shut down the active server. If failover is not a concern, you can override this behavior with the `--force` flag.

    Clients will connect to the new active server.

4.  Restart any clients that fail to reconnect to the new active server within the configured reconnection window.

The previously active server can now rejoin the cluster as a standby server. If restartable mode had been enabled, its data is first removed and then the current data is read in from the now active server.



## A Safe Cluster Shutdown Procedure {#65389}

A safe cluster shutdown should follow these steps:

1.  Shut down the standby servers using the stop-tc-server script.
2.  Shut down the clients.
    The Terracotta client will shut down when you shut down your application.
3.  Shut down the active server using the stop-tc-server script.

To restart the cluster, first start the server that was last active. If clustered data is not persisted, any of the servers could be started first as no data conflicts can take place.
