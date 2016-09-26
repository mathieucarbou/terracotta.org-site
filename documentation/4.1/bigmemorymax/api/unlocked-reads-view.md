---
---
# Unlocked Reads for Consistent Caches (UnlockedReadsView)

Certain environments require consistent cached data while also needing to provide optimized reads of that data. For example, a financial application may need to display account data as a result of a large number of requests from web clients. The performance impact of these requests can be reduced by allowing unlocked reads of an otherwise locked cache.


In cases where there is tolerance for getting potentially stale data, an unlocked (inconsistent) reads view can be created for Cache types using the UnlockedReadsView decorator. UnlockedReadsView requires that the underlying cache have Terracotta clustering and use the strong consistency mode. For example, the following cache can be decorated with UnlockedReadsView:

    <cache name="myCache"
         maxElementsInMemory="500"
         eternal="false">
       <persistence strategy="distributed"/>
       <terracotta clustered="true" consistency="strong" />
    </cache>

You can create an unlocked view of myCache programmatically:

    Cache cache = cacheManager.getEhcache("myCache");
    UnlockedReadsView unlockedReadsView = new UnlockedReadsView(cache, "myUnlockedCache");

The following table lists the API methods available with the decorator `net.sf.ehcache.constructs.unlockedreadsview.UnlockedReadsView`.

<table>
<tr>
<th>Method</th>
<th>Definition</th>
</tr>
<tr>
<td>
<code>public String getName()</code>
</td>
<td>
Returns the name of the unlocked cache view.
</td>
</tr>
<tr>
<td>
<code>public Element get(final Object key)</code>

<code>public Element get(final Serializable key)</code>
</td>
<td>
Returns the data under the given key. Returns null if data has expired.
</td>
</tr>
<tr>
<td>
<code>public Element getQuiet(final Object key)</code>

<code>public Element getQuiet(final Serializable key)</code>
</td>
<td>
Returns the data under the given key without updating cache statistics. Returns null if data has expired.
</td>
</tr>
</table>


### UnlockedReadsView and Data Freshness

By default, caches have the following attributes set as shown:

    <cache ... copyOnRead="true" ... >
    ...
      <terracotta ... consistency="strong"  ... />
    ...
    </cache>

Default settings are designed to make distributed caches more efficient and consistent in most use cases.
