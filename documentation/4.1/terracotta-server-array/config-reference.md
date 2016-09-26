---
---
# Terracotta Configuration Reference
{toc|2:3}

## Introduction
This document is a reference to all of the Terracotta configuration elements in the Terracotta configuration file, which is named `tc-config.xml` by default.

You can use a sample configuration file provided in the kit as the basis for your Terracotta configuration. Some samples have inline comments describing the configuration elements. Be sure to start with a clean file for your configuration.

The Terracotta configuration XML document is divided into the sections &lt;servers> and &lt;clients>. Each of these sections provides a number of configuration options relevant to its particular configuration topic.

<a id="config_modules"></a><a id="integration_modules"></a><a id="terracotta_integration_modules"></a>
## Configuration Variables
Certain variables can be used that are interpolated by the configuration subsystem using local values:

<table>
<tr><th>Variable</th><th>Interpolated Value</th></tr>
<tr><td>%h</td><td>The fully-qualified hostname</td></tr>
<tr><td>%i</td><td>The IP address</td></tr>
<tr><td>%o</td><td>The operating system</td></tr>
<tr><td>%v</td><td>The version of the operating system</td></tr>
<tr><td>%a</td><td>The CPU architecture</td></tr>
<tr><td>%H</td><td>The home directory of the user running the application</td></tr>
<tr><td>%n</td><td>The username of the user running the application</td></tr>
<tr><td>%t</td><td>The path to the temporary directory (for example, /tmp on *NIX)</td></tr>
<tr><td>%D</td><td>Time stamp (yyyyMMddHHmmssSSS)</td></tr>
<tr><td>%(_system property_)</td><td>The value of the given Java system property</td></tr>
</table>

These variables can be used where appropriate, including for elements or attributes that expect strings or paths for values:

* the "name", "host" and "bind" attributes of the `<server>` element
* the password file location for JMX authentication
* client logs location
* server logs location
* server data location

<table>
<caption>NOTE: Value of %i</caption>
<tr><td>The variable %i is expanded into a value determined by the host's networking setup. In many cases that setup is in a <code>hosts</code> file containing mappings that may influence the value of %i. Test this variable in your production environment to check the value it interpolates.</td></tr>
</table>

## Using Paths as Values
Some configuration elements take paths as values. Relative paths are interpreted relative to the current working directory (the directory from which the server was started). Specifying an absolute path is recommended.

<a id="override_props"></a>
## Overriding tc.properties

Every Terracotta installation has a default `tc.properties` file containing system properties. Normally, the settings in `tc.properties` are pre-tuned and should not be edited.

If tuning is required, you can override certain properties in `tc.properties` using `tc-config.xml`. This can make a production environment more efficient by allowing system properties to be pushed out to clients with `tc-config.xml`. Those system properties would normally have to be configured separately on each client.

### Setting System Properties in tc-config
To set a system property with the same value for all clients, you can add it to the Terracotta server's `tc-config.xml` file using a configuration element with the following format:


    <property name="<tc_system_property>" value="<new_value>" />


All `<property />` tags must be wrapped in a `<tc-properties>` section placed at the beginning of `tc-config.xml`.

For example, to override the values of the system properties `l1.cachemanager.enabled` and `l1.cachemanager.leastCount`, add the following to the beginning of `tc-config.xml`:

    <tc-properties>
      <property name="l1.cachemanager.enabled" value="false" />
      <property name="l1.cachemanager.leastCount" value="4" />
    </tc-properties>


### Override Priority
System properties configured in `tc-config.xml` override the system properties in the default `tc.properties` file provided with the Terracotta kit. The default `tc.properties` file should *not* be edited or moved.

If you create a _local_ `tc.properties` file in the Terracotta `lib` directory, system properties set in that file are used by Terracotta and will override system properties in the _default_ `tc.properties` file. System properties in the local `tc.properties` file are *not* overridden by system properties configured in `tc-config.xml`.

System property values passed to Java using -D override all other configured values for that system property. In the example above, if `-Dcom.tc.l1.cachemanager.leastcount=5` was passed at the command line or through a script, it would override the value in `tc-config.xml` and `tc.properties`. The order of precedence is shown in the following list, with highest precedence shown last:

1. default `tc.properties`
2. `tc-config.xml`
3. local, or user-created `tc.properties` in Terracotta `lib` directory
4. Java system properties set with -D

### Failure to Override
If system properties set in `tc-config.xml` fail to override default system properties, a warning is logged to the Terracotta logs. The warning has the following format:

~~~
The property <system_property_name> was set by local settings to <value>.
This value will not be overridden to <value> from the tc-config.xml file.
~~~

System properties used early in the Terracotta initialization process may fail to be overridden. If this happens, a warning is logged to the Terracotta logs. The warning has the following format:

~~~
The property <system_property_name> was read before initialization completed.
~~~

The warning is followed by the value assigned to `<system_property_name>`.

**NOTE:** The property `tc.management.mbeans.enabled` is known to load before initialization completes and cannot be overridden.


<a id="servers"></a>
## Servers Configuration Section
This section contains the information that defines and configures the Terracotta Server Array (TSA) and its component servers.

### /tc:tc-config/servers
This section defines the Terracotta server instances present in your cluster. One or more entries can be defined, either directly under the &lt;servers> element or in [mirror groups](#mirror-groups). If this section is omitted, Terracotta configuration behaves as if there's a single server instance with default values.

This section also defines certain global settings that affect all servers, including the attribute `secure`. This is a global control for enabling ("true") or disabling ("false" DEFAULT) [SSL-based security](tsa-security) for the entire cluster.


### /tc:tc-config/servers/server

A server stanza encapsulates the configuration for a Terracotta server instance.  The server element takes three optional attributes (see table below).

<table>
<tr><th>Attribute</th><th>Definition</th><th>Value</th><th>Default Value</th></tr>
<tr><td>host</td>
<td>The address of the machine hosting the Terracotta server</td>
<td>Host machine's IP address or resolvable hostname</td>
<td>Host machine's IP address</td></tr>
<tr><td>name</td>
<td>The symbolic name of the Terracotta server; can be passed to Terracotta scripts such as start-tc-server using <code>-n &lt;name></code> </td>
<td> user-defined string</td>
<td><host>:<tsa-port></td></tr>
<tr><td>bind</td>
<td>The network interface on which the Terracotta server listens cluster traffic; 0.0.0.0 specifies all interfaces</td><td>interface's IP address</td>
<td>0.0.0.0</td></tr>
</table>

Each Terracotta server instance needs to know which configuration it should use as it starts up. If the server's configured name is the same as the hostname of the host it runs on and no host contains more than one server instance, then configuration is found automatically.

For more information on how to use the Terracotta configuration file with servers, see the [Terracotta configuration guide](configuration-guide).

*Sample configuration snippet:*

    <server>
      <!-- my host is '%i', my name is '%i:tsa-port', my bind is 0.0.0.0 -->
      ...
    </server>
    <server host="myhostname">
      <!-- my host is 'myhostname', my name is 'myhostname:tsa-port', my bind is 0.0.0.0 -->
      ...
    </server>
    <server host="myotherhostname" name="server1" bind="192.168.1.27">
      <!-- my host is 'myotherhostname', my name is 'server1', my bind is 192.168.1.27 -->
      ...
    </server>

<a id="server-data"></a><a id="persistence"></a>
### /tc:tc-config/servers/server/data
This element specifies the path where the server should store its data for persistence.

**Default:** data (creates the directory `data` under the working directory)

<a id="server-logs"></a>
### /tc:tc-config/servers/server/logs
This section lets you declare where the server should write its logs.

**Default:** logs (creates the directory `logs` under the working directory)

You can also specify `stderr:` or `stdout:` as the output destination for log messages. For example:

    <logs>stdout:</logs>

To set the logging level, see [this FAQ entry](/documentation/bigmemorymax/technical-faq#log-level).


### /tc:tc-config/servers/server/index
This element specifies the path where the server should store its search indexes.

**Default:** index (creates the directory `index` under the working directory)

### /tc:tc-config/servers/server/data-backup
This element specifies the path where the server should store backups (if a backup call is initiated).

**Default:** data-backup (creates the directory `data-backup` under the working directory)

<a id="tsa-port"></a>
### /tc:tc-config/servers/server/tsa-port
This section lets you set the port that the Terracotta server listens to for client traffic.

The default value of "tsa-port" is 9510.

*Sample configuration snippet:*

    <tsa-port>9510</tsa-port>


<a id="jmx-port"></a>
### /tc:tc-config/servers/server/jmx-port
This section lets you set the port that the Terracotta server's JMX Connector listens to.

The default value of "jmx-port" is 9520. If tsa-port is set, this port defaults to the value of the tsa-port plus 10.

*Sample configuration snippet:*

    <jmx-port>9520</jmx-port>


<a id="tsa-group-port"></a>
### /tc:tc-config/servers/server/tsa-group-port
This section lets you set the port that the Terracotta server uses to communicate with other Terracotta servers.

The default value of "tsa-group-port" is 9530. If tsa-port is set, this port defaults to the value of the tsa-port plus 20.

*Sample configuration snippet:*

    <tsa-group-port>9530</tsa-group-port>


### /tc:tc-config/servers/server/security
This section contains the data necessary for running a secure cluster based on SSL, digital certificates, and node authentication and authorization.

See the [advanced-security page](tsa-security) for a configuration example.


#### /tc:tc-config/servers/server/security/ssl/certificate
The element specifying certificate entry and location of the certificate store. The format is:

~~~
<store-type>:<certificate-alias>@</path/to/keystore.file>
~~~

The Java Keystore (JKS) type is supported by Terracotta 3.7 and higher.

#### /tc:tc-config/servers/server/security/keychain
This element contains the following subelements:

* `<class>` &ndash; Element specifying the class defining the keychain file. If a class is not specified, `com.terracotta.management.keychain.FileStoreKeyChain` is used.
* `<url>` &ndash; The URI for the keychain file. It is passed to the keychain class to specify the keychain file.
* `<secret-provider>` &ndash; The fully qualified class name of the user implementation of `com.terracotta.management.security.SecretProviderBackEnd`. This class can read and provide the keychain file.


#### /tc:tc-config/servers/server/security/auth
This element contains the following subelements:

* `<realm>` &ndash; Element specifying the class defining the security realm. If a class is not specified, `com.tc.net.core.security.ShiroIniRealm` is used.
* `<url>` &ndash; The URI for the Realm configuration (.ini) file. It is passed to the realm class to specify authentication file. Alternatively, URIs for LDAP or Microsoft Active directory can also be used if one of these schemes is implemented instead.
* `<user>` &ndash; The username that represents the server and is authenticated by other servers. This name is part of the credentials stored in the .ini file. The default value is "terracotta".


#### /tc:tc-config/servers/server/security/management
This element contains the subelements needed to allow the Terracotta Management Server (TMS) to make a secure connection to the TSA:

* `ia` &ndash; The HTTPS URL with the domain of the TMS, followed by the port 9443 and the path `/tmc/api/assertIdentity`.
* `timeout` &ndash; The timeout value (in milliseconds) for connections from the server to the TMS.
* `hostname` &ndash; Used only if the DNS hostname of the server does not match server hostname used in its certificate. If there is a mismatch, enter the DNS address of the server here.


<a id="server-authentication"></a>
### /tc:tc-config/servers/server/authentication
Turn on JMX authentication for the Terracotta server. An empty tag (`<authentication />`) defaults to the standard Java JMX authentication mechanism referring to password and access files in `$JAVA_HOME/jre/lib/management`:

    $JAVA_HOME/jre/lib/management/jmxremote.password
    $JAVA_HOME/jre/lib/management/jmxremote.access

You must modify these files as as follows (or, if none exist create them).

**jmxremote.password**
Add a line to the end of the file declaring a username and password followed by a carriage return:

    secretusername secretpassword

**jmxremote.access**
Add the following line (with a carriage return) to the end of your file:

    secretusername      readwrite

Be sure to assign the appropriate permissions to the file. For example, in *NIX:

    $ chmod 500 jmxremote.password
    $ chown <user who will run the server> jmxremote.password

For information on alternatives to JMX authentication, see [Terracotta Cluster Security](/documentation/bigmemorymax/terracotta-server-array/managing-security).

Note that version 4.x does not support HTTP authentication.

### /tc:tc-config/servers/dataStorage

This configuration block includes the required `offheap` element, as well as the optional `hybrid` element.

* `<offheap>` must be configured for each server; the minimum amount is 4 GB.

* `<dataStorage>` specifies the maximum amount of data to store on each server, and represents the hybrid sum of off-heap DRAM plus flash SSD.

* `<hybrid/>`, if included, enables the Hybrid option.

*Sample configuration snippet:*

    <dataStorage size=”800g”>
       <offheap size=”200g”/>
       <hybrid/>
    </dataStorage>

If the `hybrid` element is present, then `dataStorage` size may exceed `offheap` size, however if the hybrid element is not present, then the `dataStorage` size must be less than or equal to the `offheap` size.


<a id="mirror-groups"></a>
### /tc:tc-config/servers/mirror-group
A mirror group is a *stripe* in a TSA, consisting of one active server and one or more mirror (or backup) servers. A configuration that does not use the &lt;mirror-group> element would produce a one-stripe TSA:

    <servers>
      <server name="A">
      ...
      </server>
      <server name="B">
      ...
      </server>
      <server name="C">
      ...
      </server>
      <server name="D">
      ...
      </server>
    ...
    </servers>

One of the named servers would assume the role of active (the one started first or that wins the election), while the remaining servers become mirrors. Note that in a typical stripe, having only one or two mirrors is sufficient and less taxing on the active server's resources (as it needs to sync with each mirror).

The following example shows the same servers split into two stripes:

    <servers>
      <mirror-group group-name="team1">
        <server name="A">
        ...
        </server>
        <server name="B">
        ...
        </server>
      </mirror-group>
      <mirror-group group-name="team2">
        <server name="C">
        ...
        </server>
        <server name="D">
        ...
        </server>
      </mirror-group>
    ...
    </servers>

Each stripe will have one active and one mirror server.

<table>
<caption>NOTE: Server vs. Mirror Group</caption>
<tr>
<td>
Under &lt;servers>, you may use either &lt;server> or &lt;mirror-group> configurations, but not both. All &lt;server> configurations directly under &lt;servers> work together as one mirror group, with one active server and the rest mirrors. To create more than one stripe, use &lt;mirror-group> configurations directly under &lt;servers>. The mirror group configurations then include one or more &lt;server> configurations.
</td>
</tr>
</table>

For more examples and information, see the [Terracotta Configuration Guide](/documentation/bigmemorymax/terracotta-server-array/configuration-guide).


<a id="garbage-collection"></a>
<a id="dgc"></a>
### /tc:tc-config/servers/garbage-collection
This section lets you configure the periodic distributed garbage collector (DGC) that runs in the TSA. The DGC collects shared data made garbage by Java garbage collection.

For many use cases, there is no need to enable periodic DGC. For caches, the more efficient automatic inline DGC is normally sufficient for clearing garbage. In addition, certain read-heavy applications will never require the periodic DGC as little shared data becomes garbage.

However, in certain situations such as when Terracotta Toolkit data structures are in use, the periodic DGC may need to be enabled. Inline DGC is not available for these data structures.

For more on how DGC functions, see [TSA Architecture](server-arrays#about-distributed-garbage-collection).

*Configuration snippet:*

    <garbage-collection>

     <!--    Default: false -->
      <enabled>true</enabled>

      <!-- If "true", additional information is logged when a
         server performs distributed garbage collection.

         Default: false
      -->
      <verbose>false</verbose>

      <!-- How often should distributed garbage collection
         be performed, in seconds?

         Default: 3600 (60 minutes)
      -->
      <interval>3600</interval>
    </garbage-collection>


### /tc:tc-config/servers/restartable
The fast-restart persistence mechanism must be explicitly enabled for the TSA using this element:

    <restartable enabled="true"/>

In case of TSA failure, fast-restart persistence allows the TSA to reload all shared cluster data.

To function, this feature requires &lt;offheap> to be enabled on each server. To make backups of TSA data, the backup feature requires this feature to be enabled.

<a id="client-reconnect-window"></a>
### /tc:tc-config/servers/client-reconnect-window
This section lets you declare the window of time servers will allow disconnected clients to reconnect to the cluster as the same client. Outside of this window, a client can only rejoin as a new client. The value is specified in seconds and the default is 120 seconds.

If adjusting value, note that a too-short reconnection window can lead to unsuccessful reconnections during failure recovery, while a too-long window lowers the efficiency of the cluster since it is paused for the time the window is in effect.

*Further reading:*
For more information on how client and server reconnection is executed in a Terracotta cluster, and on tuning reconnection properties in a high-availability environment, see [Configuring Terracotta Clusters For High Availability](/documentation/4.1/terracotta-server-array/high-availability).


## Clients Configuration Section

The clients section contains configuration about how clients should behave.

### /tc:tc-config/clients/logs

This section lets you configure where the Terracotta client writes its logs.

*Sample configuration snippet:*


    <!--     
       This value undergoes parameter substitution before being used;
       thus, a value like 'client-logs-%h' would expand to
       'client-logs-banana' if running on host 'banana'. See the
       Product Guide for more details.

       If this is a relative path, then it is interpreted relative to
       the current working directory of the client (that is, the directory
       you were in when you started the program that uses Terracotta
       services). It is thus recommended that you specify an absolute
       path here.


       Default: 'logs-%i'; this places the logs in a directory relative
       to the directory you were in when you invoked the program that uses
       Terracotta services (your client), and calls that directory, for example,
       'logs-10.0.0.57' if the machine that the client is on has assigned IP
       address 10.0.0.57.
    -->
    <logs>logs-%i</logs>


You can also specify `stderr:` or `stdout:` as the output destination for log messages. For example:

    <logs>stdout:</logs>

To set the logging level, see [this FAQ entry](/documentation/4.1/bigmemorymax/technical-faq#log-level).
