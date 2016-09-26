---
---
# Starting up TSA or CLC as Windows Service using the Service Wrapper

{toc|2:3}

A Windows service supports scheduling and automatic start and restart. You might want to run the Terracotta Server Array or the Cross-Language Connector, which are Java applications, as a __Windows service__. If so, use the Service Wrapper located inside the kit at `$installdir/server/wrapper` (or, for 3.7, `$installdir/wrapper`).

Set JAVA_HOME
-------------------------------
To start the service, set your `JAVA_HOME` in `conf/wrapper-tsa.conf` or `conf/wrapper-clc.conf`. For example:

~~~
set.JAVA_HOME=C:/Java/jdk1.7.0_21
~~~
The wrapper does not read your `JAVA_HOME` from the environment. For Windows, if you do not want to set it in the configuration file, comment it out and set `JAVA_HOME` in the registry instead.


Configuration Files
-----------------------------------------
For the Cross-Language Connector, you need these configuration files:

 * conf/cross-language-config.xml
 * conf/ehcache.xml


For the TSA, you need this configuration file:

 * conf/tc-config.xml

Overwrite those files with your own. If you want to change these file names, modify the names in the wrapper configurations.

Modify the `TSA conf/wrapper-tsa.conf` file to match the server name in your tc-config.xml:

~~~
set.SERVER_NAME=server0
~~~

where _server0_ represents the name of the server you want to start.    


Set Permissions
-------------------------
The services are controlled by an Administrator user, so you have to confirm for every action, such as install, start, stop, remove.

In addition, the Administrator user needs to have read/write permission for the "wrapper" directory.


Install and Start the Service
------------------------------------
The wrapper service is located at `$installdir/server/wrapper` .

To install the service wrapper, run the script with the install parameter:

    %> bin/tsa-service.bat install

NOTE: The examples in this section show the TSA script. For the Cross-Language Connector, use the `clc-service` or `clc-service.bat` script.

Then you can either start/stop the service:

    %> bin/tsa-service.bat start
    %> bin/tsa-service.bat stop

If you want to remove the service:

    %> bin/tsa-service.bat remove

There are more commands available when you run the script without any parameter:

    %> bin/tsa-service.bat


Changing Wrapper Configuration
--------------------------------------------------------------------
There are comments in `wrapper-tsa.conf` and `wrapper-clc.conf` to explain each parameter.
If you need to modify JVM system properties, classpath, or command line parameters,
follow the current pattern. Pay close attention to their numerical
order and parameter counts.

For more information, see [http://wrapper.tanukisoftware.com/doc/english/properties.html](http://wrapper.tanukisoftware.com/doc/english/properties.html)
