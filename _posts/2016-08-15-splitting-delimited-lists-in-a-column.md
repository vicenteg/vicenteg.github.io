---
layout: "post"
title:  "Splitting nested strings in JSON in Apache Drill"
date:   2016-08-15 17:50:30 -0500
categories: "drill"
tags: "drill"
---

What do you do when you get a comma-delimited list in a column of some data you want to query in drill?

Your JSON data might look something like this:

```json
{ "key": "chipmunks", "value": "alvin,simon,theodore" }
{ "key": "turtles", "value": "raphael,donatello,leonardo,michelangelo"}
```

When we `select *` this, we get:

```
0: jdbc:drill:> select * from dfs.`/Users/vince/data/cartoons.json`;
+------------+------------------------------------------+
|    key     |                  value                   |
+------------+------------------------------------------+
| chipmunks  | alvin,simon,theodore                     |
| turtles    | raphael,donatello,leonardo,michelangelo  |
+------------+------------------------------------------+
2 rows selected (0.341 seconds)
```

The value column is string, but it looks kinda like a list. Can we `FLATTEN` it?

```
0: jdbc:drill:> select FLATTEN(t.`value`) from dfs.`/Users/vince/data/cartoons.json` t;
Error: SYSTEM ERROR: ClassCastException: Cannot cast org.apache.drill.exec.vector.NullableVarCharVector to org.apache.drill.exec.vector.complex.RepeatedValueVector

Fragment 0:0

[Error Id: 2301eea8-40a8-4d87-ae9d-801a9c126189 on 172.30.1.187:31010] (state=,code=0)
```

No. It's just a string. And Drill tells you so - it can't cast a NullableVarCharVector (string!) to a RepeatedValueVector (list!).

BTW, note the backticks around `value`. Shame on me for choosing a reserved word for a column name. Carrying on.

Drill lacks a function to split a string on a delimiter into multiple columns. And what we want to get to is a table listing like this:

```
+------------+---------------+
|   animal   |     name      |
+------------+---------------+
| chipmunks  | alvin         |
| chipmunks  | simon         |
| chipmunks  | theodore      |
| turtles    | raphael       |
| turtles    | donatello     |
| turtles    | leonardo      |
| turtles    | michelangelo  |
+------------+---------------+
```

Once we reshape the data in this way, we can do aggregations and groupings to our hearts content, and finally learn just how many anthropomorphic animals there were in these cartoons.

We could do some wrangling of the data before it is stored to turn the string into a JSON list, which would make it `FLATTEN`able by drill. To do that in a script would be simple - split the string on the delimiter, and encode the resulting list as JSON as in the following Python snippet:

```python
split_string = input_column.split(",")
json.dumps(split_string)
```

Easy enough, but now you have extra code to maintain. And you'll then have to work out what to do about some (totally solvable) problems like how to deal with failures, how to avoid querying partially written files, and so on. It would be better if we could just leave the data as is, and keep things simple. Maybe we can solve this in SQL instead?

In drill we can use a few functions together the get the desired effect. What we do is turn the comma delimited string into a valid JSON string, then use Drill's `CONVERT_FROM` function to parse that JSON into a list. Once we have a list, we can `FLATTEN` it to get the structure we want.

Let's build the query we need to get there. First, let's use `REGEXP_REPLACE` to quote all the list elements. This is the first step to getting a valid JSON list:

```
0: jdbc:drill:> select REGEXP_REPLACE(t.`value`, '([\w-_]+)', '"$1"') from dfs.`/Users/vince/data/cartoons.json` t;
+--------------------------------------------------+
|                      EXPR$0                      |
+--------------------------------------------------+
| "alvin","simon","theodore"                       |
| "raphael","donatello","leonardo","michelangelo"  |
+--------------------------------------------------+
2 rows selected (1.393 seconds)
```

Next, let's use the `CONCAT` function to add square brackets to add the syntax we need to connote a list:

```
0: jdbc:drill:> select CONCAT('[', REGEXP_REPLACE(t.`value`, '([\w-_]+)', '"$1"'), ']') from dfs.`/Users/vince/data/cartoons.json` t;
+----------------------------------------------------+
|                       EXPR$0                       |
+----------------------------------------------------+
| ["alvin","simon","theodore"]                       |
| ["raphael","donatello","leonardo","michelangelo"]  |
+----------------------------------------------------+
2 rows selected (1.81 seconds)
```

Finally, we wrap that whole mess in a `CONVERT_FROM` to parse the JSON:

```
0: jdbc:drill:> select CONVERT_FROM(CONCAT('[', REGEXP_REPLACE(t.`value`, '([\w-_]+)', '"$1"'), ']'), 'JSON') from dfs.`/Users/vince/data/cartoons.json` t;
+----------------------------------------------------+
|                       EXPR$0                       |
+----------------------------------------------------+
| ["alvin","simon","theodore"]                       |
| ["raphael","donatello","leonardo","michelangelo"]  |
+----------------------------------------------------+
2 rows selected (1.665 seconds)
```

It looks the same, but it's not. That last output is a proper list that we can `FLATTEN`!

Let's `FLATTEN` it, and see what it looks like, with some column aliases to make it nice:

```
0: jdbc:drill:> select t.`key` animal,FLATTEN(CONVERT_FROM(CONCAT('[', REGEXP_REPLACE(t.`value`, '([\w-_]+)', '"$1"'), ']'), 'JSON')) name from dfs.`/Users/vince/data/cartoons.json` t;
+------------+---------------+
|   animal   |     name      |
+------------+---------------+
| chipmunks  | alvin         |
| chipmunks  | simon         |
| chipmunks  | theodore      |
| turtles    | raphael       |
| turtles    | donatello     |
| turtles    | leonardo      |
| turtles    | michelangelo  |
+------------+---------------+
7 rows selected (1.86 seconds)
```

Nice indeed!

And a simple subselect now to do our aggregation:

```
0: jdbc:drill:> select animal,count(name) from (select t.`key` animal,FLATTEN(CONVERT_FROM(CONCAT('[', REGEXP_REPLACE(t.`value`, '([\w-_]+)', '"$1"'), ']'), 'JSON')) name from dfs.`/Users/vince/data/cartoons.json` t) group by animal;
+------------+---------+
|   animal   | EXPR$1  |
+------------+---------+
| chipmunks  | 3       |
| turtles    | 4       |
+------------+---------+
2 rows selected (5.451 seconds)
```

Cowabunga!
