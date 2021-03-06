# Databases

If you're unsure which database to use, Postgresql is a highly reliable, well maintained project and has excellent drivers for Go. the examples shown below use psql, but can be adapted for other databases. Go has drivers for all the mainstream databases, as well as a selection of time series and key/value stores written in Go.

You can find a list of go database drivers on the Go wiki page [SQL Drivers](https://github.com/golang/go/wiki/SQLDrivers) - prefer a driver which doesn't use cgo and conforms to the [database/sql](https://golang.org/pkg/database/sql/) interface if possible.

## Importing a driver

Importing a database driver is used to register the database driver. In retrospect this is an unfortunate design decision, and it would be better to register drivers explicitly, but it won't change in Go 1. So import the driver as follows in order to register it and use a given database:

```go
import (
  "database/sql"
  // This line registers the database driver 
  // and exposes the standard database/sql interface
  _ "github.com/lib/pq" 
)
```

If code imports lots of drivers, it will be importing all the code, and registering the drivers on init, so only import those you need to use in a program. If importing drivers with the blank identifier as above you can only use the standard sql interface defined in database/sql. You may find you want to use a higher level package to simplify common operations, or write your own wrapper for the database in complex applications. 

## Opening a connection

When you open a connection to the database, you must ping it to ensure it opened correctly

```go
// Open the database (no connection is miade)
db, err := sql.Open("postgres","postgres://azurediamond:hunter2@localhost/azurediamond?sslmode=verify-full")
        "user:password@tcp(127.0.0.1:3306)/hello")
if err != nil {
    return err
}

// Check the db connection
err = db.Ping()
if err != nil {
    return err
}
```

## Closing a connection

Don't close the connection to the database frequently as it is designed to be recycled,  you should ideally create one connection per datastore which lasts for the lifetime of your application. For example you might call defer db.Close in main.

```go
defer db.Close()
```

## Querying the database

Querying the database can be driver specific, especially for more advanced features, so be sure to read the driver documentation for the particular database/sql flavour you're using.

### Parameter placeholders

The different databases use different formats for parameter placeholders, for example they don't all support ? as you might expect. The Mysql driver uses ?, ? etc, the sqlite driver uses ?, ? etc , the Postgresql driver uses $1, $2 etc and the Oracle driver uses :val1, :val2 etc. Sqlx is one of the few drivers to support named query parameters.

### Reading values

To read Values from the database, use db.Query to fetch a result:

```go
// Select from the db
sql := "select count from users where status=100"
rows, err := db.Query(sql)
if err != nil {
    return err
}
defer rows.Close()
```

And rows.Scan to scan in the rows received back:

```go
// Read the count (just one row and one col)
var count int 
for rows.Next() {
   err = rows.Scan(&count)
   if err != nil {
     return err
   }
}
```

### Reading a Row into a struct

To read a single row into a struct, you can use Query Row. You may want to add a constructor to your models which takes a row and instantiates the model from it, but the code below is a minimal example.

```go
// Query a row from the database
row := db.QueryRow("select id,name,created_at from tablename where id=?", id)

type Resource struct{
    ID int
    Name string
    CreatedAt time.Time
}
var r Resource
err := row.Scan(&r.ID, &r.Name, &r.CreatedAt) // This assumes no values are nil in the database
if err != nil {
    return err
}
```

### Scanning into a struct

Many database libraries use reflect to attempt to introspect struct fields. You should try to avoid using reflect if you can as it is slow, prone to panics, and requires you to use struct tags and a dsl invented by the database library author. Another approach is to generate a list of columns, and pass them to the struct itself to assign, since it knows all about its fields and which values should go into them. This will require the struct to validate the column values and deal with nulls.  

```go
type User struct {
    ID int
    Name string
    CreatedAt time.Time
}

func ReadUser(columns map[string]interface{}) *User {
    u := &User{}
    u.ID = validateInt(cols["id"])
    u.Name = validateString(cols["name"])
    u.CreatedAt = validateTime(cols["created_at"])
    return u
}
```

## Handling Relations

This can seem a bit more fiddly than other languages, however the best approach is a straightforward one - retrieve the indexes of relations, and then if you require all of their information, retrieve the relations separately from the database. 

You can of course retrieve them with a join at the same time, but in complex apps it helps to separate retrieving relations from retrieving the actual relation records, which is often not necessary \(for example you might need to know a user has 8 images, and which ids they have, but not all the image captions and image data\). Being forced to handle relations manually does have advantages compared to the say ActiveRecord which can pull in far too much data behind the scenes.

## Connections are recycled

Database connections may be called from many goroutines and connections are pooled by the driver. So don't use stateful sql commands like USE, BEGIN or COMMIT, LOCK TABLES etc and instead use the facilities offered by the sql driver. There’s no default limit on the number of connections, so it is possible to exhaust the number of connections allowed by the database. You can use  SetMaxOpenConns and SetMaxIdleConns to control this behaviour.

```go
db.SetMaxIdleConns(10)
db.SetMaxOpenConns(100)
```

## Null values

If your database may contain null values, you must guard against them. One way to do this is to scan into a map of empty interface and then assert that the interface{} contains the type you \(as opposed to a special null value\). If it fails, use the zero value of the type instead.

## Getting Database columns

If you wish to know which columns are in a row, you can call:

```go
// Get a list of column names for the row
cols, err := rows.Columns()
```

This might be useful if you vary selects \(e.g. sometimes select just id and created at, sometimes select the full model\), and yet wish to always use a standard constructor for models which can be passed a map\[string\]interface{} with the values.

## Writing to the database

```go
// Insert a row
row := db.QueryRow("insert into tablename VALUES($1,$2,$3)", args...)
// Retrieve the inserted id
err = row.Scan(&id)
// Return the id and any error
return id, err
```

## Multiple Statements

The database/sql interface doesn't specify that drivers should support multiple statements, which means the behaviour is undefined and it's probably best to send single statements, unless your driver explicitly supports it.

