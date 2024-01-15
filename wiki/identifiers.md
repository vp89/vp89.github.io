---
title: Identifiers
layout: wiki
wiki: true
status: in progress
create_date: 2023-01-13 14:07:00 -0500
update_date: 2023-01-14 07:52:00 -0500
---

- For apps that use a single-writer RDBMS, an 8 byte monotonically increasing unsigned integer (aka bigserial) is a good default option
- Identifiers comprised of any kind of sequence number or time inherently leak information:
    - "Bigserial" can leak how many entities have been created and whether 1 particular entity was created before another
    - UUIDv7 or ULID can leak when an entity was created
    - Snowflake IDs can leak that a database might be sharded (and to some extent <i>how</i>) as well as which shard an entity is located in
    - This point has more significance if the identifier is externally facing
- UUIDv4 contain 122 bits of entropy (6 bits representing the version/variant). This scheme does not leak any information
- Depending on which database you're using, UUIDv4 can be a poor choice as your primary key. RDBMS are heavily optimized for sequential writes so using a UUIDv4 as a primary key will result in worse `INSERT` performance
- Bigserial creates an operational burden if the database is sharded. Each shard's auto increment counter needs to be manually set to avoid clashing so there is an opportunity for mistakes
    - One solution is to carve out the id generation into it's own MySQL writer, this allows you to shard the write workload while still having an easy to manage centralized id source. This pattern was documented by Flickr as ["ticket servers"](https://code.flickr.net/2010/02/08/ticket-servers-distributed-unique-primary-keys-on-the-cheap/)
    - As mentioned in the Flickr article, the Id generation can be sharded into 2 writers each generating only odd or even numbers
    - The downsides of this pattern are that writes may now require 2 round trips to a database, done sequentially. This cost could be mitigated if each application node is able to claim a batch of Ids