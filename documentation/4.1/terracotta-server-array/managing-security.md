---
---
# Cluster Security {#74096}

{toc|2:3}

## Introduction

Note: For a brief overview of Terracotta security with links to individual topics, see <a href="/documentation/4.1/bigmemorymax/security-overview">Security Overview</a>.

The Enterprise Edition of the Terracotta kit provides standard authentication mechanisms to control access to Terracotta servers. Enabling one of these mechanisms causes a Terracotta server to require credentials before allowing a JMX connection.

You can choose one of the following to secure servers:

* SSL-based security &ndash; Authenticates all nodes (including clients) and secures the entire cluster with encrypted connections. Includes role-based authorization.
* LDAP-based authentication &ndash; Uses your organization's authentication database to secure access to Terracotta servers.
* JMX-based authentication &ndash; provides a simple authentication scheme to protect access to Terracotta servers.

Note that Terracotta scripts cannot be used with secured servers without [passing credentials to the script](#using-scripts).

## Configure SSL-based Security
To learn how to use Secure Sockets Layer (SSL) encryption and certificate-based authentication to secure enterprise versions of Terracotta clusters, see the [advanced security page](tsa-security). Note that using SSL to a Terracotta cluster reduces performance due to the overhead introduced by encrypting inter-node communication.

## Configure Security Using LDAP (via JAAS)

Lightweight Directory Access Protocol (LDAP) security is based on JAAS and requires Java 1.6. Using an earlier version of Java does not prevent Terracotta servers from running, but security will *not* be enabled.

To configure security using LDAP, follow these steps:


1.  Save the following configuration to the file `.java.login.config`:

        Terracotta {
        com.sun.security.auth.module.LdapLoginModule REQUIRED
        java.naming.security.authentication="simple"
        userProvider="ldap://orgstage:389"
        authIdentity="uid={USERNAME},ou=People,dc=terracotta,dc=org"
        authzIdentity=controlRole
        useSSL=false
        bindDn="cn=Manager"
        bindCredential="****"
        bindAuthenticationType="simple"
        debug=true;
        };

    Edit the values for `userProvider` (LDAP server), `authIdentity` (user identity), and `bindCredential` (encrypted password) to match the values for your environment.

2.  Save the file `.java.login.config` to the directory named in the Java property `user.home`.
3.  Add the following configuration to each &lt;server&gt; block in the Terracotta configuration file:

        <server host="myHost" name="myServer">
        ...
         <authentication>
           <mode>
             <login-config-name>Terracotta</login-config-name>
           </mode>
         </authentication>
        ...
        </server>

4.  Start the Terracotta server and look for a log message containing "INFO - Credentials: loginConfig[Terracotta]" to confirm that LDAP security is in effect.

    <table>
    <caption>NOTE: Incorrect Setup    </caption>
    <tr>
    <td>
    If security is set up incorrectly, the Terracotta server can still be started. However, you might not be able to shut down the server using the shutdown script (**stop-tc-server**).
    </td>
    </tr>
    </table>


## Configure Security Using JMX Authentication

Terracotta can use the standard Java security mechanisms for JMX authentication, which relies on the creation of `.access` and `.password` files with correct permissions. The default location for these files for JDK 1.5 or higher is `$JAVA_HOME/jre/lib/management`.

To configure security using JMX authentication, follow these steps:

1.  Ensure that the desired usernames and passwords for securing the target servers are in the JMX password file `jmxremote.password` and that the desired roles are in the JMX access file `jmxremote.access`.
2.  If both `jmxremote.access` and `jmxremote.password` are in the default location (`$JAVA_HOME/jre/lib/management`), add the following configuration to each &lt;server&gt; block in the Terracotta configuration file:

        <server host="myHost" name="myServer">
        ...
         <authentication />
        ...
        </server>
3.  If `jmxremote.password` is not in the default location, add the following configuration to each &lt;server&gt; block in the Terracotta configuration file:

        <server host="myHost" name="myServer">
        ...
         <authentication>
           <mode>
             <password-file>/path/to/jmx.password</password-file>
           </mode>
         </authentication>
        ...
        </server>
4.  If `jmxremote.access` is not in the default location, add the following configuration to each &lt;server&gt; block in the Terracotta configuration file:

        <server host="myHost" name="myServer">
        ...
         <authentication>
           <mode>
             <password-file>/path/to/jmxremote.password</password-file>
           </mode>
           <access-file>/path/to/jmxremote.access</access-file>
         </authentication>
        ...
        </server>


#### File Not Found Error

If the JMX password file is not found when the server starts up, an error is logged stating that the password file does not exist.

##User Roles
There are two roles available for Terracotta servers and clients:

* admin – The user with the "admin" role is the initial user who sets up security. Thereafter, the "admin" user can perform system functions such as shutting down servers, clearing or deleting caches and cache managers, and reloading configurations.

* terracotta – This is the operator role. The default username for the operator role is "terracotta". The "terracotta" user can connect to the TMC and access the read-only areas. In addition, the "terracotta" user can start a secure server. But a user must have the "admin" role in order to run the stop-tc-server script.


<a id="using-scripts"></a>
## Using Scripts Against a Server with Authentication

A script that targets a secured Terracotta server must use the correct login credentials to access the server. If you run a Terracotta script such as **backup-data** or **server-stat** against a secured server, pass the credentials using the `-u` (followed by username) and `-w` (followed by password) flags.

For example, if Server1 is secured with username "user1" and password "password", run the **server-stat** script by entering the following:

##### UNIX/LINUX

~~~
[PROMPT]${TERRACOTTA_HOME}/server/bin/server-stat.sh -s Server1 -u user1 -w password
~~~


##### MICROSOFT WINDOWS

~~~
[PROMPT]%TERRACOTTA_HOME%\server\bin\server-stat.bat -s Server1 -u user1 -w password
~~~


## Extending Server Security

JMX messages are not encrypted. Therefore, server authentication does not provide secure message transmission after valid credentials are provided by a listening client. To extend security beyond the login threshold, consider the following options:


* Place Terracotta servers in a secure location on a private network.
* Restrict remote queries to an encrypted tunnel, such as one provided by SSH or stunnel.
* If using public or outside networks, use a VPN for all communication in the cluster.
* If using Ehcache, add a cache decorator to the cache that implements your own encryption and decryption.
