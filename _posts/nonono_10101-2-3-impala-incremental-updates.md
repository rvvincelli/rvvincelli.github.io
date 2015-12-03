---
layout: post
title: Incremental updates with Impala
---

{% include google_analytics.html %}

## New tuples only

A very common usage pattern on a database is that of updation inserts: a have a new row, I want to insert it in my table possibly updating the existing row. It is usually the case that the primary
key drives this: the new row is an update for the existing table row carrying the very same key. This is actually an SQL standard too, see [here](https://en.wikipedia.org/wiki/Merge_(SQL)) and some
DBMS's have conforming operations, see for example [here](https://wiki.postgresql.org/wiki/UPSERT).

[Cloudera Impala](http://impala.io/) is the way fast cluster analytical queries are best served on Hadoop these days. While it's pretty
[standard](http://www.cloudera.com/content/www/en-us/documentation/enterprise/latest/topics/impala_langref.html) in its flavor of SQL it does not have statements to perform upserts. It is explained
[here](http://www.cloudera.com/content/www/en-us/documentation/enterprise/latest/topics/impala_dml.html) how the DML subset is not so rich because of HDFS being inherently append-only. Another
feature I miss a lot is database renaming.

So, to break it down with an example, let's say this is our base table, the current, or old if you wish, version of the data: 
<table><thead>
<tr>
<th>ID</th>
<th style="text-align: center">Car type</th>
<th style="text-align: right">Color</th>
<th style="text-align: right">Year</th>
</tr>
</thead><tbody>
<tr>
<td>1</td>
<td style="text-align: center">jeep</td>
<td style="text-align: right">red</td>
<td style="text-align: right">12-1-88</td>
</tr>
<tr>
<td>2</td>
<td style="text-align: center">auto</td>
<td style="text-align: right">blue</td>
<td style="text-align: right">13-1-88</td>
</tr>
<tr>
<td>3</td>
<td style="text-align: center">moto</td>
<td style="text-align: right">yellow</td>
<td style="text-align: right">11-9-01</td>
</tr>
</tbody></table>

And we have a second table, we refer to it as the new, or delta, table, with the same schema of course:
<table><thead>
<tr>
<th>ID</th>
<th style="text-align: center">Car type</th>
<th style="text-align: right">Color</th>
<th style="text-align: right">Year</th>
</tr>
</thead><tbody>
<tr>
<td>1</td>
<td style="text-align: center">jeep</td>
<td style="text-align: right">brown</td>
<td style="text-align: right">12-1-88</td>
</tr>
<tr>
<td>2</td>
<td style="text-align: center">auto</td>
<td style="text-align: right">green</td>
<td style="text-align: right">13-1-88</td>
</tr>
</tbody></table>

For every row in the old table it is kept if no row with the same ID is found in the new table, otherwise it is replaced.
So our final result will be:
<table><thead>
<tr>
<th>ID</th>
<th style="text-align: center">Car type</th>
<th style="text-align: right">Color</th>
<th style="text-align: right">Year</th>
</tr>
</thead><tbody>
<tr>
<td>1</td>
<td style="text-align: center">jeep</td>
<td style="text-align: right">brown</td>
<td style="text-align: right">12-1-88</td>
</tr>
<tr>
<td>2</td>
<td style="text-align: center">auto</td>
<td style="text-align: right">green</td>
<td style="text-align: right">13-1-88</td>
</tr>
<td>3</td>
<td style="text-align: center">moto</td>
<td style="text-align: right">yellow</td>
<td style="text-align: right">11-9-01</td>
</tr>
</tbody></table>

Below we will focus on this scenario, where all of the rows in the new table are fresher, so we just replace by ID, without looking at timestamp columns for example. If this is not your case, ie
maybe you want to keep the current row or maybe not according to which of the two is actually newer, take a look at [this](https://t.co/kaJ84eTgPW) post; [Hive](https://hive.apache.org) is used but
as you know the two SQL variants of Hive and Impala are almost the same thing really.

 
[this]() 