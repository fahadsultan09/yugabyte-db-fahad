---
title: Read Committed isolation level
headerTitle: Read Committed isolation level
linkTitle: Read Committed
description: Details about the Read Committed isolation level
beta: /preview/faq/general/#what-is-the-definition-of-the-beta-feature-tag
menu:
  preview:
    identifier: architecture-read-committed
    parent: architecture-acid-transactions
    weight: 1153
type: docs
---

Read Committed is one of the three isolation levels in PostgreSQL, and also its default. A unique property of this isolation level is that, for transactions running with this isolation, clients don't need to retry/handle serialization errors (40001) in application logic.

The other two isolation levels (Serializable and Repeatable Read) require apps to have retry logic for serialization errors. Read Committed in PostgreSQL works around conflicts by allowing single statements to work on an _inconsistent snapshot_ (in other words, non-conflicting rows are read as of the statement's snapshot, but conflict resolution is done by reading and attempting re-execution/ locking on the latest version of the row).

YSQL supports the Read Committed isolation level, and its behavior is the same as that of PostgreSQL's [Read Committed level (section 13.2.1)](https://www.postgresql.org/docs/13/transaction-iso.html#XACT-READ-COMMITTED).

## Semantics

The following two key semantics set apart Read Committed isolation from Repeatable Read in PostgreSQL:

1. Each statement should be able to read everything that was committed before the statement was issued. In other words, each statement runs on the latest snapshot of the database as of when the statement was issued.
1. Clients never face serialization errors (40001) in read committed isolation level. To achieve this, PostgreSQL re-evaluates statements for conflicting rows based on some rules as described in the next section.

### Read restart errors

In addition to the two key requirements, there is an extra YSQL specific requirement for its read committed isolation level: ensure that external clients don't face `kReadRestart` errors. `Read restart` errors stem from clock skew which is inherent in distributed databases due to the distribution of data across more than one physical node. PostgreSQL doesn't require defining semantics around read restart errors because it is a single node database without clock skew. When there is clock skew, the following situation can arise in a distributed database like YugabyteDB:

* A client starts a distributed transaction by connecting to YSQL on some node N1 in the YugabyteDB cluster and issuing some statement which reads data from multiple shards on different physical YB-TServers in the cluster. For this issued statement, the read point which defines the snapshot of the database at which the data will be read, is picked on some YB-TServer node M based on the current time of that YB-TServer. Depending on the scenario, node M could be the same as N1 or not, but that isn't relevant to this discussion. Consider T1 to be the chosen read time.
* The node N1 might collect data from many shards on different physical YB-TServers. In this pursuit, it will issue requests to many other nodes to read data.
* Assuming that node N1 reads from some node N2, it could be the case that there exists some data written on node N2 at time T2 (> T1) but was written before the read was issued. This can happen because of clock skew where the physical clock on node N2 might be running slightly ahead of node M, and hence the write which was actually done in the past, still has a timestamp higher than T1.
* Note that the clock skew between all nodes in the cluster is always in a [`max_clock_skew`](/preview/reference/configuration/yb-tserver/#max-clock-skew-usec) bound due to clock synchronization algorithms.
* For writes at some time higher than T1 + `max_clock_skew`, the database can be sure that they were done after the read timestamp was chosen on any node. But for writes at a time between T1 and T1 + `max_clock_skew`, node N2 can find itself in an ambiguous situation such as the following:

  * it should still return the data if the client issued the read after the data was committed, because it could be the same client connecting to YSQL from a different node and the following guarantee needs to be maintained; the database always returns data that was committed in the past.

  * it should not return the data if the write was actually performed in the future, that is, after the read point (aka snapshot) was chosen.

* In such a situation, where node N2 finds writes in the range `(T1, T1+max_clock_skew]`, to avoid breaking the strong guarantee of _a reader should always be able to read what was committed earlier_, node N2 avoids giving incorrect results and raises a `Read restart` error.

Some distributed databases handle this uncertainty due to clock skew by using algorithms to maintain a tight bound on the clock skew, and then taking the conservative approach of waiting out the clock skew before acknowledging the commit request from the client for each transaction that writes data. YugabyteDB instead uses various mechanisms internally to reduce the scope of this ambiguity, and if there is still ambiguity in rare scenarios, the error is surfaced to the client.

However, for Read Committed isolation, YugabyteDB has stronger mechanisms in place to ensure that the ambiguity is always resolved internally, and `Read restart` errors are not surfaced to the client. So, in read committed transactions, clients don't have to add any retry logic for `Read restart` errors (similar to serialization errors).

### Handling serialization errors

To handle serialization errors in the database (without surfacing them to the client), PostgreSQL takes the following steps based on the statement type.

#### UPDATE, DELETE, SELECT FOR UPDATE, FOR SHARE, FOR NO KEY UPDATE, FOR KEY SHARE

(The last two are not mentioned in the PostgreSQL documentation, but the same behavior is seen for these as follows.)

If the row of interest:

* is being updated (or deleted) by other concurrent transactions in a conflicting way, wait for the conflicting transactions to commit or rollback, and then perform [recheck steps](#recheck-steps).

* has been updated (or deleted) by other concurrent transactions in a conflicting way, perform [recheck steps](#recheck-steps).

* has been locked by other concurrent transactions in a conflicting way, wait for them to commit or rollback, then perform recheck steps.


##### Recheck steps

The recheck steps are as follows:

1. Read the latest version of the conflicting row and lock it appropriately. The latest version could have a different primary key as well. PostgreSQL finds it by following the chain of updates for a row even across primary key changes. Note that locking is necessary so that another conflict isn't seen on this row while re-evaluating the row again and possibly updating/acquiring a lock on it in step 3. If the locking faces a conflict, it would wait and resume traversing the chain further once unblocked.
1. If the updated version of a row is deleted, ignore it.
1. Apply update/delete/acquire lock on updated version of row if the where clause evaluates to true on the updated version of row.

#### INSERT

1. ON CONFLICT DO UPDATE: if a conflict occurs, wait for the conflicting transaction(s) to commit or rollback.
    1. If all conflicting transaction(s) rollback, proceed as usual.
    1. On commit of any conflicting transaction, traverse the chain of updates as described above and re-evaluate the latest version of the row for any conflict. If there is no conflict, `insert` the original row. Else, perform the `do update` part on the latest version of the row.
1. ON CONFLICT DO NOTHING: do nothing if a conflict occurs.

Note that the preceding methodology in PostgreSQL can lead to two different user visible semantics. One is the common case, and the other is a degenerate situation that can never be seen in practice, but is nevertheless possible and still upholds the semantics of Read Committed isolation. The common case is as follows:

```sql
create table test (k int primary key, v int);
insert into test values (2, 5);
```

<table class="no-alter-colors">
  <thead>
    <tr>
    <th>
    Client 1
    </th>
    <th>
    Client 2
    </th>
    </tr>
  </thead>
  <tbody>
  <tr>
   <td>

```sql
begin transaction isolation level read committed;
```

   </td>
   <td>
   </td>
  </tr>
  <tr>
   <td>
   </td>
   <td>

```sql
begin transaction isolation level read committed;
```

   </td>
  </tr>
  <tr>
   <td>

```sql
insert into test values (5, 5);
```

```output
INSERT 0 1
```

   </td>
   <td>
   </td>
  </tr>
  <tr>
   <td>

```sql
update test set v=10 where k=2;
```

```output
UPDATE 1
```

   </td>
   <td>
   </td>
  </tr>
  <tr>
   <td>
   </td>
   <td>

```sql
update test set v=100 where v>=5;
```

```output
(waits)
```

   </td>
  </tr>
  <tr>
   <td>

```sql
COMMIT;
```

   </td>
   <td>
   </td>
  </tr>

  <tr>
   <td>
   </td>
   <td>

```output
UPDATE 1
```

   </td>
  </tr>

  <tr>
   <td>
   </td>
   <td>

```sql
select * from test;
```

   </td>
  </tr>

<tr>
   <td>
   </td>
   <td>

```output
 k |  v
---+-----
 5 |   5
 2 | 100
(2 rows)
```

   </td>
  </tr>
</tbody>
</table>

As seen above, the UPDATE from transaction 2 first picks the latest snapshot of the database which only has the row (2, 5). The row satisfies the `UPDATE` statement's `WHERE` clause and hence the transaction 2 tries to update the value of `v` from 5 to 100. However, due to an existing conflicting write from transaction 1, it waits for transaction 1 to end. Once transaction 1 commits, it re-reads the latest version of only the conflicting row, and re-evaluates the `WHERE` clause. The clause is still satisfied by the new row (2, 10) and so the value is updated to 100. Note that the newly inserted row (5, 5) isn't updated even though it satisfies the `WHERE` clause of transaction 2's `UPDATE`, because it was not part of the snapshot originally picked by transaction 2's `UPDATE` statement. Hence, it is clear that, to avoid serialization errors, PostgreSQL allows a single statement to run on an inconsistent snapshot, for example: one snapshot which is picked to read all data when the statement is started, and a latest version of the row is used only for each conflicting row as and when required.

The other degenerate scenario that can occur differs in the output of the `UPDATE` in transaction 2:

<table class="no-alter-colors">
  <thead>
    <tr>
    <th>
    Client 1
    </th>
    <th>
    Client 2
    </th>
    </tr>
  </thead>
  <tbody>
  <tr>
   <td>

```sql
update test set v=10 where k=2;
```

```output
UPDATE 1
```

   </td>
   <td>
   </td>
  </tr>
  <tr>
   <td>
   </td>
   <td>

```sql
update test set v=100 where v>=5;
```

```output
(some processing before snapshot is picked, but feels like postgreSQL is waiting due to a conflict)
```

   </td>
  </tr>
  <tr>
   <td>

```sql
COMMIT;
```

   </td>
   <td>
   </td>
  </tr>

  <tr>
   <td>
   </td>
   <td>

```output
UPDATE 2
```

   </td>
  </tr>

  <tr>
   <td>
   </td>
   <td>

```sql
select * from test;
```

   </td>
  </tr>

<tr>
   <td>
   </td>
   <td>

```output
 k |  v
---+-----
 5 | 100
 2 | 100
(2 rows)
```

   </td>
  </tr>
</tbody>
</table>

The above can occur via the following step: till Client 1 commits, PostgreSQL on Client 2 is busy with some other processing and only after Client 1 commits, transaction on Client 2 is able to pick a snapshot based off the current time for the statement. This leads to both rows being read as part of the snapshot and updated, without any conflicts seen. Both outcomes are valid and satisfy the semantics of read committed isolation level. And theoretically, the user can't figure out which one will be seen because the user can't differentiate between a pause due to _waiting for a conflicting transaction_ or a pause due to the database just being _busy_ or _slow_. In the second case, the whole statement runs off a single snapshot and it is easier to reason the output.

These two possibilities show that the client can't have application logic that relies on the expectation that the common case occurs always. Given this, YugabyteDB provides a stronger guarantee that each statement always works off just a single snapshot and no inconsistency is allowed even in case of a some conflicting rows. This leads to YugabyteDB always returning output similar to the second outcome in the above example which is also simpler to reason.

This can change after [#11573](https://github.com/yugabyte/yugabyte-db/issues/11573) as mentioned in the roadmap for read committed isolation [#13557](https://github.com/yugabyte/yugabyte-db/issues/13557).

### Interaction with concurrency control

Semantics of Read Committed isolation adheres only with the [Wait-on-Conflict](../concurrency-control/#wait-on-conflict-beta-preview-faq-general-what-is-the-definition-of-the-beta-feature-tag) concurrency control policy. This is because a Read Committed transaction has to wait for other transactions to commit or rollback in case of a conflict, and then perform the re-check steps to make progress.

As the [Fail-on-Conflict](../concurrency-control/#fail-on-conflict) concurrency control policy doesn't make sense for Read Committed, even if this policy is set for use on the cluster (by having the TServer flag `enable_wait_queues=false`), transactions in Read Committed isolation will still provide `Wait-on-Conflict` semantics. For providing `Wait-on-Conflict` semantics without wait queues, YugabyteDB relyies on an indefinite retry-backoff mechanism with exponential delays when conflicts are detected. The retries are at the statement level. Each retry will use a newer snapshot of the database in anticipation that the conflicts might not occur. This is done because if the read time of the new snapshot is higher than the commit time of the earlier conflicting transaction T2, the conflicts with T2 would essentially be voided since T1's statement and T2 would no longer be "concurrent".

However, when Read Committed isolation provides Wait-on-Conflict semantics without wait queues, the following limitations exist:
- You may have to manually tune the exponential backoff parameters for performance as explained in [Performance tuning](../read-committed/#performance-tuning).
- The app may have to rely on statement timeouts to [avoid deadlocks](#avoid-deadlocks-in-read-committed-transactions).
- There may be unfairness during contention due to the retry-backoff mechanism, resulting in high P99 latencies.

## Usage

By setting the YB-TServer gflag `yb_enable_read_committed_isolation=true`, the syntactic `Read Committed` isolation in YSQL will actually map to the Read Committed implementation in DocDB. If set to false, it will have the earlier behavior of mapping syntactic `Read Committed` on YSQL to Snapshot Isolation in DocDB, meaning it will behave same as `Repeatable Read`.

The following ways can be used to start a Read Committed transaction after setting the gflag:

1. `START TRANSACTION isolation level read committed [read write | read only];`
2. `BEGIN [TRANSACTION] isolation level read committed [read write | read only];`
3. `BEGIN [TRANSACTION]; SET TRANSACTION ISOLATION LEVEL READ COMMITTED;` (this will be supported after [#12494](https://github.com/yugabyte/yugabyte-db/issues/12494))
4. `BEGIN [TRANSACTION]; SET SESSION CHARACTERISTICS AS TRANSACTION ISOLATION LEVEL READ COMMITTED;` (this will be supported after #12494)

## Examples

Start by setting up the table you'll use in all of the examples in this section.

```sql
create table test (k int primary key, v int);
```

### Avoid deadlocks in Read Committed transactions

When wait queues are not enabled, that is, `enable_wait_queues=false`, configure a statement timeout to avoid deadlocks. A different statement timeout can be set for each session using the `statement_timeout` session variable. Also, a single statement timeout can be applied globally to all sessions by setting the `statement_timeout` session variable in `ysql_pg_conf_csv` YB-TServer gflag on cluster startup.

```sql
truncate table test;
insert into test values (1, 5);
insert into test values (2, 5);
```

<table class="no-alter-colors">
  <thead>
    <tr>
    <th>
    Client 1
    </th>
    <th>
    Client 2
    </th>
    </tr>
  </thead>
  <tbody>
  <tr>
   <td>

```sql
begin transaction isolation level read committed;
```

   </td>
   <td>
   </td>
  </tr>
  <tr>
   <td>
   </td>
   <td>

```sql
begin transaction isolation level read committed;
```

   </td>
  </tr>
  <tr>
   <td>
   </td>
   <td>

```sql
set statement_timeout=2000;
```

   </td>
  </tr>
  <tr>
   <td>

```sql
update test set v=5 where k=1;
```

```output
UPDATE 1
```

   </td>
   <td>
   </td>
  </tr>
  <tr>
   <td>
   </td>
   <td>

```sql
update test set v=5 where k=2;
```

```output
UPDATE 1
```

   </td>
  </tr>
  <tr>
   <td>

```sql
update test set v=5 where k=2;
```

```output
(waits)
```

   </td>
   <td>
   </td>
  </tr>
  <tr>
   <td>
   </td>
   <td>

```sql
update test set v=5 where k=1;
```

```output
ERROR:  cancelling statement due to statement timeout
```

   </td>
  </tr>
  <tr>
   <td>
   </td>
   <td>

```sql
rollback;
```

   </td>
  </tr>
  <tr>
   <td>

```output
UPDATE 1
```

   </td>
   <td>
   </td>
  </tr>
  <tr>
   <td>

```sql
commit;
```

   </td>
   <td>
   </td>
  </tr>
</tbody>
</table>

### SELECT behavior without explicit locking

```sql
truncate table test;
insert into test values (1, 5);
```

<table class="no-alter-colors">
  <thead>
    <tr>
    <th>
    Client 1
    </th>
    <th>
    Client 2
    </th>
    </tr>
  </thead>
  <tbody>
  <tr>
   <td>

```sql
begin transaction isolation level read committed;
```

   </td>
   <td>
   </td>
  </tr>
  <tr>
   <td>
   </td>
   <td>

```sql
begin transaction isolation level read committed;
```

   </td>
  </tr>
  <tr>
   <td>

```sql
select * from test where v=5;
```

```output
 k | v
---+---
 1 | 5
(1 row)
```

   </td>
   <td>
   </td>
  </tr>
  <tr>
   <td>
   </td>
   <td>

```sql
insert into test values (2, 5);
```

```output
INSERT 0 1
```

   </td>
  </tr>
  <tr>
   <td>

```sql
select * from test where v=5;
```

```output
 k | v
---+---
 1 | 5
(1 row)
```

   </td>
   <td>
   </td>
  </tr>
  <tr>
   <td>

```sql
insert into test values (3, 5);
```

```output
INSERT 0 1
```

   </td>
   <td>
   </td>
  </tr>
  <tr>
   <td>

```sql
select * from test where v=5;
```

```output
 k | v
---+---
 1 | 5
 3 | 5
(2 rows)
```

   </td>
   <td>
   </td>
  </tr>
  <tr>
   <td>
   </td>
   <td>

```sql
commit;
```

   </td>
  </tr>
  <tr>
   <td>

```sql
select * from test where v=5;
```

```output
 k | v
---+---
 1 | 5
 2 | 5
 3 | 5
(3 rows)
```

   </td>
   <td>
   </td>
  </tr>
  <tr>
   <td>

```sql
commit;
```

   </td>
   <td>
   </td>
  </tr>
</tbody>
</table>

### UPDATE behavior

```sql
truncate table test;
insert into test values (0, 5), (1, 5), (2, 5), (3, 5), (4, 1);
```

<table class="no-alter-colors">
  <thead>
    <tr>
    <th>
    Client 1
    </th>
    <th>
    Client 2
    </th>
    </tr>
  </thead>
  <tbody>
  <tr>
   <td>

```sql
begin transaction isolation level read committed;
```

   </td>
   <td>
   </td>
  </tr>
  <tr>
   <td>
   </td>
   <td>

```sql
begin transaction isolation level read committed;
```

   </td>
  </tr>
  <tr>
   <td>
   </td>
   <td>

```sql
insert into test values (5, 5);
```

```output
INSERT 0 1
```

   </td>
  </tr>
  <tr>
   <td>
   </td>
   <td>

```sql
update test set v=10 where k=4;
```

```output
UPDATE 1
```

   </td>
  </tr>
  <tr>
   <td>
   </td>
   <td>

```sql
delete from test where k=3;
```

```output
DELETE 1
```

   </td>
  </tr>
  <tr>
   <td>
   </td>
   <td>

```sql
update test set v=10 where k=2;
```

```output
UPDATE 1
```

   </td>
  </tr>
  <tr>
   <td>
   </td>
   <td>

```sql
update test set v=1 where k=1;
```

```output
UPDATE 1
```

   </td>
  </tr>
  <tr>
   <td>
   </td>
   <td>

```sql
update test set k=10 where k=0;
```

```output
UPDATE 1
```

   </td>
  </tr>
  <tr>
   <td>

```sql
update test set v=100 where v>=5;
```

```output
(waits)
```

   </td>
   <td>
   </td>
  </tr>
  <tr>
   <td>
   </td>
   <td>

```sql
commit;
```

   </td>
  </tr>
  <tr>
   <td>

```output
UPDATE 4
```

```sql
select * from test;
```

```output
 k  |  v
----+-----
  5 | 100
  1 |   1
 10 | 100
  4 | 100
  2 | 100
(5 rows)
```

   </td>
   <td>
   </td>
  </tr>
  <tr>
   <td>

```sql
commit;
```

   </td>
   <td>
   </td>
  </tr>
</tbody>
</table>

### SELECT FOR UPDATE behavior

```sql
truncate table test;
insert into test values (0, 5), (1, 5), (2, 5), (3, 5), (4, 1);
```

<table class="no-alter-colors">
  <thead>
    <tr>
    <th>
    Client 1
    </th>
    <th>
    Client 2
    </th>
    </tr>
  </thead>
  <tbody>
  <tr>
   <td>

```sql
begin transaction isolation level read committed;
```

   </td>
   <td>
   </td>
  </tr>
  <tr>
   <td>
   </td>
   <td>

```sql
begin transaction isolation level read committed;
```

   </td>
  </tr>
  <tr>
   <td>
   </td>
   <td>

```sql
insert into test values (5, 5);
```

```output
INSERT 0 1
```

   </td>
  </tr>
  <tr>
   <td>
   </td>
   <td>

```sql
update test set v=10 where k=4;
```

```output
UPDATE 1
```

   </td>
  </tr>
  <tr>
   <td>
   </td>
   <td>

```sql
delete from test where k=3;
```

```output
DELETE 1
```

   </td>
  </tr>
  <tr>
   <td>
   </td>
   <td>

```sql
update test set v=10 where k=2;
```

```output
UPDATE 1
```

   </td>
  </tr>
  <tr>
   <td>
   </td>
   <td>

```sql
update test set v=1 where k=1;
```

```output
UPDATE 1
```

   </td>
  </tr>
  <tr>
   <td>
   </td>
   <td>

```sql
update test set k=10 where k=0;
```

```output
UPDATE 1
```

   </td>
  </tr>
  <tr>
   <td>

```sql
select * from test where v>=5 for update;
```

```output
(waits)
```

   </td>
   <td>
   </td>
  </tr>
  <tr>
   <td>
   </td>
   <td>

```sql
commit;
```

   </td>
  </tr>
  <tr>
   <td>

```output
 k  | v
----+----
  5 |  5
 10 |  5
  4 | 10
  2 | 10
(4 rows)
```

   </td>
   <td>
   </td>
  </tr>
  <tr>
   <td>

```sql
commit;
```

   </td>
   <td>
   </td>
  </tr>
</tbody>
</table>

### INSERT behavior

**Insert a new key that is also just changed by another transaction**

```sql
truncate table test;
insert into test values (1, 1);
```

<table class="no-alter-colors">
  <thead>
    <tr>
    <th>
    Client 1
    </th>
    <th>
    Client 2
    </th>
    </tr>
  </thead>
  <tbody>
  <tr>
   <td>

```sql
begin transaction isolation level read committed;
```

   </td>
   <td>
   </td>
  </tr>
  <tr>
   <td>
   </td>
   <td>

```sql
begin transaction isolation level read committed;
```

   </td>
  </tr>
  <tr>
   <td>
   </td>
   <td>

```sql
update test set k=2 where k=1;
```

```output
UPDATE 1
```

   </td>
  </tr>
  <tr>
   <td>

```sql
insert into test values (2, 1);
```

```output
(waits)
```

   </td>
   <td>
   </td>
  </tr>
  <tr>
   <td>
   </td>
   <td>

```sql
commit;
```

   </td>
  </tr>
  <tr>
   <td>

```output
ERROR:  duplicate key value violates unique constraint "test_pkey"
```

   </td>
   <td>
   </td>
  </tr>
  <tr>
   <td>

```sql
rollback;
```

   </td>
   <td>
   </td>
  </tr>
</tbody>
</table>

**Same as previous, but with ON CONFLICT**

```sql
truncate table test;
insert into test values (1, 1);
```

<table class="no-alter-colors">
  <thead>
    <tr>
    <th>
    Client 1
    </th>
    <th>
    Client 2
    </th>
    </tr>
  </thead>
  <tbody>
  <tr>
   <td>

```sql
begin transaction isolation level read committed;
```

   </td>
   <td>
   </td>
  </tr>
  <tr>
   <td>
   </td>
   <td>

```sql
begin transaction isolation level read committed;
```

   </td>
  </tr>
  <tr>
   <td>
   </td>
   <td>

```sql
update test set k=2 where k=1;
```

```output
UPDATE 1
```

   </td>
  </tr>
  <tr>
   <td>

```sql
insert into test values (2, 1) on conflict (k) do update set v=100;
```

```output
(waits)
```

   </td>
   <td>
   </td>
  </tr>
  <tr>
   <td>
   </td>
   <td>

```sql
commit;
```

   </td>
  </tr>
  <tr>
   <td>

```output
INSERT 0 1
```

   </td>
   <td>
   </td>
  </tr>
  <tr>
   <td>

```sql
select * from test;
```

```output
 k |  v
---+-----
 2 | 100
(1 row)
```

   </td>
   <td>
   </td>
  </tr>
  <tr>
   <td>

```sql
commit;
```

   </td>
   <td>
   </td>
  </tr>
</tbody>
</table>

**INSERT old key that is removed by other transaction**

```sql
truncate table test;
insert into test values (1, 1);
```

<table class="no-alter-colors">
  <thead>
    <tr>
    <th>
    Client 1
    </th>
    <th>
    Client 2
    </th>
    </tr>
  </thead>
  <tbody>
  <tr>
    <td>

```sql
begin transaction isolation level read committed;
```

  </td>
  <td>
  </td>
  </tr>
  <tr>
   <td>
   </td>
   <td>

```sql
begin transaction isolation level read committed;
```

   </td>
  </tr>
  <tr>
   <td>
   </td>
   <td>

```sql
update test set k=2 where k=1;
```

```output
UPDATE 1
```

   </td>
  </tr>
  <tr>
   <td>

```sql
insert into test values (1, 1);
```

```output
(waits)
```

   </td>
   <td>
   </td>
  </tr>
  <tr>
   <td>
   </td>
   <td>

```sql
commit;
```

   </td>
  </tr>
  <tr>
   <td>

```output
INSERT 0 1
```

   </td>
   <td>
   </td>
  </tr>
  <tr>
   <td>

```sql
select * from test;
```

```output
 k | v
---+---
 1 | 1
 2 | 1
(2 rows)
```

   </td>
   <td>
   </td>
  </tr>
  <tr>
   <td>

```sql
commit;
```

   </td>
   <td>
   </td>
  </tr>
</tbody>
</table>

**Same as previous, but with ON CONFLICT**

```sql
truncate table test;
insert into test values (1, 1);
```

<table class="no-alter-colors">
<thead>
  <tr>
   <th>
   Client 1
   </th>
   <th>
   Client 2
   </th>
  </tr>
</thead>
<tbody>
  <tr>
   <td>

```sql
begin transaction isolation level read committed;
```

   </td>
   <td>
   </td>
  </tr>
  <tr>
   <td>
   </td>
   <td>

```sql
begin transaction isolation level read committed;
```

   </td>
  </tr>
  <tr>
   <td>
   </td>
   <td>

```sql
update test set k=2 where k=1;
```

```output
UPDATE 1
```

   </td>
  </tr>
  <tr>
   <td>

```sql
insert into test values (1, 1) on conflict (k) do update set v=100;
```

```output
(waits)
```

   </td>
   <td>
   </td>
  </tr>
  <tr>
   <td>
   </td>
   <td>

```sql
commit;
```

   </td>
  </tr>
  <tr>
   <td>

```output
INSERT 0 1
```

   </td>
   <td>
   </td>
  </tr>
  <tr>
   <td>

```sql
select * from test;
```

```output
 k |  v
---+-----
 1 |  1
 2 |  1
(2 rows)
```

   </td>
   <td>
   </td>
  </tr>
  <tr>
   <td>

```sql
commit;
```

   </td>
   <td>
   </td>
  </tr>
</tbody>
</table>

## Cross feature interaction

This feature interacts with the following features:

1. **Follower reads (integration in progress):** When follower reads is turned on, the read point for each statement in a Read Committed transaction will be picked as _Now()_ - _yb_follower_read_staleness_ms_ (if the transaction/statement is known to be explicitly/implicitly read only).

## Limitations

Refer to [#13557](https://github.com/yugabyte/yugabyte-db/issues/13557) for limitations.

## Noteworthy considerations

This isolation level allows both phantom and non-repeatable reads (as in [SELECT behavior without explicit locking](#select-behavior-without-explicit-locking)).

Adding this new isolation level doesn't affect the performance of existing isolation levels.

### Performance tuning

If a statement in the Read Committed isolation level faces a conflict, it will be retried with exponential backoff till the statement times out. The following parameters control the backoff:

* _retry_max_backoff_ is the maximum backoff in milliseconds between retries.
* _retry_min_backoff_ is the minimum backoff in milliseconds between retries.
* _retry_backoff_multiplier_ is the multiplier used to calculate the next retry backoff.

You can set these parameters on a per-session basis, or in the `ysql_pg_conf_csv` YB-TServer gflag on cluster startup.

If the [Wait-on-Conflict](../concurrency-control/#wait-on-conflict) concurrency control policy is enabled, there won't be a need to manually tune these parameters for performance. Statements will restart only when all conflicting transactions have committed or rolled back, instead of retrying with an exponential backoff.
