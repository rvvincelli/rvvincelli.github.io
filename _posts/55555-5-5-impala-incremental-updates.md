---
layout: post
title: Incremental updates with Impala
---

{% include google_analytics.html %}

## New tuples only

A very common usage pattern on a database is that of updation inserts: I have a new row, I want to insert it in my table possibly updating the existing corresponding row. It is usually the case that the primary
key drives this: the new row is an update for the existing table row carrying the very same key. This is actually an SQL standard too, see [here](https://en.wikipedia.org/wiki/Merge_(SQL)) and some
DBMS's have conforming operations, see for example [here](https://wiki.postgresql.org/wiki/UPSERT).

[Cloudera Impala](http://impala.io/) is the way fast cluster SQL analytical queries are best served on Hadoop these days. While it's pretty
[standard](http://www.cloudera.com/content/www/en-us/documentation/enterprise/latest/topics/impala_langref.html) in its flavor of SQL it does not have statements to perform upserts. It is explained
[here](http://www.cloudera.com/content/www/en-us/documentation/enterprise/latest/topics/impala_dml.html) how the DML subset is not so rich because of HDFS being inherently append-only. Another feature I miss a lot is database renaming.

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
<td style="text-align: right">1988-01-12</td>
</tr>
<tr>
<td>2</td>
<td style="text-align: center">auto</td>
<td style="text-align: right">blue</td>
<td style="text-align: right">1988-01-13</td>
</tr>
<tr>
<td>3</td>
<td style="text-align: center">moto</td>
<td style="text-align: right">yellow</td>
<td style="text-align: right">2001-01-09</td>
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
<td style="text-align: right">1988-01-12</td>
</tr>
<tr>
<td>2</td>
<td style="text-align: center">auto</td>
<td style="text-align: right">green</td>
<td style="text-align: right">1988-01-13</td>
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
<td style="text-align: right">1988-01-12</td>
</tr>
<tr>
<td>2</td>
<td style="text-align: center">auto</td>
<td style="text-align: right">green</td>
<td style="text-align: right">1988-01-13</td>
</tr>
<td>3</td>
<td style="text-align: center">moto</td>
<td style="text-align: right">yellow</td>
<td style="text-align: right">2001-01-09</td>
</tr>
</tbody></table>

Below we will focus on this scenario, where all of the rows in the new table are fresher, so we just replace by ID, without looking at timestamp columns for example. If this is not your case, ie
maybe you want to keep the current row or maybe not according to which of the two is actually newer, take a look at [this](https://t.co/kaJ84eTgPW) post; [Hive](https://hive.apache.org) is used but
as you know the two SQL variants of Hive and Impala are almost the same thing really. Or even better you can always amend our easy strategy here.

 
## How we do
We will use a small CSV file as our base table and some other as the delta. Data and code can be found [here]([this](https://github.com/rvvincelli/impala-upserts) ). 
The whole procedure can be divided into three logical steps:

1. creation of the base table, *una tantum*
2. creation of the delta table, every time there is a new batch to merge
3. integration, merging the two tables

This procedure may be executed identically multiple times on the same folders as it is we may say idempotent: thanks to the `IF NOT EXISTS` we don't assume the base table exists or not and every new delta is guaranteed to be added to the original data. Notice that no records history is maintained, ie we don't save the different versions a row is updated through.

First of all, a new database:

```sql
CREATE DATABASE IF NOT EXISTS dealership;
```

```sql
USE dealership;
```

So we create the table to begin with, we import as an Impala table our text file by means of a `CREATE EXTERNAL` specifying the location of the directory containing the file. It is a directory, we can of course import multiple files and it makes sense provided that they all share the same schema to match the one we specify in the `CREATE`; the directory must be an HDFS path, it cannot be a local one.

```sql
CREATE EXTERNAL TABLE IF NOT EXISTS cars(
       id STRING,
       type STRING,
       color STRING,
       year TIMESTAMP
)
ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t'
LOCATION '/user/ricky/dealership/cars/base_table';
```

If the timestamp field is [properly](http://www.cloudera.com/content/www/en-us/documentation/enterprise/latest/topics/impala_timestamp.html) formatted it is parsed automatically into the desired type, otherwise it will turn out null. This is the base table and needs to be created only once - if the script is retried it won't be recreated.

In the very same way we create now a table importing the new data:

```sql
CREATE EXTERNAL TABLE cars_new(
       id STRING,
       type STRING,
       color STRING,
       year TIMESTAMP
)
ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t'
LOCATION '/user/ricky/dealership/cars/delta_of_the_day';
```
This creation is not optional, no `IF NOT EXISTS`, as this table is temporary for us to store the new rows, it will be dropped afterwards.

To make things clearer let us refer to these two tables as 'old' and 'new' and do:

```sql ALTER TABLE cars RENAME TO cars_old;```

This is of course equivalent to:
```sql
CREATE TABLE cars_old AS SELECT * FROM cars;
DROP TABLE cars;
```
which might be your actual pick in case the Hive metastore bugs you with permission issues.

And now to the merge:

```sql
CREATE TABLE cars 
AS 
    SELECT *
    FROM cars_old
    WHERE id NOT IN (
        SELECT id
        FROM cars_new
    )
    UNION (
        SELECT *
        FROM cars_new
    );  
```
This may be broken down into two steps actually:

1. extraction of the rows that stay current, those we don't have updates for
2. union of these rows with the new row set, to define the new dataset

As you can see it is driven by the ID: from the old table we simply keep those rows whose ID's do not occur in the delta.
Of course if the ID matches there will be a replacement, no matter what: more advanced updation logics are always possible. Again, this scales linearly with the size of the starting table, assuming the deltas are of negligible size.

Finally the step to guarantee we can re-execute everything safely the next time new data is available:

DROP TABLE cars_new;
DROP TABLE cars_old;

## How to run

First of all create the data files on HDFS on the paths from the `LOCATION`s above.

Just fire up your `impala-shell`, `connect` to the `impalad` and issue the statements! You can work from the Impala UI provided by [Hue](http://gethue.com/) as well but be careful as you may not see eventual errors.

For batch mode put all of the statements in a text file and run the [--query_file](http://www.cloudera.com/content/www/en-us/documentation/cdh/5-1-x/Impala/Installing-and-Using-Impala/ciiu_shell_options.html) option:
`impala-shell -i impala.daemon.host:21000 -f impala_upserts.sql`
In this case the process will abort if query execution errors occur.

That's * FROM me.here!
