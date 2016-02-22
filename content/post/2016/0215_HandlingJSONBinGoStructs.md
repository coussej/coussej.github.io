+++
date = "2016-02-16T10:33:11Z"
description = ""
title = "Handling JSONB in Go Structs"
+++

In a [previous]({{< relref "post/2016/0114_ReplacingEAVwithJSONBinPostgreSQL.md" >}}) post I already described how much database design can be simplified by using the PostgreSQL JSONB datatypes for storing entity properties. Here, I'll show how you can map this design directly onto your structs in Go. 

We want to handle this kind of entity in our application:

```go
{
  id:          1
  name:        "test entity 1"
  description: "a test entity for some guy's blog"
  properties:  {
    color:        "red"
    lenght:       120
    width:        3.1882420
    hassomething: true
    country:      "Belgium"
  } 
}
```

To store this kind of entity, we create the following table in a PostgreSQL database:

```sql
CREATE TABLE entity (
  id          SERIAL PRIMARY KEY, 
  name        TEXT, 
  description TEXT,
  properties  JSONB
);
```

#### Handling in Go

In go, wel'll create a struct with the same fields as our database columns:

```go
type Entity struct {
	Id          int         `db:"id"`
	Name        string      `db:"name"`
	Description string      `db:"description"`
	Properties  ???         `db:"properties"`
}
```

But what type will we give to the Properties field? Turns out that when querying the JSONB column, the [lib/pq](https://github.com/lib/pq) driver will return a bytestring. The most convenient way to work with JSONB coming from a database would be in the form of a *map[string]interface{}*, not in the form of a JSON object and most certainly not as bytes. Luckely, the Go standard library has 2 built-in interfaces we can implement to create our own database compatible type: *sql.Scanner* & *driver.Valuer*

>> For more info on these interfaces, check out [this](http://jmoiron.net/blog/built-in-interfaces) excellent post. In summary, when you have a type that implements these 2 interfaces, you can use that type with the database/sql package.

First, we create the type for our properties field. Note that if you have different kinds of entities (orders, customers, books, ...), you can simple re-use this type if they have a similar field: 

```go
type PropertyMap map[string]interface{}
```

Then we implement the interface. Let's start with *driver.Valuer*. To satisfy this interface, we must implement the *Value* method, which must transform our type to a database driver compatible type. In our case, we'll marshall the map to JSONB data (= []byte):

```go
func (p PropertyMap) Value() (driver.Value, error) {
	j, err := json.Marshal(p)
	return j, err
}
```

That's it. Time for the second interface, *sql.Scanner*, which requires us to implement a *Scan* method. This method must take the raw data that comes from the database and transform it to our new type. In our case, the database will return JSONB ([]byte) that we must transform to our type (the reverse of what we did with *driver.Valuer*):

```go
func (p *PropertyMap) Scan(src interface{}) error {
	source, ok := src.([]byte)
	if !ok {
		return errors.New("Type assertion .([]byte) failed.")
	}

	var i interface{}
	err := json.Unmarshal(source, &i)
	if err != nil {
		return err
	}

	*p, ok = i.(map[string]interface{})
	if !ok {
		return errors.New("Type assertion .(map[string]interface{}) failed.")
	}

	return nil
}
```

That's it. Now we can use this type as any other type with the database/sql package:

```go
e := Entity{Id:1}

err = db.QueryRow("SELECT name, description, properties FROM entity WHERE id = $1", 
              e.Id).Scan(&e.Name, &e.Description, &e.Properties)
              
fmt.Printf("%+v\n", e)
```

which results in 

```
{Id:1 Name:test entity 1 Description:a test entity for some guy's blog Properties:map[color:red width:3.1882420 length:120 country:Belgium hassomething:true]}
```

Accessing an individual property can then be done as follows:

```go
width, ok := e.Properties["width"].(float64)
color, ok := e.Properties["color"].(string)
```

If you want even more simplicity, I suggest you take a look at the [sqlx](https://github.com/jmoiron/sqlx) package, which extends the standard sql package with some very useful features. For example, instead of selecting a number of rows and scanning them row by row into a struct, sqlx allows you to do this:

```go
var test []Entity
db.Select(&test, "SELECT * FROM entity")
```

Ah, how minimal. 




