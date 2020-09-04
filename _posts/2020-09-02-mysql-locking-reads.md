---
layout: post
title:  "MySQL Locking Reads"
date:   2020-09-02 00:00:00 -0500
categories: programming
mermaid: true
---

[Locking reads](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking-reads.html) in MySQL allow you to place exclusive locks on rows (or ranges of rows) by selecting said rows inside of a transaction. They seem to be a more fine-grained way to opt-in to `SERIALIZABLE` isolation for specific queries in a session, but they are quite unintuitive and arguably buggy. Misuse of these can reduce the concurrency of operations that's possible on a given table as well as opening you up to deadlocks.

### Setup

To demonstrate how this clause works, I will use the following dummy table:

```
CREATE SCHEMA lockingreads;
USE lockingreads;

CREATE TABLE justpk
(
    A INT,
    B INT,
    PRIMARY KEY (A)
);

INSERT INTO justpk (A, B) VALUES (1, 1), (4, 1), (5, 1);
```

We can therefore visualize the data in the index like so:

<div class="mermaid">
graph
  Root[PK] --> A[1 Gap]
  Root[PK] --> B[1 Record]
  Root[PK] --> C[4 Gap]
  Root[PK] --> D[4 Record]
  Root[PK] --> E[5 Record]
  Root[PK] --> F[Supremum pseudo-record]
</div>

All locking-reads or inserts done in subsequent tests will be preceded by executing `BEGIN;` so that the transaction is left open. This allows us to observe the various locks held after each statement. After that I am running `ROLLBACK;` to release the locks and/or undo the `UPDATE/INSERT`. I am omitting these BEGIN / ROLLBACK calls for brevity as there will be a lot of different test cases here.

I am running these examples on MySQL 8.x because previous versions don't emit sufficient detail about locks held, but according to the documentation these observed behaviors should also apply for versions such as 5.6, for example.

### Viewing locks currently held

In MySQL 8.x, we can query the `performance_schema`.`data_locks` table to view details about specific locks held for a given table. If you are following along, the easiest way to ensure these details are being captured is to use MySQL Workbench and under Administration -> Performance -> Performance Schema Setup choose `Fully Enabled`. If you wanted to capture these details in a Production environment you should do so in a more [fine-grained way](https://dev.mysql.com/doc/refman/8.0/en/performance-schema-quick-start.html).

```
SELECT ENGINE_TRANSACTION_ID, INDEX_NAME, LOCK_TYPE, LOCK_MODE, LOCK_STATUS, LOCK_DATA
FROM `performance_schema`.`data_locks`
WHERE object_name = 'justpk';
```

Column|Description
---|---
ENGINE_TRANSACTION_ID|The transaction the lock is held by
INDEX_NAME|If this is a `RECORD` lock then this will state the specific index whose record the lock is held on
LOCK_TYPE|Can be `TABLE` or `RECORD` for "row-level" locks
LOCK_MODE|Different types of locks are documented [here](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html). Though as we will see later, the annotations used here are not consistent for gap locks, it really depends on what kind of query led to the gap lock being granted.
LOCK_STATUS|Can be `GRANTED` or `WAITING`
LOCK_DATA|For `RECORD` locks this will show the key of the index record being locked. If there is a gap lock held on an empty table or at the "end" of the index then the LOCK_DATA would be `supremum pseudo-record`.

### Locking reads against existing records

Let's start simple and observe what happens if we make a locking read for a row that we know already exists.

```
SELECT * FROM justpk WHERE A = 1 FOR UPDATE;
```

We can see that we have taken an exclusive record (non-gap) lock against the index record whose key is `1`.

INDEX_NAME|LOCK_TYPE|LOCK_MODE|LOCK_DATA
---|---|---|---
PRIMARY|RECORD|X,REC_NOT_GAP|1

We can confirm everything is working as expected by running the following statements in another session:

```
INSERT INTO justpk (A, B) VALUES (0, 1); # allowed
UPDATE justpk SET B = 2 WHERE A = 1; # blocked
INSERT INTO justpk (A, B) VALUES (2, 1); # allowed
INSERT INTO lockingreads.justpk (A, B) VALUES (3, 1); # allowed
UPDATE lockingreads.justpk SET B = 2 WHERE A = 4; # allowed
UPDATE lockingreads.justpk SET B = 2 WHERE A = 5; # allowed
INSERT INTO lockingreads.justpk (A, B) VALUES (6, 1); # allowed
```

<div class="mermaid">
graph
  Root[PK] --> A[1 Gap]
  Root[PK] --> B[1 Record]
  Root[PK] --> C[4 Gap]
  Root[PK] --> D[4 Record]
  Root[PK] --> E[5 Record]
  Root[PK] --> F[Supremum pseudo-record]
  style B fill:red
</div>

### Locking reads against ranges

Another use case may be to place a locking read against a range of records. The behavior of the locking read varies depending on what records exist in the range that is being locked.

```
SELECT * FROM justpk WHERE A BETWEEN 1 AND 4 FOR UPDATE;
```

INDEX_NAME|LOCK_TYPE|LOCK_MODE|LOCK_DATA
---|---|---|---
PRIMARY|RECORD|X,REC_NOT_GAP|1
PRIMARY|RECORD|X|4

We observe to see what is blocked or allowed in a separate session:

```
INSERT INTO lockingreads.justpk (A, B) VALUES (0, 1); # allowed
UPDATE lockingreads.justpk SET B = 2 WHERE A = 1; # blocked
INSERT INTO lockingreads.justpk (A, B) VALUES (2, 1); # blocked
INSERT INTO lockingreads.justpk (A, B) VALUES (3, 1); # blocked
UPDATE lockingreads.justpk SET B = 2 WHERE A = 4; # blocked
UPDATE lockingreads.justpk SET B = 2 WHERE A = 5; # allowed
INSERT INTO lockingreads.justpk (A, B) VALUES (6, 1); # allowed
```

<div class="mermaid">
graph
  Root[PK] --> A[1 Gap]
  Root[PK] --> B[1 Record]
  Root[PK] --> C[4 Gap]
  Root[PK] --> D[4 Record]
  Root[PK] --> E[5 Record]
  Root[PK] --> F[Supremum pseudo-record]
  style B fill:red
  style C fill:red
  style D fill:red
</div>

These results are not surprising, we have blocked updates to records 1 and 4 and blocked inserts into the gap between 1 and 4.

Now let's try the following:

```
SELECT * FROM justpk WHERE A BETWEEN 1 AND 5 FOR UPDATE;
```

INDEX_NAME|LOCK_TYPE|LOCK_MODE|LOCK_DATA
---|---|---|---
PRIMARY|RECORD|X,REC_NOT_GAP|1
PRIMARY|RECORD|X|4
PRIMARY|RECORD|X|5
PRIMARY|RECORD|X|supremum pseudo-record

```
INSERT INTO lockingreads.justpk (A, B) VALUES (0, 1); # allowed
UPDATE lockingreads.justpk SET B = 2 WHERE A = 1; # blocked
INSERT INTO lockingreads.justpk (A, B) VALUES (2, 1); # blocked
INSERT INTO lockingreads.justpk (A, B) VALUES (3, 1); # blocked
UPDATE lockingreads.justpk SET B = 2 WHERE A = 4; # blocked
UPDATE lockingreads.justpk SET B = 2 WHERE A = 5; # blocked
INSERT INTO lockingreads.justpk (A, B) VALUES (6, 1); # blocked
```

<div class="mermaid">
graph
  Root[PK] --> A[1 Gap]
  Root[PK] --> B[1 Record]
  Root[PK] --> C[4 Gap]
  Root[PK] --> D[4 Record]
  Root[PK] --> E[5 Record]
  Root[PK] --> F[Supremum pseudo-record]
  style B fill:red
  style C fill:red
  style D fill:red
  style E fill:red
  style F fill:red
</div>

It is surprising to see here that the insert into the `supremum pseudo-record` is blocked, but inserting into the "1 gap" isn't. The engine should know that taking this lock is not necessary, because we are dealing with a unique index. It seems to know that when it doesn't take a lock on the 1 gap.

Let's see what happens if we select a range that overlaps the min/max values of the index in both directions:

```
SELECT * FROM justpk WHERE A BETWEEN 0 AND 5 FOR UPDATE;
```

INDEX_NAME|LOCK_TYPE|LOCK_MODE|LOCK_DATA
---|---|---|---
PRIMARY|RECORD|X|1
PRIMARY|RECORD|X|4
PRIMARY|RECORD|X|5
PRIMARY|RECORD|X|supremum pseudo-record

Interestingly the lock on the value `1` is no longer marked as `REC_NOT_GAP`.

```
INSERT INTO lockingreads.justpk (A, B) VALUES (0, 1); # blocked
UPDATE lockingreads.justpk SET B = 2 WHERE A = 1; # blocked
INSERT INTO lockingreads.justpk (A, B) VALUES (2, 1); # blocked
INSERT INTO lockingreads.justpk (A, B) VALUES (3, 1); # blocked
UPDATE lockingreads.justpk SET B = 2 WHERE A = 4; # blocked
UPDATE lockingreads.justpk SET B = 2 WHERE A = 5; # blocked
INSERT INTO lockingreads.justpk (A, B) VALUES (6, 1); # blocked
```

<div class="mermaid">
graph
  Root[PK] --> A[1 Gap]
  Root[PK] --> B[1 Record]
  Root[PK] --> C[4 Gap]
  Root[PK] --> D[4 Record]
  Root[PK] --> E[5 Record]
  Root[PK] --> F[Supremum pseudo-record]
  style A fill:red
  style B fill:red
  style C fill:red
  style D fill:red
  style E fill:red
  style F fill:red
</div>

We can see now that the `1 Gap` is locked which makes sense.

### Gap locks are not marked consistently in performance_schema.data_locks

As we can see in the examples above, when doing a locking read on a range of records. Gaps within that range are locked, but there is no explicit gap lock marked in the `data_locks` table. This behavior differs if you try to do a locking read for a record that doesn't exist:

```
SELECT * FROM lockingreads.justpk WHERE A = 2 FOR UPDATE;
```

INDEX_NAME|LOCK_TYPE|LOCK_MODE|LOCK_DATA
---|---|---|---
PRIMARY|RECORD|X,GAP|4

```
INSERT INTO lockingreads.justpk (A, B) VALUES (0, 1); # allowed
UPDATE lockingreads.justpk SET B = 2 WHERE A = 1; # allowed
INSERT INTO lockingreads.justpk (A, B) VALUES (2, 1); # blocked
INSERT INTO lockingreads.justpk (A, B) VALUES (3, 1); # blocked
UPDATE lockingreads.justpk SET B = 2 WHERE A = 4; # allowed
UPDATE lockingreads.justpk SET B = 2 WHERE A = 5; # allowed
INSERT INTO lockingreads.justpk (A, B) VALUES (6, 1); # allowed
```

<div class="mermaid">
graph
  Root[PK] --> A[1 Gap]
  Root[PK] --> B[1 Record]
  Root[PK] --> C[4 Gap]
  Root[PK] --> D[4 Record]
  Root[PK] --> E[5 Record]
  Root[PK] --> F[Supremum pseudo-record]
  style C fill:red
</div>

We can see that the only lock taken is on the `4 Gap`, but it's marked differently in the `data_locks` table. It seems that if you do a locking read on a range, gaps are not explicitly stated whereas if you select specific records then it is.

### Exclusive gap locks are not exclusive

Another interesting finding is that exclusive gap locks DO block inserts into the gap, but > 1 transaction can take an exclusive lock on the same gap at the same time. This is mentioned in the docs but it's a foot-gun and it's quite likely this behavior will prove to be undesirable to you.

```
# session 1
SELECT * FROM lockingreads.justpk WHERE A = 2 FOR UPDATE;

# session 2
SELECT * FROM lockingreads.justpk WHERE A = 3 FOR UPDATE;
```

TRANSACTION_ID|INDEX_NAME|LOCK_TYPE|LOCK_MODE|LOCK_DATA
---|---|---|---|---
1|PRIMARY|RECORD|X,GAP|4
2|PRIMARY|RECORD|X,GAP|4

<div class="mermaid">
graph
  Root[PK] --> A[1 Gap]
  Root[PK] --> B[1 Record]
  Root[PK] --> C[4 Gap]
  Root[PK] --> D[4 Record]
  Root[PK] --> E[5 Record]
  Root[PK] --> F[Supremum pseudo-record]
  style C fill:red
</div>

```
# session 1
INSERT INTO lockingreads.justpk VALUES (2, 1); # blocked

# session 2
INSERT INTO lockingreads.justpk VALUES (3, 1); # blocked, deadlock
```

In this case, our intention was to protect another transaction from inserting a record with a value of 2. Because multiple gap locks are allowed though, we end up deadlocking transactions that want to insert other records into the same gap. These are not clashing inserts so in this case transaction 2 "should" have been able to continue. Just another thing to be aware of if you do choose to use these.

### Conclusion

The behavior of these locking reads is unintuitive at best, buggy at worst. Even with the use of a unique index, it is likely for the locking behavior of these to be "over-eager" and > 1 "exclusive" lock being allowed on the same gap opens you up to unwanted deadlocks. I would test your specific use of these very carefully and recommend you avoid the feature in general as it's likely they won't work for you as you expect.
