---
---
# Search Performance and Best Practices
####For Search API and BigMemory SQL


{toc|2:3}



## Search Implementation and Performance

###BigMemory Max Backed by the Terracotta Server Array (TSA)

This implementation uses indexes that are maintained on each Terracotta server. With distributed BigMemory Max, the data is sharded across the number of active nodes in the cluster, and the index for each shard is maintained on the server for that shard. Searches are performed using the Scatter-Gather pattern. The query executes on each node and the results are then aggregated in the BigMemory Max that initiated the search.

Search operations perform in O(log n / number of shards) time. Performance is excellent. To improve the performance still further, consider adding more servers to the TSA. Search results are returned over the network, and the data returned might be very large, so techniques to limit return size are recommended. For more information, see [Best Practices](#best-practices).

###Standalone BigMemory Max
BigMemory uses a Search index that is maintained at the local node. The index is stored under a directory in the DiskStore and is available whether or not persistence is enabled. Any overflow from the on-heap tier of the cache is searched using indexes.

Search operations perform in O(log(n)) time. For tips that can aid performance, see [Best Practices](#best-practices).

For caches that are on-heap only, Attributes are extracted during query execution rather than ahead of time, and indexes are not used. Instead, the cache takes advantage of the fast access to do the equivalent of a table scan for each query. Each element in the cache is only visited once.

On-heap search operations perform in O(n) time. To see performance results, see [Maven-based performance test](http://svn.terracotta.org/svn/forge/offHeap-test/), where an average of representative queries takes 4.6 ms for a 10,000 entry cache, and 427 ms for a 1,000,000 entry cache.

##Best Practices for Optimizing Searches
1. Construct searches by including only the data that is actually required.
  *  Only use `includeKeys()` and/or `includeAttribute()` if those values are required for your application logic.
  *  If you don't need values or attributes, be careful not to burden your queries with unnecessary work. For example, if `result.getValue()` is not called in the search results, do not use `includeValues()` in the query.
  *  Consider if it would be sufficient to get attributes or keys on demand. For example, instead of running a search query with `includeValues()` and then `result.getValue()`, run the query for keys and include `cache.get()` for each individual key.

  **Note**: `includeKeys()` and `includeValues()` have lazy deserialization, which means that keys and values are de-serialized only when `result.getKey()` or `result.getValue()` is called. However, calls to `includeKeys()` and `includeValues()` do take time, so consider carefully when constructing your queries.

2. Searchable keys and values are automatically indexed by default. If you are not including them in your query, turn off automatic indexing with the following:

        <cache name="cacheName" ...>
          <searchable keys="false" values="false"/>
          ...
          </searchable>
        </cache>

3. Limit the size of the result set. Depending on your use case, you might consider maxResults, an Aggregator, or pagination:
  *  If getting a subset of all the possible results quickly is more important than receiving all the results, consider using `query.maxResults(int number_of_results)`  Sometimes maxResults is useful where the result set is ordered such that the items you want most are included within the maxResults.
  *  If all you want is a summary statistic, use a built-in Aggregator function, such as `count()`. For details, see the `net.sf.ehcache.search.aggregator` package in the <a href="http://www.ehcache.org/apidocs/2.8.5/index.html">Ehcache Javadoc</a>.
  *  If you want to avoid an `OutOfMemoryError` while allowing your Terracotta client to receive an extremely large result set, consider using the Pagination feature. Pagination limits how many of the total results appear on the client at a time, so that you can view the results in page-sized batches. Instead of calling the parameterless version of the execute method `query.execute()`, pass in an `ExecutionHints` object that specifies the page size you want:

            query.execute(new ExecutionHints().setResultBatchSize(pageSize))

      If you call for results after issuing a query with `ExecutionHints`, all results are returned (same behavior as a regular query), except that only the number of results specified as the `ResultBatchSize` will appear on the client. For example, if your query would have 500 results and you use a `ResultBatchSize` of 100, you will still get all 500 results, but you can scroll through them in pages of 100.

      You can enable search result pagination for the execution phase of a query whether the query was constructed using the Search API or BigMemory SQL.

      Limitations of search result pagination:
      *  Results from GroupBy queries (created with `Query.addGroupBy()`) cannot be paginated regardless of server topology.
      *  In multi-stripe (active/active) TSA topologies, pagination is not supported for the following query types:
          *  Result-size capped queries with aggregate functions, for example, those
constructed with `Query.includeAggregator().maxResults()` - with the exception that `count()` is the one aggregator that does work with all topologies
          *  Queries that request result ordering, for example, those created with
	`Query.addOrderBy()`

6. Make your search as specific as possible.
  * Queries with `iLike` criteria and fuzzy (wildcard) searches might take longer than more specific queries.
  * If you are using a wildcard, try making it the trailing part of the string instead of the leading part (`"321*"` instead of `"*123"`).
  * TIP: If you want leading wildcard searches, you should create a `<searchAttribute>` with the string value reversed in it, so that your query can use the trailing wildcard instead.

7. When possible, use the query criteria "Between" instead of "LessThan" and "GreaterThan", or "LessThanOrEqual" and "GreaterThanOrEqual". For example, instead of using `le(startDate)` and `ge(endDate)`, try `not(between(startDate,endDate))`.  

8. Index dates as integers. This can save time and can also be faster if you have to do a conversion later on.

9. Searches of eventually consistent BigMemory Max data sets are fast because queries are executed immediately, without waiting for the commit of pending transactions at the local node. **Note**: This means that if a thread adds an element into an eventually consistent cache and immediately runs a query to fetch the element, it will not be visible in the search results until the update is published to the server.

## Concurrency Notes
Unlike cache operations, which have selectable concurrency control or transactions, queries are asynchronous and Search results are eventually consistent with the caches.

#### Index Updating
Although indexes are updated synchronously, their state lags slightly behind that of the cache. The only exception is when the updating thread performs a search.

For caches with concurrency control, an index does not reflect the new state of the cache until:

* The change has been applied to the cluster.
* For a cache with transactions, when `commit` has been called.


#### Query Results
Unexpected results might occur if:

*   A search returns an Element reference that no longer exists.
*   Search criteria select an Element, but the Element has been updated.
*   Aggregators, such as `sum()`, disagree with the same calculation done by redoing the calculation yourself by re-accessing the cache for each key and repeating the calculation.
*   A value reference refers to a value that has been removed from the cache, and the cache has not yet been reindexed. If this happens, the value is null but the key and attributes supplied by the stale cache index are non-null. Because values in Ehcache are also allowed to be null, you cannot tell whether your value is null because it has been removed from the cache after the index was last updated or because it is a null value.

#### Recommendations
Because the state of the cache can change between search executions, the following is recommended:

*	Add all of the aggregators you want for a query at once, so that the returned aggregators are consistent.
*	Use null guards when accessing a cache with a key returned from a search.

<a id="nulls"></a>
## Options for Working with Nulls

BigMemory SQL supports using the presence or absence of null as a search criterion:

    select * from searchable where birthDate is null
    select * from searchable where birthDate is not null

The Search API supports the same criteria:

	myQuery.addCriteria(cache.getAttribute("middle_name").isNull());

The opposite case: require that a value for the attribute must be present:

	myQuery.addCriteria(cache.getAttribute("middle_name").notNull());

which is equivalent to:

	myQuery.addCriteria(cache.getAttribute("middle_name").isNull().not());

Alternatively, you can call constructors to set up equivalent logic:

	Criteria isNull = new IsNull("middle_name");
	Criteria notNull = new NotNull("middle_name");  
