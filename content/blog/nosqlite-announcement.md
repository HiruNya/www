+++
title = "NoSqlite: A NoSQL layer over SQLite!"
date = 2020-07-06

[taxonomies]
tags = ["Announcement", "Rust"]
+++

[Link to repository.](https://github.com/HiruNya/nosqlite)

## Preface

As I was making [Nyanko], I needed a way of storing the user's data
more efficiently than a JSON file as I would have to serialise a large JSON file every time I made a change to the data.
In this situation, most developers reach out to [sqlite]
because you can store a whole database in a single file and read and write to it very efficiently.
I also intend on porting [Nyanko] to Android and [sqlite] is a very common data storage format for Android apps.

There's just one problem - I prefer nosql databases.
This is probably due to my naivety, but I prefer nosql databases due to them not being restricted by a schema
and thus not requiring migrations which I feel would be harder in [Nyanko] due to the database/file being stored on the user's computer.

There are key-value storage systems that store everything into one file and tick many of my boxes (like [sled])
however I also desire the ability to query the entries in a table like you can with [AWS DynamoDB] for example.

I found [hotpot-db] which was a great idea!
We could just use [sqlite] as a nosql database by having it store JSON objects
and using [sqlite]'s [`json1`] extension.
Unfortunately [hotpot-db] didn't seem to allow more complex queries
and didn't have a lot of the features I would have like to use like sorting and limiting the number of entries.

So I decided to make my own crate that made use of [rusqlite] and [sqlite]'s [`json1`] extension.
I call it [nosqlite]!

## Features

- [JSON Document Store + Serde](#json)
- [Querying](#querying)
- [Updating Values](#update)
- [SQL Table Interop](#sql)
- [Indexes](#index)

### JSON Document Store + Serde {#json}

Because the crate works by essentially writing the data into a JSON string,
we can use the powerful [serde] framwork.

```rust
#[derive(Deserialize, Serialize)]
struct User {
	name: String,
	age: u8,
}

// Inserts a Json object into the table.
table.insert(User { name: "Hiruna".into(), age: 19 }, &connection)?;
table.insert(User { name: "Bob".into(), age: 13 }, &connection)?;

// Gets the first Json object in the table. (Not the one we just inserted unless the table was empty).
let data: Entry<i64, User> = table.get(1).entry(&connection)??;
```

But you aren't restricted to concrete types - you can put in any object you want with [serde_json]'s `json!` macro.

```rust
table.insert(json!({ first_name: "Hiruna", last_name: "Jayamanne" }), &connection)?;
```

You also don't have to get everything, you can decide if you want just the json object, id, or a field.

```rust
// Get botht the id and the JSON object
let entry: Entry<i64, User> = table.get(1).entry(&connection)??;

// Get only the JSON object
let data: User = table.get(1).data(&connection)??;

// Get only the id
let id: i64 = table.get(1).id(&connection)??;

// Get a specific field in the JSON object
let field: String = table.get(1).field("name", &connection)??;
```

You can also get the value of fields that have been nested within another object.

```rust
table.insert(json!({ "name": "Alex", "nested": { "x": 3, "y": 4 } }), &connection)?;

// Gets the "x" field of the "nested" object
table.iter().field("nested.x", connection)?
```

Arrays are also not a problem.

```rust
table.insert(json!({ "name": "Alex", "array": [1, 2, 3, 4] }), &connection);

// Gets the 2nd item out of the array
table.iter().field("array[1]", connection)?
```

### Querying {#querying}

Writing queries was one of the main things I desired.
You can filter by the values of an object, limit the number of results, etc.

```rust
table.insert(&User{ name: "Hiruna".into(), age: 18 }, &connection)?;
table.insert(&User{ name: "Bob".into(),  age: 13 }, &connection)?;
table.insert(&User{ name: "Callum".into(), age: 25 }, &connection)?;
table.insert(&User{ name: "Alex".into(), age: 20 }, &connection)?;
// Iterate over the entries in the table
table.iter()
    // Sort by Age
    .sort(field("age").ascending())
    // Only get people who are 18+
    .filter(field("age").gte(18))
    // Gets the name and age fields of the JSON object
    .fields::<(String, u8), _, _, _>(&["name", "age"], &connection)?
    .into_iter()
    .for_each(|(name, age)| println!("{:10} : {}", name, age));
```

I tried it to make as idiomatic as possible and so I named the method `iter`
however it should be noted that it returns a builder struct *not* an actual iterator.

We can also have some more complex filters.

```rust
// Get users whose age is greater than equal to 18 and less than 50
// Or if their name is "Bob"
field("age").gte(18).and(field("age").lt(50)).or(field("name").eq("Bob"))
```

### Updating Values {#update}

We also might want to update a certain field of an entry that has already been inserted.
In that case there are three possible ways of updating an object: set, insert, and patch.

**Set** only updates an object's field if that field already exists in the object.

```rust
// Has an `age` field
table.insert(json!({ "first_name": "Hiruna", "age": 19 }), &connection)?;
// Does not have an `age` field
table.insert(json!({ "first_name": "Bob" }), &connection)?;

// Only the first entry's "age" will be set to 13
table.iter().set("age", 13, &connection);
```

**Insert** only updates an object's field if that field does not exist in the object.

```rust
// Has an `age` field
table.insert(json!({ "first_name": "Hiruna", "age": 19 }), &connection)?;
// Does not have an `age` field
table.insert(json!({ "first_name": "Bob" }), &connection)?;

// The second entry will get a new "age" field set to 13
table.iter().insert("age", 13, &connection);
```

**Patch** updates a value no matter what and allows you to update multiple fields at once by taking in a JSON object.

```rust
table.insert(json!({ "first_name": "Hiruna", "age": 19 }), &connection)?;
table.insert(json!({ "first_name": "Bob" }), &connection)?;

// Both entries will have "age" set to 13 and "score" set to 5
table.iter().patch(json!({ "age": 13, "score": 5 }), &connection);
```

### SQL Table Interop {#sql}

Sometimes you're in a situation where you already have a sql table that you don't want to mess with
or maybe you are sure that a certain value should be it's own sql column instead of a field in a JSON object
(e.g. because it's more efficient).

A [nosqlite] `Connection` implements `AsRef<SqliteConnection>`
so you can still execute normal SQL statements.

```rust
connection.as_ref().execute(r#"
    CREATE TABLE custom(
        id INTEGER PRIMARY KEY,
        data TEXT NOT NULL,
        weight INTEGER
    )
"#, NO_PARAMS)?;
```

You can then turn this sql table into a [nosqlite] table.

```rust
let table = EntryTable::<i64>::unchecked("custom", "id", "data");
```

Unfortunately, you don't get the nice insertion API so you'll have to use SQL statements.
The only benefit that this crate can provide is the `Json` newtype struct which wraps a type that implements
[serde]'s `Serialize` trait and implements `ToSql` on it.

```rust
let mut statement = connection.as_ref().prepare("INSERT INTO custom (data, weight) VALUES (?, ?)")?;

statement.execute(&[&Json(User{ name: "Hiruna".into(), age: 18 }) as &dyn ToSql, &100 as &dyn ToSql])?;
statement.execute(&[&Json(User{ name: "Bob".into(),  age: 13 }) as &dyn ToSql, &50 as &dyn ToSql])?;
statement.execute(&[&Json(User{ name: "Callum".into(), age: 25 }) as &dyn ToSql, &25 as &dyn ToSql])?;
statement.execute(&[&Json(User{ name: "Alex".into(), age: 20 }) as &dyn ToSql, &75 as &dyn ToSql])?;
```

However now you can use those extra sql columns in your filter.

```rust
table.iter()
    // Only get people who are 18+ and their weighting is over 50
    .filter(field("age").gte(18).and(column("weight").gt(50)))
    .field::<String, _>("name", &connection)?;
```

### Indexes {#index}

This crate however is not as efficient as [sqlite] is
because [sqlite] was never meant to store JSON objects in the first place.
What I believe happens is that in a query, the [`json1`] extension parses every string column into a JSON extension.
As you can imagine, this is extremely inefficient.
I'm not really interested in high performance due to [nyanko] not needing it
but this could prove to be an issue if I have a lot of entries in my table.
Thankfully, [sqlite] allows [indexes on expressions](https://sqlite.org/expridx.html)
which should allow us to somewhat cache the results of the parsing function.
This is very useful in a situation where you know what fields will be queried most often.

```rust
table.index("my_index", &[field("name"), field("age")], connection)?;
```

Unfortunately whether or not the index is used is unclear and decided by [sqlite] at runtime.
I would recommend running some kind of benchmark to figure out if the index actually does anything.
I would have preferred something like [AWS DynamoDB] queries where you can *force* the use of an index
but it does not seem to possible with [sqlite]
(there is an [`INDEXED BY`](https://sqlite.org/lang_indexedby.html) clause that I could possibly use
but it says that I should only be using it for regression testing so I'll avoid it for now).

## Downsides

I've mentioned it before but speed is a very big downside of using this library.
I haven't done any proper benchmarks, but I can imagine that it does quite a lot worse than
normal [sqlite] and worse than a possible future implementation that purely focuses on being a nosql database.
Again [indexes](#index) do help but this crate is definitely not for people who want high performance
because I can't promise it.

Another problem is that since this has no schema, it is very difficult to catch errors at compile time
like you can with ORMs like [diesel](https://github.com/diesel-rs/diesel) and [sqlx](https://github.com/launchbadge/sqlx).
For example, if you expect a field "A" to have a `String` in it but it actually has a number or the field doesn't exist at all,
then [nosqlite] will silently ignore that entry.
This is useful if you, for example, only want entries that contain a certain field.
But is painful if you get a bug where an entry is not being returned
which would require you to track down the offending piece of code.

## Conclusion

While I don't see it being used for anything big,
I wanted to share it for anyone else who wants an easy-to-use storage system for their application.
I have not yet uploaded this crate to [crates.io](https://crates.io) (and I'm not sure if I should!)
so for now you would have to point to the github link in your `Cargo.toml` file.

Thanks:
- [rusqlite] for the great safe sqlite bindings.
- [sqlite] for the great storage system and for making a `json1` extension.
- [hotpot-db] for the inspiration.
- You, for reading my first blog post!

[nosqlite]: https://github.com/HiruNya/nosqlite
[Nyanko]: https://github.com/HiruNya/nyanko
[sqlite]: https://www.sqlite.org/
[sled]: https://github.com/spacejam/sled
[AWS DynamoDB]: https://aws.amazon.com/dynamodb/
[hotpot-db]: https://github.com/drbh/hotpot-db
[`json1`]: https://www.sqlite.org/json1.html
[rusqlite]: https://github.com/rusqlite/rusqlite
[serde]: https://github.com/serde-rs/serde
[serde_json]: https://github.com/serde-rs/json
