---
title: Identifiers
layout: wiki
wiki: true
status: in progress
create_date: 2023-01-13 14:07:00 -0500
update_date: 2023-01-16 08:03:00 -0500
---

- For apps that use a single-writer RDBMS, an 8 byte monotonically increasing unsigned integer (aka bigserial) is a good default option
- Identifiers can leak information, this can have varying implications depending on whether the identifier is externally visible, which scheme is used and what entity it's attached to:
    - "Bigserial" can leak how many entities have been created and whether 1 particular entity was created before another
    - [UUIDv7](https://www.ietf.org/archive/id/draft-peabody-dispatch-new-uuid-format-01.html) or [ULID](https://github.com/ulid/spec) can leak when an entity was created
    - [Snowflake IDs](https://en.wikipedia.org/wiki/Snowflake_ID) contain a "machine id" to facilitate sharding. This leaks that the database is sharded, how many machines exist and whether 2 entities are colocated together
- [UUIDv4s](https://datatracker.ietf.org/doc/html/rfc4122#section-4.4) contain 122 bits of entropy (6 bits representing the version/variant). This scheme does not leak any information
- If using an RDBMS, UUIDv4 will result in much worse `INSERT` performance once the table no longer fits in memory. This is because the part of the index you are modifying is less likely to already be in memory, resulting in an additional read IOP on every insert.
    - Benchmarks on [MySQL](https://www.percona.com/blog/uuids-are-popular-but-bad-for-performance-lets-discuss/) and [Postgres](https://www.2ndquadrant.com/en/blog/sequential-uuid-generators/) show a ~3-6x performance hit
- Bigserial creates an operational burden if the database is sharded. Each shard's auto increment counter needs to be manually set to avoid clashing so there is an opportunity for mistakes
    - One solution is to run a ["ticket server"](https://code.flickr.net/2010/02/08/ticket-servers-distributed-unique-primary-keys-on-the-cheap/) which is a dedicated centralized database server used solely to generate ids. This allows you to shard the write workload while still having an easy to manage centralized id source.
    - The ticket server could be easily sharded into 2 writers each generating only odd or even numbers.
    - The downsides of this pattern are that `INSERT`s require 2 round trips to a database, done sequentially. This cost could be amortized over many `INSERT`s by lazily claiming a batch of Ids from the ticket server instead of just 1
    - <img src="/images/ticket-server.drawio.png" />