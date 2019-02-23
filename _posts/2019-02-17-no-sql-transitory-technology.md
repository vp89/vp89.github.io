---
layout: post
title:  "NoSQL, a transitory technology"
date:   2019-02-19 00:00:00 -0000
categories: programming
---

I recently watched this talk on [Advanced DynamoDB Design Patterns](https://www.youtube.com/watch?v=HaEPXoXVf2k) in preparation for some upcoming work. I ended up watching it multiple times and sharing it with colleagues because I felt it gave me, a relative outsider to the NoSQL world, a good feel for what it's all about and why people have been using it.

The presenter states that we have yet to hit the "slope of enlightenment" with regards to NoSQL adoption because most people who are currently using it are not using it properly.

<img src="/dynamo-hype-curve.png" />

The gist of it is that NoSQL is quite unforgiving and you need to be very clever up front in the way you model your data to fit all your access patterns. One of the recommendations made is to put all your data in the same table or key space. You then use clever combinations of composite keys in your partition and sort keys to differentiate the types of data and have efficient look-ups. This is reiterated in this AWS documentation about [NoSQL design best practices](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/bp-general-nosql-design.html#bp-general-nosql-design-concepts).

It must be a lot of fun to be a NoSQL consultant. You go in to a company, work with them to enumerate all their access patterns and then design an extremely clever one-table contraption that performs well and is capable of holding practically infinite amounts of data. You then hold the new table design up in the air like a solved Rubik's cube, everyone is happy and a big check clears in your bank account for a job well done. It would be interesting to see how that "solution" holds up 1 or 2 years down the line.

If proper NoSQL design must be done "up-front" and with complete knowledge of the access patterns, then it is by definition a type of database that is only suitable for mature systems/workloads that are unlikely to drastically change in the future.

There are lots of disadvantages to a lack of schema. Optimal read/write patterns must be documented somewhere and maintained. When you have a defined schema they are up-front and obvious. By looking at the column types and the indexes you can easily know the most optimal ways to access this table.

If I use one table and index overloading, then all my records must contain within themselves some metadata about their type. For example, for `Orders` I have to write the partition key as `Order_<key>` instead of just `<key>`. No big deal, disk is cheap in the cloud right? Yes. But IO is not. Every time you are reading that record you are reading an extra 8 bytes. DynamoDB offsets that somewhat by charging you in units of 1KB but different systems may charge you based on total IO. The extraneous metadata may also cause some records to spill over from 1 write unit to 2 write units.

Operationally, things become tricky too. Let's say you discover a bug with your pricing engine and you need to update all your `Orders` to reflect the new correct price. How do you know you got all of them? How long would it take to check a 50TB mega-table that all your orders are properly formed? How complex and error-prone would that script that performs the check need to be? Renaming record types goes from being a quick DDL command to a full table scan!

The combination of schema and a declarative API are extremely powerful and the lack of these two things is really why I don't think NoSQL is here to stay.

NoSQL databases allow us to have database clusters that where the physical nodes are abstracted away. This is very useful as it allows you to not clutter up your application code with database sharding logic. These databases generally achieved this by stripping away a lot of the functionality and guarantees of a SQL database. We are now seeing these same databases trying to play catch-up on functionality, DynamoDB transactions or Cassandra Query Language are some examples. It will be tough sledding for them to catch up on the decades of development that have gone into Postgres, MySQL or SQL Server.

What if you could solve the scaling problem without losing all the functionality and guarantees of a SQL database? That is harder and evidently has taken more time than the approach that NoSQL databases took. There are currently many ongoing attempts to solve this problem, I would class them as `cloud-native databases` and `sharding middleware`.

Cloud-native databases are de-coupling the database process from the storage layer and leveraging the existing high-scale storage services that those companies have built. Examples of these are Aurora, Spanner, Cosmos. I am not as familiar with the latter two, but the Aurora project was wise to start off as a fork of MySQL and Postgres.

The sharding middleware components allow the cluster to be abstracted away. You write `SELECT foo FROM bar WHERE tenantId = 'abc'` and the query router runs that query on the node which contains that data. Examples of these are Citus (Postgres) and Vitess (MySQL).

To my knowledge there is not yet a cohesive well executed combination of these two approaches, but when it comes there would be very little reason to use NoSQL databases.
