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

[1]: https://github.com/cznic/ql
[2]: http://golang.org/doc/effective_go.html#blank
