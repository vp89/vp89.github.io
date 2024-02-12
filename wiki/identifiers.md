---
title: Identifiers
layout: wiki
wiki: true
status: in progress
create_date: 2024-01-13 14:07:00 -0500
update_date: 2024-01-27 08:03:00 -0500
---

- For apps that use a single-writer RDBMS, an 8 byte monotonically increasing unsigned integer (aka bigserial) is a good default option
- Identifiers can leak information, this can have varying implications depending on whether the identifier is externally visible, which scheme is used and what entity it's attached to:
    - "Bigserial" can leak how many entities have been created and whether 1 particular entity was created before another
    - [UUIDv7](https://datatracker.ietf.org/doc/draft-ietf-uuidrev-rfc4122bis/) or [ULID](https://github.com/ulid/spec) can leak when an entity was created
    - [Snowflake IDs](https://en.wikipedia.org/wiki/Snowflake_ID) contain a "machine id" to facilitate sharding. This leaks that the database is sharded, how many machines exist and whether 2 entities are colocated together
- [UUIDv4s](https://datatracker.ietf.org/doc/html/rfc4122#section-4.4) contain 122 bits of entropy (6 bits representing the version/variant). This scheme does not leak any information
- If using an RDBMS, UUIDv4 will result in much worse `INSERT` performance once the table no longer fits in memory. This is because the part of the index you are modifying is less likely to already be in memory, resulting in an additional read IOP on every insert.
    - Benchmarks on [MySQL](https://www.percona.com/blog/uuids-are-popular-but-bad-for-performance-lets-discuss/) and [Postgres](https://www.2ndquadrant.com/en/blog/sequential-uuid-generators/) show a ~3-6x performance hit
- Bigserial creates an operational burden if the database is sharded. Each shard's auto increment counter needs to be manually set to avoid clashing so there is an opportunity for mistakes. If shard count is relatively low and shard merging is not needed this solution is likely manageable

- The ["ticket server"](https://code.flickr.net/2010/02/08/ticket-servers-distributed-unique-primary-keys-on-the-cheap/) pattern allows you to shard your writes while still having a simple, centralized source of ids:
    - The main downside of this pattern is that `INSERT`s require 2 round trips to a database, done sequentially. This cost could be amortized over many `INSERT`s by lazily claiming a batch of Ids from the ticket server instead of just 1
    - <img src="/images/ticket-server.drawio.png" />

- If using anything but bigserial or UUIDv4, you will need to build up a toolchain around it to make it easy to work with. ULID is more mature than UUIDv7 at the time of writing but this could change
    - Server-side generator, encode from human-readable string, decode to human-readable string, generate from timestamp (for time-based range query)

- A nice benefit of schemes which prefix the id with a timestamp is that an additional secondary index on a `created_at` column is not needed to enable querying or pruning of data based on when it was created

    ```
    SET @cutoff = ULID_FROM_DATETIME('2024-01-01');
    DELETE FROM table WHERE id < @cutoff;
    ```

- Time-sortable 64 bit id is highly efficient but difficult to provide global uniqueness without some kind of "shard id":
    - 44 bits for timestamp
        - 10 bits for milliseconds
        - 34 bits for seconds from an epoch = 544 years
    - 10 bits for shard id (supports up to 1024 physical shards)
    - 10 bits for internal sequence number (supports up to 1024 ids per millisecond per shard)
    - In MySQL, the primary key is stored at the end of each secondary index record and may also be stored in other tables as foreign keys. PKs for "hot tables" tend to experience such amplification so minimizing their size while retaining sortability and global uniqueness is very compelling