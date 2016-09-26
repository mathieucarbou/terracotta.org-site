---
---
# BigMemory Search API Queries

{toc|2:3}

Prerequisite: This topic assumes that you have read [BigMemory Search Setup](/documentation/4.1/bigmemorymax/search/introduction).

## Introduction

The BigMemory Max Search API allows you to execute arbitrarily complex queries against caches with pre-built indexes. Alternative indexes on values allow data to be looked up based on multiple criteria. Otherwise, data can only be looked up based on keys.

Searchable attributes can be extracted from both keys and values. Keys, values,
or summary values (Aggregators) can all be returned. Here is a simple example: Search for 32-year-old males and return the cache values.

    Results results = cache.createQuery().includeValues()
      .addCriteria(age.eq(32).and(gender.eq("male"))).execute();



## Creating a Query
BigMemory Max Search uses a fluent, object-oriented Query API, following the principles of a domain specific language (DSL), which should be familiar to Java programmers. For example:

    Query query = cache.createQuery().addCriteria(age.eq(35)).includeKeys().end();
    Results results = query.execute();

### Using Attributes in Queries
 If declared and available, the well-known attributes are referenced by their names or the convenience attributes are used directly:

    Results results = cache.createQuery().addCriteria(Query.KEY.eq(35)).execute();
    Results results = cache.createQuery().addCriteria(Query.VALUE.lt(10)).execute();

Other attributes are referenced by the names in the configuration:

    Attribute<Integer> age = cache.getSearchAttribute("age");
    Attribute<String> gender = cache.getSearchAttribute("gender");
    Attribute<String> name = cache.getSearchAttribute("name");


### Expressions

A Query is built up using Expressions. Expressions can include logical operators such as &lt;and&gt; and &lt;or&gt;, and comparison operators such as &lt;ge&gt; (&gt;=), &lt;between&gt;, and &lt;like&gt;.
The configuration `addCriteria(...)` is used to add a clause to a query. Adding a further clause automatically "&lt;and&gt;s" the clauses.

    query = cache.createQuery().includeKeys()
                 .addCriteria(age.le(65))
		 .add(gender.eq("male"))
		 .end();

Both logical and comparison operators implement the `Criteria` interface.
To add a criterion with a different logical operator, explicitly nest it within a new logical operator Criteria Object. For example, to check for age = 35 or gender = female:

    query.addCriteria(new Or(age.eq(35),
                 gender.eq(Gender.FEMALE))
                );

More complex compound expressions can be created through additional nesting.
For a complete list of expressions, see [Expression JavaDoc](http://www.ehcache.org/apidocs/2.8.5/index.html).

### List of Operators
Operators are available as methods on attributes, so they are used by adding a ".". For example, "lt" means "less than" and is used as `age.lt(10)`, which is a shorthand way of saying `age LessThan(10)`.

| Shorthand | Criteria Class | Description
|----------|--------------|--------------
| and    |  And          | The Boolean AND logical operator
| between | Between      | A comparison operator meaning between two values
| eq     | EqualTo       | A comparison operator meaning Java "equals to" condition
| gt     | GreaterThan   | A comparison operator meaning greater than.
| ge     | GreaterThanOrEqual | A comparison operator meaning greater than or equal to.
| in     | InCollection  | A comparison operator meaning in the collection given as an argument
| lt     | LessThan      | A comparison operator meaning less than.
| le     | LessThanOrEqual| A comparison operator meaning less than or equal to
| ilike  | ILike          | A regular expression matcher. "?" and "*" may be used. Note that placing a wildcard in front of the expression will cause a table scan. ILike is always case insensitive.
| isNull | IsNull        | Tests whether the value of attribute with given name is null
| notNull| NotNull       | Tests whether the value of attribute with given name is NOT null
| not    | Not           | The Boolean NOT logical operator
| ne     | NotEqualTo    | A comparison operator meaning not the Java "equals to" condition
| or     | Or            | The Boolean OR logical operator

**Note**: For Strings, the operators are case-insensitive.

### Making Queries Immutable

 By default, a query can be executed, modified, and re-executed. If `end` is called,
 the query is made immutable.


## Obtaining and Organizing Query Results

Queries return a `Results` object that contains a list of objects of class `Result`. Each `Element` in the cache that a query finds is represented as a `Result` object. For example, if a query finds 350 elements, there will be 350 `Result` objects. However, if no keys or attributes are included but aggregators are included, there is exactly one `Result` present.

A Result object can contain:

*    the Element key - when `includeKeys()` is added to the query,
*    the Element value - when `includeValues()` is added to the query,
*    predefined attribute(s) extracted from an Element value - when `includeAttribute(...)` is added to the query. To access an attribute from a Result, use `getAttribute(Attribute<T> attribute)`.
*    aggregator results - Aggregator results are summaries computed for the search. They are available through `Result.getAggregatorResults`, which returns a list of `Aggregator`s in the same order in which they were used in the `Query`.

### Aggregators
Aggregators are added with `query.includeAggregator(\<attribute\>.\<aggregator\>)`.
For example, to find the sum of the age attribute:

    query.includeAggregator(age.sum());

For a complete list of aggregators, see the [Aggregators JavaDoc](http://www.ehcache.org/apidocs/2.8.5/index.html).

### Ordering Results
Query results can be ordered in ascending or descending order by adding an `addOrderBy` clause to the query, which takes
as parameters the attribute to order by and the ordering direction. For example, to order the results by ages in ascending order:

    query.addOrderBy(age, Direction.ASCENDING);


### Grouping Results
BigMemory Max query results can be grouped similarly to using an SQL GROUP BY statement. The BigMemory GroupBy feature provides the option to group results according to specified attributes. You can add an `addGroupBy` clause to the query, which takes as parameters the attributes to group by. For example, you can group results by department and location:

    Query q = cache.createQuery();
    Attribute<String> dept = cache.getSearchAttribute("dept");
    Attribute<String> loc = cache.getSearchAttribute("location");
    q.includeAttribute(dept);
    q.includeAttribute(loc);
    q.addCriteria(cache.getSearchAttribute("salary").gt(100000));
    q.includeAggregator(Aggregators.count());
    q.addGroupBy(dept, loc);


The GroupBy clause groups the results from `includeAttribute()` and allows aggregate functions to be performed on the grouped attributes. To retrieve the attributes that are associated with the aggregator results, you can use:

		String dept = singleResult.getAttribute(dept);
		String loc = singleResult.getAttribute(loc);


####GroupBy Rules
Grouping query results adds another step to the query--first results are returned, and second the results are grouped. Note the following rules and considerations:

*  In a query with a GroupBy clause, any attribute specified using `includeAttribute()` should also be included in the GroupBy clause.
*  Special KEY or VALUE attributes cannot be used in a GroupBy clause. This means that `includeKeys()` and `includeValues()` cannot be used in a query that has a GroupBy clause.
*  Adding a GroupBy clause to a query changes the semantics of any aggregators passed in, so that they apply only within each group.
*  As long as there is at least one aggregation function specified in a query, the grouped attributes are not required to be included in the result set, but they are typically requested anyway to make result processing easier.
*  An `addCriteria()` clause applies to all results prior to grouping.
*  If OrderBy is used with GroupBy, the ordering attributes are limited to those listed in the GroupBy clause.


### Limiting the Size of Results

By default, a query can return an unlimited number of results. For example, the following
query will return all keys in the cache.

    Query query = cache.createQuery();
    query.includeKeys();
    query.execute();

If too many results are returned, it could cause an OutOfMemoryError
The `maxResults` clause is used to limit the size of the results.
For example, to limit the above query to the first 100 elements found:

    Query query = cache.createQuery();
    query.includeKeys();
    query.maxResults(100);
    query.execute();

**Note**: When maxResults is used with GroupBy, it limits the number of groups.

When you are done with the results, call `discard()` to free up resources.
In the distributed implementation with Terracotta, resources may be used to hold results for paging or return.

[See also "pagination" in [Best Practices for Search](/documentation/4.1/bigmemorymax/search/introduction#best-practices)]

### Interrogating Results
To determine what a query returned, use one of the interrogation methods on `Results`:

* `hasKeys()`
* `hasValues()`
* `hasAttributes()`
* `hasAggregators()`


## Sample Application
To get started with BigMemory Search, you can use [a simple standalone sample application](http://github.com/sharrissf/Ehcache-Search-Sample/downloads/) with few dependencies. You can also check out the source:

    git clone git://github.com/sharrissf/Ehcache-Search-Sample.git

For examples on how to use each Search feature, see the [Ehcache Test Sources](http://www.ehcache.org/apidocs/2.8.5/index.html) page.

## Scripting Environments
You can use scripting with BigMemory Search. The following example shows how to use it with BeanShell:

    Interpreter i = new Interpreter();
    //Auto discover the search attributes and add them to the interpreter's context
    Map<String, SearchAttribute> attributes =
        cache.getCacheConfiguration().getSearchAttributes();
    for (Map.Entry<String, SearchAttribute> entry : attributes.entrySet()) {
    i.set(entry.getKey(), cache.getSearchAttribute(entry.getKey()));
    LOG.info("Setting attribute " + entry.getKey());
    }
    //Define the query and results. Add things which would be set in the GUI i.e.
    //includeKeys and add to context
    Query query = cache.createQuery().includeKeys();
    Results results = null;
    i.set("query", query);
    i.set("results", results);
    //This comes from the freeform text field
    String userDefinedQuery = "age.eq(35)";
    //Add on the things that we need
    String fullQueryString =
        "results = query.addCriteria(" + userDefinedQuery + ").execute()";
    i.eval(fullQueryString);
    results = (Results) i.get("results");
    assertTrue(2 == results.size());
    for (Result result : results.all()) {
    LOG.info("" + result.getKey());
    }
