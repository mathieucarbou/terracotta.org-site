---
---
# Management and Monitoring using JMX {#JMX-Management-and-Monitoring}

{toc|2:3}

## Introduction

JMX creates a standard way of instrumenting classes and making them available to a management and monitoring infrastructure. This provides an alternative to the [Terracotta Management Server](/documentation/4.1/tms/tms) for custom or third-party tools.

## JMX Overview {#jmx}
The `net.sf.ehcache.management` package contains MBeans and a `ManagementService` for JMX management of BigMemory Max. It is in a separate package so that JMX libraries are only required if you want to use it - there is no leakage of JMX dependencies
into the core Ehcache package.

Use `net.sf.ehcache.management.ManagementService.registerMBeans(...)` static method
to register a selection of MBeans to the MBeanServer provided to the method. If you wish to monitor Ehcache but not use JMX, use the existing public methods on `Cache` and `CacheStatistics`.

The Management package is illustrated in the follwing image.

![Ehcache Image](/images/documentation/management_package.png)

## MBeans {#MBeans}
BigMemory Max supports Standard MBeans. MBeans are available for the following:

* CacheManager
* Cache
* CacheConfiguration
* CacheStatistics

All MBean attributes are available to a local MBeanServer. The CacheManager MBean allows
traversal to its collection of Cache MBeans. Each Cache MBean likewise allows traversal to
its CacheConfiguration MBean and its CacheStatistics MBean.

## JMX Remoting {#JMXr-Remoting}
The Remote API allows connection from a remote JMX Agent to an MBeanServer via an <a id="MBeanServerConnection"></a>`MBeanServerConnection`.
Only `Serializable` attributes are available remotely. The following Ehcache MBean attributes are available remotely:

* limited CacheManager attributes
* limited Cache attributes
* all CacheConfiguration attributes
* all CacheStatistics attributes

Most attributes use built-in types. To access all attributes, add ehcache.jar to the remote JMX client's
classpath. For example, `jconsole -J-Djava.class.path=ehcache.jar`.

## `ObjectName` naming scheme
* CacheManager - "net.sf.ehcache:type=CacheManager,name=&lt;CacheManager>"
* Cache - "net.sf.ehcache:type=Cache,CacheManager=&lt;cacheManagerName>,name=&lt;cacheName>"
* CacheConfiguration
- "net.sf.ehcache:type=CacheConfiguration,CacheManager=&lt;cacheManagerName>,name=&lt;cacheName>"
* CacheStatistics - "net.sf.ehcache:type=CacheStatistics,CacheManager=&lt;cacheManagerName>,name=&lt;cacheName>"

## The Management Service {#Management-Service}
The `ManagementService` class is the API entry point.
![Ehcache Image](/images/documentation/ManagementService.png)
There is only one method, `ManagementService.registerMBeans` which is used to initiate JMX registration
of a CacheManager's instrumented MBeans.
The `ManagementService` is a `CacheManagerEventListener` and
is therefore notified of any new Caches added or disposed and updates the MBeanServer appropriately.
Initiated MBeans remain registered in the MBeanServer until the CacheManager shuts down, at which time
the MBeans are deregistered. This ensures correct behavior in application servers where applications are
deployed and undeployed.

<pre><code>
/**
* This method causes the selected monitoring options to be be registered
* with the provided MBeanServer for caches in the given CacheManager.
*
* While registering the CacheManager enables traversal to all of the other
*  items,
* this requires programmatic traversal. The other options allow entry points closer
* to an item of interest and are more accessible from JMX management tools like JConsole.
* Moreover CacheManager and Cache are not serializable, so remote monitoring is not
* possible * for CacheManager or Cache, while CacheStatistics and CacheConfiguration are.
* Finally * CacheManager and Cache enable management operations to be performed.
*
* Once monitoring is enabled caches will automatically added and removed from the
* MBeanServer * as they are added and disposed of from the CacheManager. When the
* CacheManager itself * shutsdown all registered MBeans will be unregistered.
*
* @param cacheManager the CacheManager to listen to
* @param mBeanServer the MBeanServer to register MBeans to
* @param registerCacheManager Whether to register the CacheManager MBean
* @param registerCaches Whether to register the Cache MBeans
* @param registerCacheConfigurations Whether to register the CacheConfiguration MBeans
* @param registerCacheStatistics Whether to register the CacheStatistics MBeans
*/
public static void registerMBeans(
   net.sf.ehcache.CacheManager cacheManager,
   MBeanServer mBeanServer,
   boolean registerCacheManager,
   boolean registerCaches,
   boolean registerCacheConfigurations,
   boolean registerCacheStatistics) throws CacheException {
</code></pre>

## JConsole Example {#JConsole-Example}
This example shows how to register CacheStatistics in the JDK platform MBeanServer, which
works with the JConsole management agent.

    CacheManager manager = new CacheManager();
    MBeanServer mBeanServer = ManagementFactory.getPlatformMBeanServer();
    ManagementService.registerMBeans(manager, mBeanServer, false, false, false, true);

CacheStatistics MBeans are then registered.
![Ehcache Image](/images/documentation/JConsoleExample.png) *CacheStatistics MBeans in JConsole*

## Hibernate statistics {#Hibernate-statistics}
If you are running Terracotta clustered caches as hibernate second-level cache provider, it is possible to access
the hibernate statistics and Ehcache stats via JMX.
`EhcacheHibernateMBean` is the main interface that exposes all the APIs via JMX. It basically extends
two interfaces -- `EhcacheStats` and `HibernateStats`. Please look into the specific interface for more
details. You may also refer to this [online tutorial](http://weblogs.java.net/blog/maxpoon/archive/2007/06/extending_the_n_2.html).


## Performance
Collection of cache statistics is not entirely free of overhead, however, the statistics API switches on/off automatically according to usage. If you need few statistics, you incur little overhead; on the other hand, as you use more statistics, you can incur more. Statistics are off by default.

## SSL-Secured JMX Monitoring

Note: For a brief overview of Terracotta security with links to individual topics, see <a href="/documentation/4.1/bigmemorymax/security-overview">Security Overview</a>.

This page documents setting up an SSL-enabled connection for remote monitoring of a Terracotta Server from a simple Java client using Java Management Extensions (JMX) technology.

Before creating this connection, be sure to complete the instructions on the [Securing Terracotta Clusters with SSL](/documentation/4.1/terracotta-server-array/tsa-security) page.


###Compile the Client

Use the sample client code below, but adapt the host, port, username, and password variables according to your setup.

		import java.util.HashMap;
		import java.util.Map;

		import javax.management.remote.JMXConnector;
		import javax.management.remote.JMXConnectorFactory;
		import javax.management.remote.JMXServiceURL;
		import javax.management.remote.rmi.RMIConnectorServer;
		import javax.rmi.ssl.SslRMIClientSocketFactory;
		import javax.rmi.ssl.SslRMIServerSocketFactory;

		public class Main {
		  public static void main(String[] args) throws Exception {
			String host = "terracotta-server-host";
			String port = "9520";
			String username = "terracotta";
			String password = "terracotta-user-password";

			Object[] credentials = { username, password.toCharArray() };

			SslRMIClientSocketFactory csf = new SslRMIClientSocketFactory();
			SslRMIServerSocketFactory ssf = new SslRMIServerSocketFactory();

			Map<String, Object> env = new HashMap<String, Object>();
			env.put(RMIConnectorServer.RMI_CLIENT_SOCKET_FACTORY_ATTRIBUTE, csf);
			env.put(RMIConnectorServer.RMI_SERVER_SOCKET_FACTORY_ATTRIBUTE, ssf);
			env.put("com.sun.jndi.rmi.factory.socket", csf);
			env.put("jmx.remote.credentials", credentials);

			JMXServiceURL serviceURL = new JMXServiceURL("service:jmx:rmi://" + host + ":" + port +
				"/jndi/rmi://" + host + ":" + port + "/jmxrmi");
			JMXConnector jmxConnector = JMXConnectorFactory.connect(serviceURL, env);

			// do some work with the JMXConnector

			jmxConnector.close();
		  }
		}

###Run the Client

After compiling your client, configure the JVM with a truststore containing your Terracotta Server's certificate. You can simply re-use the one created for the Terracotta Server (refer to [Securing Terracotta Clusters with SSL](/documentation/4.1/terracotta-server-array/tsa-security)).

		% java -Djavax.net.ssl.trustStore=/your/path/to/truststore.jks \
		  -Djavax.net.ssl.trustStorePassword=your_truststore_password \
		  Main

###About the Credentials

In the above example, the client's credentials are encoded as an array of Objects. The Object array contains the username as a String in the array's first slot, and the password as a char[] in the array's second slot. The Object array is then passed to the connection as the "jmx.remote.credentials" entry. Passing the credentials in this format is necessary to avoid an authentication failure, except for the following exception. If you are using the JConsole tool, the credentials are sent as String[]{String,String} instead of String[]{String,char[]}.

###Troubleshooting
####Password stack trace

The stack trace below indicates that the password you specified in the `javax.net.ssl.trustStorePassword` system property is not the same as in the truststore you specified in the `javax.net.ssl.trustStore` system property.


		Exception in thread "main" java.io.IOException: Failed to retrieve RMIServer stub: javax.naming.CommunicationException [Root exception is java.rmi.ConnectIOException: Exception creating connection to: localhost; nested exception is:
			java.net.SocketException: java.security.NoSuchAlgorithmException: Error constructing implementation (algorithm: Default, provider: SunJSSE, class: com.sun.net.ssl.internal.ssl.DefaultSSLContextImpl)]
			at javax.management.remote.rmi.RMIConnector.connect(RMIConnector.java:338)
			at javax.management.remote.JMXConnectorFactory.connect(JMXConnectorFactory.java:248)
			at Main.main(Main.java:31)
			at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
			at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:39)
			at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:25)
			at java.lang.reflect.Method.invoke(Method.java:597)
			at com.intellij.rt.execution.application.AppMain.main(AppMain.java:120)
		Caused by: javax.naming.CommunicationException [Root exception is java.rmi.ConnectIOException: Exception creating connection to: localhost; nested exception is:
			java.net.SocketException: java.security.NoSuchAlgorithmException: Error constructing implementation (algorithm: Default, provider: SunJSSE, class: com.sun.net.ssl.internal.ssl.DefaultSSLContextImpl)]
			at com.sun.jndi.rmi.registry.RegistryContext.lookup(RegistryContext.java:101)
			at com.sun.jndi.toolkit.url.GenericURLContext.lookup(GenericURLContext.java:185)
			at javax.naming.InitialContext.lookup(InitialContext.java:392)
			at javax.management.remote.rmi.RMIConnector.findRMIServerJNDI(RMIConnector.java:1886)
			at javax.management.remote.rmi.RMIConnector.findRMIServer(RMIConnector.java:1856)
			at javax.management.remote.rmi.RMIConnector.connect(RMIConnector.java:255)
			... 7 more
		Caused by: java.rmi.ConnectIOException: Exception creating connection to: localhost; nested exception is:
			java.net.SocketException: java.security.NoSuchAlgorithmException: Error constructing implementation (algorithm: Default, provider: SunJSSE, class: com.sun.net.ssl.internal.ssl.DefaultSSLContextImpl)
			at sun.rmi.transport.tcp.TCPEndpoint.newSocket(TCPEndpoint.java:614)
			at sun.rmi.transport.tcp.TCPChannel.createConnection(TCPChannel.java:198)
			at sun.rmi.transport.tcp.TCPChannel.newConnection(TCPChannel.java:184)
			at sun.rmi.server.UnicastRef.newCall(UnicastRef.java:322)
			at sun.rmi.registry.RegistryImpl_Stub.lookup(Unknown Source)
			at com.sun.jndi.rmi.registry.RegistryContext.lookup(RegistryContext.java:97)
			... 12 more
		Caused by: java.net.SocketException: java.security.NoSuchAlgorithmException: Error constructing implementation (algorithm: Default, provider: SunJSSE, class: com.sun.net.ssl.internal.ssl.DefaultSSLContextImpl)
			at javax.net.ssl.DefaultSSLSocketFactory.throwException(SSLSocketFactory.java:179)
			at javax.net.ssl.DefaultSSLSocketFactory.createSocket(SSLSocketFactory.java:192)
			at javax.rmi.ssl.SslRMIClientSocketFactory.createSocket(SslRMIClientSocketFactory.java:105)
			at sun.rmi.transport.tcp.TCPEndpoint.newSocket(TCPEndpoint.java:595)
			... 17 more
		Caused by: java.security.NoSuchAlgorithmException: Error constructing implementation (algorithm: Default, provider: SunJSSE, class: com.sun.net.ssl.internal.ssl.DefaultSSLContextImpl)
			at java.security.Provider$Service.newInstance(Provider.java:1245)
			at sun.security.jca.GetInstance.getInstance(GetInstance.java:220)
			at sun.security.jca.GetInstance.getInstance(GetInstance.java:147)
			at javax.net.ssl.SSLContext.getInstance(SSLContext.java:125)
			at javax.net.ssl.SSLContext.getDefault(SSLContext.java:68)
			at javax.net.ssl.SSLSocketFactory.getDefault(SSLSocketFactory.java:102)
			at javax.rmi.ssl.SslRMIClientSocketFactory.getDefaultClientSocketFactory(SslRMIClientSocketFactory.java:192)
			at javax.rmi.ssl.SslRMIClientSocketFactory.createSocket(SslRMIClientSocketFactory.java:102)
			... 18 more
		Caused by: java.io.IOException: Keystore was tampered with, or password was incorrect
			at sun.security.provider.JavaKeyStore.engineLoad(JavaKeyStore.java:771)
			at sun.security.provider.JavaKeyStore$JKS.engineLoad(JavaKeyStore.java:38)
			at java.security.KeyStore.load(KeyStore.java:1185)
			at com.sun.net.ssl.internal.ssl.TrustManagerFactoryImpl.getCacertsKeyStore(TrustManagerFactoryImpl.java:202)
			at com.sun.net.ssl.internal.ssl.DefaultSSLContextImpl.getDefaultTrustManager(DefaultSSLContextImpl.java:70)
			at com.sun.net.ssl.internal.ssl.DefaultSSLContextImpl.<init>(DefaultSSLContextImpl.java:40)
			at sun.reflect.NativeConstructorAccessorImpl.newInstance0(Native Method)
			at sun.reflect.NativeConstructorAccessorImpl.newInstance(NativeConstructorAccessorImpl.java:39)
			at sun.reflect.DelegatingConstructorAccessorImpl.newInstance(DelegatingConstructorAccessorImpl.java:27)
			at java.lang.reflect.Constructor.newInstance(Constructor.java:513)
			at java.lang.Class.newInstance0(Class.java:357)
			at java.lang.Class.newInstance(Class.java:310)
			at java.security.Provider$Service.newInstance(Provider.java:1221)
			... 25 more
		Caused by: java.security.UnrecoverableKeyException: Password verification failed
			at sun.security.provider.JavaKeyStore.engineLoad(JavaKeyStore.java:769)
			... 37 more


####Truststore stack trace

The stack trace below indicates that the truststore you specified in the `javax.net.ssl.trustStore` system property does not exist or cannot be read.


		Exception in thread "main" java.io.IOException: Failed to retrieve RMIServer stub: javax.naming.CommunicationException [Root exception is java.rmi.ConnectIOException: error during JRMP connection establishment; nested exception is:
			javax.net.ssl.SSLException: java.lang.RuntimeException: Unexpected error: java.security.InvalidAlgorithmParameterException: the trustAnchors parameter must be non-empty]
			at javax.management.remote.rmi.RMIConnector.connect(RMIConnector.java:338)
			at javax.management.remote.JMXConnectorFactory.connect(JMXConnectorFactory.java:248)
			at Main.main(Main.java:31)
			at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
			at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:39)
			at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:25)
			at java.lang.reflect.Method.invoke(Method.java:597)
			at com.intellij.rt.execution.application.AppMain.main(AppMain.java:120)
		Caused by: javax.naming.CommunicationException [Root exception is java.rmi.ConnectIOException: error during JRMP connection establishment; nested exception is:
			javax.net.ssl.SSLException: java.lang.RuntimeException: Unexpected error: java.security.InvalidAlgorithmParameterException: the trustAnchors parameter must be non-empty]
			at com.sun.jndi.rmi.registry.RegistryContext.lookup(RegistryContext.java:101)
			at com.sun.jndi.toolkit.url.GenericURLContext.lookup(GenericURLContext.java:185)
			at javax.naming.InitialContext.lookup(InitialContext.java:392)
			at javax.management.remote.rmi.RMIConnector.findRMIServerJNDI(RMIConnector.java:1886)
			at javax.management.remote.rmi.RMIConnector.findRMIServer(RMIConnector.java:1856)
			at javax.management.remote.rmi.RMIConnector.connect(RMIConnector.java:255)
			... 7 more
		Caused by: java.rmi.ConnectIOException: error during JRMP connection establishment; nested exception is:
			javax.net.ssl.SSLException: java.lang.RuntimeException: Unexpected error: java.security.InvalidAlgorithmParameterException: the trustAnchors parameter must be non-empty
			at sun.rmi.transport.tcp.TCPChannel.createConnection(TCPChannel.java:286)
			at sun.rmi.transport.tcp.TCPChannel.newConnection(TCPChannel.java:184)
			at sun.rmi.server.UnicastRef.newCall(UnicastRef.java:322)
			at sun.rmi.registry.RegistryImpl_Stub.lookup(Unknown Source)
			at com.sun.jndi.rmi.registry.RegistryContext.lookup(RegistryContext.java:97)
			... 12 more
		Caused by: javax.net.ssl.SSLException: java.lang.RuntimeException: Unexpected error: java.security.InvalidAlgorithmParameterException: the trustAnchors parameter must be non-empty
			at com.sun.net.ssl.internal.ssl.Alerts.getSSLException(Alerts.java:190)
			at com.sun.net.ssl.internal.ssl.SSLSocketImpl.fatal(SSLSocketImpl.java:1747)
			at com.sun.net.ssl.internal.ssl.SSLSocketImpl.fatal(SSLSocketImpl.java:1708)
			at com.sun.net.ssl.internal.ssl.SSLSocketImpl.handleException(SSLSocketImpl.java:1691)
			at com.sun.net.ssl.internal.ssl.SSLSocketImpl.handleException(SSLSocketImpl.java:1617)
			at com.sun.net.ssl.internal.ssl.AppOutputStream.write(AppOutputStream.java:105)
			at java.io.BufferedOutputStream.flushBuffer(BufferedOutputStream.java:65)
			at java.io.BufferedOutputStream.flush(BufferedOutputStream.java:123)
			at java.io.DataOutputStream.flush(DataOutputStream.java:106)
			at sun.rmi.transport.tcp.TCPChannel.createConnection(TCPChannel.java:211)
			... 16 more
		Caused by: java.lang.RuntimeException: Unexpected error: java.security.InvalidAlgorithmParameterException: the trustAnchors parameter must be non-empty
			at sun.security.validator.PKIXValidator.<init>(PKIXValidator.java:57)
			at sun.security.validator.Validator.getInstance(Validator.java:161)
			at com.sun.net.ssl.internal.ssl.X509TrustManagerImpl.getValidator(X509TrustManagerImpl.java:108)
			at com.sun.net.ssl.internal.ssl.X509TrustManagerImpl.checkServerTrusted(X509TrustManagerImpl.java:204)
			at com.sun.net.ssl.internal.ssl.X509TrustManagerImpl.checkServerTrusted(X509TrustManagerImpl.java:249)
			at com.sun.net.ssl.internal.ssl.ClientHandshaker.serverCertificate(ClientHandshaker.java:1188)
			at com.sun.net.ssl.internal.ssl.ClientHandshaker.processMessage(ClientHandshaker.java:135)
			at com.sun.net.ssl.internal.ssl.Handshaker.processLoop(Handshaker.java:593)
			at com.sun.net.ssl.internal.ssl.Handshaker.process_record(Handshaker.java:529)
			at com.sun.net.ssl.internal.ssl.SSLSocketImpl.readRecord(SSLSocketImpl.java:943)
			at com.sun.net.ssl.internal.ssl.SSLSocketImpl.performInitialHandshake(SSLSocketImpl.java:1188)
			at com.sun.net.ssl.internal.ssl.SSLSocketImpl.writeRecord(SSLSocketImpl.java:654)
			at com.sun.net.ssl.internal.ssl.AppOutputStream.write(AppOutputStream.java:100)
			... 20 more
		Caused by: java.security.InvalidAlgorithmParameterException: the trustAnchors parameter must be non-empty
			at java.security.cert.PKIXParameters.setTrustAnchors(PKIXParameters.java:183)
			at java.security.cert.PKIXParameters.<init>(PKIXParameters.java:103)
			at java.security.cert.PKIXBuilderParameters.<init>(PKIXBuilderParameters.java:87)
			at sun.security.validator.PKIXValidator.<init>(PKIXValidator.java:55)
			... 32 more


####Conditional stack trace

The stack trace below indicates that one of the following conditions is true:

* The username you specified does not exist.
* The password you specified is incorrect.
* The credentials field is not an Object array containing the username as a String in the array's first slot and the password as a char[] in the array's second slot.


		Exception in thread "main" java.lang.SecurityException: Username and/or password is not valid!
			at com.tc.management.EnterpriseL2Management$1.authenticate(EnterpriseL2Management.java:202)
			at javax.management.remote.rmi.RMIServerImpl.doNewClient(RMIServerImpl.java:213)
			at javax.management.remote.rmi.RMIServerImpl.newClient(RMIServerImpl.java:180)
			at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
			at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:39)
			at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:25)
			at java.lang.reflect.Method.invoke(Method.java:597)
			at sun.rmi.server.UnicastServerRef.dispatch(UnicastServerRef.java:303)
			at sun.rmi.transport.Transport$1.run(Transport.java:159)
			at java.security.AccessController.doPrivileged(Native Method)
			at sun.rmi.transport.Transport.serviceCall(Transport.java:155)
			at sun.rmi.transport.tcp.TCPTransport.handleMessages(TCPTransport.java:535)
			at sun.rmi.transport.tcp.TCPTransport$ConnectionHandler.run0(TCPTransport.java:790)
			at sun.rmi.transport.tcp.TCPTransport$ConnectionHandler.run(TCPTransport.java:649)
			at java.util.concurrent.ThreadPoolExecutor$Worker.runTask(ThreadPoolExecutor.java:895)
			at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:918)
			at java.lang.Thread.run(Thread.java:680)
			at sun.rmi.transport.StreamRemoteCall.exceptionReceivedFromServer(StreamRemoteCall.java:255)
			at sun.rmi.transport.StreamRemoteCall.executeCall(StreamRemoteCall.java:233)
			at sun.rmi.server.UnicastRef.invoke(UnicastRef.java:142)
			at javax.management.remote.rmi.RMIServerImpl_Stub.newClient(Unknown Source)
			at javax.management.remote.rmi.RMIConnector.getConnection(RMIConnector.java:2327)
			at javax.management.remote.rmi.RMIConnector.connect(RMIConnector.java:277)
			at javax.management.remote.JMXConnectorFactory.connect(JMXConnectorFactory.java:248)
			at Main.main(Main.java:31)
			at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
			at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:39)
			at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:25)
			at java.lang.reflect.Method.invoke(Method.java:597)
			at com.intellij.rt.execution.application.AppMain.main(AppMain.java:120)
