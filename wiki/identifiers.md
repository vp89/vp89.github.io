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
- If the identifier is entirely internal and expected to remain so, then this aspect holds less importance
- For public identifiers, some obfuscation could be warranted but generally it comes at a cost
- UUIDv4 contain 122 bits of entropy + 6 bits representing the version/variant. This makes them suitable when you want full control over what information about your entity is exposed
- Depending on which database you're using, UUIDv4 can be a poor choice as your primary key. RDBMS are heavily optimized for sequential writes so using a UUIDv4 as a primary key will result in worse `INSERT` performance
