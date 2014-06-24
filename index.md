# upper.io/db

The `upper.io/db` package for [Go][2] provides a single interface for
interacting with different data sources through the use of adapters that wrap
mature database drivers.

Usage:

```go
import(
  // Main package.
  "upper.io/db"
  // PostgreSQL adapter.
  "upper.io/db/postgresql"
)
```

As of today, `upper.io/db` fully supports MySQL, PostgreSQL and SQLite (CRUD +
Transactions) and provides partial support for MongoDB and QL (CRUD only).

The following code example pulls data from a collection and populates the
`people` array with results of the query.

```go
// This code works the same for all supported databases.
var people []Person
res = col.Find(db.Cond{"name": "Max"}).Limit(2).Sort("-input")
err = res.All(&people)
```

`upper.io/db` is not a full-featured ORM, and thus it does not impose any hard
restrictions on data structures nor automatic table creation, indexing or any
additional magic, it just manages the most common operations so you can focus
on the complex stuff, if you need to do some complicated database query you'll
have to do it by hand, so having a good understanding of the database you're
working on is required.

This is the documentation site, you may also find useful information in the
[source code repository][7] at [github][7].

## Required software

The `upper.io/db` package depends on the [Go compiler and tools][4]. [Version
1.1+][5] is preferred since the underlying `mgo` driver for the `mongo` adapter
depends on a method (`reflect.Value.Convert()`) introduced in go1.1.  However,
using a lower Go version (down to go1.0.1) could still be possible with other
adapters.

In order to use `go get` to fetch and install [Go][4] packages, you'll also
need the [git][3] version control system. Please refer to the [git][3] project
site for specific instructions on how to install it in your operative system.

### Installation

Once you got [Go][4] installed, you can download and install the `upper.io/db`
package using `go get`:

```sh
go get upper.io/db
```

*Note:* If the `go get` program fails with something like:

```sh
package upper.io/db: exec: "git": executable file not found in $PATH
```

it means that a required program is missing, `git` in this case. To fix this
error install the missing application and try again.

## Database adapters

Installing the main package just provides base structures and interfaces but in
order to actually communicate with a database you'll also need a database
adapter.

Here's a list of available database adapters. Look into the adapter's link to
see installation instructions that are specific to each adapter.

* [MySQL](./db/mysql/)
* [MongoDB](./db/mongo)
* [PostgreSQL](./db/postgresql)
* [QL](./db/ql)
* [SQLite](./db/sqlite/)

## Learn by example

Through this example we'll be featuring the usage of the SQLite adapter.

Fire up a terminal and check if the `sqlite3` command is installed. If the
program is missing install it like this:

```go
# Installing sqlite3 in Debian
sudo apt-get install sqlite3 -y
```

Then, run the `sqlite3` command and create a `test.db` database:

```sh
sqlite3 test.db
```

The `sqlite3` program will welcome you with a prompt, like this:

```
sqlite>
```

From within the `sqlite3` prompt, create a demo table:

```sql
CREATE TABLE demo (
  first_name VARCHAR(80),
  last_name VARCHAR(80),
  bio TEXT
);
```

After creating the table, type `.exit` to end the `sqlite3` session.

```
sqlite> .exit
```

### Connection setup

Now you're ready to code. Create a `main.go` file and import both the
`upper.io/db` and the recently installed adapter `upper.io/db/sqlite`:

```go
# main.go
package main

import (
  "upper.io/db"
  "upper.io/db/sqlite"
)
```

Then, configure the database credentials using the `db.Settings{}` struct. This
struct is used to store authentication settings that `db.Open()` will use to
connect to a database:

```go
// Connection and authentication data.
type Settings struct {
  // Database server hostname or IP. Leave blank if
  // using unix sockets.
  Host string
  // Database server port. Leave blank if using
  // unix sockets.
  Port int
  // Name of the database.
  Database string
  // Username (for authentication).
  User string
  // Password (for authentication).
  Password string
  // A path of a UNIX socket file. Leave blank if
  // using host and port.
  Socket string
  // Database charset.
  Charset string
}
```

In this example we'll be using a SQLite3 database with no authentication so
most `db.Settings{}` fields like `User`, `Password` or `Hostname` are not
required:

```go
# main.go
var settings = db.Settings{
  // A SQLite database is a plain file.
  Database: `example.db`,
}
```

After configuring the database settings create a `main()` function and use
`db.Open()` inside. This method creates a connection to a database using the
given adapter. The first argument must be the adapter's name
(`upper.io/db/$NAME`), the second one should be a `db.Settings{}` variable,
such as the `settings` we've created above.

```go
// Using db.Open() to open the sqlite database
// specified by the settings variable.
sess, err = db.Open(sqlite.Adapter, settings)
```

At this point, you can use the `sess` variable to get a `db.Collection{}`
reference.

## Working with collections

Collections are sets of objects of the same class. SQL tables and NoSQL
collections are both known as just "collections" in the `upper.io/db` context.

### Getting a collection reference

In order to use and query collections you'll need a collection reference, use
the `db.Database.Collection()` method on the previously defined `sess` variable
in order to get a collection reference:

```go
// Pass the table/collection name to get a collection
// reference.
col, err = sess.Collection("demo")
```

### Creating objects (The C in CRUD)

Use the `col` variable and a map to insert a new item into the collection:

```go
// You can use maps or structs, in this case we'll
// be appending a new row defined by a map.
item = map[string]interface{}{
  "first_name": "Hayao",
  "last_name": "Miyazaki",
  "bio": "Japanese film director.",
}
col.Append(item)
```

Using structs to define the format of collection items is not strictly required
(we could use maps too), but it's recommended. If you'd like to create a struct
datatype for the demo table we've created before, then you should define each
column as a field of the struct:

```go
// Use the "db" tag to match database column names with Go
// struct property names.
type Demo struct {
  FirstName string `db:"first_name"`
  LastName  string `db:"last_name"`
  Bio       string `db:"bio,omitempty"`
}
```

And instead of using a map, we can insert a `Demo{}` value with the
`db.Collection.Append()` call.

```go
// Inserting a struct is as easy as inserting a map.
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
  "log"
  "upper.io/db"
  "upper.io/db/sqlite"
)

var settings = db.Settings{
  Database: "test.db",
}

type Demo struct {
  FirstName string `db:"first_name"`
  LastName  string `db:"last_name"`
  Bio       string `db:"bio"`
}

func main() {
  var err error
  var sess db.Database
  var col db.Collection

  sess, err = db.Open(sqlite.Adapter, settings)

  if err != nil {
    log.Fatal(err)
  }

  defer sess.Close()

  col, err = sess.Collection("demo")

  if err != nil {
    log.Fatal(err)
  }

  err = col.Append(Demo{
    FirstName: "Hayao",
    LastName: "Miyazaki",
    Bio: "Japanese film director.",
  })

  if err != nil {
    log.Fatal(err)
  }
}
```

compile and run it, like this:

```sh
go build main.go
./main
```

## Mapping structs to columns

While `upper.io/db` will work fine with Go maps, using structs is the
recommended way for mapping table rows or collection elements into Go values,
as they provide more control and customization on how columns are mapped.

### Defining a struct

You can map database column names to struct fields in a similar way the
`encoding/json` package does.

In this example:

```go
type Foo struct {
  // Will match the column named "id".
  Id      int64
  // Will match the column named "title".
  Title   string
  // Will be ignored, as it's not an exported field.
  private bool
}
```

The `Id` and `Title` fields begin with an uppercase letter, so they are
exported fields. Exported fields of a struct are mapped to table columns
according to their name (letter case and underscores won't matter), while the
`private` field is unexported and will be ignored by `upper.io/db`.

`upper.io/db` assumes that a field will be matched to one and only one column,
trying to map multiple fields to the same column is currently not supported.

### Custom column names and field options (usage of the `db` tag).

If the name of the exported field is different from the name of the column, you
could use a `db` tag in the field definition to bind it to a custom column
name:

```go
type Foo struct {
  Id      int64
  // Will be mapped to the "foo_title" column.
  Title   string `db:"foo_title"`
  private bool
}
```

Besides specifying column names, the `db` tag could be used to pass additional
options for fields. You can specify more than one option by separating them
using commas:

```go
type Foo struct {
  Id      int64
  Title   string `db:"title,opt1,opt2,..."`
  private bool
}
```

### Skipping empty fields

If you'd like to avoid using an exported field in a statement when it's empty,
you can pass the `omitempty` option to the `db` tag, like this:

```go
type Foo struct {
  // Will be skipped when Id == 0.
  Id      int64  `db:"id,omitempty"`
  Title   string `db:"foo_title"`
  private bool
}
```

### Ignoring an exported field

If you need to skip an exported field you can set its name to "-" using a `db`
tag:

```go
type Foo struct {
  Id            int64
  Title         string
  private       bool
  IgnoredField  string `db:"-"`
}
```

You can have as name fields named `"-"` as you need:

```go
type Foo struct {
  Id            int64
  Title         string
  private       bool
  IgnoredField  string `db:"-"`
  IgnoredField1 string `db:"-"`
  IgnoredField2 string `db:"-"`
}
```

### Embedded structs

If you need to embed one struct into another and you'd like the two of them
being considered as if they were part of the same struct (at least on
`upper.io/db` context), you can pass the `inline` option to the field name,
like this:

```go
type Foo struct {
  Id      int64
  Title   string `db:"-"`
  private bool
}
```

```go
type Bar struct {
  EmbeddedFoo Foo     `db:",inline"`
  ExtraField  string  `db:"extra_field"`
}
```

Embedding with `inline` also works for anonymous fields:

```go
type Foo struct {
  Id      int64
  Title   string `db:"-"`
  private bool
}
```

```go
type Bar struct {
  Foo         `db:",inline"`
  ExtraField  string `db:"extra_field"`
}
```

## Working with result sets

You can use the `db.Collection.Find()` to define result sets, you can use a
result set to search for the recently appended item. Result sets can be
iterated (`db.Collection.Next()`), dumped to a pointer (`db.Result.One()`) or
dumped to a pointer of array of items (`db.Result.All()`).

```go
// SELECT * FROM people WHERE last_name = "Miyazaki"
res = col.Find(db.Cond{"last_name": "Miyazaki"})
```

## Retrieving objects (The R in CRUD)

Once you have a result set (`res` in this example), you can choose to fetch
results into an array, providing a pointer to an array of structs or maps, as
in the following example.

```go
// Define birthdays as an array of Birthday{} and fetch
// the contents of the result set into it using
// `db.Result.All()`.
var birthdays []Birthday
err = res.All(&birthdays)
```

Filling an array could be expensive if you're working with a lot of rows, if
you're working with big result sets looping over one result at a time would
perform better. Use `db.Result.Next()` to fetch one row at a time:

```go
var birthday Birhday
for {
  // Walking over the result set.
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
// Remember to close the result set when using
// db.Result.Next()
res.Close()
```

If you need only one element of the result set, the `db.Result.One()` method
could be better suited for the task.

```go
var birthday Birhday
err = res.One(&birthday)
```

## Narrowing result sets

Once you have a basic understanding of result sets, you can start using
conditions, limits and offsets to reduce the amount of rows returned in a
query.

Use the `db.Cond{}` type to define conditions for `db.Collection.Find()`.

```go
type db.Cond map[string]interface{}
```

```go
// SELECT * FROM users WHERE user_id = 1
res = col.Find(db.Cond{"user_id": 1})
```

If you want to add multiple conditions just provide more keys to the
`db.Cond{}` map:

```go
// SELECT * FROM users where user_id = 1
//  AND email = "ser@example.org"
res = col.Find(db.Cond{
  "user_id": 1,
  "email": "user@example.org",
})
```

provided conditions will be grouped under an *AND* conjunction, by default.

If you want to use an *OR* disjunction instead, the `db.Or{}` type is
available. The following code:

```go
// SELECT * FROM users WHERE
// email = "user@example.org"
// OR email = "user@example.com"
res = col.Find(db.Or{
  db.Cond{
    "email": "user@example.org",
  },
  db.Cond{
    "email": "user@example.com",
  }
})
```

uses *OR* disjunction instead of *AND*.

Complex *AND* filters can be delimited by the `db.And{}` type.

This example:

```go
res = col.Find(db.And{
  db.Or{
    db.Cond{
      "first_name": "Jhon",
    },
    db.Cond{
      "first_name": "John",
    },
  },
  db.Or{
    db.Cond{
      "last_name": "Smith",
    },
    db.Cond{
      "last_name": "Smiht",
    },
  },
})
```

means `(first_name = "Jhon" OR first_name = "John") AND (last_name = "Smith" OR
last_name = "Smiht")`.

## Result sets are chainable

A `col.Find()` instruction returns a `db.Result{}` interface, and some methods of
`db.Result{}` return the same interface, so they can be called in a chainable
fashion.

This example:

```go
res = col.Find().Skip(10).Limit(8).Sort("-name")
```

skips ten rows, counts up to eight rows and sorts the results by name
(descendent).

If you want to know how many items does the set hold, use the
`db.Result.Count()` call:

```go
c, err := res.Count()
```

this call will ignore `Offset` and `Limit` settings, so the returned result is
the total size of the result set.

## Closing result sets

When you're done using the result set, remember to close it.

```go
res.Close()
```

## More operations with result sets

Result sets are not only capable of returning rows, they can also be used to
update or delete all the rows that match the given conditions.

### Updating objects (The U in CRUD)

If you want to update the whole set of rows you can use the
`db.Result.Update()` method.

```go
res = col.Find(db.Cond{"name": "Old name"})
err = res.Update(map[string]interface{}{
  "name": "New name",
})
```

### Deleting objects (The D in CRUD)

If you want to delete a set of rows, use the `db.Result.Remove()` call on a
result set.

```go
res = col.Find(db.Cond{"active": 0})

res.Remove()
```

## Closing result sets

Make sure to use `db.Result.Close()` when finishing using the result set.

```go
res.Close()
```

## Working with databases

There are many more things you can do with a `db.Database{}` reference besides
getting a collection.

For example, you could get a list of all collections within the database:

```go
all, err = sess.Collections()
for _, name := range all {
  fmt.Printf("Got collection %s.\n", name)
}
```

If you need to switch databases, you can use the `db.Database.Use()` method

```go
err = sess.Use("another_database")
```

## Tips and tricks

### Logging

You can enable the logging of generated SQL statements and errors to standard
output by using the `UPPERIO_DB_DEBUG` environment variable:

```console
UPPERIO_DB_DEBUG=1 ./go-program
```

You can also use this environment variable when running tests.

```console
cd $GOPATH/src/upper.io/db/sqlite
UPPERIO_DB_DEBUG=1 go test
...
2014/06/22 05:15:20
  SQL: SELECT "tbl_name" FROM "sqlite_master" WHERE ("type" = 'table')

2014/06/22 05:15:20
  SQL: SELECT "tbl_name" FROM "sqlite_master" WHERE ("type" = ? AND "tbl_name" = ?)
  ARG: [table artist]
...
```

### Transactions

You can use the `db.Database.Transaction()` function to start a transaction (if
the database adapter supports such feature). This will return a clone of the
session (type `db.Tx{}`) with two added functions: `db.Tx.Commit()` and
`db.Tx.Rollback()` that you can use to save the transaction or to abort it.

```go
var tx db.Tx
if tx, err = sess.Transaction(); err != nil {
  log.Fatal(err)
}

var artist db.Collection
if artist, err = tx.Collection("artist"); err != nil {
  log.Fatal(err)
}

if _, err = artist.Append(row); err != nil {
  log.Fatal(err)
}

if err = tx.Commit(); err != nil {
  log.Fatal(err)
}
```

### Working with the underlying driver

Some situations will require you to use methods that are specific to the
underlying driver, for example, if you're in the need of using the
[mgo.Session.Ping](http://godoc.org/labix.org/v2/mgo#Session.Ping) method, you
can retrieve the underlying `*mgo.Session` as an `interface{}`, cast it with
the appropriate type and use the `mgo.Session.Ping()` method on it, like this:

```go
drv = sess.Driver().(*mgo.Session)
err = drv.Ping()
```

or this:

```go
drv = sess.Driver().(*sql.DB)
rows, err = drv.Query("SELECT name FROM users WHERE age=?", age)
```

if you're using a SQL adapter.

### Using sqlutil

Sometimes you'll need to run complex SQL queries with joins and database
specific magic, there is an extra package `sqlutil` that you could use in this
situation:

```go
import "upper.io/db/util/sqlutil"
```

This is an example for `sqlutil.FetchRows`:

```go
  var sess db.Database
  var rows *sql.Rows
  var err error
  var drv *sql.DB

  type publication_t struct {
    Id       int64  `db:"id,omitempty"`
    Title    string `db:"title"`
    AuthorId int64  `db:"author_id"`
  }

  if sess, err = db.Open(Adapter, settings); err != nil {
    t.Fatal(err)
  }

  defer sess.Close()

  drv = sess.Driver().(*sql.DB)

  rows, err = drv.Query(`
    SELECT
      p.id,
      p.title AS publication_title,
      a.name AS artist_name
    FROM
      artist AS a,
      publication AS p
    WHERE
      a.id = p.author_id
  `)

  if err != nil {
    t.Fatal(err)
  }

  var all []publication_t

  // Mapping to an array.
  if err = sqlutil.FetchRows(rows, &all); err != nil {
    t.Fatal(err)
  }

  if len(all) != 9 {
    t.Fatalf("Expecting some rows.")
  }
```

You can also use `sqlutil.FetchRow(*sql.Rows, interface{})` for mapping results
obtained from `sql.DB.Query()` statements to a pointer of a single struct
instead of a pointer to an array of structs. Please note that there is no
support for `sql.DB.QueryRow()` and that you must provide a `*sql.Rows` value
to both `sqlutil.FetchRow()` and `sqlutil.FetchRows()`.

## Method reference

You can see the [full method reference][6] for `upper.io/db` at [godoc.org][6].

## How to contribute

Thanks for taking the time to contribute. There are many ways you can help this
project:

### Reporting bugs and suggestions

The [source code page][7] at github includes a nice [issue tracker][8], please
use this interface to report bugs.

### Hacking the source

The [source code page][7] at github includes an [issue tracker][8], see the
issues or create one, then [create a fork][11], hack on your fork and when
you're done create a [pull request][12], so that the code contribution can get
merged into the main package. Note that not all contributions can be merged to
`upper.io/db`, so please be very explicit on justifying the proposed change and
on explaining how the package users can get the greater benefit from your hack.

### Improving the documentation

There is a special [documentation repository][9] at github were you can also
file [issues][10]. If you find any spot were you would like the docs to be more
specific, please open an issue to let us know; and if you're in the possibility
of helping us fixing grammar errors, typos, code examples or even documentation
issues, please [create a fork][11], edit the documentation files and then
create a [pull request][12], so that the contribution can be merged into the
main repository.

## License

The MIT license:

> Copyright (c) 2013-2014 José Carlos Nieto, https://menteslibres.net/xiam
>
> Permission is hereby granted, free of charge, to any person obtaining
> a copy of this software and associated documentation files (the
> "Software"), to deal in the Software without restriction, including
> without limitation the rights to use, copy, modify, merge, publish,
> distribute, sublicense, and/or sell copies of the Software, and to
> permit persons to whom the Software is furnished to do so, subject to
> the following conditions:
>
> The above copyright notice and this permission notice shall be
> included in all copies or substantial portions of the Software.
>
> THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND,
> EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF
> MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND
> NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE
> LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION
> OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION
> WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.

[1]: https://upper.io
[2]: http://golang.org
[3]: http://git-scm.com/
[4]: http://golang.org/doc/install
[5]: http://golang.org/doc/go1.1
[6]: http://godoc.org/upper.io/db
[7]: https://github.com/upper/db
[8]: https://github.com/upper/db/issues
[9]: https://github.com/upper/db-docs
[10]: https://github.com/upper/db-docs/issues
[11]: https://help.github.com/articles/fork-a-repo
[12]: https://help.github.com/articles/fork-a-repo#pull-requests
