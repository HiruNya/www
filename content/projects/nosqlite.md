+++
title = "NoSQLite"
description = "A library that lets you use SQLite as a NoSQL JSON document store."
weight = 0

[extra]
git = "https://github.com/HiruNya/nosqlite"
+++

NoSQLite is a library that allows you to benefit from the stable and performant SQLite database
but with a NoSQL-like JSON-based API which uses SQLite's `json1` extension.

## Features
- Insert JSON objects.
- Get and remove JSON objects by primary key.
- Iterate through multiple entries in a table.
- Filter and Sort the entries by using the fields in a JSON object
or columns in the SQL table.
- Get *only* the primary key, JSON object,
or specific field(s) from the JSON object.
- Set, insert, replace, and remove fields in a JSON object.
- 'Patch' JSON objects with other JSON objects.

## Example
```rust
#[derive(Serialize)]
struct User {
	name: String,
	age: u8,
}

fn main() {
	let connection = Connection::in_memory().unwrap();
	let table = connection.table("people").unwrap();
	table.insert(&User{ name: "Hiruna".into(), age: 18 }, &connection).unwrap();
	table.insert(&User{ name: "Bob".into(),  age: 13 }, &connection).unwrap();
	table.insert(&User{ name: "Callum".into(), age: 25 }, &connection).unwrap();
	table.insert(&User{ name: "Alex".into(), age: 20 }, &connection).unwrap();
	// Iterate over the entries in the table
	table.iter()
		// Sort by Age
		.sort(field("age").ascending())
		// Only get people who are 18+
		.filter(field("age").gte(18))
		// Gets the name and age fields of the JSON object
		.fields::<(String, u8), _, _, _>(&["name", "age"], &connection)
		.unwrap()
		.into_iter()
		.for_each(|(name, age)| println!("{:10} : {}", name, age));
}
```
