# QL adapter for upper.io/db

The `upper.io/db/ql` adapter for the [QL][1] is a wrapper of the
`github.com/cznic/ql/ql` driver written by [Jan Mercl][1].

## Installation

Use `go get` to download and install the adapter:

```go
go get upper.io/db/ql
```

## Usage

To use this adapter, import `upper.io/db` and the `upper.io/db/ql` packages.
Note that the adapter must be imported to the [blank identifier][2].

```go
# main.go
package main

import (
  "upper.io/db"
  _ "upper.io/db/ql"
)
```

Then, you can use the `db.Open()` method to open a QL database file:

```go
var settings = db.Settings{
  Database: `/path/to/example.db`, // Path to a QL database file.
}

sess, err = db.Open("ql", settings)
```

## Example

The following SQL statement creates a table with "name" and "born"
columns.

```sql
--' example.sql

DROP TABLE IF EXISTS birthdays;

CREATE TABLE birthdays (
  name string,
  born time
);
```

Use the `ql` command line tool to create a `example.db` database
file.

```
rm -f example.db
cat example.sql | ql -db example.db
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
	"upper.io/db"      // Imports the main db package.
	_ "upper.io/db/ql" // Imports the ql adapter.
)

var settings = db.Settings{
	Database: `example.db`, // Path to database file.
}

type Birthday struct {
	// Maps the "Name" property to the "name" column of the "birthdays" table.
	Name string `db:"name"`
	// Maps the "Born" property to the "born" column of the "birthdays" table.
	Born time.Time `db:"born"`
}

func main() {

	// Attemping to open the "example.db" database file.
	sess, err := db.Open("ql", settings)

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

[1]: https://github.com/cznic/ql
[2]: http://golang.org/doc/effective_go.html#blank
