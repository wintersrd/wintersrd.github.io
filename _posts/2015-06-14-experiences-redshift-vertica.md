---
layout: post
title: Redshift vs Vertica, an analyst's experience
tags:
- Redshift
- Vertica
---

When it comes to conducting analysis on very large structured data sets, there are a limited number of viable players which won't require a second mortgage and army of consultants to use (looking at you, Exadata). For the last several years my platform of choice has been HP Vertica: it's fast, scales well, relatively easy to manage/performance tune, and is fairly middle of the pack for pricing. However, for the last several months I've been using Redshift as my primary data warehouse and feel ready to make a comparison on where the platforms materially differ.

###Data Loading: Structured flat files

* **Vertica**: Vertica's standard loading paradigm is to load the file via a local copy command and loads into memory. This creates a potentially volatile loading pattern (dependent on network performance) but is extremely fast a parallelizes well if the machine executing the copy statement is in the same network. When running loading jobs like "copy all tables from a production DB" or "I need ad hoc data in the DB for analysis", Vertica was significantly faster/easier than Redshift.

* **Redshift**: Refshift can load from S3, EMR, or DynamoDB (it can also load from another machine via SSH, but the mechanism to do so is enormously cumbersome). Since it is lacking in any COPY FROM LOCAL functionality, loading data from a local source is both more cumbersome and much slower. However, loading data from Hadoop (presuming it's an EMR cluster) is much better in Redshift.


###Time and Time Series

* **Redshift**: Time series analysis is where Redshift falls flat; the database is lacking certain basic functions like _TO\_TIMESTAMP_... god help you if you need to work with Unix epochs. Most basic date functions are represented (ex. _DATEDIFF_, _DATETRUNC_) but there are very few analytical or window functions available.

* **Vertica**: In contrast, Vertica has an enormous number of date functions available to assist in analysis of event stream data (although not all functions are implemented in ISO/ANSI standards). Additionally, it's possible to do advanced time analytics in raw SQL. For example, sessionization of an event stream can be done using _CONDITIONAL\_TRUE\_EVENT_.


###JSON Handling

* **Redshift**: Redshift allows direct loading of JSON objects to tables and will map each key to the appropriate column during loading. This solution works well when the arrays are consistently defined as it will offer high performance on read but is a liability if new keys are constantly being added to your arrays - those new records will be ignored until the column is added to the table. Redshift also has excellent support for querying JSON objects stored in strings (ex. _select json\_extract\_array\_element\_text('[111,112,113]', 2);_)

* **Vertica**: In contrast, Vertica stores the JSON object as written in a Flex Table, preserving the whole record and using a view to allow access via SQL to the array. The DBA can choose which (if any) columns should be materialized as database columns; this approach is more complicated from an end-user perspective but ensures that data is never lost. Vertica also does not support querying JSON strings directly except by abusing functions developed for use on Flex tables.


###Physical Structure Management

* **Vertica**: The reason that column stores are so performant is the physical sorting and partitioning of the data by column; like indices, tuning for performance is a long term process and may require many iterations. Vertica allows multiple sort and compression designs for each table, allowing one to optimize for a variety of queries at the cost of load performance. Additionally, changes can be executed on the fly without impacting end users or dropping tables.

* **Redshift**: Redshift allows the same level of sorting/node distribution logic, but each table can only have one physical manifestation. Additionally, changes to the physical structure require the table to be dropped and recreated. This significantly slows down the DBA's work on performance tuning.


###Conclusion

So, which one is "better"? Objectively, neither - both platforms bring strengths and weaknesses to the table. I would argue that Vertica has a more mature functionality set and is easier/more flexible to work in, with the downsides of higher cost and more administrative work required.
