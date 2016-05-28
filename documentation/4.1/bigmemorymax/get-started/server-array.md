---
---
#Adding the Terracotta Server Array

The Terracotta Server Array is an array of in-memory data servers that
plug in seamlessly to provide applications with high-performance, low and
predictable latency data access to terabytes of in-memory data and
enterprise capabilities like high availability, scalability, fault tolerance
and more.

Let's try starting up a Terracotta server and connecting our data sets to it.

## Preparation

* Add a valid Terracotta license key to the top level of your Terracotta
  installation directory. If you don't have one yet, you can register for a
  license at <a href="/downloads/bigmemorymax" target="_blank">terracotta.org/downloads/bigmemorymax</a>
* Add the following jar in the bigmemory-max-4.x.x download kit to your
  classpath:
  * apis/toolkit/lib/terracotta-toolkit-runtime-ee-4.x.x.jar
* Add the terracotta configuration to the configuration file (see below)
* Make sure data classes serializable
* Start a Terracotta server instance

## Create Configuration File

Create a new configuration file called "ehcache-server-array.xml" in your
classpath and add a new cache called "server-array" (you can name it anything
you want, as long as you use the correct name in your code).

	<ehcache xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
		 xsi:noNamespaceSchemaLocation="http://www.ehcache.org/ehcache.xsd"
		 name="SampleConfig" maxBytesLocalHeap="64M">
	  <cache name="hello-world"/>
	  <cache name="crud"/>
	  <cache name="search">
		<searchable>
			<searchAttribute name="height"/>
			<searchAttribute name="weight"/>
			<searchAttribute name="bodyMassIndex" />
		</searchable>
	  </cache>
	  <cache name="sort">
		<searchable>
			<searchAttribute name="height"/>
			<searchAttribute name="weight"/>
		</searchable>
	  </cache>
	  <cache name="group">
		<searchable>
			<searchAttribute name="height"/>
			<searchAttribute name="gender"/>
			<searchAttribute name="bmi" expression="value.getBodyMassIndex()"/>
		</searchable>
	  </cache>
	  <cache name="server-array">
		<searchable>
			<searchAttribute name="height"/>
			<searchAttribute name="gender"/>
			<searchAttribute name="bmi" expression="value.getBodyMassIndex()"/>
		</searchable>
		  <!-- Add the terracotta element to this cache. This causes this data
		  set to be managed by the Terracotta server array. For the purposes of
		  demonstration, we set the consistency to "strong" to ensure that data
		  is always consistent across the entire distributed system. There are
		  other consistency settings that may be more suitable for different
		  data sets and applications. -->
		<terracotta consistency="strong"/>
	  </cache>
	  <!-- Add the terracottaConfig element to specify where to find the
	  configuration specific to the server array.  In this case, the configuration
	  is retrieved from the server array itself. -->
	  <terracottaConfig url="localhost:9510"/>
	</ehcache>

## Create ServerArrayTest.java

Create and compile a Java class called ServerArrayTest:

	import java.io.Serializable;
	import java.util.List;

	import net.sf.ehcache.Cache;
	import net.sf.ehcache.CacheManager;
	import net.sf.ehcache.Element;
	import net.sf.ehcache.search.Attribute;
	import net.sf.ehcache.search.Direction;
	import net.sf.ehcache.search.Query;
	import net.sf.ehcache.search.Result;
	import net.sf.ehcache.search.Results;
	import net.sf.ehcache.search.aggregator.Aggregators;

	public class ServerArrayTest {

	  public static void main(final String[] args) {

		// Create a cache manager using the factory method...
		final CacheManager cacheManager = CacheManager.newInstance(Group.class
			.getResource("/ehcache-server-array.xml"));

		// Retrieve the "sort" data set...
		final Cache dataSet = cacheManager.getCache("server-array");

		// Check to see if there's any data in it yet...
		Element person1 = dataSet.get(1L);
		if (person1 == null) {
		  System.out.println("We didn't find any data in the server array."
			  + " This must be the first execution.");
		  // The data isn't in the server array yet (this must be the first time we
		  // ran the program), so let's create some objects...
		  final Person janeAusten = new Person(1, "Jane", "Austen", "Female",
			  5 * 12 + 7, 130);
		  final Person charlesDickens = new Person(2, "Charles", "Dickens", "Male",
			  5 * 12 + 9, 160);
		  final Person janeDoe = new Person(3, "Jane", "Doe", "Female", 5 * 12 + 5,
			  145);
		  final Person alexSmith = new Person(4, "Alex", "Smith", "Other",
			  5 * 12 + 5, 160);

		  // Put the objects into the data set...
		  dataSet.put(new Element(janeAusten.getId(), janeAusten));
		  dataSet.put(new Element(charlesDickens.getId(), charlesDickens));
		  dataSet.put(new Element(janeDoe.getId(), janeDoe));
		  dataSet.put(new Element(alexSmith.getId(), alexSmith));
		} else {

		  System.out.println("We found data in the server array from a previous"
			  + " execution. No need to create new data.");

		}

		// Fetch the search attributes that we'll use in our query and when fetching
		// our results...
		Attribute<Integer> height = dataSet.getSearchAttribute("height");
		Attribute<String> gender = dataSet.getSearchAttribute("gender");
		Attribute<Float> bmi = dataSet.getSearchAttribute("bmi");

		// Create a query object. (This query is designed to select the entire data
		// set.)
		final Query query = dataSet.createQuery().addCriteria(height.gt(5 * 12));

		// Group by the gender attribute so we can perform aggregations on each
		// gender...
		query.addGroupBy(gender);
		query.includeAttribute(gender);

		// Include an aggregation of the average height and bmi of each gender in
		// the results...
		query.includeAggregator(Aggregators.average(height));
		query.includeAggregator(Aggregators.average(bmi));

		// Include the count of the members of each gender
		query.includeAggregator(Aggregators.count());

		// Make the results come back sorted by gender in alphabetical order
		query.addOrderBy(gender, Direction.ASCENDING);

		// Execute the query...
		final Results results = query.execute();

		// Print the results...
		for (Result result : results.all()) {
		  String theGender = result.getAttribute(gender);
		  List<Object> aggregatorResults = result.getAggregatorResults();
		  Float avgHeight = (Float) aggregatorResults.get(0);
		  Float avgBMI = (Float) aggregatorResults.get(1);
		  Integer count = (Integer) aggregatorResults.get(2);
		  System.out.println("Gender: " + theGender + "; count: " + count
			  + "; average height: " + avgHeight + "; average BMI: " + avgBMI);
		}
	  }

	  public static class Person implements Serializable {
		private static final long serialVersionUID = 1L;
		private final String firstName;
		private final String lastName;
		private final long id;
		private final int height;
		private final int weight;
		private String gender;

		public Person(final long id, final String firstName, final String lastName,
			final String gender, final int height, final int weight) {
		  this.id = id;
		  this.firstName = firstName;
		  this.lastName = lastName;
		  this.gender = gender;
		  this.height = height;
		  this.weight = weight;
		}

		public String getFirstName() {
		  return firstName;
		}

		public String getLastName() {
		  return lastName;
		}

		public String getGender() {
		  return gender;
		}

		public int getHeight() {
		  return height;
		}

		public int getWeight() {
		  return weight;
		}

		public float getBodyMassIndex() {
		  return ((float) weight / ((float) height * (float) height)) * 703;
		}

		public long getId() {
		  return id;
		}

		public String toString() {
		  return "[id=" + id + ", firstName=" + firstName + ", lastName="
			  + lastName + ", height=" + height + " in" + ", weight=" + weight
			  + " lbs" + ", bmi=" + getBodyMassIndex() + "]";
		}
	  }
	}

## Start a Terracotta Server Instance

In a terminal, go to the "server" directory in your BigMemory Max
distribution.  Then run the start-tc-server command:

    %> cd /path/to/bigmemory-max-4.x.x/server
	%> ./bin/start-tc-server.sh

## Execute

The first time you run the ServerArrayTest program in a terminal, there is no
data in the server array, so you should see output like this:

<pre style="overflow: auto; white-space: nowrap">
We didn't find any data in the server array. This must be the first execution.<br/>
Gender: Female; count: 2; average height: 66.0; average BMI: 22.242641<br/>
Gender: Male; count: 1; average height: 69.0; average BMI: 23.625288<br/>
Gender: Other; count: 1; average height: 65.0; average BMI: 26.622484<br/>
</pre>

If you run the ServerArrayTest program again (without restarting the Terracotta
server instance that is already running), the data will be automatically
retrieved from the server array, so you should see output like this:

<pre style="overflow: auto; white-space: nowrap">
We found data in the server array from a previous execution. No need to create new data.<br/>
Gender: Female; count: 2; average height: 66.0; average BMI: 22.242641<br/>
Gender: Male; count: 1; average height: 69.0; average BMI: 23.625288<br/>
Gender: Other; count: 1; average height: 65.0; average BMI: 26.622484<br/>
</pre>

In our default configuration, the server keeps all data in memory only (though,
you can add easily add data persistence with a fast restartable store
configuration). To clear all data, restart the server before you next run the
ServerArrayTest program.

## Next Steps

This is just a short example of using BigMemory Max with the full power of the
server array to help get you started. Keep in mind that the settings we used
here were chosen to make it as easy as possible to get started. We encourage
you to read the rest of the documentation and experiment with the different
configuration options, features and deployment topologies available to best
meet your application's performance and scale needs.
