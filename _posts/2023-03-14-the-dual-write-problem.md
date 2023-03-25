---
layout: post
title:  "The minimalist's guide to the dual write problem"
date:   2023-03-20 00:00:00 -0500
categories: programming
---

In microservice/event-driven architectures, services typically write to their own database as well as publish domain events to a message broker for interested parties to consume. This need to perform dual writes against two or more distinct systems exposes the broader system to potential data inconsistency.

<!-- insert diagram here -->

In some contexts, it could be reasonable to *do nothing* or make a low-effort attempt to reduce but not eliminate the probability of this bad outcome. Disparate systems can often be reconciled ex-post through some time consuming manual remediation work. Normalcy bias is real and the high availability of managed cloud infrastructure can make it hard to justify going the extra mile to properly handle rare failure modes.

Due to it's pervasiveness, many solutions have been devised to solve this problem. The transactional outbox pattern involves writing domain events to an "outbox" table. The table is polled like a queue by the publishing service, using `UPDATE FOR SKIP LOCKED` to acquire a lease on messages that need to be published. Once published, the message can be deleted. The benefits of this approach are that no additional components are required and all the code is self-contained within the publishing service. The downside is that this pattern results in at least 3 additional writes for each given domain event.

<!-- insert diagram here -->

If DynamoDB is your transactional database of choice then DynamoDB Streams provides a good solution to this problem that avoids a lot of the more common drawbacks. Write amplification can be kept to a minimum because domain events can sometimes be derived from item-level change events. When an outbox table *is* needed, it can be easily managed over time using the TTL feature and Dynamo does have a transaction API now. The stream could be consumed from the publishing service itself or from a Lambda deployable built from the same code base, mitigating some of the change management and testability challenges mentioned earlier.

<!-- insert diagram here -->

For those not using DynamoDB, tools like Debezium or AWS DMS are available. These tools consume the database's transaction log and produce events to a separate log after applying some configurable transforms. Similarly to the DynamoDB streams solution, the outbox table can sometimes be obviated by deriving domain events from row-level DML change events. The cost and complexity increase from deploying these additional components can be prohibitive however. They may also require you to encode some of this translation logic outside of the publishing service's code base which can introduce some change management and testability challenges.

<!-- insert diagram here -->

Fortunately, simpler solutions do exist for users of relational databases. Before illustrating them in detail, let's step back and think about the problem from first principles. We are trying to solve for dual writes across a transactional database and a message broker, but transactional databases themselves perform dual writes. For any given write, it's first applied to the log and later flushed to the data file which holds the database "state" at any given point in time. The dual write problem is fundamentally a *database* problem, therefore it follows that the database would already provide a good solution for it. By tailing the transaction log directly from the publishing service we can solve the problem in a simple, efficient way. The solution is entirely self-contained but more efficient than polling the outbox table.

<!-- insert diagram here -->

 <!-- explain postgres details -->
 
 Postgres provides a flavor of logical replication which allows a publishing service to directly tail the transaction log and receive JSON events that represent row-level DML changes.

<!-- explain mysql details -->
