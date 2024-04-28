---
title: "Exploring the Enigma of Database Vacuuming"
date: 2024-04-26T19:37:25+04:00
draft: false
categories: [databases]
tags: [postgres, performance]
---

Before we jump into what `VACUUM` does and its gotachs. We need to understand how the data is actually stored on disk. What happens when a tuple is inserted, updated, or deleted? Understanding this will help us understand what `VACUUM` does and its implications.

## Tuples, Page & TOAST

A **tuple** in database terms, both a table's row version and index entries. A tuple is a single row of a table. It contains the actual data. A tuple is made up of the following things.

- **header**: it's a 23-byte header that contains the tuple metadata
- **data**: the actual data of the tuple.

A **page** in the database is a single unit of storage of a fixed size, usually between 4KiB to 16KiB. In Postgres, it's 8KiB. A single page can hold multiple rows, indexes, etc. When the data is written or read from disk, it's done in terms of whole pages. 

```sql
select current_setting('block_size');
 current_setting
-----------------
 8192             -- the size of a single page in Postgres is 8KiB
(1 row)
```
A single page is made up of the following things.

- **header**: it's a 28-byte header that contains the page metadata
- **array of tuple pointers(table of contents)**: the number of tuple pointers is determined by the size of the tuple. Each tuple pointer is 4 bytes. 
- **free space**: the free space on the page is used to store new tuples. Once the free space is exhausted, a new page will be created. 
- **tuples**: rows of the table. Each tuple is 23 bytes plus the size of the data. 
- **special space**: stores things like visibility map, free space map, etc. 

So, a typical page layout looks like the one below.

<img src="images/page_layout.svg" width="50%" style="border:none;" alt="page_layout"/>

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
It returns the following columns
- **lp**: location pointer
- **lp_len**: length of the tuple
- **t_xmin**: transaction ID which inserted the tuple
- **t_xmax**: transaction id which updated or deleted the tuple
- **t_ctid**: tuple id
- **t_data**: bytea representation of the tuple 

Looking into the data, we see that the transaction id is 775 and has a page entry. For this tuple, the **t_xmax** is 0, which means that it's a new entry and has not yet been updated or deleted.

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

Now, if we look at the page items, we see that the transaction ids 765 and 766 have a page entry. For this tuple, the **t_xmax** is 765, which means that it's a new entry and has not yet been updated or deleted. We can see that there are multiple entries for the same tuple. The old tuple is still present on the disk and has not been erased.

```sql
select lp, lp_len, t_xmin, t_xmax, t_ctid, t_data from heap_page_items(get_raw_page('test_table', 0));
 lp | lp_len | t_xmin | t_xmax | t_ctid |        t_data
----+--------+--------+--------+--------+----------------------
  1 |     33 |    775 |    776 | (0,2)  | \x010000000b74657374  -- old tuple, inserted by txn 775. (1, 'test')
  2 |     33 |    776 |      0 | (0,2)  | \x010000000b70726f64  -- new tuple, updated by txn 776. (1, 'prod')
(2 rows)
```

**TOAST (The Oversized-Attribute Storage Technique)**

The obvious question is, can a row not exceed 8KiB? Actually, it can. Postgres has TOAST to store tuples larger than 8KiB. A TOAST table is a regular table. However, rows of these tables are handled in such a way that they are never updated; they can be either added or
deleted, so no versioning. Whenever the tuple is inserted in a table, it is wider than the `TOAST_TUPLE_THRESHOLD` bytes (typically 2 KiB). The columns larger than 2KiB are stored in a separate table, and a reference to the TOAST table is stored in the main table. This might impact the performance if there are a lot of updates on the columns stored in the TOAST table.

We list the tuples larger than 8KiB using the below query. 
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

## [Vacuuming](https://www.postgresql.org/docs/current/sql-vacuum.html)

Previously, we saw that when a tuple is updated or deleted, the old tuple is still present on the disk. Over time, these dead tuples can grow and eat up disk space. Routine vaccuming removes these dead tuples and cleans up disk space. It also keeps the stats table updated. These stats tables are used by the query optimizer to identify the best possible way to execute a query. 

There are two types of vacuuming in Postgres
- **VACUUM**: This is the most basic form of vacuuming. It removes dead tuples versions in the table. It does not reclaim the space occupied by the dead tuples.
- **VACUUM FULL**: This is a more aggressive form of vacuuming. It removes dead tuples and reclaims the space. Full vacuuming is a blocking operation. It locks the table and prevents any `DML` operations on the table. 

While the vacuuming processing is running any `DDL` operations are blocked.

*Let us create a table and insert some data.*

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

*Now, let's update and delete the tuple.* 

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

`VAACUMM` removes the dead tuples and updates the stats table. It does not reclaim the space occupied by the dead tuples in all the cases. There are cases where the space is reclaimed by acquiring an `ACCESS EXCLUSIVE` lock on the table.

```sql
vacuum test_table;
```
After vacuuming, we see that the dead tuples have been removed, but the disk space has not been reclaimed.

```sql
select lp, lp_len, t_xmin, t_xmax, t_ctid, t_data from heap_page_items(get_raw_page('test_table', 0));
 lp | lp_len | t_xmin | t_xmax | t_ctid |           t_data
----+--------+--------+--------+--------+----------------------------
  1 |      0 |        |        |        |                             -- dead tuple removed but space not reclaimed
  2 |      0 |        |        |        |                             -- dead tuple removed but space not reclaimed
  3 |     36 |    780 |      0 | (0,3)  | \x010000001173746167696e67  -- active tuple
(3 rows)
```

`VACUUM FULL` removes the dead tuples, updates the stats table, and reclaims the space occupied by them. 

```sql
vacuum full test_table;
```

After full vacuuming, we see that the dead tuples have been removed, and the disk space has also been reclaimed.

```sql
select lp, lp_len, t_xmin, t_xmax, t_ctid, t_data from heap_page_items(get_raw_page('test_table', 0));
 lp | lp_len | t_xmin | t_xmax | t_ctid |           t_data
----+--------+--------+--------+--------+----------------------------
  1 |     36 |    780 |      0 | (0,1)  | \x010000001173746167696e67  -- active tuple
(1 row)
```

### Phases

The vacuuming process is divided into multiple phases. 

#### Heap Scan

The first phase is the heap scan. In this phase, the visibility map is checked. The idea of visibility map is to mark the pages that do not have any dead tuples. So only the pages that are not on the visibility map will be scanned. The IDs of the dead tuples are added to a special array called `tids`, which is used in the next phase.

#### Index Vacuuming

In this phase, all the indexes on the table being vacuumed are scanned. It identifies the index entries that point to the IDs in the `tids` array. These index entries are removed from pages. This phase does not leave any index references to the actual dead tuples, but the dead tuples are still present in the heap.

#### Heap Vacuuming

In this phase, the dead tuples are removed from the heap. The tid array is used to identify the dead tuples. The dead tuples are removed from the heap. It can safely remove the dead tuples since the index entries have been removed in the previous phase.

#### Heap Truncating

In this phase, if the process identifies that several pages towards the end of the file are empty, the file will be truncated to save disk space. The truncation might require an `ExclusiveLock` on the table, which can be a blocking operation.

### Access Exclusive Lock during plain VACUUM

The `VACUUM` operation does not require an `ACCESS EXCLUSIVE` lock in all the cases. But there are cases where the plain `VACUUM` requires an `ACCESS EXCLUSIVE` lock, mentioned in the Heap Truncating phase. If an `ACCESS EXCLUSIVE` has been acquired, then the table is locked during the `VACUUM` operation. The master can detect if any DML operations are being performed on the table and can release the lock if it has been aquired. But once these locks are propagated to the replicas. On replicas, there is no way to detect if a query is being blocked by a `VACUUM` operation, so the read queries can be blocked indefinitely on the replicas and can cause an incident in a high-traffic environment.

**How can we avoid this?** 

We can disable the `TRUNCATE` operation in the `VACUUM` operation by setting the `vacuum_truncate` parameter to `off.` This will prevent the truncation of pages, and hence, the `ACCESS EXCLUSIVE` will not be acquired. However, this will not reclaim the space occupied by the dead tuples, which can be a problem in the long run. 

1. The best way to handle this is to perform the `VACUUM` operation during off-peak hours without disabling the `TRUNCATE` operation. This will ensure that the space occupied by the dead tuples is reclaimed.
2. If the application can handle the downtime, then the `VACUUM FULL` operation can be performed. This will reclaim the space occupied by the dead tuples.
3. If the application cannot handle the downtime, then the `VACUUM` operation can be performed with `vacuum_truncate` set to `off.` We can explore other options like [pg_repack](https://reorg.github.io/pg_repack/), which can be used to reclaim the space without acquiring the `ACCESS EXCLUSIVE.`

{{< notice tip >}}
If you have a running job that might update or delete many rows, it is better to disable the `TRUNCATE` operation in the `VACUUM` operation so that the `ACCESS EXCLUSIVE` is not acquired and the DML operations are not blocked.
{{< /notice >}}

## Further Reading

- [Postgres Wiki](https://wiki.postgresql.org/wiki/Introduction_to_VACUUM,_ANALYZE,_EXPLAIN,_and_COUNT)
- [Sql Vacuum](https://www.postgresql.org/docs/current/sql-vacuum.html)
- [Routine Vacuuming](https://www.postgresql.org/docs/current/routine-vacuuming.html)
- [Pg_repack](https://reorg.github.io/pg_repack/)