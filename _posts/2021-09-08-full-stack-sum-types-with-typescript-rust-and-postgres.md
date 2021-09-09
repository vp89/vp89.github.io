---
layout: post
title:  "Full-stack sum types with TypeScript, Rust and Postgres"
date:   2021-09-08 00:00:00 -0500
categories: programming
---

[Sum types](https://en.wikipedia.org/wiki/Tagged_union) allow you to model that a particular value can be typed in one of many, pre-defined ways. This provides a lot of the flexibility you get from using untyped dynamic values without giving up on compile-time safety. Using sum types to implement variants leads to clean, straight-forward code. Languages that don't have sum types force their users to effectively emulate them using verbose, complicated "patterns". In the database layer, storing variants as scalar values allows for simple access patterns that perform well at scale and should be portable to pretty much any database you might be using. The ability to model these variants in a relatively consistent way across all layers of the stack makes applications easier to debug and reason about.

At the heart of our exploration will be Rust. Rust provides first-class support for sum types, which it calls [`enums`](https://doc.rust-lang.org/book/ch06-01-defining-an-enum.html). Rust has [powerful, exhaustive pattern matching](https://doc.rust-lang.org/book/ch18-03-pattern-syntax.html) on enums that provide elegant, ergonomic solutions for avoiding the "null problem" and treating errors as values. In regular application code, the combination of sum types and pattern matching result in straight forward code that can be as flexible as you require without extraneous layers and/or un-natural hierarchical structures that you typically see with more "object-oriented" languages.

While Rust enums are great, unfortunately in the *real world* we can't just write everything in Rust. We need to save things to databases and display things in browsers! We need to figure out how to get components that let us do those things to play nice with Rust enums. For our database we will be looking at Postgres and it's support for JSON encoded values. In the frontend layer we will write some standalone TypeScript code for brevity, aiming to leverage it's [`discriminated unions`](https://basarat.gitbook.io/typescript/type-system/discriminated-unions).

### A suitable contrivance

Let's suppose we have to implement a content catalog that contains entries for different pieces of content. The catalog should be able to easily accomodate new types of content as the business expands without requiring significant refactoring. Each content entry holds metadata that varies depending on the type of content we are dealing with. We'd prefer to model the catalog as a single table, to avoid introducing the need for complex access patterns requiring many joins. In the frontend, users should see a single tabular view of all the content that allows them to filter on various aspects of a given piece of content's metadata. In order to preserve developer velocity on this application, we must ensure that content metadata is consistently structured in the various layers so that SQL reports remain accurate and frontends provide useful, comprehensive filtering.

### SQL

```
CREATE DATABASE content_catalog;

\c content_catalog

CREATE TABLE IF NOT EXISTS content_entries (
    entry_id SERIAL PRIMARY KEY,
    content_url VARCHAR(500) NOT NULL,
    content_type_id INT NOT NULL,
    content_metadata JSONB NOT NULL
);
```

entry_id|content_url|content_type_id|content_metadata
---|---|---|---
1|http://google.com|1|{"type": "Music","artist":"Jay-Z","label":"Rocafella","genre":"Rap","tracks":10}
2|http://google.com|2|{"type": "Television","director":"Mr Barbaz","producer":"Mr Blargh","seasons":8,"episodes":100}
3|http://google.com|3|{"type": "Film","director":"Steven Spielberg","producer":"Mr Foobar","duration_mins":123}

### Rust

We can model each content entry using a struct:

```
#[derive(Debug, Deserialize, Serialize)]
struct ContentEntry {
    entry_id: i32,
    content_url: String,
    content_metadata: Json<ContentMetadata>
}
```

Each content entry can be of a different type and thus have different `metadata`. This is where enums come in:

```
#[derive(Debug, Deserialize, Serialize)]
#[serde(tag = "type")]
enum ContentMetadata {
    Music { artist: String, label: String, genre: String, tracks: i32 },
    Television { director: String, producer: String, seasons: i32, episodes: i32 },
    Film { director: String, producer: String, duration_mins: i32 }
}

impl ContentMetadata {
    fn content_type_id(&self) -> i32 {
        match self {
            ContentMetadata::Music { .. } => 1,
            ContentMetadata::Television { .. } => 2,
            ContentMetadata::Film { .. } => 3
        }
    }
}
```

Of note here is the `#[serde(tag = "type")]` attribute. This tells the serialization library how we want to encode our enums using the "internally tagged" [enum representation](https://serde.rs/enum-representations.html). We are making a trade-off by choosing a more human-readable JSON representation at the expense of not being able to use tuple variants in our enum. Struct variants seem more suitable for this use case, so this feels like a net positive.

We have also implemented a function `content_type_id` on any variant of `ContentMetadata` to extract a `content_type_id` from it. This function may or may not be required but I am including it to illustrates how concisely we can implement methods that handle every case. The ability to elide any fields of each struct variant you don't need keeps these methods nice and concise.

We can get the JSON encoded content metadata out of the database and into the appropriate ContentMetadata variant using the following code:

```
let content_entry = sqlx::query_as!(
    ContentEntry,
    r#"
    SELECT entry_id, content_url, content_metadata as "content_metadata: Json<ContentMetadata>"
    FROM content_entries
    WHERE entry_id = 1;
    "#)
.fetch_one(&**pool)
.await;
```

Note that we must explicitly return the `content_metadata` as `content_metadata: Json<ContentMetadata>` so that serde can match and deserialize to the correct field.

To insert a new content entry, we don't have to do anything too interesting. Simply pass your ContentMetadata variant into `Json` and pass that in as a query parameter:

```
let content_metadata = ContentMetadata::Music {
    artist: "Jon Bon Jovi".to_string(),
    label: "EMI".to_string(),
    genre: "Rock".to_string(),
    tracks: 14
};

let result = sqlx::query!(
    r#"
    INSERT INTO content_entries (content_metadata, content_type_id, content_url)
    VALUES ( $1, $2, $3 );
    "#,
    Json(&content_metadata) as _,
    &content_entry.content_type_id(),
    "http://google.com"
)
.execute(&**pool)
.await;
```


### TypeScript

TypeScript has [discriminated unions]() which is just another word for sum types. TypeScript's story around working with sum types is not as nice as Rust's. As you'll see below, we need to employ a few tricks to get exhaustive pattern matching enforced by the compiler, but it *is* possible without requiring a ton of effort:

```
type ContentEntry = {
    entry_id: number,
    content_url: string,
    content_metadata: Music | Television | Movie
}

type Music = {
    type: ContentType.Music,
    artist: string,
    label: string,
    genre: string,
    tracks: number
}

type Television = {
    type: ContentType.Television,
    director: string,
    producer: string,
    seasons: number,
    episodes: number
}

type Movie = {
    type: ContentType.Movie,
    director: string,
    producer: string,
    duration_mins: number
}

enum ContentType {
    Music = "Music",
    Television = "Television",
    Movie = "Movie"
}
```

As you can see above we have a type for the top-level ContentEntry and `content_metadata` as a field that can have 1 of 3 variants. Each of those variants has a string literal called `type` that we have wrapped with a `ContentType` enum to make renames within this codebase safer.

Here is some dummy code to deal with these variants in an exhaustive way:

```
import axios from 'axios';

(async () => {
    const response = (await axios.get<ContentEntry>('http://localhost:4000/')).data;

    switch (response.content_metadata.type) {
        case ContentType.Music:
            console.log(`Artist: ${response.content_metadata.artist}\nLabel: ${response.content_metadata.label}\nGenre: ${response.content_metadata.genre}\nTracks: ${response.content_metadata.tracks}\n`);
            break;
        case ContentType.Television:
            console.log(`Director: ${response.content_metadata.director}\nProducer: ${response.content_metadata.producer}\nSeasons: ${response.content_metadata.seasons}\nEpisodes: ${response.content_metadata.episodes}\n`);
            break;
        case ContentType.Movie:
            console.log(`Director: ${response.content_metadata.director}\nProducer: ${response.content_metadata.producer}\nDuration Mins: ${response.content_metadata.duration_mins}\n`);
            break;
        default:
            exhaustive(response.content_metadata);
    }
})();

function exhaustive(x: never): never {
    throw new Error("Unhandled animal kind in the system");
}
```

The above code uses a TypeScript technique called [narrowing](https://www.typescriptlang.org/docs/handbook/2/narrowing.html#discriminated-unions) on the `type` string literal as well as a trick using the `never` type to provide exhaustive pattern matching. The TypeScript compiler knows all the possible string literals in `ContentEntry.content_metadata.type` so it can fall through to the default block if it sees that one isn't handled. This sets off a compiler error because this block is calling a function that should never return.

### Evolving the enum

If we want to add fields to any of our variants, we would first add them to both the Rust and TypeScript code bases as "nullable":

```
#[derive(Debug, Deserialize, Serialize)]
#[serde(tag = "type")]
enum ContentMetadata {
    Music { artist: String, label: String, genre: String, tracks: i32 },
    Television { director: String, producer: String, seasons: i32, episodes: i32 },
-   Film { director: String, producer: String, duration_mins: i32 }
+   Film { director: String, producer: String, duration_mins: i32, genre: Option<String> }
}
```

```
type Movie = {
    type: ContentType.Movie,
    director: string,
    producer: string,
-   duration_mins: number
+   duration_mins: number,
+   genre: string | null
}
```

This would allow your code to deal with previously encoded variants where this field didn't exist. You could then "backfill" this field into the existing JSON encoded values using some code like this:

```
UPDATE content_entries
SET content_metadata = jsonb_set(content_metadata, '{genre}', '"Unknown"', TRUE)
WHERE content_type_id = 3
AND NOT content_metadata ? 'genre';
```

The TypeScript and Rust variants could now safely be encoded to not support "nullable" values.

### Drawbacks

One drawback of this technique is that we are encoding data in a database in the way that Rust and the `serde` crate choose to represent it. If we had other applications trying to consume this same data directly, their language may not be able to easily work with data structured in this way. The advent of *microservices* has brought about improved practices around avoiding this, so this should be less of a concern.

Another, similar, drawback is that the database itself does not provide any type checking on our JSON-encoded variants. We should therefore only permit writes of these values by the application itself or by carefully tested migration scripts. If it was a requirement that human-users should be able to insert or modify existing values, those write operations could be made available through stored procedures that perform validation on whether the supplied JSON value is properly formed as one of the variants.

### Source code

Full source code for this exploration is available on [GitHub](https://github.com/vp89/rust-ts-sum-types)
