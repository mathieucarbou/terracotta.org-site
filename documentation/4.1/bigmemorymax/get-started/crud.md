---
---
# Basic CRUD

Now that you've created your first instance of BigMemory with the Ehcache
interface, let's exercise its basic CRUD functions.

## Create Configuration File

Create a new configuration file called "ehcache-crud.xml" in your classpath and
add a new cache called "crud."

	<ehcache xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
		 xsi:noNamespaceSchemaLocation="http://www.ehcache.org/ehcache.xsd"
		 name="CRUDConfig" maxBytesLocalHeap="64M">
	  <cache name="hello-world"/>
	  <cache name="crud"/>
	</ehcache>

_(Notice that the maxByteLocalHeap setting has been moved to the top-level
&lt;ehcache&gt; element&mdash;BigMemory gives you the ability to sculpt the
memomory profile of your individual data sets as well as apply bulk settings to
all data sets.)_

## Create Crud.java

Create and compile a Java class called Crud:

	import net.sf.ehcache.Cache;
	import net.sf.ehcache.CacheManager;
	import net.sf.ehcache.Element;

	public class Crud {
		public static void main(final String[] args) {
			// Create a cache manager using the factory method
			// AND specify the new configuration file
            final CacheManager cacheManager =
			      CacheManager.newInstance(
				    Crud.class.getResource("/ehcache-crud.xml"));

			// Get the "crud" cache from the cache manager...
			final Cache dataStore = cacheManager.getCache("crud");

			// Set up the first data element...
            final String myKey = "My Key";
			final String myData = "My Data";
			final Element createElement = new Element(myKey, myData);

			// CREATE data using the put(Element) method...
			dataStore.put(createElement);
			System.out.println("Created data: " + createElement);

			// READ data using the get(Object) method...
			Element readElement = dataStore.get(myKey);
			System.out.println("Read data:    " + readElement);

			// Check to make sure the data is the same...
			if (! myData.equals(readElement.getObjectValue())) {
				throw new RuntimeException("My data doesn't match!");
			}

			// UPDATE data by mapping a new value to the same key...
			final String myNewData = "My New Data";
			final Element updateElement = new Element(myKey, myNewData);
			dataStore.put(updateElement);
			System.out.println("Updated data: " + updateElement);

			// Test to see that the data is updated...
			readElement = dataStore.get(myKey);
			if (! myNewData.equals(readElement.getObjectValue())) {
				throw new RuntimeException("My data doesn't match!");
			}

			// DELETE data using the remove(Object) method...
			final boolean wasRemoved = dataStore.remove(myKey);
			System.out.println("Removed data: " + wasRemoved);
			if (! wasRemoved) {
				throw new RuntimeException("My data wasn't removed!");
			}

			// Be polite and release the CacheManager resources...
			cacheManager.shutdown();
		}
	}

# Execute

When you run the Crud program in a terminal, you should see output like this:
<pre style="overflow: auto">
Created data: [ key = My Key, value=My Data, version=1, hitCount=0,<br/>
          CreationTime = 1361401376761, LastAccessTime = 1361401376761 ]<br/>
Read data:    [ key = My Key, value=My Data, version=1, hitCount=1, <br/>
          CreationTime = 1361401376761, LastAccessTime = 1361401376775 ]<br/>
Updated data: [ key = My Key, value=My New Data, version=1, hitCount=0, <br/>
          CreationTime = 1361401376776, LastAccessTime = 1361401376776 ]<br/>
Removed data: true<br/>
</pre>

# Next Step

[Next Step: Search &rsaquo;](search)
