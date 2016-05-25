+++
date = "2016-05-24T14:51:08Z"
description = ""
title = "A Minus Operator For PostgreSQL's JSONB"

+++

Those who are familiar with the PostgreSQL [hstore](http://www.postgresql.org/docs/current/static/hstore.html) extension will probably know that you have the possibility to delete matching pairs from 2 hstore objects by substracting them from one another:

```
SELECT 'a=>1, b=>2, c=>3'::hstore - 'a=>4, b=>2'::hstore
```
which results in 
```
"a"=>"1", "c"=>"3"
```

This is particularly usefull for audit triggers that store the row data in a hstore column, where you can easily see which columns were changed and to what by substracting the new row from the old row (as hstore of course):

```
hstore(NEW_ROW) - hstore(OLD_ROW)
```

JSONB on the other hand does not have such an operator, which is unfortunate as many people are looking at JSONB for use cases where before they would choose hstore. Luckily, PostgreSQL allows you to create custom operators, which gives you the possibility to create a minus operator for JSONB yourself.

> Update: After I wrote this, I found [another implementation](http://schinckel.net/2014/09/29/adding-json(b)-operators-to-postgresql/) for this functionality. However, that one does not provide a way to also include nested jsonb objects, and this one does (see below).

First, let's create a function that gives us the desired result:

```
CREATE OR REPLACE FUNCTION jsonb_minus ( arg1 jsonb, arg2 jsonb ) RETURNS jsonb
AS $$

SELECT 
	COALESCE(json_object_agg(key, value), '{}')::jsonb
FROM 
	jsonb_each(arg1)
WHERE 
	arg1 -> key <> arg2 -> key 
	OR arg2 -> key IS NULL

$$ LANGUAGE SQL;
```

Let's break this down. The **jsonb_each** function returns a table with colums *key (text)* and *value (jsonb)*, with a row for each key in the argument. From that table, we only keep the rows for which the keys have a different value in *arg1* than in *arg2*, or keys that are not present in *arg2*. With the remaining rows, we build a new jsonb object using the **json_object_agg** function. If there are no rows, this function will return *NULL*, which is why the function is surrounded by COALESCE to return an empty JSONB object in that case.

After this, we can turn this into an actual operator by executing the following:

```
CREATE OPERATOR - (
    PROCEDURE = jsonb_minus,
    LEFTARG   = jsonb,
    RIGHTARG  = jsonb )
```

Now you can just use the **-** operator like you would do with hstore. Let's look at the result:
```
SELECT '{"a":1, "b":{"c":123, "d":"test"}}'::jsonb 
	- '{"a":1, "b":{"c":321, "d":"test"}}'::jsonb
```
results in
```
{"b": {"c": 123, "d": "test"}}
```

Great! "a" was removed from arg1 as it had the same value, "b" is still there as it has a different value. But only "b"->"c" is different, yet "b"->"d" is still there. This is because our function only works on the first level of the json object. With a little more effort, we can expand it so that it recursively looks into all the nested properties as well:

```
CREATE OR REPLACE FUNCTION jsonb_minus ( arg1 jsonb, arg2 jsonb )
 RETURNS jsonb
AS $$

SELECT 
	COALESCE(json_object_agg(
        key,
        CASE
            -- if the value is an object and the value of the second argument is
            -- not null, we do a recursion
            WHEN jsonb_typeof(value) = 'object' AND arg2 -> key IS NOT NULL 
			THEN jsonb_minus(value, arg2 -> key)
            -- for all the other types, we just return the value
            ELSE value
        END
    ), '{}')::jsonb
FROM 
	jsonb_each(arg1)
WHERE 
	arg1 -> key <> arg2 -> key 
	OR arg2 -> key IS NULL

$$ LANGUAGE SQL;
```

Instead of just selecting the value, we now check if the value is a json object. If it is, we call jsonb_minus recursively to substract the nested objects from one another.

Executing the example from earlier now gives us:

```
{"b": {"c": 123}}
```

As you can see, the function now also works for nested jsonb properties. 