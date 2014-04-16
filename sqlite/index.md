# SQLite adapter for upper.io/db

The `upper.io/db/sqlite` adapter for the [SQLite3 database][3] is a wrapper of
the `github.com/mattn/go-sqlite3` driver written by [Yasuhiro Matsumoto][1].

## Installation

This package uses cgo, so in order to compile and install it you'll also need a
C compiler, such as `gcc`:

```
# Debian
sudo apt-get install gcc

# FreeBSD
sudo pkg install gcc
sudo ln -s /usr/local/bin/gcc47 /usr/local/bin/gcc
```

Otherwise, you'll end with an error like this:

```
# github.com/mattn/go-sqlite3
exec: "gcc": executable file not found in $PATH
```

Once `gcc`is installed, use `go get` to download, compile and install the
sqlite adapter.

```
go get upper.io/db/sqlite
```

## Usage

To use this adapter, import `upper.io/db` and the `upper.io/db/sqlite`
packages. Note that the adapter must be imported to the [blank identifier][2].

```go
# main.go
package main

import (
  "upper.io/db"
  _ "upper.io/db/sqlite"
)
```

Then, you can use the `db.Open()` method to open a SQLite3 database file:

```go
var settings = db.Settings{
  Database: `/path/to/example.db`, // Path to a sqlite3 database file.
}

sess, err = db.Open("sqlite", settings)
```

## Example

The following SQL statement creates a table with "name" and "born"
columns.

```sql
--' example.sql

DROP TABLE IF EXISTS "birthdays";

CREATE TABLE "birthdays" (
  "name" varchar(50) DEFAULT NULL,
  "born" varchar(12) DEFAULT NULL
);
```

Use the `sqlite3` command line tool to create a `example.db`
database file.

```
rm -f example.db
cat example.sql | sqlite3 example.db
```

The Go code below will add some rows to the "birthdays" table and then will
print the same rows that were inserted.

```go
// example.go

package main

import (
  "fmt"
  "log"
  "time"
  "upper.io/db"          // Imports the main db package.
  _ "upper.io/db/sqlite" // Imports the sqlite adapter.
)

var settings = db.Settings{
  Database: `example.db`, // Path to database file.
}

type Birthday struct {
  // Maps the "Name" property to the "name" column of the "birthdays" table.
  Name string `field:"name"`
  // Maps the "Born" property to the "born" column of the "birthdays" table.
  Born time.Time `field:"born"`
}

func main() {

  // Attemping to open the "example.db" database file.
  sess, err := db.Open("sqlite", settings)

  if err != nil {
    log.Fatalf("db.Open(): %q\n", err)
  }

  // Remember to close the database session.
  defer sess.Close()

  // Pointing to the "birthdays" table.
  birthdayCollection, err := sess.Collection("birthdays")

  if err != nil {
    log.Fatalf("sess.Collection(): %q\n", err)
  }

  // Attempt to remove existing rows (if any).
  err = birthdayCollection.Truncate()

  if err != nil {
    log.Fatalf("Truncate(): %q\n", err)
  }

  // Inserting some rows into the "birthdays" table.

  birthdayCollection.Append(Birthday{
    Name: "Hayao Miyazaki",
    Born: time.Date(1941, time.January, 5, 0, 0, 0, 0, time.Local),
  })

  birthdayCollection.Append(Birthday{
    Name: "Nobuo Uematsu",
    Born: time.Date(1959, time.March, 21, 0, 0, 0, 0, time.Local),
  })

  birthdayCollection.Append(Birthday{
    Name: "Hironobu Sakaguchi",
    Born: time.Date(1962, time.November, 25, 0, 0, 0, 0, time.Local),
  })

  // Let's query for the results we've just inserted.
  var res db.Result

  res = birthdayCollection.Find()

  var birthdays []Birthday

  // Query all results and fill the birthdays variable with them.
  err = res.All(&birthdays)

  if err != nil {
    log.Fatalf("res.All(): %q\n", err)
  }

  // Printing to stdout.
  for _, birthday := range birthdays {
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

[1]: https://github.com/mattn/go-sqlite3
[2]: http://golang.org/doc/effective_go.html#blank
[3]: http://www.sqlite.org/
