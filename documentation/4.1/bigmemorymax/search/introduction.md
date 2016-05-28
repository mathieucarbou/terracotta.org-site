---
---
# BigMemory Search Setup
####For Search API and BigMemory SQL


{toc|2:3}

#####New for Search in 4.1
For support for nulls, see [Options for Working with Nulls](/documentation/4.1/bigmemorymax/search/best-practices#options-for-working-with-nulls).

For handling large result sets, see #3 of [Best Practices](/documentation/4.1/bigmemorymax/search/best-practices#best-practices-for-optimizing-searches).

## Introduction

BigMemory Search allows you to execute arbitrarily complex queries against caches with pre-built indexes. The development of alternative indexes on values provides the ability for data to be looked up based on multiple criteria instead of just keys.

Searchable attributes may be extracted from both keys and values. Keys, values,
or summary values (Aggregators) can all be returned. Here is a simple example: Search for 32-year-old males and return the cache values.

    Results results = cache.createQuery().includeValues()
      .addCriteria(age.eq(32).and(gender.eq("male"))).execute();

Queries can be formulated using the either the BigMemory Search API or BigMemory SQL (Structured Query Language).

    // BigMemory Search API:
    Attribute<Integer> age = cache.getSearchAttribute("age");
    Person.createQuery().addCriteria(age.gt(30)).includeValues().execute();

    // BigMemory SQL:
    QueryManager queryManager =
    		QueryManagerBuilder
    				.newQueryManagerBuilder()
    				.addCache(Person)
    				.addCache(Address)
    				.build();
    Query personQuery = queryManager.createQuery(
    	"select * from Person where age > 30");

Before creating a query, the Ehcache configuration must be prepared. This page contains information about preparing for searches. For constructing queries, go to either:

* [BigMemory Search API](/documentation/4.1/bigmemorymax/search/api-queries)
* [BigMemory SQL](/documentation/4.1/bigmemorymax/search/bigmemory-sql)

#### What is Searchable?

Searches can be performed against Element keys and values, but they must be treated as attributes. Some Element keys and values are directly searchable and can simply be added to the search index as attributes. Some Element keys and values must be made searchable by extracting attributes with supported search types out of the keys and values. It is the attributes themselves which are searchable.


## Making a Cache Searchable
Caches can be made searchable, on a per cache basis, either by configuration or programmatically.

### By Configuration

Caches are made searchable by adding a `<searchable/>` tag to the ehcache.xml file.

~~~
<cache name="cache2" maxBytesLocalHeap="16M" eternal="true" maxBytesLocalOffHeap="256M">
	<persistence strategy="localRestartable"/>
	<searchable/>
</cache>
~~~

This configuration will scan keys and values and, if they are of supported search types, add them as
attributes called "key" and "value" respectively. If you do not want automatic indexing of keys and values,
you can disable it with:

~~~
<cache name="cacheName" ...>
	<searchable keys="false" values="false">
	   ...
	</searchable>
</cache>
~~~

You might want to do this if you have a mix of types for your keys or values. The automatic indexing will throw
an exception if types are mixed.

If you think that you will want to add search attributes after the cache is initialized, you can explicitly indicate the dynamic search configuration. Set the `allowDynamicIndexing` attribute to "true" to enable use of the dynamic attributes extractor (described in the [Defining Attributes](#defining-attributes) section below):

~~~
<cache name="cacheName" ...>
	<searchable allowDynamicIndexing="true">
	   ...
	</searchable>
</cache>
~~~

Often keys or values will not be directly searchable and instead you will need to extract searchable attributes from the keys or values.
The following example shows a more typical case. Attribute Extractors are explained in more detail in the following section.

~~~
<cache name="cache3" maxEntriesLocalHeap="10000" eternal="true" maxBytesLocalOffHeap="10G">
	<persistence strategy="localRestartable"/>
	<searchable>
   		<searchAttribute name="age" class="net.sf.ehcache.search.TestAttributeExtractor"/>
   		<searchAttribute name="gender" expression="value.getGender()"/>
	</searchable>
</cache>
~~~


### Programmatically

The following example shows how to programmatically create the cache configuration with search attributes.

    Configuration cacheManagerConfig = new Configuration();
    CacheConfiguration cacheConfig = new CacheConfiguration("myCache", 0).eternal(true);
    Searchable searchable = new Searchable();
    cacheConfig.addSearchable(searchable);
    // Create attributes to use in queries.
    searchable.addSearchAttribute(new SearchAttribute().name("age"));
    // Use an expression for accessing values.
    searchable.addSearchAttribute(new SearchAttribute()
        .name("first_name")
        .expression("value.getFirstName()"));
    searchable.addSearchAttribute(new SearchAttribute()
         .name("last_name")
	 .expression("value.getLastName()"));
    searchable.addSearchAttribute(new SearchAttribute()
         .name("zip_code")
	 .className("net.sf.ehcache.search.TestAttributeExtractor"));
    cacheManager = new CacheManager(cacheManagerConfig);
    cacheManager.addCache(new Cache(cacheConfig));
    Ehcache myCache = cacheManager.getEhcache("myCache");
    // Now create the attributes and queries, then execute.
    ...

To learn more about BigMemory Search, see the `net.sf.ehcache.search*` packages in the [Javadoc for http://www.ehcache.org/apidocs/2.8.5/index.html](http://www.ehcache.org/apidocs/2.8.5/index.html).

### Disk usage with the Terracotta Server Array

Search indexes are stored on Terracotta server disks. The default path is the same directory as the configured `<data>` location. You can customize the path using the `<index>` element in the server's `tc-config.xml` configuration file.

The Terracotta Server Array can be configured to be restartable in addition to including searchable caches, but both of these features require disk storage. When both are enabled, be sure that enough disk space is available. Depending upon the number of searchable attributes, the amount of disk storage required might be 3 times the amount of in-memory data.

It is highly recommended to store the search index (`<index>`) and the Fast Restart data (`<data>`) on separate disks.


## Defining Attributes

In addition to configuring a cache to be searchable, you must define the attributes to be used in searches.

Attributes are extracted from keys or values during search by using `AttributeExtractor`s. An extracted attribute must be one of the following types:

*   Boolean
*   Byte
*   Character
*   Double
*   Float
*   Integer
*   Long
*   Short
*   String
*   java.util.Date
*   java.sql.Date
*   Enum

These types correspond to the AttributeType enum specified at [Javadoc](http://www.ehcache.org/apidocs/2.8.5/index.html).

Type name matching is case sensitive. For example, Double resolves to the java.lang.Double class type, and double is interpreted as the primitive double type.


#### Examples of Attributes

##### API Example
	<searchable>
	<searchAttribute name="age" type="Integer"/>
	</searchable>

##### BigMemory SQL Example

    // no cast required for String or int
    select * from Person where age = '11'

If an attribute cannot be found or is of the wrong type, an AttributeExtractorException is thrown on search execution.

**Note**: On the first use of an attribute, the attribute type is detected, validated against supported types, and saved automatically. Once the type is established, it cannot be changed. For example, if an integer value was initially returned for attribute named "Age" by the attribute extractor, it is an error for the extractor to return a float for this attribute later on.

### Well-known Attributes

The parts of an Element that are well-known attributes can be referenced by some predefined, well-known names.
If a key and/or value is of a supported search type, it is added automatically as an attribute with the name
"key" or "value".
These well-known attributes have the convenience of being constant attributes made available on the `Query` class.
For example, the attribute for "key" can be referenced in a query by `Query.KEY`. For even greater readability, statically import so that, in this example, you would use `KEY`.

| Well-known Attribute Name | Attribute Constant |
| --- | --- |
| key               |     Query.KEY |
| value             |     Query.VALUE |


### Reflection Attribute Extractor

The `ReflectionAttributeExtractor` is a built-in search attribute extractor that uses JavaBean conventions and also understands a simple form of expression. Where a JavaBean property is available and it is of a searchable type, it can be declared:

    <cache>
      <searchable>
        <searchAttribute name="age"/>
      </searchable>
    </cache>

The expression language of the `ReflectionAttributeExtractor` also uses method/value dotted expression chains. The expression chain must start with "key", "value", or "element". From the starting object, a chain of method calls or field names follows. Method calls and field names can be freely mixed in the chain:

    <cache>
      <searchable>
        <searchAttribute name="age" expression="value.person.getAge()"/>
      </searchable>
    </cache>
    <cache>
      <searchable>
         <searchAttribute name="name" expression="element.toString()"/>
      </searchable>
    </cache>

 **Note**: The method and field name portions of the expression are case-sensitive.

<a id="custom-extractor"></a>
### Custom Attribute Extractor

In more complex situations, you can create your own attribute extractor by implementing the `AttributeExtractor` interface. The interface's `attributeFor` method returns the attribute value for the element and attribute name you specify.

**Note**: These examples assume there are previously created Person objects containing attributes such as name, age, and gender.

Provide your extractor class:

    <cache name="cache2" maxEntriesLocalHeap="0" eternal="true">
      <persistence strategy="none"/>
      <searchable>
         <searchAttribute name="age" class="net.sf.ehcache.search.TestAttributeExtractor"/>
      </searchable>
    </cache>

A custom attribute extractor could be passed an Employee object to extract a specific attribute:

    returnVal = employee.getdept();

If you need to pass state to your custom extractor, specify properties:

    <cache>
      <searchable>
        <searchAttribute name="age"
        class="net.sf.ehcache.search.TestAttributeExtractor"
        properties="foo=this,bar=that,etc=12" />
      </searchable>
    </cache>

If properties are provided, the attribute extractor implementation must have a public constructor that accepts a single `java.util.Properties` instance.

### Dynamic Attributes Extractor

The `DynamicAttributesExtractor` provides flexibility by allowing the search configuration to be changed after the cache is initialized. This is done with one method call, at the point of element insertion into the cache. The `DynamicAttributesExtractor` method returns a map of attribute names to index and their respective values. This method is called for every Ehcache.put() and replace() invocation.

Assuming that we have previously created Person objects containing attributes such as name, age, and gender, the following example shows how to create a dynamically searchable cache and register the `DynamicAttributesExtractor`:

	Configuration config = new Configuration();
	config.setName("default");
  	CacheConfiguration cacheCfg = new CacheConfiguration("PersonCache");
  	cacheCfg.setEternal(true);
	cacheCfg.terracotta(new TerracottaConfiguration().clustered(true));
  	Searchable searchable = new Searchable().allowDynamicIndexing(true);

	cacheCfg.addSearchable(searchable);
	config.addCache(cacheCfg);

	CacheManager cm = new CacheManager(config);
	Ehcache cache = cm.getCache("PersonCache");
	final String attrNames[] = {"first_name", "age"};
	// Now you can register a dynamic attribute extractor to index
	// the cache elements, using a subset of known fields
	cache.registerDynamicAttributesExtractor(new DynamicAttributesExtractor() {
		Map<String, Object> attributesFor(Element element) {
			Map<String, Object> attrs = new HashMap<String, Object>();
			Person value = (Person)element.getObjectValue();
			// For example, extract first name only
			String fName = value.getName() == null ? null : value.getName().split("\\s+")[0];
			attrs.put(attrNames[0], fName);
			attrs.put(attrNames[1], value.getAge());
			return attrs;
		}
	});
	// Now add some data to the cache
	cache.put(new Element(10, new Person("John Doe", 34, Person.Gender.MALE)));

Given the code above, the newly put element would be indexed on values of name and age fields, but not gender. If, at a later time, you would like to start indexing the element data on gender, you would need to create a new `DynamicAttributesExtractor` instance that extracts that field for indexing.

Similarly, consider the following scenario. A Customer object has three fields: First Name, Middle Name, and Last Name. Two custom extractors exist, one for FirstName and one for LastName. To index and search on Middle Name, you must add a new, third extractor specifically for that purpose.    

#####Dynamic Search Rules

* To use the `DynamicAttributesExtractor`, the cache must be configured to be searchable and dynamically indexable. See [Making a Cache Searchable](#making-a-cache-searchable).

* A dynamically searchable cache must have a dynamic extractor registered BEFORE data is added to it. (This is to prevent potential races between extractor registration and cache loading which might result in an incomplete set of indexed data, leading to erroneous search results.)

* Each call on the `DynamicAttributesExtractor` method replaces the previously registered extractor, because there can be at most one extractor instance configured for each such cache.

* If a dynamically searchable cache is initially configured with a predefined set of search attributes, this set of attributes is always be queried for extracted values, regardless of whether or not a dynamic search attribute extractor has been configured.

* The initial search configuration takes precedence over dynamic attributes, so if the dynamic attribute extractor returns an attribute name already used in the initial searchable configuration, an exception is thrown.

* Clustered BigMemory clients do not share dynamic extractor instances or implementations. In a clustered searchable deployment, the initially configured attribute extractors cannot vary from one client to another. This is enforced by propagating them across the cluster. However, for dynamic attribute extractors, each clustered client maintains its own dynamic extractor instance. Each distributed application using dynamic search must therefore maintain its own attribute extraction consistency.
