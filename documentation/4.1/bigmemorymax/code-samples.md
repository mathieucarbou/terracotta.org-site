---
---
#Code Samples

{toc|2:2}

This page is a companion to the code samples provided in the BigMemory Max kit. For BigMemory tutorials, visit [Hello, World!](/documentation/4.1/bigmemorymax/get-started/hello-world)

## Introduction
The following code samples illustrate various features of BigMemory Max. They
are also available in the BigMemory Max kit in the /code-samples directory.

<table>
  <thead>
    <tr>
      <th>Title</th>
      <th>Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td width="200"><a href="#example01-config-file">example01-config-file</a></td>
      <td>BigMemory may be configured declaratively,
	using an XML configuration file, or programmatically via the fluent
	configuration API. This sample shows how to configure a basic instance
	of BigMemory Max declaratively with the XML configuration file.</td>
    </tr>

    <tr>
      <td><a href="#example02-config-programmatic">example02-config-programmatic</a></td>
      <td>Configure a basic instance of BigMemory Max programmatically with the fluent
	configuration API.</td>
    </tr>

    <tr>
      <td><a href="#example03-crud">example03-crud</a></td>
      <td>Basic create, retrieve, update and delete (CRUD) operations available in
	BigMemory Max.</td>
    </tr>

    <tr>
      <td><a href="#example04-search">example04-search</a></td>
      <td>Basic in-memory search features of BigMemory Max.</td>
    </tr>

    <tr>
      <td><a href="#example05-nonstop">example05-nonstop</a></td>
      <td>
	The nonstop feature allows operations to proceed on clients that have become disconnected from the cluster. The rejoin feature then allows clients to identify a source of Terracotta configuration and rejoin a cluster. This sample demonstrates the nonstop and rejoin features of BigMemory Max.
      </td>
    </tr>
    <tr>
      <td><a href="#example06-arc">example06-arc</a></td>
      <td>
	Automatic Resource Control (ARC) is a powerful capability of BigMemory Max
	that gives users the ability to control how much data is stored in heap memory
	and off-heap memory. This sample shows the basic configuration options for data
	tier sizing using ARC.
      </td>
    </tr>
    <tr>
      <td><a href="#example07-cache">example07-cache</a></td>
      <td>BigMemory Max is a powerful in-memory data
	management solution. Among its many applications, BigMemory Max may be used as a
	cache to speed up access to data from slow or expensive databases and other
	remote data sources. This example shows how to enable and configure the caching
	features available in BigMemory Max.</td>
    </tr>
  </tbody>
</table>

To run the code samples with Maven, you will need to add the Terracotta Maven repositories to your Maven settings.xml file. Add the following repository information to your settings.xml file:

    <repository>
        <id>terracotta-repository</id>
        <url>http://www.terracotta.org/download/reflector/releases</url>
        <releases>
            <enabled>true</enabled>
        </releases>
    </repository>

For further information, refer to [Working with Apache Maven](http://terracotta.org/documentation/more/apache-maven).

<a id="example01-config-file"></a>
## Example 1: Declarative Configuration via XML

To configure BigMemory declaratively with an XML file, create a
CacheManager instance, passing the a file name or an URL object to the
constructor.

The following example shows how to create a CacheManager with an URL of the XML
file in the classpath at `/xml/ehcache.xml`.


    CacheManager manager = CacheManager.newInstance(
                              getClass().getResource("/xml/ehcache.xml"));
    try {
      Cache bigMemory = manager.getCache("BigMemory");
      // now do stuff with it...

    } finally {
      if (manager != null) manager.shutdown();
    }

Here are the contents of the XML configuration file used by this sample:

    <ehcache xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
             xsi:noNamespaceSchemaLocation="http://www.ehcache.org/ehcache.xsd"
             name="config">
      <cache name="BigMemory"
             maxBytesLocalHeap="512M"
             maxBytesLocalOffHeap="32G"
             copyOnRead="true"
             statistics="true"
             eternal="true">
      </cache>
    </ehcache>

The configuration element `maxBytesLocalOffHeap` lets you set how much off-heap
memory to use. BigMemory's unique off-heap memory storage lets you use all of
the memory available on a server in a single JVM&mdash;from gigabytes to
multiple terabytes&mdash;without causing garbage
collection pauses.

<a id="example02-config-programmatic"></a>
## Example 2: Programmatic Configuration

To configure BigMemory Max programmatically, use the [Ehcache fluent
configuration API](http://www.ehcache.org/apidocs/2.8.5/index.html).

    Configuration managerConfiguration = new Configuration()
        .name("bigmemory-config")
        .cache(new CacheConfiguration()
            .name("BigMemory")
            .maxBytesLocalHeap(512, MemoryUnit.MEGABYTES)
            .maxBytesLocalOffHeap(1, MemoryUnit.GIGABYTES)
            .copyOnRead(true)
            .statistics(true)
            .eternal(true)
        );

    CacheManager manager = CacheManager.create(managerConfiguration);
    try {
      Cache bigMemory = manager.getCache("BigMemory");
      // now do stuff with it...

    } finally {
      if (manager != null) manager.shutdown();
    }

<a id="example03-crud"></a>
## Example 3: Create, Read, Update and Delete (CRUD)
The CRUD sample demonstrates basic create, read, update and delete operations.

First, we create a BigMemory data store configured to use 128 MB of heap memory:

    Configuration managerConfiguration = new Configuration();
    managerConfiguration.name("config")
        .terracotta(new TerracottaClientConfiguration().url("localhost:9510"))
        .cache(new CacheConfiguration()
            .name("bigMemory-crud")
            .maxBytesLocalHeap(128, MemoryUnit.MEGABYTES)
            .terracotta(new TerracottaConfiguration())
        );

    CacheManager manager = CacheManager.create(managerConfiguration);
    Cache bigMemory = manager.getCache("bigMemory-crud");

Now that we have a BigMemory instance configured and available, we can start
creating data in it:

    final Person timDoe = new Person("Tim Doe", 35, Person.Gender.MALE,
        "eck street", "San Mateo", "CA");
    bigMemory.put(new Element("1", timDoe));

Then, we can read data from it:

    final Element element = bigMemory.get("1");
    System.out.println("The value for key 1 is  " + element.getObjectValue());

And update it:

    final Person pamelaJones = new Person("Pamela Jones", 23, Person.Gender.FEMALE,
        "berry st", "Parsippany", "LA");
    bigMemory.put(new Element("1", pamelaJones));
    final Element updated = bigMemory.get("1");
    System.out.println("The value for key 1 is now " + updated.getObjectValue() +
                         ". key 1 has been updated.");

And delete it:

    bigMemory.remove("1");
    System.out.println("Try to retrieve key 1.");
    final Element removed = bigMemory.get("1");
    System.out.println("Value for key 1 is " + removed +
                       ". Key 1 has been deleted.");

You can also create or update multiple entries at once:

    Collection<Element> elements = new ArrayList<Element>();
    elements.add(new Element("1", new Person("Jane Doe", 35,
        Person.Gender.FEMALE, "eck street", "San Mateo", "CA")));
    elements.add(new Element("2", new Person("Marie Antoinette", 23,
        Person.Gender.FEMALE, "berry st", "Parsippany", "LA")));
    elements.add(new Element("3", new Person("John Smith", 25,
        Person.Gender.MALE, "big wig", "Beverly Hills", "NJ")));
    elements.add(new Element("4", new Person("Paul Dupont", 25,
        Person.Gender.MALE, "big wig", "Beverly Hills", "NJ")));
    elements.add(new Element("5", new Person("Juliet Capulet", 25,
        Person.Gender.FEMALE, "big wig", "Beverly Hills", "NJ")));

    bigMemory.putAll(elements);

And read multiple entries at once:

    final Map<Object, Element> elementsMap = bigMemory.getAll(
        Arrays.asList("1", "2", "3"));

And delete multiple entries at once:

    bigMemory.removeAll(Arrays.asList("1", "2", "3"));

And delete everything at once:

    bigMemory.removeAll();


<a id="example04-search"></a>
## Example 4: Search
BigMemory Max comes with powerful in-memory search capabilities. This sample shows
how to perform basic search operations on your in-memory data using the Search API.
You can also issue queries using BigMemory SQL, either from the command line or from the Terracotta Management Console (TMC). This SQL-like language is documented on the [BigMemory SQL](/documentation/4.1/bigmemorymax/search/bigmemory-sql) page.

First, create an instance of BigMemory Max with searchable attributes:

    Configuration managerConfig = new Configuration()
        .cache(new CacheConfiguration().name("MySearchableDataStore")
            .eternal(true)
            .maxBytesLocalHeap(512, MemoryUnit.MEGABYTES)
            .maxBytesLocalOffHeap(32, MemoryUnit.GIGABYTES)
            .searchable(new Searchable()
                .searchAttribute(new SearchAttribute().name("age"))
                .searchAttribute(new SearchAttribute().name("gender")
                    .expression("value.getGender()"))
                .searchAttribute(new SearchAttribute().name("state")
                    .expression("value.getAddress().getState()"))
                .searchAttribute(new SearchAttribute().name("name")
                    .className(NameAttributeExtractor.class.getName()))
            )
        );

    CacheManager manager = CacheManager.create(managerConfig);
    Ehcache bigMemory = manager.getEhcache("MySearchableDataStore");

Next, insert data:

    bigMemory.put(new Element(1, new Person("Jane Doe", 35, Gender.FEMALE,
        "eck street", "San Mateo", "CA")));
    bigMemory.put(new Element(2, new Person("Marie Antoinette", 23, Gender.FEMALE,
        "berry st", "Parsippany", "LA")));
    bigMemory.put(new Element(3, new Person("John Smith", 25, Gender.MALE,
        "big wig", "Beverly Hills", "NJ")));
    bigMemory.put(new Element(4, new Person("Paul Dupont", 45, Gender.MALE,
        "cool agent", "Madison", "WI")));
    bigMemory.put(new Element(5, new Person("Juliet Capulet", 30, Gender.FEMALE,
        "dah man", "Bangladesh", "MN")));
    for (int i = 6; i < 1000; i++) {
      bigMemory.put(new Element(i, new Person("Juliet Capulet" + i, 30,
          Person.Gender.MALE, "dah man", "Bangladesh", "NJ")));
    }


Create some search attributes and construct a query:

    Attribute<Integer> age = bigMemory.getSearchAttribute("age");
    Attribute<Gender> gender = bigMemory.getSearchAttribute("gender");
    Attribute<String> name = bigMemory.getSearchAttribute("name");
    Attribute<String> state = bigMemory.getSearchAttribute("state");

    Query query = bigMemory.createQuery();
    query.includeKeys();
    query.includeValues();
    query.addCriteria(name.ilike("Jul*").and(gender.eq(Gender.FEMALE)))
        .addOrderBy(age, Direction.ASCENDING).maxResults(10);

Execute the query and look at the results:

    Results results = query.execute();
    System.out.println(" Size: " + results.size());
    System.out.println("----Results-----\n");
    for (Result result : results.all()) {
      System.out.println("Maxt: Key[" + result.getKey()
                         + "] Value class [" + result.getValue().getClass()
                         + "] Value [" + result.getValue() + "]");
    }

We can also use Aggregators to perform computations across query
results. Here's an example that computes the average age of all people in the
data set:

    Query averageAgeQuery = bigMemory.createQuery();
    averageAgeQuery.includeAggregator(Aggregators.average(age));
    System.out.println("Average age: "
                       + averageAgeQuery.execute().all().iterator().next()
                           .getAggregatorResults());

We can also restrict the calculation to a subset based on search
attributes. Here's an example that computes the average age of all people in
the data set between the ages of 30 and 40:

    Query agesBetween = bigMemory.createQuery();
    agesBetween.addCriteria(age.between(30, 40));
    agesBetween.includeAggregator(Aggregators.average(age));
    System.out.println("Average age between 30 and 40: "
                       + agesBetween.execute().all().iterator().next()
                           .getAggregatorResults());

Using Aggregators, we can also find number of entries that match our search
criteria.  Here's an example that finds the number of people in the data set
who live in New Jersey:

    Query newJerseyCountQuery = bigMemory.createQuery().addCriteria(
                                    state.eq("NJ"));
    newJerseyCountQuery.includeAggregator(Aggregators.count());
    System.out.println("Count of people from NJ: "
                       + newJerseyCountQuery.execute().all().iterator().next()
                           .getAggregatorResults());

<a id="example05-nonstop"></a>
## Example 5: Nonstop/Rejoin
The nonstop feature allows certain operations to proceed on clients that have become disconnected from the cluster, and it allows operations to proceed even if they cannot complete by the nonstop timeout value. The rejoin feature then allows clients to identify a source of Terracotta configuration, so that clients can rejoin a cluster after having been either disconnected from that cluster or timed out by a Terracotta server.  

This example demonstrates the nonstop and rejoin features of BigMemory Max. After loading the configuration below, use the scripts provided to start and run the server. Then stop the server and verify that the nonstop feature is working. Start the server again and watch the rejoin feature in action.


    Configuration managerConfig = new Configuration()
        .terracotta(new TerracottaClientConfiguration().url("localhost:9510").rejoin(true))
        .cache(new CacheConfiguration().name("nonstop-sample")
            .persistence(new PersistenceConfiguration().strategy(DISTRIBUTED))
            .maxBytesLocalHeap(128, MemoryUnit.MEGABYTES)
            .maxBytesLocalOffHeap(1, MemoryUnit.GIGABYTES)
            .terracotta(new TerracottaConfiguration()
	        .nonstop(new NonstopConfiguration()
	        .immediateTimeout(false)
                .timeoutMillis(10000).enabled(true)
            )
            ));

    CacheManager manager = CacheManager.create(managerConfig);
    Ehcache bigMemory = manager.getEhcache("nonstop-sample");

    try {
      System.out.println("**** Put key 1 / value timDoe. ****");
      final Person timDoe = new Person("Tim Doe", 35, Person.Gender.MALE,
          "eck street", "San Mateo", "CA");
      bigMemory.put(new Element("1", timDoe));

      System.out.println("**** Get key 1 / value timDoe. ****");
      System.out.println(bigMemory.get("1"));
      waitForInput();

      System.out
          .println("**** Now you have to kill the server using the
	       stop-sample-server.bat on Windows or stop-sample-server.sh otherwise ****");
      try {
        while (true) {
          bigMemory.get("1");
        }
      } catch (NonStopCacheException e) {
        System.out
            .println("**** Server is unreachable - NonStopException received when trying
	        to do a get on the server. NonStop is working ****");
      }

      System.out
          .println("**** Now you have to restart the server using the start-sample-server.bat
	      on Windows or start-sample-server.sh otherwise ****");

      boolean serverStart = false;
      while (serverStart  == false) {
        try {
          bigMemory.get("1");
          //if server is unreachable, exception is thrown when doing the get
          serverStart = true;
        } catch (NonStopCacheException e) {
        }
      }
      System.out
          .println("**** Server is reachable - No More NonStopException received when
	      trying to do a get on the server. Rejoin is working ****");
    } finally

    {
      if (manager != null) manager.shutdown();
    }


<a id="example06-arc"></a>
## Example 6: Automatic Resource Control (ARC)

Automatic Resource Control (ARC) is a powerful capability of BigMemory Max
that gives users the ability to control how much data is stored in heap memory
and off-heap memory.

The following XML configuration instructs ARC to allocate a maximum of 512 M of heap
memory and 8 G of off-heap memory. In this example, when 512 M of heap memory is used, ARC will
automatically move data into off-heap memory up to a maximum of 8 G.  The
amount of off-heap memory you can use is limited only by the amount of physical
RAM you have available.

    <ehcache xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:noNamespaceSchemaLocation="http://ehcache.org/ehcache.xsd"
         updateCheck="false" monitoring="autodetect"
         dynamicConfig="true"
         name="MyManager" maxBytesLocalHeap="512M"
         maxBytesLocalOffHeap="8G">

      <defaultCache>
      </defaultCache>

      <cache name="BigMemory1">
      </cache>

      <cache name="BigMemory2">
      </cache>

    </ehcache>

This instructs the ARC capability of BigMemory to keep a maximum of 512 MB of its data in heap
for nanosecond to microsecond access.  In this example, as the 512 MB of heap memory fills up,
ARC will automatically move data to the 8 GB off-heap store where it is
available at microsecond speed. This configuration keeps heap sizes small to
avoid garbage collection pauses and tuning, but still uses large amounts of
in-process memory for ultra-fast access to data.

**Important:** *BigMemory Max is capable of addressing gigabytes to terabytes of
in-memory data in a single JVM. However, to avoid swapping take care not to configure BigMemory Max to use more memory than is physically available on your hardware.*

It's also possible to allocate resources on a per-data set basis.  Here's
an example of allocating 4 G of off-heap memory to the BigMemory1 data set and
4 G of off-heap memory to the BigMemory2 data set:

    <ehcache xmlns
	 ...
         name="MyManager"
         maxBytesLocalHeap="512M"
         maxBytesLocalOffHeap="32G"
         maxBytesLocalDisk="128G">

      <cache name="BigMemory1"
             maxBytesLocalOffHeap="4G">
      </cache>

      <cache name="BigMemory2"
             maxBytesLocalOffHeap="4G">
      </cache>

    </ehcache>

<a id="example07-cache"></a>
## Example 7: Using BigMemory As a Cache

BigMemory Max is a powerful in-memory data
management solution. Among its many applications, BigMemory Max may be used as a
cache to speed up access to data from slow or expensive databases and other
remote data sources. This example shows how to enable and configure the caching
features available in BigMemory Max.

The following programmatic configuration snippet shows how to set time-to-live
(TTL) and time-to-idle (TTI) policies on a data set:

    Configuration managerConfiguration = new Configuration();
    managerConfiguration.name("cacheManagerCompleteExample")
        .terracotta(new TerracottaClientConfiguration().url("localhost:9510"))
        .cache(
            new CacheConfiguration()
                .name("sample-cache")
                .maxBytesLocalHeap(128, MemoryUnit.MEGABYTES)
                .timeToLiveSeconds(4)
                .timeToIdleSeconds(2)
                .terracotta(new TerracottaConfiguration())
        );


    CacheManager manager = CacheManager.create(managerConfiguration);


The `timeToLiveSeconds` directive sets the maximum age of an element in the
data set.  Elements older than the maximum TTL will not be returned from the
data store.  This is useful when BigMemory is used as a cache of external data
and you want to ensure the freshness of the cache.

The `timeToIdleSeconds` directive sets the maximum time since last access of an
element. Elements that have been idle longer than the maximum TTI will not be
returned from the data store. This is useful when BigMemory is being used as a
cache of external data and you want to bias the eviction algorithm towards
removing idle entries.

If neither TTL nor TTI are set (or set to zero), data will stay in BigMemory until it is
explicitly removed.
