# upper.io/db

`upper.io/db` is a [Go][2] package with the ability to communicate with many
different kinds of databases.

`upper.io/db` is able to perform the most common operations on SQL and NoSQL
databases such as creating, searching, updating and removing items through the
use of special subpackages, known as adapters.

This package is not a magical fully-featured ORM, it simply allows the
developer to tell databases what to do and then it tries to stay out of the
way.

## Requisites

The `upper.io/db` package depends on:

The [Go compiler and tools][4], of course, version 1.3 is highly recommended.

The [git][3] version control system. You can install `git` on a Debian-based
system like this:

```sh
sudo apt-get install git -y
```

### Getting the package

You can download and install the `upper.io/db` package using `go get`:

```sh
go get upper.io/db
```

*Note:* If the `go get` program fails with something like:

```sh
package upper.io/db: exec: "git": executable file not found in $PATH
```

it means that a required program is missing, `git` in this case. To overcome
this error install the missing application and try again.

## Database adapters

Installing the main package is not enough, in order to communicate with a
database you'll also need a *database adapter*.

Here's a list of available database adapters. Look into the adapter's link to
see installation instructions specific to each adapter.

* [SQLite 3](./sqlite/)
* [MySQL](./mysql/)
* [PostgreSQL](./postgresql)
* [MongoDB](./mongo)

## Learn by example

Through this example, we'll be featuring the SQLite3 database and the `sqlite`
adapter.

Fire up a terminal and check if the `sqlite3` command is installed. If the
program is missing install it like this:

```go
# Installing sqlite3 in Debian
sudo apt-get install sqlite3 -y
```

Then, run the sqlite3 command and create a `test.db` database:

```sh
sqlite3 test.db
```

The sqlite3 program will welcome you with a prompt, like this:

```
sqlite>
```

From within the sqlite3 prompt, create a demo table with the following code:

```sql
CREATE TABLE demo (
  first_name VARCHAR(80),
  last_name VARCHAR(80),
  bio TEXT
);
```

After creating the table, type `.exit` to end the sqlite3 session.

```
sqlite> .exit
```

Create a `main.go` file and use the following code to import both the
`upper.io/db` and the recently installed adapter:

```go
# main.go
package main

import (
  "upper.io/db"
  _ "upper.io/db/sqlite"
)
```

Then, configure the database credentials using the `db.Settings` struct. We are
using sqlite3 so the username, password and hostname are not required.

```go
# main.go
var settings = db.Settings{
  Database: `example.db`,
}
```

Create a `main()` entry and put the instructions below to get a `db.Database`
reference:

```go
sess, err = db.Open("sqlite", settings)
```

You can use the `sess` variable to get a `db.Collection` reference.

## Working with collections

A collection is a group of items, in the SQL world, collections are known as
tables. `upper.io/db` considers both SQL tables and NoSQL collections as just
**collections**.

In order to use and query collections you'll need a collection reference, use
the `db.Database.Collection()` method on the previously defined `sess`
variable to get one:

```go
// Pass the table/collection name to get a collection reference.
col, err = sess.Collection("demo")
```

use the collection reference to insert a new row in the database:

```go
item = map[string]interface{}{
  "first_name": "Hayao",
  "last_name": "Miyazaki",
  "bio": "Japanese film director.",
}
col.Append(item)
```

Using structs to define the format of collection items is not strictly required
(we could use maps too), but it is recommended. If you'd like to create a
struct datatype for the demo table we've created before, then you should define
each column as a field of the struct:

```go
type Demo struct {
  FirstName string `field:"first_name"`
  LastName string `field:"last_name"`
  Bio string `field:"bio"`
}
```

And instead of using a map, we can insert a `Demo{}` value with the
`db.Collection.Append()` call.

```go
item = Demo{
  "Hayao",
  "Miyazaki",
  "Japanese film director.",
}
col.Append(item)
```

If you have followed the example until here, then you should have a snippet
that looks like this:

```go
# main.go

package main

import (
  "upper.io/db"
  _ "upper.io/db/sqlite"
)

var settings = db.Settings{
  Database: "test.db",
}

type Demo struct {
  FirstName string `field:"first_name"`
  LastName string `field:"last_name"`
  Bio string `field:"bio"`
}

func main() {
  sess, err := db.Open("sqlite", settings)

  if err != nil {
    panic(err.Error())
  }

  col, err := sess.Collection("demo")

  col.Append(Demo{
    "Hayao",
    "Miyazaki",
    "Japanese film director.",
  })

  if err != nil {
    panic(err.Error())
  }
}
```

compile and run it using `go run main.go`.

You can use the `db.Collection.Find()` method to search for the recently
appended item. This method creates a result set, this result set can then be
iterated or dumped to a pointer or a pointer to slice.

```go
res = col.Find(db.Cond{"last_name": "Miyazaki"})
```

Once you have a result set (`res` in this example), you can choose to fetch
results into a slice, providing a pointer to a slice of structs or maps, as in
the following example.

```go
var birthdays []Birthday
err = res.All(&birthdays)
```

Filling a slice could be expensive if you're working with a lot of rows, if you
need to optimize memory usage for big result sets looping over the result set
could be better, use `db.Result.Next()`.

```go
var birthday Birhday
for {
  err = res.Next(&birthday)
  if err == nil {
    // No error happened.
  } else if err == db.ErrNoMoreRows {
    // Natural end of the result set.
    break;
  } else {
    // Another kind of error, should be taken care of.
    return res
  }
}
```

If you need only one element of the result set, the `db.Result.One()` method
could be better suited for the task.

```go
var birthday Birhday
err = res.One(&birthday)
```

## More on result sets

Once you have a basic understanding of result sets, you can start using
conditions and limits to reduce the amount of rows.

The `db.Cond{}` type, can be used to provide conditions, use it as an argument
for `db.Collection.Find()`.

```go
res = col.Find(db.Cond{"user_id": 1})
```

If you want to use more than one condition, just provide more keys to the
`db.Cond{}` map:

```go
res = col.Find(db.Cond{
  "user_id": 1,
  "email": "user@example.org",
})
```

the provided conditions will be grouped under an AND conjunction.

If you want to use an OR instead, the `db.Or{}` type is also available:

```go
res = col.Find(db.Or{
  db.Cond{
    "email": "user@example.org",
  },
  db.Cond{
    "email": "user@example.com",
  }
})
```

Complex AND filters should be delimited by the `db.And{}` type

```go
res = col.Find(db.And{
  db.Or{
    db.Cond{
      "name": "Jhon",
    },
    db.Cond{
      "name": "John",
    },
  },
  db.Or{
    db.Cond{
      "name": "Smith",
    },
    db.Cond{
      "name": "Smiht",
    },
  },
})
```

The resulting result set can be delimited using `db.Result.Limit()` and
`db.Result.Skip()` or sorted by value, using the `db.Result.Sort()` function.

This example skips ten rows, counts up to eight rows and sorts the results by
name (descendent).

```go
res = col.Find().Skip(10).Limit(8).Sort("-name")
```

If you want to know how many rows the query is returning, use the
`db.Result.Count()` call.

```go
c, err := res.Count()
```

## More operations with result sets

Result sets are not only capable of returning rows, they can also be used to
update or delete all the rows within the given conditions.

If you want to update a whole set of rows, first limit the set.

```go
res = col.Find(db.Cond{"name": "Old name"})

err = res.Update(map[string]interface{}{
  "name": "New name",
})
```

If you want to delete a set of rows, use the `db.Result.Remove()` call on a
result set.

```go
res = col.Find(db.Cond{"active": 0})

res.Remove()
```

## Closing results after using

Make sure to use `db.Result.Close()` when finishing using the result set.

```go
res.Close()
```

## Working with databases

There are many more things you can do with a `db.Database` reference besides
getting a collection.

You can get a list of all collections within the database:

```go
all, err = sess.Collections()
for name := range all {
  fmt.Printf("Got collection %s.\n", name)
}
```

If you ever need to connect to another database, you can use the
`db.Database.Use` method

```go
err = sess.Use("another_database")
```

## Transactions

If the database server you're using supports transactions, you can use the
`db.Database.Begin()` and `db.Database.End()` methods to delimit transactions:

```go
sess.Begin()
col.Append(item)
...
col.Append(item)
sess.End()
```

## Working with the underlying driver

Some situations will require you to use methods that are specific to the
underlying driver, for example, if you're in the need of using the
[mgo.Session.Ping](http://godoc.org/labix.org/v2/mgo#Session.Ping) method, you
can retrieve the underlying `*mgo.Session` and use it to ping.

```go
drv = sess.Driver().(*mgo.Session)
err = drv.Ping()
```

[1]: http://upper.io
[2]: http://golang.org
[3]: http://git-scm.com/
[4]: http://golang.org/doc/install
