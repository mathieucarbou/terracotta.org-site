---
---
# Hello, World!

The first step to using BigMemory is to set up one or  more instances of
Ehcache. BigMemory uses Ehcache as its main programming interface.

* Download and unpack the BigMemory Max download kit.
* Add the license key (terracotta-license.key) to your classpath.
* Add the following jars in the BigMemory Max download kit to your classpath:

  * `apis/ehcache/lib/ehcache-ee-<version>.jar`
  * `apis/ehcache/lib/slf4j-api-<version>.jar`
  * `apis/ehcache/lib/slf4j-jdk-<version>.jar` - only some versions include this jar


## Create Configuration File

Create a basic configuration file, name it "ehcache.xml" and put it in your classpath:

    <ehcache xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
             xsi:noNamespaceSchemaLocation="http://www.ehcache.org/ehcache.xsd"
             name="HelloWorldConfig">
      <cache name="hello-world" maxBytesLocalHeap="64M"/>
    </ehcache>

This tells BigMemory that you have a data store called "hello-world" and that
it can use a maximum of 64 megabytes of heap in the local Java Virtual
Machine.

## Create HelloWorld.java

Create and compile a Java class called HelloWorld:

	import net.sf.ehcache.Cache;
	import net.sf.ehcache.CacheManager;
	import net.sf.ehcache.Element;

	public class HelloWorld {

		public static void main(final String[] args) {

			// Create a cache manager
			final CacheManager cacheManager = new CacheManager();

			// create the data store called "hello-world"
			final Cache dataStore = cacheManager.getCache("hello-world");

			// create a key to map the data to
			final String key = "greeting";

			// Create a data element
			final Element putGreeting = new Element(key, "Hello, World!");

			// Put the element into the data store
			dataStore.put(putGreeting);

			// Retrieve the data element
			final Element getGreeting = dataStore.get(key);

			// Print the value
			System.out.println(getGreeting.getObjectValue());
		}
    }

## Execute

When you run the program in a terminal, you will see BigMemory print out its
license and startup info, then the string "Hello, World!".

## Next Step
[Next Step: Basic Create, Read, Update and Delete (CRUD) &rsaquo;](/documentation/4.1/bigmemorymax/get-started/crud)
