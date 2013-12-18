# upper.io/db

[Upper DB][1] is a [Go][2] package for saving and retrieving [Go][2] structs to
and from permanent storage with ease.

[Upper DB][1] can be used as a tool to perform the most common operations on
SQL and NoSQL databases such as appending, searching, updating and removing
items.

## Why upper.io/db?

* Provides a common interface to communicate with a variety of databases.  Does
* not try to interfere with model definitions or your way of programming, it
just stays out of the way.

## Getting started

First, download the `upper.io/db` package:

```
go get upper.io/db
```

Then install a database adapter. Available adapters are listed below.

* [SQLite](./sqlite/)
* [MySQL](./mysql/)
* [PostgreSQL](./postgresql)
* [MongoDB](./mongodb)

Import both the `upper.io/db` and the adapter packages

```go
import (
  "upper.io/db"
  _ "upper.io/db/sqlite"
)
```

Define your database settings using the `db.Settings` struct.

```go
var settings = db.Settings{
  Database: `example.db`,
}
```

Connect to the database using the adapter (`sqlite` in this example).

```go
sess, err = db.Open("sqlite", settings)
```

If no error happened, then you're ready to make some queries.

## Working with collections

In order to use and query collections you'll need a collection reference, use
the `db.Database.Collection()` method on the previously defined `sess`
variable:

```go
// Pass the table/collection name to get a collection reference.
col, err = sess.Collection("birthdays")
```

Using this collection reference, you can start adding saving rows to the
collection.

```go
// You can save a map.
item = map[string]interface{}{
  "first_name": "Hayao",
  "last_name": "Miyazaki",
  "date": time.Now(),
}
col.Append(item)

// And you can save a struct.
item = Birthday{
  "Hayao",
  "Miyazaki",
  time.Now(),
}
col.Append(item)
```

You can use the `db.Collection.Find()` method to search for the recently
appended item. This method creates a result set, this result set can then be
iterated or dumped to a pointer or a pointer to slice.

```go
res = col.Find(db.Cond{"last_name": "Miyazaki"})
```

Depending on the number of results you're expecting, resultsets can be fetched
as values, slices or maps.

```go

// Fetching the first element of the resultset.
var birthday Birthday
err = res.One(&birthday)

// Fetching the first element of the resultset.
var birthdays []Birthday
err = res.All(&birthdays)
```

[1]: http://upper.io
[2]: http://golang.org
