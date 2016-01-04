+++
date = "2015-09-15T22:23:45Z"
title = "Listening to generic JSON notifications from PostgreSQL in Go"
+++

It's already widely known that PostgreSQL is the leading open source relational database when it comes to features. One of those features is great JSON support, an other is LISTEN/NOTIFY, a nifty pub-sub sytem exclusive to PostgreSQL. When combining these two, you get a good basis for a real-time push notification system. In this post, I will explain how to create a generic trigger function to generate JSON notifications for any table change, and how to listen to them in Go.

## The trigger function

First, we will create the trigger function. The function will notify a channel *events* with the table name, the action and the old/new row data, depending on the action. As there are no table specific references, you can use the same trigger on all tables you want notifications from.

``` sql
CREATE OR REPLACE FUNCTION notify_event() RETURNS TRIGGER AS $$

    DECLARE 
        data json;
        notification json;
    
    BEGIN
    
        -- Convert the old or new row to JSON, based on the kind of action.
        -- Action = DELETE?             -> OLD row
        -- Action = INSERT or UPDATE?   -> NEW row
        IF (TG_OP = 'DELETE') THEN
            data = row_to_json(OLD);
        ELSE
            data = row_to_json(NEW);
        END IF;
        
        -- Contruct the notification as a JSON string.
        notification = json_build_object(
                          'table',TG_TABLE_NAME,
                          'action', TG_OP,
                          'data', data);
        
                        
        -- Execute pg_notify(channel, notification)
        PERFORM pg_notify('events',notification::text);
        
        -- Result is ignored since this is an AFTER trigger
        RETURN NULL; 
    END;
    
$$ LANGUAGE plpgsql;
```

Now that the trigger function is created, let's create a sample table...

``` sql
CREATE TABLE products (
  id SERIAL,
  name TEXT,
  quantity FLOAT
);
```

... and add a trigger to it. As you can see, we add the trigger for all three actions in only one statement:

``` sql
CREATE TRIGGER products_notify_event
AFTER INSERT OR UPDATE OR DELETE ON products
    FOR EACH ROW EXECUTE PROCEDURE notify_event();
```

## Result
Fire up *psql* or any other PostgreSQL client and execute the following:

```
exampledb=# LISTEN events;
LISTEN
exampledb=# INSERT INTO products (name, quantity)
exampledb-# VALUES ('pen', 10200);
INSERT 0 1
Asynchronous notification "events" with payload "{"table" : "products", 
  "action" : "INSERT", "data" : {"id":1,"name":"pen","quantity":10200}}" 
  received from server process with PID 799.
exampledb=#
```

Et voila, you now get notifications for any action on the table. 

## Listening for the notifications in Go.

Receiving notifications in a terminal doesn't have a lot of real-life value, so the next step would be to capture these events in a server application. In Go, this is fairly easy using the [pq](https://godoc.org/github.com/lib/pq) package, which already includes functionality for listening to PostgreSQL notifications. 

The app below is based on the [sample app](http://godoc.org/github.com/lib/pq/listen_example) in the pqdocs.

``` go
package main

import (
	"bytes"
	"database/sql"
	"encoding/json"
	"fmt"
	"time"
	"github.com/lib/pq"
)

func waitForNotification(l *pq.Listener) {
	for {
		select {
		case n := <-l.Notify:
			fmt.Println("Received data from channel [", n.Channel, "] :")
			// Prepare notification payload for pretty print
			var prettyJSON bytes.Buffer
			err := json.Indent(&prettyJSON, []byte(n.Extra), "", "\t")
			if err != nil {
				fmt.Println("Error processing JSON: ", err)
				return
			}
			fmt.Println(string(prettyJSON.Bytes()))
			return
		case <-time.After(90 * time.Second):
			fmt.Println("Received no events for 90 seconds, checking connection")
			go func() {
				l.Ping()
			}()
			return
		}
	}
}

func main() {
	var conninfo string = "dbname=exampledb user=webapp password=webapp"

	_, err := sql.Open("postgres", conninfo)
	if err != nil {
		panic(err)
	}

	reportProblem := func(ev pq.ListenerEventType, err error) {
		if err != nil {
			fmt.Println(err.Error())
		}
	}

	listener := pq.NewListener(conninfo, 10*time.Second, time.Minute, reportProblem)
	err = listener.Listen("events")
	if err != nil {
		panic(err)
	}

	fmt.Println("Start monitoring PostgreSQL...")
	for {
		waitForNotification(listener)
	}
}
```

Now, run the app and make a change on the products table. You will see the notifications are being handled by the app:

```
Received data from channel [ events ] :
{
        "table": "products",
        "action": "INSERT",
        "data": {
                "id": 1,
                "name": "pen",
                "quantity": 10200
        }
}  
```

## What's next?

Of course, this example app isn't going to be of much use in real life. A better example would be to create a Websocket http handler that notifies any subscribed clients of the changes, so you can forward the database updates in real time to the clients. I'll take this up in a next post.