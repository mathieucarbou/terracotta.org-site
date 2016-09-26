---
---
#Sorting Results

Now that you've experimented with searching through data sets, let's try
sorting those results.

## Create Configuration File

Create a new configuration file called "ehcache-sort.xml" in your classpath and
add a new cache called "sort."

	<ehcache xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
		 xsi:noNamespaceSchemaLocation="http://www.ehcache.org/ehcache.xsd"
		 name="SortConfig" maxBytesLocalHeap="64M">
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
	</ehcache>

## Create Sort.java

Create and compile a Java class called Sort:

	import net.sf.ehcache.Cache;
	import net.sf.ehcache.CacheManager;
	import net.sf.ehcache.Element;
	import net.sf.ehcache.search.Attribute;
	import net.sf.ehcache.search.Direction;
	import net.sf.ehcache.search.Query;
	import net.sf.ehcache.search.Result;
	import net.sf.ehcache.search.Results;

	public class Sort {

	  public static void main(final String[] args) {

		// Create a cache manager using the factory method...
		final CacheManager cacheManager = CacheManager.newInstance(Sort.class
			.getResource("/ehcache-sort.xml"));

		// Retrieve the "sort" data set...
		final Cache dataSet = cacheManager.getCache("sort");

		// Create some objects...
		final Person janeAusten = new Person(1, "Jane", "Austen", 5 * 12 + 7, 130);
		final Person charlesDickens = new Person(2, "Charles", "Dickens",
			5 * 12 + 9, 160);
		final Person janeDoe = new Person(3, "Jane", "Doe", 5 * 12 + 5, 145);
		final Person alexSmith = new Person(4, "Alex", "Smith", 5 * 12 + 5, 160);

		// Put the objects into the data set...
		dataSet.put(new Element(janeAusten.getId(), janeAusten));
		dataSet.put(new Element(charlesDickens.getId(), charlesDickens));
		dataSet.put(new Element(janeDoe.getId(), janeDoe));
		dataSet.put(new Element(alexSmith.getId(), alexSmith));

		// Fetch the height and weight attributes that we'll use in the query...
		Attribute<Integer> height = dataSet.getSearchAttribute("height");
		Attribute<Integer> weight = dataSet.getSearchAttribute("weight");

		// Create a query for all people with a height greater than 5' 0", ordered
		// first by height, then by weight... this will retrieve the entire data
		// set, but we'll get to see how the results are sorted.
		final Query query = dataSet.createQuery().addCriteria(height.gt(5 * 12))
			.addOrderBy(height, Direction.DESCENDING)
			.addOrderBy(weight, Direction.DESCENDING).includeValues();

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

		public Person(final long id, final String firstName, final String lastName,
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

When you run the Sort program in a terminal, you should see output like this:

<pre style="overflow: auto; white-space: nowrap">
[id=2, firstName=Charles, lastName=Dickens, height=69 in, weight=160 lbs, bmi=23.625288]<br/>
[id=1, firstName=Jane, lastName=Austen, height=67 in, weight=130 lbs, bmi=20.358654]<br/>
[id=4, firstName=Alex, lastName=Smith, height=65 in, weight=160 lbs, bmi=26.622484]<br/>
[id=3, firstName=Jane, lastName=Doe, height=65 in, weight=145 lbs, bmi=24.126627]<br/>
</pre>

The results are in descending order by height.  The last two results both have
the same height, so they are further ordered by weight.

## Next Step

[Next Step: Grouping Results &rsaquo;](group)
