---
---
# BigMemory Max Quick Start

{toc|2:2}

## Install BigMemory Max
Installing BigMemory Max is as easy as downloading the kit and ensuring that the correct files are on your application's classpath. The only platform requirement is using JDK 1.6 or higher.

1. If you do not have a BigMemory Max kit, download it from [http://terracotta.org/downloads/bigmemorymax](http://terracotta.org/downloads/bigmemorymax).

    The kit is packaged as a tar.gz file. Unpack it on the command line or with the appropriate decompression application.

2. Add the following JARs from in the kit to your application's classpath:

    * `apis/ehcache/lib/ehcache-ee-<version>.jar` &ndash; This file contains the API to BigMemory Max.

    * `apis/ehcache/lib/slf4j-api-<version>.jar` &ndash; This file is the bridge, or logging facade, to the BigMemory Max logging framework.

    * `apis/toolkit/lib/terracotta-toolkit-runtime-ee-<version>.jar` &ndash; This JAR contains the libraries for the Terracotta Server Array.

3. Save the BigMemory Max license-key file to the BigMemory Max home directory. This file, called `terracotta-license.key`, was attached to an email you received after registering for the BigMemory Max download.

    Alternatively, you can add the license-key file to your application's classpath, or specify it with the following Java system property:

        -Dcom.tc.productkey.path=/path/to/terracotta-license.key

4. BigMemory Max uses Ehcache as its user-facing interface. To configure BigMemory Max, create an `ehcache.xml` configuration file, or update the one that is provided in the config-samples/ directory of the BigMemory Max kit. For example:


        <ehcache xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
             xsi:noNamespaceSchemaLocation="http://ehcache.org/ehcache.xsd"
             name="myBigMemoryMaxConfig">

          <!-- Tell BigMemory where to write its data to disk. -->
          <diskStore path="/path/to/my/disk/store/directory"/>

          <!-- Set 'maxBytesLocalOffHeap' to the amount of off-heap in-memory
          storage you want to use. This memory is invisible to the Java garbage
          collector, providing for gigabytes to terabytes of in-memory data without
          garbage collection pauses. -->
          <cache name="myBigMemoryMaxStore"
                maxBytesLocalHeap="512M"
                maxBytesLocalOffHeap="8G">

			<!-- Tell BigMemory to use the "localRestartable" persistence
			strategy for fast restart (optional). -->
            <persistence strategy="localRestartable"/>

            <!-- Include the terracotta element so that the data set will be
            managed as a client of the Terracotta server array.  -->
            <terracotta/>
          </cache>

          <!-- Specify where to find the server array configuration. In this
          case, the configuration is retrieved from the local server. -->
          <terracottaConfig url="localhost:9510" />

        </ehcache>


	Place your `ehcache.xml` file in the top-level of your classpath.

	For more information on configuration options, see the configuration of
	[ "Storage Tiers"](/documentation/4.1/bigmemorymax/configuration/storage-options)
	and to the reference ehcache.xml configuration file in the `config-samples`
	directory of the BigMemory Max kit.

5. Use the `-XX:MaxDirectMemorySize` Java option to allocate
   enough direct memory in the JVM to accomodate the off-heap storage specified
   in your configuration, plus at least 250MB to allow for other direct memory
   usage that might occur in your application. For example:

        -XX:MaxDirectMemorySize=9G

    Set `MaxDirectMemorySize` to the amount of BigMemory you have. For more
    information about this step, refer to
    [Allocating Direct Memory in the JVM](/documentation/4.1/bigmemorymax/configuration/storage-options#allocating-direct-memory).

   Also, allocate at least enough heap using the `-Xmx` Java option to accomodate the
   on-heap storage specified in your configuration, plus enough extra heap to
   run the rest of your application.  For example:

        -Xmx1g

   Finally, if necessary, define the JAVA_HOME environment variable.

6. Learn BigMemory basics from the [Tutorials](/documentation/4.1/bigmemorymax/get-started/hello-world), or look through the [Code Samples](/documentation/4.1/bigmemorymax/code-samples) for examples of how to employ the various features and capabilities of BigMemory Max.

##Start the Terracotta Server and Management Console
Large data sets in BigMemory Max can be distributed across the Terracotta Server Array (TSA) and managed with the Terracotta Management Console (TMC).

1. To configure the Terracotta server, create a `tc-config.xml` configuration file, or update the one that is provided in the config-samples/ directory of the BigMemory Max kit. For example:

        <?xml version="1.0" encoding="UTF-8" ?>
        <tc:tc-config xmlns:tc="http://www.terracotta.org/config"
              xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
              xsi:schemaLocation="http://www.terracotta.org/schema/terracotta-9.xsd">
          <servers>
            <server host="localhost" name="My Server Name">
              <!-- Specify the path where the server should store its data. -->
              <data>/local/disk/path/to/terracotta/server1-data</data>
               <!-- Specify the port where the server should listen for client
               traffic. -->
               <tsa-port>9510</tsa-port>
               <jmx-port>9520</jmx-port>
               <tsa-group-port>9530</tsa-group-port>
               <!-- Enable BigMemory on the server. -->
               <dataStorage size=”8g”>
                 <offheap size=”4g”/>
                 <hybrid/>
               </dataStorage>             
            </server>
            <!-- Add the restartable element for Fast Restartability (optional). -->
            <restartable enabled="true"/>
          </servers>
          <clients>
            <logs>logs-%i</logs>
          </clients>
        </tc:tc-config>

	Place your `tc-config.xml` file in the Terracotta server/ directory.

	For more information about configuration options, refer to the
	[TSA configuration documentation](/documentation/4.1/terracotta-server-array/configuration-guide).


2. In a terminal, change to your Terracotta server/ directory. Then execute the start-tc-server command:

        %> cd /path/to/bigmemory-max-<version>/server
    	%> ./bin/start-tc-server.sh

   You should see confirmation in the terminal that the server started.

   **Note**: For Microsoft Windows installations, use the BAT scripts, and where forward slashes ("/") are given in directory paths, substitute back slashes ("\").

3. In a terminal, change to your Terracotta tools/management-console/ directory. Then execute the start-tmc command:

        %> cd /path/to/bigmemory-max-<version>/tools/management-console
	    %> ./bin/start-tmc.sh

4. In a browser, enter the URL `http://localhost:9889/tmc`. When you first connect to the TMC, the authentication setup page appears, where you can choose to run the TMC with authentication or without.

    ![Terracotta Management Console](/images/documentation/InitialTMCAuthenticationSetup2.png)

5. Use the TMC to manage all of the clients and servers in your deployment. For more information about the TMC, refer to the
	[TMC documentation](/documentation/4.1/tms/tms).

    ![Terracotta Management Console](/images/documentation/TMCOVerview1-1.png)



## Additional Configuration Topics
For a general overview to configuring BigMemory Max, see this [Configuring BigMemory Max](/documentation/4.1/bigmemorymax/configuration/configuration). Specific configuration topics are introduced below.

### Automatic Resource Control

Automatic Resource Control (ARC) gives you fine-grained controls for tuning performance and enabling trade-offs between throughput, latency and data access. Independently adjustable configuration parameters include differentiated tier-based sizing and pinning hot or eternal data in the most effective tier.

#### Dynamically Sizing Stores
Tuning often involves sizing stores appropriately. There are a number of ways to size the different BigMemory Max data tiers using simple configuration sizing attributes. The [Sizing](/documentation/4.1/bigmemorymax/configuration/cache-size) page explains how to tune tier sizing by configuring dynamic allocation of memory and automatic balancing.

#### Pinning Data
One of the most important aspects of running an in-memory data store involves managing the life of the data in each BigMemory Max tier. See the [Data Life](/documentation/4.1/bigmemorymax/configuration/data-life) page for more information on the pinning, expiration, and eviction of data.

### Fast Restartability
BigMemory Max has full fault tolerance, allowing for continuous access to in-memory data after a planned or unplanned shutdown, with the option to store a fully consistent record of the in-memory data on the local disk at all times. The [Fast Restart](/documentation/4.1/bigmemorymax/configuration/fast-restart) page covers data persistence, fast restartability, and using the local disk as a storage tier for in-memory data (both heap and off-heap stores).


### Hybrid Data Storage
[BigMemory Hybrid](/documentation/4.1/terracotta-server-array/hybrid) extends BigMemory distributed in a Terracotta Server Array so that data can be stored across a hybrid mixture of RAM and SSD/Flash.

### Search
Search billions of entries&mdash;gigabytes, even terabytes of data&mdash;with results returned in less than a second. Data is indexed without significant overhead, and features like "GroupBy', direct support for handling null values, and optimization around handling huge results sets are included. [BigMemory Search](/documentation/4.1/bigmemorymax/search/introduction) provides the ability for data to be looked up based on multiple criteria instead of just keys. You can query BigMemory data using either simple SQL statements or the Search API.

### Transactional Caching
Transactional modes are a powerful extension for performing atomic operations on data stores, keeping your data in sync with your database. The [Transactions](/documentation/4.1/bigmemorymax/api/jta) page covers the background and configuration information for BigMemory Max transactional modes. [Explicit Locking](/documentation/4.1/bigmemorymax/api/explicitlocking) is another API that can be used as a custom alternative to XA Transactions or Local transactions.


### Administration and Monitoring

The [Terracotta Management Console](/documentation/4.1/tms) (TMC) is a web-based monitoring and administration application for tuning cache usage, detecting errors, and providing an easy-to-use access point to integrate with production management systems.

As an alternative to the TMC, standard [JMX-based administration and monitoring](/documentation/4.1/bigmemorymax/operations/jmx) is available.

For logging, BigMemory Max uses the flexible [SLF4J logging framework](/documentation/4.1/bigmemorymax/operations/logging).


## Scale Up and Scale Out

* See the [Distributed Configuration](/documentation/4.1/bigmemorymax/configuration/distributed-configuration) page to learn more about configuration for large amounts of in-memory data.

* See the [Terracotta Server Array](/documentation/4.1/terracotta-server-array/introduction) pages to learn how to use the potential of BigMemory Max.
