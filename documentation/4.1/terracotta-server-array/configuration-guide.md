---
---
# Working with Terracotta Configuration Files {#38131}

{toc|2:3}


## Introduction

Terracotta XML configuration files set the characteristics and behavior of Terracotta server instances and Terracotta clients. The easiest way to create your own Terracotta configuration file is by editing a copy of one of the sample configuration files available with the Terracotta BigMemory Max kit.

Where you locate the Terracotta configuration file, or how your Terracotta server and client configurations are loaded, depends on the stage your project is at and on its architecture. This document covers the following cases:

* Development stage, 1 Terracotta server
* Development stage, 2 Terracotta servers
* Deployment stage

This document discusses cluster configuration in the Terracotta Server Array. To learn more about the Terracotta server instances, see <a href="../terracotta-server-array/server-arrays#67714">Terracotta Server Array Architecture</a>.

For a comprehensive and fully annotated configuration file, see `config-samples/tc-config-reference.xml` in the Terracotta kit.

## Quick Start Configuration

To configure the Terracotta server, create a `tc-config.xml` configuration file, or update the one that is provided in the config-samples/ directory of the BigMemory Max kit. For example:

        <?xml version="1.0" encoding="UTF-8" ?>
        <tc:tc-config xmlns:tc="http://www.terracotta.org/config"
              xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
              xsi:schemaLocation="http://www.terracotta.org/schema/terracotta-9.xsd">
          <servers>
            <server host="localhost" name="My Server Name1">
              <!-- Specify the path where the server should store its data. -->
              <data>/local/disk/path/to/terracotta/server1-data</data>
               <!-- Specify the port where the server should listen for client
               traffic. -->
               <tsa-port>9510</tsa-port>
               <jmx-port>9520</jmx-port>
               <tsa-group-port>9530</tsa-group-port>
               <!-- Enable BigMemory on the server. -->
               <dataStorage size=”800g”>
                  <offheap size=”200g”/>
                  <!-- Hybrid storage is optional. -->
                  <hybrid/>
               </dataStorage>
             </server>
            <server host="localhost" name="My Server Name2">
              <data>/local/disk/path/to/terracotta/server2-data</data>
               <tsa-port>9510</tsa-port>
               <jmx-port>9520</jmx-port>
               <tsa-group-port>9530</tsa-group-port>
               <dataStorage size=”200g”>
                  <offheap size=”200g”/>
               </dataStorage>
             </server>
            <!-- Add the restartable element for Fast Restartability (optional). -->
            <restartable enabled="true"/>
          </servers>
          <clients>
            <logs>logs-%i</logs>
          </clients>
        </tc:tc-config>


To successfully configure a Terracotta Server Array using the Terracotta configuration file, note the following:

* Two or more servers should be defined in the &lt;servers&gt; section of the Terracotta configuration file.

* &lt;tsa-group-port&gt; is the port used by the Terracotta server to communicate with other Terracotta servers.

* Under &lt;servers>, use either &lt;server> or &lt;mirror-group> configurations, but not a mixture. You may configure multiple servers or multiple mirror groups. &lt;server> instances under &lt;servers> work together as a mirror group. To create more than one stripe, use &lt;mirror-group> instances.

* Terracotta server instances must not share data directories. Each server&#x27;s &lt;data&gt; element should point to a different and preferably local data directory.

* For data persistence, configure fast restartability. Enabling &lt;restartable> means that the shared in-memory data is backed up and, in case of failure, it is automatically restored. Setting &lt;restartable> to "false" or omitting the &lt;restartable> element are two ways to configure no persistence.

* Each server requires an off-heap store, which allows all data to be stored in-memory, limited only by the amount of memory in your server. The minimum &lt;offheap> size is 4 GB. Additional hybrid storage is optional. Specify the &lt;dataStorage> size and the &lt;offheap> size. The &lt;offheap> size can be set to the amount of memory available in your server for data. If you enable &lt;hybrid>, then the &lt;dataStorage> size can exceed the &lt;offheap> size.

* All servers and clients should be running the same version of Terracotta and Java.


<a id="18319"></a>
## How Terracotta Servers Get Configured {#pgfId-998357}

At startup, Terracotta servers load their configuration from one of the following sources:

* A default configuration included with the Terracotta kit
* A local or remote XML file

These sources are explored below.

### Default Configuration {#pgfId-998366}

If no configuration file is specified *and* no `tc-config.xml` exists in the directory in which the Terracotta instance is started, then default configuration values are used.


### Local XML File (Default) {#pgfId-998369}

The file `tc-config.xml` is used by default if it is located in the directory in which a Terracotta instance is started *and* no configuration file is explicitly specified.


### Local or Remote Configuration File {#pgfId-998372}

You can explicitly specify a configuration file by passing the -f option to the script used to start a Terracotta server. For example, to start a Terracotta server on UNIX/Linux using the provided script, enter:

~~~
start-tc-server.sh -f <path_to_configuration_file>
~~~

where &lt;path_to_configuration_file&gt; can be a URL or a relative directory path. In Microsoft Windows, use `start-tc-server.bat`.

**Note**: Cygwin (on Windows) is not supported for this feature.


## How Terracotta Clients Get Configured

At startup, Terracotta clients load their configuration from one of the following sources:

* <a href="#29148">Local or Remote XML File</a>
* <a href="#70775">Terracotta Server</a>
* An Ehcache configuration file (using the [`<terracottaConfig>`](/documentation/4.1/bigmemorymax/configuration/configuration) element) used with BigMemory Max and BigMemory Go.
* A Quartz properties file (using the `org.quartz.jobStore.tcConfigUrl` property) used with Quartz Scheduler.
* A Filter (in `web.xml`) element used with containers and Terracotta Web Sessions.
* The client constructor (`TerracottaClient()`) used when a client is instantiated programmatically using the Terracotta Toolkit.

Terracotta clients can load customized configuration files to specify &lt;client&gt; and &lt;application&gt; configuration. However, the &lt;servers&gt; block of every client in a cluster must match the &lt;servers&gt; block of the servers in the cluster. If there is a mismatch, the client will emit an error and fail to complete its startup. However, there are options you can set for [How Server Settings Can Override Client Settings](/documentation/4.1/bigmemorymax/configuration/reference-guide#how-server-settings-affect-client-settings).

<table>
<caption>NOTE:  Error with Matching Configuration Files</caption>
<tr>
<td>
On startup, a Terracotta client may emit a configuration-mismatch error if its &lt;servers> block does not match that of the server it connects to. However, under certain circumstances, this error may occur even if the &lt;servers> blocks appear to match.

The following suggestions may help prevent this error:

- Use `-Djava.net.preferIPv4Stack` consistently. If it is explicitly set on the client, be sure to explicitly set it on the server.

- Ensure `etc/hosts` file does not contain multiple entries for hosts running Terracotta servers.

- Ensure that DNS always returns the same address for hosts running Terracotta servers.
</td>
</tr>
</table>


### Local or Remote XML File {#29148}

See the discussion for local XML file (default) in <a href="#18319">How Terracotta Servers Get Configured</a>.

To specify a configuration file for a Terracotta client, see <a href="#36126">Clients in Development</a>.

<table>
<caption>NOTE:  Fetching Configuration from the Server</caption>
<tr>
<td>
On startup, Terracotta clients must fetch certain configuration properties from a Terracotta server. A client loading its own configuration will attempt to connect to the Terracotta servers named in that configuration. If none of the servers named in that configuration are available, the client cannot complete its startup.
</td>
</tr>
</table>


<a id="998396"></a>
### Terracotta Server {#pgfId-998396}

Terracotta clients can load configuration from an active Terracotta server by specifying its hostname and TSA port (see <a href="#98555">Clients in Production</a>).


## Configuration in a Development Environment {#pgfId-998399}

In a development environment, using a different configuration file for each Terracotta client facilitates the testing and tuning of configuration options. This is an efficient and effective way to gain valuable insight on best practices for clustering your application with Terracotta.


### One-Server Setup in Development

For one Terracotta server, the default configuration is adequate.

<img src="/images/documentation/tc-config-dev-A.png" alt="Terracotta development cluster with one server." class="panel" >


To use the default configuration settings, start your Terracotta server using the `start-tc-server.sh` (or `start-tc-server.bat)` script in a directory that does *not* contain the file `tc-config.xml`:

~~~
[PROMPT] ${TERRACOTTA_HOME}\bin\start-tc-server.sh
~~~

To specify a configuration file, use one of the approaches discussed in <a href="#18319">How Terracotta Servers Get Configured</a>.


### Two-Server Setup in Development {#pgfId-998416}

 A two-server setup, sometimes referred to as an active-mirror setup, has one active server instance and one "hot standby" (the mirror) that should load the same configuration file.

<img src="/images/documentation/tc-config-dev-AP.png" alt="Terracotta development cluster with two servers." />

The configuration file loaded by the Terracotta servers must define each server separately using &lt;server&gt; elements. For example:

    <tc:tc-config xsi:schemaLocation="http://www.terracotta.org/schema/terracotta-9.xsd"
    xmlns:tc="http://www.terracotta.org/config"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
    ...
    <!-- Use an IP address or a resolvable host name for the host attribute. -->
      <server host="123.456.7.890" name="Server1">
    ...
      <server host="myResolvableHostName" name="Server2">
    ...
    </tc:tc-config>

<br class="clear"/>

Assuming Server1 is the active server, using the same configuration allows Server2 to be the mirror and maintain the environment in case of failover. If you are running both Terracotta servers on the same host, the only port that has to be specified in configuration is the &lt;tsa-port>; the values for &lt;jmx-port> and &lt;tsa-group-port> are filled in automatically.

<table>
<caption>NOTE:  Running Two Servers on the Same Host</caption>
<tr>
<td>
If you are running the servers on the same machine, some elements in the &lt;server&gt; section, such as &lt;tsa-port&gt; and &lt;server-logs&gt;, must have different values for each server.
</td>
</tr>
</table>


#### Server Names for Startup {#pgfId-998437}

With multiple &lt;server&gt; elements, the name attribute may be required to avoid ambiguity when starting a server:

~~~
start-tc-server.sh -n Server1 -f <path_to_configuration_file>
~~~

In Microsoft Windows, use `start-tc-server.bat`.

For example, if you are running Terracotta server instances on the same host, you must specify the name attribute to set an unambiguous target for the script.

However, if you are starting Terracotta server instances in an unambiguous setup, specifying the server name is optional. For example, if the Terracotta configuration file specifies different IP addresses for each server, the script assumes that the server with the IP address corresponding to the local IP address is the target.


### Clients in Development {#36126}

You can explicitly specify a client's Terracotta configuration file by passing `-Dtc.config=path/to/my-tc-config.xml` when you start your application with the Terracotta client.

~~~
 -Dtc.config=path/to/my-tc-config.xml -cp classes myApp.class.Main
~~~

where `myApp.class.Main` is the class used to launch the application you want to cluster with Terracotta.

If `tc-config.xml` exists in the directory in which you run Java, it can be loaded without `-Dtc.config`.



## Configuration in a Production Environment {#pgfId-998467}

For an efficient production environment, it's recommended that you maintain one Terracotta configuration file. That file can be loaded by the Terracotta server (or servers) and pushed out to clients. While this is an optional approach, it's an effective way to centralize and decrease maintenance.

<img src="/images/documentation/tc-config-prod-AP.png" alt="Terracotta production cluster with two servers." />

If your Terracotta configuration file uses "%i" for the hostname attribute in its server element, change it to the actual hostname in production. For example, if in development you used the following:

    <server host="%i" name="Server1">

and the production host's hostname is myHostName, then change the host attribute to the myHostName:

    <server host="myHostName" name="Server1">


<a id="98555"></a>
### Clients in Production {#pgfId-998485}

For clients in production, you can set up the Terracotta environment before launching your application.


<a id="99727"></a>
#### Setting Up the Terracotta Environment {#pgfId-998488}

To start your application with the Terracotta client using your own scripts, first set the following environment variables:

    TC_INSTALL_DIR=<path_to_local_Terracotta_home>
    TC_CONFIG_PATH=<path/to/tc-config.xml>

or

    TC_CONFIG_PATH=<server_host>:<tsa-port>

where `<server_host>:<tsa-port>` points to the running Terracotta server. The specified Terracotta server will push its configuration to the Terracotta client.

Alternatively, a client can specify that its configuration come from a server by setting the _tc.config_ system propery:

~~~
-Dtc.config=serverHost:tsaPort
~~~

If more than one Terracotta server is available, enter them in a comma-separated list:

    TC_CONFIG_PATH=<server_host1>:<tsa-port>,<server_host2>:<tsa-port>

If &lt;server_host1&gt; is unavailable, &lt;server_host2&gt; is used.



#### Terracotta Products

Terracotta products can set a configuration path using their own configuration files.

For BigMemory Max and BigMemory Go, use the `<terracottaConfig>` element in the Ehcache configuration file (`ehcache.xml` by default):

    <terracottaConfig url="localhost:9510" />

For Quartz, use the `org.quartz.jobStore.tcConfigUrl` property in the Quartz properties file (`quartz.properties` by default):

    org.quartz.jobStore.tcConfigUrl = /myPath/to/tc-config.xml

For Terracotta Web Sessions, use the appropriate elements in `web.xml` or `context.xml` (see <a href="/documentation/4.1/web-sessions/installation-guide">Web Sessions Installation</a>).


## Binding Ports to Interfaces

Normally, the ports you specify for a server in the Terracotta configuration are bound to the interface associated with the host specified for that server. For example, if the server is configured with the IP address "12.345.678.8" (or a hostname with that address), the server's ports are bound to that same interface:

    <server host="12.345.678.8" name="Server1">
     ...
     <tsa-port>9510</tsa-port>
     <jmx-port>9520</jmx-port>
     <tsa-group-port>9530</tsa-group-port>
    </server>

However, in certain situations it may be necessary to specify a different interface for one or more of a server's ports. This is done using the `bind` attribute, which allows you bind a port to a different interface. For example, a JMX client may only be able connect to a certain interface on a host. The following configuration shows a JMX port bound to an interface different than the host's:

    <server host="12.345.678.8" name="Server1">
     ...
     <tsa-port>9510</tsa-port>
     <jmx-port bind="12.345.678.9">9520</jmx-port>
     <tsa-group-port>9530</tsa-group-port>
    </server>



## Which Configuration? {#pgfId-998530}

Each server and client must maintain separate log directories. By default, server logs are written to `%(user.home)/terracotta/server-logs` and client logs to `%(user.home)/terracotta/client-logs`.

To find out which configuration a server or client is using, search its logs for an INFO message containing the text "Configuration loaded from".
