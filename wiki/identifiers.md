---
title: Identifiers
layout: wiki
wiki: true
status: in progress
create_date: 2023-01-14 14:07:00 -0500
update_date: 2023-01-14 14:07:00 -0500
---

- For apps that use a single-writer RDBMS, an 8 byte monotonically increasing unsigned integer (aka bigserial) is a good default option
- Identifiers not entirely comprised of entropy inherently leak information:
    - "Bigserial" can leak how many entities have been created and whether 1 particular entity was created before another
    - UUIDv7 or ULID can leak when an entity was created
    - Snowflake IDs can leak that a database might be sharded (and to some extent <i>how</i>) as well as which shard an entity is located in
- If the identifier is entirely internal and expected to remain so, then this aspect holds less importance
- For public identifiers, some obfuscation could be warranted but generally it comes at a cost
- UUIDv4s are 128 bits of entropy. A UUIDv4 doesn't leak any information and so it can be a suitable "external" identifier
- Depending on which database you're using, UUIDv4 can be a poor choice. RDBMS are heavily optimized for sequential writes so using a UUIDv4 as a primary key will result in worse `INSERT` performance