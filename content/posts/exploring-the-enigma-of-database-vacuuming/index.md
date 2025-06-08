---
title: "Exploring the Enigma of Database Vacuuming"
date: 2024-04-26T19:37:25+04:00
draft: false
categories: [databases]
tags: [postgres, performance]
images: ["/posts/exploring-the-enigma-of-database-vacuuming/images/thumbnail.png"]
---

Before we discuss what `VACUUM`[^1] does and its implications, we need to understand how data is actually stored on disk. What happens when a tuple is inserted, updated, or deleted? Understanding this will help us understand what `VACUUM` does, why it's needed, and its implications.

We will primarily focus on **Postgres**[^2], as it's one of the most widely used open-source databases. Different databases might have different implementations.

## Tuples, Page & TOAST

A **tuple** in database terms is a table's row version and index entries. A tuple is a single row of a table. It contains the actual data. A tuple is made up of the following things.

- **header**: it's a 23-byte header that contains the tuple metadata.
- **data**: the actual data of the tuple.

A **page** in the database is a single unit of storage of a fixed size, usually between 4KiB and 16KiB. In Postgres, it's 8KiB. A single page can hold multiple rows, indexes, etc. When the data is written or read from disk, it's done in terms of whole pages. 

```sql
select current_setting('block_size');
 current_setting
-----------------
 8192             -- the size of a single page in Postgres is 8KiB
(1 row)
```
A single page is made up of the following things.

- **header**: it's a 28-byte header that contains the page metadata.
- **array of tuple pointers(table of contents)**: pointers to the tuples on the page. Its 4 bytes per tuple pointer.
- **free space**: the free space on the page is used to store new tuples. Once the free space is exhausted, a new page will be created. 
- **tuples**: rows of the table. Each tuple is 23 bytes plus the size of the actual data. 
- **special space**: stores things like visibility map, free space map, etc. 

So, a typical page layout looks like the one below.

<img src="images/page_layout.svg" width="60%" style="border:none;" alt="page_layout"/>

In fact, we can view this database using the `pageinspect` extension.  Let us create a table, add some data, and display the transaction ID performing the insert.
```sql
create extension pageinspect;
create table if not exists test_table (
 id int primary key,
 name varchar(255)
);

begin;
select txid_current(); -- current txn id performing insert
 txid_current
--------------
          775
(1 row)
insert into test_table (id, name) values (1, 'test');
commit;
```
The page header details can be queried using the below query.
```sql
select lower, upper, special, pagesize from page_header(get_raw_page('test_table', 0));
 lower | upper | special | pagesize
-------+-------+---------+----------
    28 |  8152 |    8192 |     8192
(1 row)
```

**A word about transaction IDs**

Every DML statement in Postgres is wrapped around a unique transaction ID performing the operation. These IDs act as a time filter, indicating the duration for which the tuple was active. They are the backbone of MVCC architecture. 

- **t_xmin** is the transaction that inserted the tuple.
- **t_xmax** is the transaction that updated or deleted the tuple.

We can use the `heap_page_items` function to view the page items/rows on a page. It takes the raw page data as input and returns the page items. 

We will only focus on the following columns.
- **lp**: location pointer.
- **lp_len**: length of the tuple.
- **t_xmin**: transaction id which inserted the tuple.
- **t_xmax**: transaction id which updated or deleted the tuple.
- **t_ctid**: tuple id.
- **t_data**: bytea representation of the tuple.

Looking into the data, we see that the transaction id is 775 and has a page entry. The **t_xmax** is 0, meaning it's a new entry and has not yet been updated or deleted by any transaction.

```sql
select lp, t_xmin, t_xmax, t_ctid from heap_page_items(get_raw_page('test_table', 0));
 lp | t_xmin | t_xmax | t_ctid
----+--------+--------+--------
  1 |    775 |      0 | (0,1)
(1 row)
```

Now, let's update the tuple and see what the page looks like. 

```sql
begin;
select txid_current(); -- current txn id performing update
 txid_current
--------------
          776
(1 row)

update test_table set name = 'prod' where id = 1;
commit;
```

Now, if we look at the page items, we see that the transaction ids `765` and `766` have a page entry. For the last tuple the **t_xmax** is set to ``765`` which means that it has been updated and the tuple is not invalid. There is a new tuple with **t_xmin** set to `766` and **t_xmax** set to `0`, which is the new version of the tuple. Notice the **t_ctid** for the old tuple is `(0,2)` and for the new tuple is `(0,2)`. The first number is the block number, and the second number is an index in the block. Previously, the tuple was at index `1`; now, it's at index `2`, which means the new tuple is inserted next to the old tuple. We can also infer that the old tuple has not been removed from the disk. 

```sql
select lp, lp_len, t_xmin, t_xmax, t_ctid, t_data from heap_page_items(get_raw_page('test_table', 0));
 lp | lp_len | t_xmin | t_xmax | t_ctid |        t_data
----+--------+--------+--------+--------+----------------------
  1 |     33 |    775 |    776 | (0,2)  | \x010000000b74657374  -- old tuple, inserted by txn 775. (1, 'test')
  2 |     33 |    776 |      0 | (0,2)  | \x010000000b70726f64  -- new tuple, updated by txn 776. (1, 'prod')
(2 rows)
```

**TOAST (The Oversized-Attribute Storage Technique)**

> *“The best thing since sliced bread”*
>
> *- Postgres developers*

{{< notice note >}}
*While this segment may appear abstract and potentially unrelated to vacuuming, it's still beneficial to grasp how Postgres manages large tuples. However, if your primary interest lies solely in vacuuming, you can safely skip over this section.*
{{< /notice >}}

The obvious question is, can a tuple not exceed 8KiB? Actually, it can. Postgres has TOAST to store tuples larger than 8KiB. A TOAST table is a regular table. Whenever the tuple is inserted in a table, it is wider than the `TOAST_TUPLE_THRESHOLD` bytes (typically 2 KiB). The columns larger than 2KiB are stored in a separate table, and a reference to the TOAST table is stored in the main table. This might impact the performance if there are a lot of updates on the columns stored in the TOAST table.

We can identify the tuples larger than 8KiB using the below query.

```sql
select 
 r.id as row_id,
 sum(pg_column_size(r.*)) as row_size
from <table_name> as r
group by r.id
having sum(pg_column_size(r.*)) > 8192
order by sum(pg_column_size(r.*)) desc
limit 10;
```

Let's create a new table and insert a tuple large enough to be stored in the TOAST table.

```sql
create table if not exists test_table_toast (
 id serial primary key,
 large_text text
);
insert into test_table_toast (large_text) select repeat('lorem ipsum dolor sit amet, consectetur adipiscing elit. ', 10000);

select r.id as row_id, sum(pg_column_size(r.*)) as row_size from test_table_toast as r group by r.id;
 row_id | row_size
--------+----------
      1 |     6626 -- the row size is ~6.5KiB
(1 row)
```

If we check the page items, the **lp_len** is only **46 bytes**. But the actual size of the tuple is **6626 bytes**. This is because the actual data is stored in the TOAST table. The `t_data` column contains the reference to the TOAST table.

```sql
select lp, lp_len, t_xmin, t_xmax, t_ctid, t_data from heap_page_items(get_raw_page('test_table_toast', 0));
 lp | lp_len | t_xmin | t_xmax | t_ctid |                     t_data
----+--------+--------+--------+--------+------------------------------------------------
  1 |     46 |    787 |      0 | (0,1)  | \x01000000011294b20800c21900006440000060400000
(1 row)
```

Every table has a corresponding TOAST table. The TOAST table name is `pg_toast.pg_toast_<table_oid>`. The `table_oid` can be found using the below query.

{{< highlight postgresql "linenos=table" >}}
select oid, relname from pg_class where relname = 'test_table_toast';
  oid  |     relname
-------+------------------
 16476 | test_table_toast
(1 row)

\d pg_toast.pg_toast_16476;
TOAST table "pg_toast.pg_toast_16476"
   Column   |  Type
------------+---------
 chunk_id   | oid
 chunk_seq  | integer
 chunk_data | bytea
Owning table: "public.test_table_toast"
Indexes:
    "pg_toast_16476_index" PRIMARY KEY, btree (chunk_id, chunk_seq)
{{< /highlight >}}

## Vacuuming[^1]

Previously, we saw that when a tuple is updated or deleted, the old tuple is still present on the disk. Over time, these dead tuples can grow and eat up disk space. Routine[^5] vaccuming removes these dead tuples and cleans up disk space.

There are two types of vacuuming in Postgres
- **VACUUM**: This is the most basic form of vacuuming. It removes dead tuple versions in the table. It does not reclaim the space occupied by the dead tuples. There can be scenarios where the space is reclaimed by acquiring an `ACCESS EXCLUSIVE` lock on the table.
- **VACUUM FULL**: This is a more aggressive form of vacuuming. It removes dead tuples and reclaims the space. Full vacuuming is a blocking operation. It locks the table and prevents any Data Manipulation Language (DML) operations on the table.

While the vacuuming processing is running, any Data Definition Language (DDL) operations are blocked.

Let us create a table and insert some data.
```sql
truncate test_table; -- remove the existing data

begin;
select txid_current();
 txid_current
--------------
          778  -- txn id 778 performing insert
(1 row)
insert into test_table (id, name) values (1, 'test');
commit;

begin;
select txid_current();
 txid_current
--------------
          779 -- txn id 779 performing insert
(1 row)
insert into test_table (id, name) values (2, 'prod');
commit;
```

```sql
select lp, lp_len, t_xmin, t_xmax, t_ctid, t_data from heap_page_items(get_raw_page('test_table', 0));
 lp | lp_len | t_xmin | t_xmax | t_ctid |        t_data
----+--------+--------+--------+--------+----------------------
  1 |     33 |    778 |      0 | (0,1)  | \x010000000b74657374
  2 |     33 |    779 |      0 | (0,2)  | \x020000000b70726f64
(2 rows)
```

Now, let's update and delete the tuple.

```sql
begin;
select txid_current();
 txid_current
--------------
          780 -- txn id 780 performing update
(1 row)
update test_table set name = 'staging' where id = 1;
commit;

begin;
select txid_current();
 txid_current
--------------
          781 -- txn id 781 performing delete
(1 row)
delete from test_table where id = 2;
commit;
```

```sql
select lp, lp_len, t_xmin, t_xmax, t_ctid, t_data from heap_page_items(get_raw_page('test_table', 0));
 lp | lp_len | t_xmin | t_xmax | t_ctid |           t_data
----+--------+--------+--------+--------+----------------------------
  1 |     33 |    778 |    780 | (0,3)  | \x010000000b74657374       -- inserted by txn 778, updated by txn 780. (1, 'test')
  2 |     33 |    779 |    781 | (0,2)  | \x020000000b70726f64       -- inserted by txn 779, deleted by txn 781. (2, 'prod')
  3 |     36 |    780 |      0 | (0,3)  | \x010000001173746167696e67 -- updated tuple by txn 780. (1, 'staging')
(3 rows)
```


The output below shows that the plain `VACUUM` operation removes the dead tuples but does not reclaim the space they occupy. 

```sql
vacuum test_table;
select lp, lp_len, t_xmin, t_xmax, t_ctid, t_data from heap_page_items(get_raw_page('test_table', 0));
 lp | lp_len | t_xmin | t_xmax | t_ctid |           t_data
----+--------+--------+--------+--------+----------------------------
  1 |      0 |        |        |        |                             -- dead tuple removed but space not reclaimed
  2 |      0 |        |        |        |                             -- dead tuple removed but space not reclaimed
  3 |     36 |    780 |      0 | (0,3)  | \x010000001173746167696e67  -- active tuple
(3 rows)
```

In the below output, we can see that `VACUUM FULL` removes the dead tuples and reclaims the space they occupied. The previous dead tuples with **t_xmin** `778` and `779` are removed and the space is reclaimed.

```sql
vacuum full test_table;
select lp, lp_len, t_xmin, t_xmax, t_ctid, t_data from heap_page_items(get_raw_page('test_table', 0));
 lp | lp_len | t_xmin | t_xmax | t_ctid |           t_data
----+--------+--------+--------+--------+----------------------------
  1 |     36 |    780 |      0 | (0,1)  | \x010000001173746167696e67  -- active tuple
(1 row)
```

#### Phases

The vacuuming process is divided into multiple phases. 

##### Heap Scan

The first phase is the heap scan. In this phase, the visibility map is checked. The idea of a visibility map is to keep track of all the pages that do not have any dead tuples. So only the pages that are not on the visibility map will be scanned. The IDs of the dead tuples are added to a special array called `tids`, which is used in the next phase.

##### Index Vacuuming

This phase scans all the indexes on the table being vacuumed. It identifies the index entries that point to the IDs in the `tids` array and removes them from pages. This phase does not leave any index references to the actual dead tuples, but the dead tuples are still present in the heap.

##### Heap Vacuuming

In this phase, the dead tuples are removed from the heap. The `tids` array is used to identify the dead tuples. The dead tuples are removed from the heap. It can safely remove the dead tuples since the index entries have been removed in the previous phase.

##### Heap Truncating

In this phase, if the process identifies that several pages towards the end of the file are empty, the file will be truncated to save disk space. The truncation might require an `ACCESS EXCLUSIVE` on the table, which can be a blocking operation.

#### Access Exclusive Lock during plain VACUUM

**Why does the `ACCESS EXCLUSIVE` lock get acquired during the `VACUUM` operation?**

This was a TIL[^3] moment, as I always thought plain vacuuming did not require an exclusive lock. 

Performing a plain vacuum operation on a database table does not always require an exclusive lock, except during the [Heap Truncating phase](#heap-truncating). If the vacuum process identifies that several pages towards the end of the file are empty, it will truncate the file to save disk space, which requires an `ACCESS EXCLUSIVE` lock.

The primary node can monitor ongoing Data Manipulation Language (DML) operations on the table and release the lock if needed. However, the replicas cannot distinguish whether a query is blocked by a `VACUUM` operation, which can cause problems for read queries on the replicas, especially in high-traffic environments.


**How can we avoid this?** 

We can disable the `TRUNCATE` operation in the `VACUUM` operation by setting the `vacuum_truncate` parameter to `off.` This will prevent the truncation of pages, so the lock will not be acquired. However, this will not reclaim the space occupied by the dead tuples, which can be a problem in the long run. 

1. The best way to handle this is to vacuum during off-peak hours without disabling the `vacuum_truncate` option. This will ensure that the space occupied by the dead tuples is reclaimed.
2. If the application can handle the downtime, then the `VACUUM FULL` operation can be performed. This will reclaim the space occupied by the dead tuples.
3. If the application cannot handle the downtime, then the vacuuming operation can be performed with `vacuum_truncate` set to `off.` We can explore other options like pg_repack[^4], which can be used to reclaim the space without acquiring the `ACCESS EXCLUSIVE.`

{{< notice tip >}}
If you have a running job that might update or delete a large number of rows, it is better to disable the `TRUNCATE` option in the `VACUUM` operation so that the `ACCESS EXCLUSIVE` is not acquired, which might impact the production traffic.
{{< /notice >}}


[^1]: https://www.postgresql.org/docs/current/sql-vacuum.html
[^2]: https://www.postgresql.org
[^3]: https://knowyourmeme.com/memes/today-i-learned-til
[^4]: https://reorg.github.io/pg_repack/
[^5]: https://www.postgresql.org/docs/current/routine-vacuuming.html