# MySQL adapter for upper.io/db

The `mysql` adapter for [MySQL][2] wraps the `github.com/go-sql-driver/mysql`
driver written by [Julien Schmidt][1].

This adapter supports basic CRUD queries, transactions, simple join queries and
raw SQL.

## Installation

Use `go get` to download and install the adapter:

```
go get upper.io/db/mysql
```

## Setting up database access

The `mysql.ConnectionURL{}` struct is defined like this:

```go
// ConnectionURL implements a MySQL connection struct.
type ConnectionURL struct {
  User     string
  Password string
  Address  db.Address
  Database string
  Options  map[string]string
}
```

The `db.Address` interface can be satisfied by the `db.Host()`,
`db.HostPort()` or `db.Socket()` functions.

Alternatively, a `mysql.ParseURL()` function is provided:

```go
// ParseURL parses s into a ConnectionURL struct.
mysql.ParseURL(s string) (ConnectionURL, error)
```

You may use `mysql.ConnectionURL` as argument for `db.Open()`.

## Usage

To use this adapter, import `upper.io/db` and the `upper.io/db/mysql` packages.

```go
# main.go
package main

import (
  "upper.io/db"
  "upper.io/db/mysql"
)
```

Then, you can use the `db.Open()` method to connect to a MySQL server:

```go
var settings = mysql.ConnectionURL{
  Address:  db.Host("localhost"), // MySQL server IP or name.
  Database: "peanuts",            // Database name.
  User:     "cbrown",             // Optional user name.
  Password: "snoopy",             // Optional user password.
}

sess, err = db.Open(mysql.Adapter, settings)
```

## Example

The following SQL statement creates a table with "name" and "born"
columns.

```sql
--' example.sql
DROP TABLE IF EXISTS birthday;

CREATE TABLE birthday (
  `name` VARCHAR(50),
  `born` DATE
);
```

Use the `mysql` command line tool to create the birthday table on the
upperio_tests database.

```
cat example.sql | mysql -uupperio -pupperio upperio_tests
```

The Go code below will add some rows to the "birthday" table and then will
print the same rows that were inserted.

```go
// example.go

package main

import (
  "fmt"
  "time"
  "upper.io/db"         // Imports the main db package.
  "upper.io/db/mysql"   // Imports the mysql adapter.
)

var settings = mysql.ConnectionURL{
  Database: `upperio_tests`,                            // Database name
  Address:   db.Socket(`/var/run/mysqld/mysqld.sock`),  // Using unix sockets.
  User:     `upperio`,                                  // Database username.
  Password: `upperio`,                                  // Database password.
}

type Birthday struct {
  // Maps the "Name" property to the "name" column of the "birthday" table.
  Name string `db:"name"`
  // Maps the "Born" property to the "born" column of the "birthday" table.
  Born time.Time `db:"born"`
}

func main() {

  // Attemping to establish a connection to the database.
  sess, err := db.Open(mysql.Adapter, settings)

  if err != nil {
    log.Fatalf("db.Open(): %q\n", err)
  }

  // Remember to close the database session.
  defer sess.Close()

  // Pointing to the "birthday" table.
  birthdayCollection, err := sess.Collection("birthday")

  if err != nil {
    log.Fatalf("sess.Collection(): %q\n", err)
  }

  // Attempt to remove existing rows (if any).
  err = birthdayCollection.Truncate()

  if err != nil {
    log.Fatalf("Truncate(): %q\n", err)
  }

  // Inserting some rows into the "birthday" table.

  birthdayCollection.Append(Birthday{
    Name: "Hayao Miyazaki",
    Born: time.Date(1941, time.January, 5, 0, 0, 0, 0, time.UTC),
  })

  birthdayCollection.Append(Birthday{
    Name: "Nobuo Uematsu",
    Born: time.Date(1959, time.March, 21, 0, 0, 0, 0, time.UTC),
  })

  birthdayCollection.Append(Birthday{
    Name: "Hironobu Sakaguchi",
    Born: time.Date(1962, time.November, 25, 0, 0, 0, 0, time.UTC),
  })

  // Let's query for the results we've just inserted.
  var res db.Result

  res = birthdayCollection.Find()

  var birthday []Birthday

  // Query all results and fill the birthday variable with them.
  err = res.All(&birthday)

  if err != nil {
    log.Fatalf("res.All(): %q\n", err)
  }

  // Printing to stdout.
  for _, birthday := range birthday {
    fmt.Printf("%s was born in %s.\n", birthday.Name, birthday.Born.Format("January 2, 2006"))
  }

}
```

Running the example above:

```
go run main.go
```

Expected output:

```
Hayao Miyazaki was born in January 5, 1941.
Nobuo Uematsu was born in March 21, 1959.
Hironobu Sakaguchi was born in November 25, 1962.
```

## Unique adapter features

### Simple JOIN queries

Querying from multiple tables is possible using `db.Database.Collection()`,
just pass the name of all the tables separating them by commas. You can also
use the `AS` keyword to define an alias that you could later use in conditions
to refer to the original table.

```
var err error
var artistPublication db.Collection

// Querying from two tables.
artistPublication, err = sess.Collection(`artist AS a`, `publication AS p`)

if err != nil {
  log.Fatal(err)
}

res := artistPublication.Find(
  // Use db.Raw{} to enclose statements that you'd like to pass without
  // filtering.
  db.Raw{`a.id = p.author_id`},
).Select(
  `p.id`, // We defined "p" as an alias for "publication".
  `p.title as publication_title`, // The "AS" is recognized as column alias.
  db.Raw{`a.name AS artist_name`},
)

type artistPublication_t struct {
  ID               int64  `db:"id"`
  PublicationTitle string `db:"publication_title"`
  ArtistName       string `db:"artist_name"`
}

all := []artistPublication_t{}

if err = res.All(&all); err != nil {
  log.Fatal(err)
}
```

If you're working with more than one collection, the first one you pass becomes
your primary collection. Calls to `db.Collection.Append()`,
`db.Collection.Remove()` and `db.Collection.Update()` will be performed on your
primary collection.

### Auto-incremental keys

If you want to use auto-increment keys with a MySQL database,
you must define the column type as `NOT NULL AUTO_INCREMENT`, like this:

```sql
CREATE TABLE foo(
  id BIGINT(20) UNSIGNED NOT NULL AUTO_INCREMENT, PRIMARY KEY(id),
  title VARCHAR(255)
);
```

Also, you must provide the `omitempty` option in the `db` tag when defining the
struct:

```go
type Foo struct {
  ID    int64   `db:"id,omitempty"`
  Title string  `db:"title"`
}
```

In order for the ID to be returned by `db.Collection.Append()`, the primary key
must be named "id", this is a known limitation that will be fixed on future
releases.

### Raw SQL

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
    ID       int64  `db:"id,omitempty"`
    Title    string `db:"title"`
    AuthorID int64  `db:"author_id"`
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
obtained from `sql.DB.Query()` calls to a pointer of a single struct instead of
a pointer to an array of structs. Please note that there is no support for
`sql.DB.QueryRow()` and that you must provide a `*sql.Rows` value to both
`sqlutil.FetchRow()` and `sqlutil.FetchRows()`.

### Using `db.Raw` and `db.Func`

If you need to provide a raw parameter for a method you can use the `db.Raw`
type. Plese note that raw means that the specified value won't be filtered:

```go
res = sess.Find().Select(db.Raw{`DISTINCT(name)`})
```

`db.Raw` also works for condition values.

Another useful type that you could use to create an equivalent statement is
`db.Func`:

```go
res = sess.Find().Select(db.Func{`DISTINCT`, `name`})
```

[1]: https://github.com/go-sql-driver/mysql
[2]: http://www.mysql.com
