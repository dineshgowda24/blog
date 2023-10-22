---
title: "Demystifying Database Transactions"
date: 2023-07-22T20:55:32+05:30
draft: false
categories: [databases]
tags: [transactions, postgres, concurrency]
---

The most compelling feature of relational database systems is ACID(Atomicity Consistency Isolation Durability). Isolation is achieved using transactions. Isolation is needed to avoid race conditions when concurrent actors act upon the same row.

SQL standard defines isolation with different levels. Every isolation level offers certain guarantees and possible anomalies that can occur. Understanding the internals of isolation levels and their anomalies helps us identify the right level for the use case and work with the anomalies.

This post will look at the isolation levels and their anomalies. We will also look at how Postgres implements isolation levels.

## Sample Database

We will use the following table for all the examples.

```postgresql
create table wallet(id int, name text, balance int);
insert into wallet values (1, 'luffy', 1000), (2, 'zoro', 500), (3, 'luffy', 500);
select * from wallet;

 id | name  | balance
----+-------+---------
  1 | luffy |  1000
  2 | zoro  |  500
  3 | luffy |  500
(3 rows)
```

## Isolation Levels

As per SQL standards, there are four isolation levels, each offering certain guarantees and possible anomalies that can occur. No one knows how many anomalies actually exist in the wild. The following table shows the isolation levels and the anomalies that can occur.

| Isolation Levels                   | Dirty Read             | Non Repeatable Read  | Phantom Read       |
|------------------------------------|------------------------|----------------------|--------------------|
| Read Uncommited                    | possible(not in PG)    | possible             | possible           |
| Read Commited                      | not possible           | possible             | possible           |
| Repeatable Read/Snapshot Isolation | not possible           | not possible         | possible(not in PG)           |
| Seralizable                        | not possible           | not possible         | not possible       |

### Read Uncommitted

In read uncommitted, any concurrent transactions can read all the uncommitted data of other transactions. Though SQL standard has it, most database systems do not implement it as there is no practical use. Postgres, by default, uses read committed isolation level. Even if we set the isolation level to read uncommitted, Postgres will use the read-committed isolation level.

```postgresql
show default_transaction_isolation;
 default_transaction_isolation
 -------------------------------
 read committed
(1 row)
```

### Read Committed

As the name suggests, the read committed isolation level guarantees that any transaction can read only committed data. This is the default isolation level in Postgres.

#### Non-Repeatable Read

A non-repeatable read is a phenomenon in which a transaction reads the same row twice and gets different results. This can happen when a transaction reads a row and another transaction updates it before the first transaction rereads it.

The anomaly can be prevented using higher isolation levels like repeatable read or serializable. But this comes at a cost of performance. The performance cost is because the database system has to take a snapshot of the database when the transaction starts. The snapshot is used for all read operations in the transaction. Any changes made by other transactions after the snapshot are not visible to the transaction.

|timeline|t1|t2|comments|
|:----|:----|:----|:----|
|1|`begin; set transaction isolation level read committed`| | t1 starts | |
|2| |`begin; set transaction isolation level read committed`| t2 starts | |
|3|`select balance as b1 from wallet where id = 1`| | returns 1000 | |
|4| |`update wallet set balance = 1100 where id = 1`| | |
|5| |`commit`| t2 ends | |
|6|`select balance as b1 from wallet where id = 1`| | returns 1100(value of b1 changed when the row was read twice within the same transcation) | |
|8|`commit`| | t1 ends| |

#### Read Skew

Read skew is a phenomenon where a transaction reads data that is inconsistent. This can happen when a transaction reads data from two different transactions. The data read from the first transaction is consistent, but the data read from the second transaction is inconsistent.

|timeline|t1|t2|comments|
|:----|:----|:----|:----|
|1|`begin; set transaction isolation level read committed`| | t1 starts | |
|2| |`begin; set transaction isolation level read committed`| t2 starts | |
|3|`select balance as b1 from wallet where id = 1`| | returns 1000 | |
|4| |`update wallet set balance = 1100 where id = 1`| | |
|5| |`update wallet set balance = 400 where id = 2`| | |
|6| |`commit`| t2 ends | |
|7|`select balance as b2 from wallet where id = 2`| | 400 | |
|8| b1 + b2 | | 1400(inconsistent data due to read skew) | |
|8|`commit`| | t1 ends| |

#### Lost Updates(W-W Conflict)

A lost update happens when one transaction overwrites the changes of another transaction. The overwriting transaction may be committed later, but the changes made by the first transaction are lost. The database system does not know that 1000$ is related to wallet.balance, so it cannot circumvent the lost update.
One of the ways to prevent lost updates is to use atomic operations.

|timeline|t1|t2|comments|
|:----|:----|:----|:----|
|1|`begin; set transaction isolation level read committed`| | t1 starts | |
|2| |`begin; set transaction isolation level read committed`| t2 starts | |
|3|`select balance from wallet where id = 1`| | returns 1000 | |
|4| |`select balance from wallet where id = 1`| returns 1000| |
|5|`update wallet set balance = 1000+ 500 where id = 1`||adds 500 to the balance| |
|6| |`update wallet set balance = 1000+ 500 where id = 1`|adds 500 to the balance||
|7|`commit`| | t1 ends | |
|8||`commit`| t2 ends | |

```postgresql
select balance from wallet where id = 1; -- returns 1500(inconsistent data due to lost update)
```

#### Atomic Operations

Atomic write operations read records and write them back with the new values. The read and write operations are performed in a single step. This prevents lost updates.

|timeline|t1|t2|comments|
|:----|:----|:----|:----|
|1|`begin; set transaction isolation level read committed`| | t1 starts | |
|2| |`begin; set transaction isolation level read committed`| t2 starts | |
|3|`select balance from wallet where id = 1`| | returns 1000 | |
|4| |`select balance from wallet where id = 1`| returns 1000| |
|5|`update wallet set balance = (balance + 500) where id = 1`||adds 500 to the balance(atomic update)| |
|6| |`update wallet set balance = (balance + 500) where id = 1`|adds 500 to the balance(atomic update), t2 is blocked until t1 commits||
|7|`commit`| | t1 ends | |
|8||`commit`| t2 ends | |

```postgresql
select balance from wallet where id = 1; -- returns 2000
```

#### Explicit Locking

Explicit locking can prevent lost updates. A transaction can lock a row and prevent other transactions from modifying it. The lock is released when the transaction commits or rolls back. Any transaction that tries to alter a locked row is blocked until the lock is released.

|timeline|t1|t2|comments|
|:----|:----|:----|:----|
|1|`begin; set transaction isolation level read committed`| | t1 starts | |
|2| |`begin; set transaction isolation level read committed`| t2 starts | |
|3|`select balance from wallet where id = 1 for update`| | returns returns 1000 and row is locked| |
|4| |`select balance from wallet where id = 1 for update`| returns 1500 once lock is released(its blocked until T1 commits with updated balance due to explicit lock)| |
|5|`update wallet set balance = 1500 where id = 1`||adds 500 to the balance| |
|6| |`update wallet set balance = 2000 where id = 1`|adds 500 to the balance, t2 is blocked until T1 commits||
|7|`commit`| | t1 ends | |
|8||`commit`| t2 ends | |

```postgresql
select balance from wallet where id = 1; -- returns 2000
```

The wait time to acquire the lock can be configured using [`lock_timeout`](https://www.postgresql.org/docs/current/runtime-config-client.html#GUC-LOCK-TIMEOUT). The default value is `0`, which disables timeout and the transaction waits indefinitely.

```postgresql
show lock_timeout;
 lock_timeout
--------------
 0
(1 row)
```

### Repeatable Read/Snapshot Isolation

At this isolation level, the standard guarantees repeatable reads. This means that if a transaction reads a row twice, it will get the same result. This is achieved by taking a snapshot of the database when the transaction starts. Any changes made by other transactions after the snapshot are not visible to the transaction.

Postgres guarantees that non-repeatable and phantom read anomalies are impossible at this isolation level. We can see that in the following example.

|timeline|t1|t2|comments|
|:----|:----|:----|:----|
|1|`begin; set transaction isolation level repeatable read;`| | t1 starts | |
|2|`select balance from wallet where id = 1;`| | returns 1000| |
|3|`select balance from wallet where balance > 250;`| | returns 1000 and 500| |
|4| |`begin;`| t2 starts with read committed isolation | |
|5| |`update wallet set balance = 100 where id = 1;`| | |
|6| |`commit;`| t2 ends | |
|7|`select balance from wallet where id = 1;`| | still returns 1000 i.e no non repeatable read| |
|8|`select balance from wallet where balance > 250;`| | still returns 1000 and 500, i.e no phantom read| |
|9|`commit;`| | t1 ends | |

#### Multi Version Concurrency Control(MVCC)

Postgres uses MVCC(Multi-Version Concurrency Control) to implement repeatable read isolation levels. MVCC creates a new version, i.e., a snapshot of a row when it is updated. The old version is retained and is visible to transactions that started before the update. The new version is visible to transactions that initiate after the update.

#### Detecting Lost Updates

We know lost updates can be prevented using atomic operations or explicit locking at the read committed isolation level, making the transaction serializable. But at a repeatable read isolation level, the transaction can run concurrently with other transactions. The database system can detect the lost updates and roll back the transaction with a serialization failure.

```postgresql
ERROR:  could not serialize access due to concurrent update
```

The application can then retry the update transaction.

#### Write skew

Write skew happens when the data consistency is violated by two transactions operating on two different rows. Suppose it is allowed to have a negative wallet balance, given the cumulative balance of all wallets is positive. If two transactions try to update the balance of two different wallets of the same user, the user's balance can become negative.

|timeline|t1|t2|comments|
|:----|:----|:----|:----|
|1|`begin; set transaction isolation level repeatable read;`| | t1 starts | |
|2|`select sum(balance) from wallet where name = 'luffy';`| | returns 500| |
|3||`begin; set transaction isolation level repeatable read;`| t2 starts | |
|4||`select sum(balance) from wallet where name = 'luffy';`| returns 1500| |
|5|`update wallet set balance = balance - 1000 where id = 1;`| | total balance of luffy is 750| |
|6||`update wallet set balance = balance - 1000 where id = 3;`| t2 also deducts 1000 from the balance of luffy| |
|7|`commit;`| | t1 ends | |
|8||`commit;`| t2 ends | |

The database system cannot know if the two updates are related, so it cannot prevent the write skew and the data consistency is lost.

```postgresql
select * from wallet where name = 'luffy';
 id | name  | balance
----+-------+---------
  1 | luffy |  0
  3 | luffy |  -500

(2 rows)
```

The write skew can be prevented by using explicit locking. All the rows of a user are locked before updating any of them. This prevents other transactions from editing any of the rows.

```postgresql
select * from wallet where name = 'luffy' for update;
```

#### Read Only

It was assumed that the read-only transactions are always serializable under Snapshot isolation, i.e. SI, provided the concurrent update transactions are serializable. But this is not true. This is because all SI reads return values from a single instant of time(a snapshot) when all committed transactions have completed their writes, and no writes of uncommitted transactions are visible before the read-only transaction starts. This implies that read-only transactions will not read anomalous results as long as the updated transactions they execute do not write such results.

Consider the following example. Let's make luffy's wallet balance 0.

```postgresql
update wallet set balance = 0 where name = 'luffy';
```

|timeline|t1|t2|t3|comments|
|:----|:----|:----|:----|:----|
|1|`begin; set transaction isolation level repeatable read;`| | | t1 starts | |
|2|`update wallet set balance = 1000 where id = '1';`| | | deposit 1000 to luffy's wallet(1)| |
|3||`begin; set transaction isolation level repeatable read;`| | t2 starts | |
|4||`update wallet set balance = -11 where id = '3';`| | withdraw 10 from luffy's wallet(3) and add a overdraft fee of 1 since the balance is negative| |
|5|`commit;`| | | t1 ends | |
|6|||`begin; set transaction isolation level repeatable read;`| t3 starts | |
|7|||`select * from wallet where name = 'luffy';`| returns 1000 and 0| |
|8|||`commit;`| t3 ends | |
|9||`commit;`| | t2 ends | |

```postgresql
select * from wallet where name = 'luffy';
 id | name  | balance
----+-------+---------
  1 | luffy |  1000
  3 | luffy |  -11

(2 rows)
```

Serializability states that the result is the same as if the operations were executed in some sequential order, meaning there is no overlap in execution.

When the read-only transaction t3 read the balance of Luffy's wallet, the balance of wallet(3) was 0, and the balance of wallet(1) was 1000. This is an anomalous result. This indicates that the deposit of 1000 came first and then the withdrawal of 10. If that was the case, then the overdraft fee 1 should not have been applied in t2 as the balance was not negative when the withdrawal happened. This is a violation of serializability.

{{< notice tip >}}
If your application uses the Repeatable Read isolation level for writing, it must be ready to retry rolled-back transactions with a serialization failure.
{{< /notice >}}

### Serializable

Serializable isolation level guarantees that all transactions are executed serially. This is the highest isolation level and comes at a cost of performance. The application must retry the transaction if it fails due to serialization failure.

Standard states that this isolation level has no anomalies. But no one knows how many anomalies actually exist in the wild. Postgres guarantees that all anomalies are not possible at this isolation level.

However, due to performance cost, a large portion of retry overhead makes it less attractive.

## Follow up

If you have any questions, feel free to reach out to me on [X](https://twitter.com/_dineshgowda) or comment below.

**Update**: *The post was featured on few newletters.*

{{< tweet user="ModernSQL" id="1708732428305498182" >}}
{{< tweet user="eatonphil" id="1707485078870163614" >}}
