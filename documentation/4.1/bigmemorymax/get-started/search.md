---
---
# Search

Now that you've experimented with BigMemory's basic CRUD operations, let's take
a look at some of the search capabilities.

We will create a data set of Person objects that has some searchable
attributes, including height, weight and body mass index.

## Create Configuration File

Create a new configuration file called "ehcache-search.xml" in your classpath
and add a new cache called "search."

	<ehcache xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
		 xsi:noNamespaceSchemaLocation="http://www.ehcache.org/ehcache.xsd"
		 name="SearchConfig" maxBytesLocalHeap="64M">
	  <cache name="hello-world"/>
	  <cache name="crud"/>
	  <cache name="search">
		<searchable>
			<searchAttribute name="height"/>
			<searchAttribute name="weight"/>
			<searchAttribute name="bodyMassIndex" />
		</searchable>
	  </cache>
	</ehcache>

## Create Search.java

Create and compile a Java class called Search:

	import net.sf.ehcache.Cache;
	import net.sf.ehcache.CacheManager;
	import net.sf.ehcache.Element;
	import net.sf.ehcache.search.Attribute;
	import net.sf.ehcache.search.Query;
	import net.sf.ehcache.search.Result;
	import net.sf.ehcache.search.Results;

	public class Search {

	  public static void main(final String[] args) {

		// Create a cache manager using the factory method...
		final CacheManager cacheManager = CacheManager.newInstance(Crud.class
			.getResource("/ehcache-search.xml"));

		// Retrieve the "search" data set...
		final Cache dataSet = cacheManager.getCache("search");

		// Create some searchable objects...
		final Person janeAusten = new Person(1, "Jane", "Austen", 5 * 12 + 7, 130);
		final Person charlesDickens = new Person(2, "Charles", "Dickens",
			5 * 12 + 9, 160);
		final Person janeDoe = new Person(3, "Jane", "Doe", 5 * 12 + 5, 145);

		// Put the searchable objects into the data set...
		dataSet.put(new Element(janeAusten.getId(), janeAusten));
		dataSet.put(new Element(charlesDickens.getId(), charlesDickens));
		dataSet.put(new Element(janeDoe.getId(), janeDoe));

		// Fetch the Body Mass Index (BMI) attribute...
		Attribute<Object> bmi = dataSet.getSearchAttribute("bodyMassIndex");

		// Create a query for all people with a BMI greater than 23...
		final Query query = dataSet.createQuery().addCriteria(bmi.gt(23F))
			.includeValues();

		// Execute the query...
		final Results results = query.execute();

		// Print the results...
		for (Result result : results.all()) {
		  System.out.println(result.getValue());
		}
	  }

	  public static class Person {
		private final String firstName;
		private final String lastName;
		private final long id;
		private final int height;
		private final int weight;

		public Person(long id, final String firstName, final String lastName,
			final int height, final int weight) {
		  this.id = id;
		  this.firstName = firstName;
		  this.lastName = lastName;
		  this.height = height;
		  this.weight = weight;
		}

		public String getFirstName() {
		  return firstName;
		}

		public String getLastName() {
		  return lastName;
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

## Execute

When you run the Search program in a terminal, you should see output like this:

<pre style="overflow: auto; white-space: nowrap">
[id=2, firstName=Charles, lastName=Dickens, height=69 in, weight=160 lbs, bmi=23.625288]<br/>
[id=3, firstName=Jane, lastName=Doe, height=65 in, weight=145 lbs, bmi=24.126627]
</pre>

## Next Step

[Next Step: Sorting Results &rsaquo;](sort)
