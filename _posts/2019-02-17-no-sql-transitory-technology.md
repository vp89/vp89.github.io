---
layout: post
title:  "NoSQL, a transitory technology"
date:   2019-02-19 00:00:00 -0000
categories: programming
---

I recently watched this talk on [Advanced DynamoDB Design Patterns](https://www.youtube.com/watch?v=HaEPXoXVf2k) in preparation for some upcoming work. I ended up watching it multiple times and sharing it with colleagues because I felt it gave me, a relative outsider to the NoSQL world, a good feel for what it's all about and why people have been using it.

The presenter states that we have yet to hit the "slope of enlightenment" with regards to NoSQL adoption because most people who are currently using it are not using it properly.

<img src="/dynamo-hype-curve.png" />

The presenter then goes on to explain what it means to use it properly. The gist of it is that NoSQL is quite unforgiving and you need to be very clever up front in the way you model your data to fit all your access patterns. One of the recommendations made is to put all your data in the same key-space and to use clever combinations of composite keys in your partition and sort keys to get efficient look-ups.

This recommendation is reiterated in this AWS documentation about [NoSQL design best practices](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/bp-general-nosql-design.html#bp-general-nosql-design-concepts).

I can imagine it must be a lot of fun to be a NoSQL consultant. You go in to a company, work with them to enumerate all their access patterns and then design an extremely clever one-table contraption that performs well and is capable of holding practically infinite amounts of data. Everyone is happy, you look like a genius and a big check clears in your bank account for a job well done. It would be interesting to see how that "solution" holds up 1 or 2 years down the line.

If proper NoSQL design is done "up-front" and with complete knowledge of the access patterns, then it is by definition a type of database that is only suitable for mature systems/workloads that are unlikely to drastically change in the future.

There are lots of disadvantages to a lack of schema. Optimal read/write patterns must be documented somewhere and maintained. When you have a defined schema they are up-front and obvious. By looking at the column types and the indexes you can easily know the most optimal ways to access this table.

If I use one table and index overloading, then all my records must contain within themselves some metadata about their type. For example, for `Orders` I have to write the partition key as `Order_<key>` instead of just `<key>`. No big deal, disk is cheap in the cloud right? Yes. But IO is not. Every time you are reading that record you are reading an extra 8 bytes. Dynamo offsets that somewhat by charging you in units of 1KB but different systems may charge you based on total IO. There would also be some records written that may spill over from 1 write unit to 2 write units, for example, because of this extraneous metadata.

Operational work on your data becomes a nightmare. Let's say you discover a bug with your pricing engine and you need to update all your `Orders` to reflect the new correct price. How do you know you got all of them? Imagine if your `Orders` table was >1TB, how would you know if all the `Orders` were properly formed? What if your `Orders` are 1TB of a 50TB table. How long would it take to check that your orders are properly formed and how complex and error-prone would that script be?

What if you make a mistake in your naming and want to rename your `Orders` to `Purchases`? Instead of a simple one-field update in a metadata table, you now have to scan through the entire table to identify your `Orders` and re-write each of them for the new naming convention.

The combination of schema and a declarative API are extremely powerful and the lack of these two things is really why I don't think NoSQL is here to stay.

The problem that NoSQL solved is the ability to have your database span across multiple physical machines but behave logically as 1 database. This allowed developers to remove the complicated sharding logic they were managing in their apps. The way that NoSQL databases solved the scaling problem is by drastically stripping away a lot of the functionality and guarantees of a SQL database. We are now seeing NoSQL systems trying to play catch-up on functionality that was initially missing (DynamoDB transactions or Cassandra Query Language, as examples) but it will be tough sledding for them to catch up on the decades of development that relational databases have. Why do you think Amazon forked MySQL and Postgres when they started Aurora? There is decades of man-hours and practical `magic` in those projects and it would be foolish as an industry to turn our backs on that and start over.

What if you could solve the scaling problem without losing all the functionality and guarantees of a SQL database? That is harder and evidently has taken more time than the approach that NoSQL took, but it looks like we are beginning to get some solid, mature solutions such as Amazon Aurora, Citus and Vitess. The cloud-native databases are solving the problem by de-coupling the database process from the storage layer and leveraging existing storage services they have built. The sharding middleware components allow developers to treat their database cluster as 1 logical component. The one to many logical/physical concerns are pushed down to the database level which makes application code easier to reason about.

A cohesive well executed combination of these two approaches will be the one that wins out in the end.
