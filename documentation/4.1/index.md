---
---
# Welcome to the Terracotta Product Documentation

{toc|2:4}

For a listing of current and previous releases, and links to release notes and platform compatibility tables, go to the [Release Information](http://www.terracotta.org/confluence/display/release/Home) page.

####New in BigMemory 4.1

[BigMemory Hybrid](/documentation/4.1/terracotta-server-array/hybrid) allows you to expand the capacity of BigMemory by including SSD/Flash as a storage medium.

[Cross-Language Clients](/documentation/4.1/cross-language) make BigMemory data accessible to applications written in .NET/C# and C++.

[BigMemory SQL](/documentation/4.1/bigmemorymax/search/bigmemory-sql) allows you to query BigMemory data with simple SQL statements that are very close to SQL-92 syntax.

[WAN Replication Service](/documentation/4.1/wan/introduction) is a new approach to WAN replication that provides fast, reliable support for multiple data centers.

[Data Lifecycle](/documentation/4.1/tms/tmc-using#off-line-data) operations have been added to the TMC for more control and visibility of clustered, offline data.

[Search Improvements](/documentation/4.1/bigmemorymax/search/introduction) include direct support for handling null values, and optimization for handling huge result sets on the client side that can improve reliability.

[Compare and Swap (CAS)](/documentation/4.1/bigmemorymax/configuration/reference-guide#30971) for eventual consistency: Enable highly multi-threaded applications (eg.CEP engines, Storm) to efficiently support accuracy of data in eventual consistency mode.

Data Upgradability provides the capability to seamlessly upgrade BigMemory data from version 4.1 to subsequent versions with the restartable option.

####Documentation Table of Contents

| Product or Component | Description |
|:-------|:------------|
| ***Products*** ||
|[BigMemory Max](/documentation/4.1/bigmemorymax/overview)|In-memory data management for real-time access to Big Data distributed across a server array.|
|[BigMemory Go](/documentation/4.1/bigmemorygo/index)|In-memory data management for real-time Big Data access on a standalone JVM.|
|[Quartz Scheduler](/documentation/4.1/quartz-scheduler/introduction)|Scalable Java job scheduler.|
|[Web Sessions](/documentation/4.1/web-sessions/get-started)|Solution for clustering web sessions.|
|[Universal Messaging](http://um.terracotta.org/developers/)|Real-time data streaming among Enterprise, Web, and Mobile platforms.|
| ***Components***||
|[Cross-Language Clients and Connector](/documentation/4.1/cross-language)|Access to BigMemory data from multiple platforms.|
|[Terracotta Server Array (TSA)](/documentation/4.1/terracotta-server-array/introduction)|Scalable, highly available, predictable, distributed in-memory data management.|
|[Terracotta Management Console (TMC)](/documentation/4.1/tms/tms)|Browser-based administration and monitoring application.|
|[WAN Replication Service](/documentation/4.1/wan/introduction)|Reliably replicates data on a per-cache basis across two or more regions connected by wide area networks.|



If you don’t find what you’re looking for in the docs, post a question to the [Terracotta support forums](https://groups.google.com/forum/#!forum/terracotta-oss).
