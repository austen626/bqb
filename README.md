# Basic Query Builder

# Compatibility

This has been tested with sqlite, PostGres, and MySQL, using `database/sql`, `pq`, `pgx`, and `sqlx`.
By the nature of how it works it should be fully compatible with any DB interface and database that uses `?` or `$` parameter syntax.

_Note: Go `v1.20+` is required for BQB `>= v1.4.0`. Go `v1.17+` is required for BQB `<= v1.3.0`._

# Why

1. Simple, lightweight, and fast
2. Supports any and all syntax by the nature of how it works
3. Doesn't require learning special syntax or operators
4. 100% test coverage

# Examples

## Basic

```golang
q := bqb.New("SELECT * FROM places WHERE id = ?", 1234)
sql, params, err := q.ToSql()
```

Produces

```sql
SELECT * FROM places WHERE id = ?
```

```
PARAMS: [1234]
```

### Escaping `?`

Use the double question mark `??` value to escape the `?` in Postgres queries.
For example:

```golang
q := bqb.New("SELECT * FROM places WHERE json_obj_column ?? 'key'")
sql, params, err := q.ToPgsql()
```

This query uses the `?` operator for jsonb types in Postgres to test an object
for the presence of a key. It should not be interpreted as an escaped value by
bqb.

```sql
SELECT * FROM places WHERE json_obj_column ? 'key'
```

```
PARAMS: []
```

## Postgres - ToPgsql()

Just call the `ToPgsql()` method instead of `ToSql()` to convert the query to Postgres syntax

```golang
q := bqb.New("DELETE FROM users").
    Space("WHERE id = ? OR name IN (?)", 7, []string{"delete", "remove"}).
    Space("LIMIT ?", 5)
sql, params, err := q.ToPgsql()
```

Produces

```sql
DELETE FROM users WHERE id = $1 OR name IN ($2, $3) LIMIT $4
```

```
PARAMS: [7, "delete", "remove", 5]
```

## Raw - ToRaw()

_Obvious warning: You should not use this for user input_

The `ToRaw()` call returns a string with the values filled in rather than parameterized

```golang
q := New("a = ?, b = ?, c = ?", "my a", 1234, nil)
sql, err := q.ToRaw()
```

Produces

```
a = 'my a', b = 1234, c = NULL
```

## Types

```golang
q := bqb.New(
    "int:? string:? []int:? []string:? Query:? JsonMap:? nil:? []intf:?",
    1, "2", []int{3, 3}, []string{"4", "4"}, bqb.New("5"), bqb.JsonMap{"6": 6}, nil, []interface{}{"a",1,true},
)
sql, _ := q.ToRaw()
```

Produces

```
int:1 string:'2' []int:3,3 []string:'4','4' Query:5 JsonMap:'{"6":6}' nil:NULL []intf:'a',1,true
```

### driver.Valuer

The [driver.Valuer](https://pkg.go.dev/database/sql/driver#Valuer) interface is supported for types that are able to convert
themselves to a sql driver value. See [examples/main.go:valuer](./examples/main.go#L102).

```
q := bqb.New("?", valuer)
```

### Embedder

BQB provides an Embedder interface for directly replacing `?` with a string returned by the `RawValue` method on the Embedder implementation.

This can be useful for changing sort direction or embedding table and column names. See [examples/main.go:embedder](./examples/main.go#L122) for an example.

_Note: Since this is a raw value, special attention should be paid to ensure user-input is checked and sanitized._

## Query IN

Arguments of type `[]string`,`[]*string`, `[]int`,`[]*int`, or `[]interface{}` are automatically expanded.

```golang
    q := bqb.New(
        "strs:(?) *strs:(?) ints:(?) *ints:(?) intfs:(?)",
        []string{"a", "b"}, []*string{}, []int{1, 2}, []*int{}, []interface{}{3, true},
    )
    sql, params, _ := q.ToSql()
```

Produces

```
SQL: strs:(?,?) *strs:(?) ints:(?,?) *ints:(?) intfs:(?,?)
PARAMS: [a b <nil> 1 2 <nil> 3 true]
```

## Json Arguments

There are two helper structs, `JsonMap` and `JsonList` to make JSON conversion a little simpler.

```golang

sql, err := bqb.New(
    "INSERT INTO my_table (json_map, json_list) VALUES (?, ?)",
    bqb.JsonMap{"a": 1, "b": []string{"a","b","c"}},
    bqb.JsonList{"string",1,true,nil},
).ToRaw()
```

Produces

```sql
INSERT INTO my_table (json_map, json_list)
VALUES ('{"a": 1, "b": ["a","b","c"]}', '["string",1,true,null]')
```

## Query Building

Since queries are built in an additive way by reference rather than value, it's easy to mutate a query without
having to reassign the result.

### Basic Example

```golang
sel := bqb.New("SELECT")

...

// later
sel.Space("id")

...

// even later
sel.Comma("age").Comma("email")
```

Produces

```sql
SELECT id,age,email
```

### Advanced Example

The `Optional(string)` function returns a query that resolves to an empty string if no query parts have
been added via methods on the query instance, and joins with a space to the next query part. 
For example `q := Optional("SELECT")` will resolve to an empty string unless parts have been added by one of the methods,
e.g `q.Space("* FROM my_table")` would make `q.ToSql()` resolve to `SELECT * FROM my_table`.

```golang

sel := bqb.Optional("SELECT")

if getName {
    sel.Comma("name")
}

if getId {
    sel.Comma("id")
}

if !getName && !getId {
    sel.Comma("*")
}

from := bqb.New("FROM my_table")

where := bqb.Optional("WHERE")

if filterAdult {
    adultCond := bqb.New("name = ?", "adult")
    if ageCheck {
        adultCond.And("age > ?", 20)
    }
    where.And("(?)", adultCond)
}

if filterChild {
    where.Or("(name = ? AND age < ?)", "youth", 21)
}

q := bqb.New("? ? ?", sel, from, where).Space("LIMIT ?", 10)
```

Assuming all values are true, the query would look like:

```sql
SELECT name,id FROM my_table WHERE (name = 'adult' AND age > 20) OR (name = 'youth' AND age < 21) LIMIT 10
```

If `getName` and `getId` are false, the query would be

```sql
SELECT * FROM my_table WHERE (name = 'adult' AND age > 20) OR (name = 'youth' AND age < 21) LIMIT 10
```

If `filterAdult` is `false`, the query would be:

```sql
SELECT name,id FROM my_table WHERE (name = 'youth' AND age < 21) LIMIT 10
```

If all values are `false`, the query would be:

```sql
SELECT * FROM my_table LIMIT 10
```

## Methods

Methods on the bqb `Query` struct follow the same pattern.

All query-modifying methods take a string (the query text) and variable length interface (the query args).

For example `q.And("abc")` will add `AND abc` to the query.

Take the following

```golang
q := bqb.Optional("WHERE")
q.Empty() // returns true
q.Len() // returns 0
q.Space("1 = 2") // query is now WHERE 1 = 2
q.Empty() // returns false
q.Len() // returns 1
q.And("b") // query is now WHERE 1 = 2 AND b
q.Or("c") // query is now WHERE 1 = 2 AND b OR c
q.Concat("d") // query is now WHERE 1 = 2 AND b OR cd
q.Comma("e") // query is now WHERE 1 = 2 AND b OR cd,e
q.Join("+", "f") // query is now WHERE 1 = 2 AND b OR cd,e+f
```

Valid `args` include `string`, `int`, `floatN`, `*Query`, `[]int`, `Embedder`, `Embedded`, `driver.Valuer` or `[]string`.
