---
layout: post
title:  "Querying NYC Motor Vehicle Collision data with Apache Drill"
date:   2016-04-15 18:49:54 -0500
categories: drill
---


# Querying NYC Motor Vehicle Collision data with Apache Drill

The New York City government has opened up their data sets in a massive way, and that provides an awesome amount of real-world data to play with.

Today, I want to use some of this data in Drill.

First, I've downloaded the motor vehicle collision data covering July 1 2012 to March 14 2016. In that time, the NYPD collected data on over 1.3 million motor vehicle collisions (wow!). I've downloaded the data in CSV format (using the NYC Open Data website, I chose the “Export” option and selected “CSV for Excel” as the format).

We're using Apache Drill, so you'll need that if you want to follow along. Go to https://drill.apache.org and follow the instructions there to get Drill and up and running in embedded mode on your computer.

# Start Drill in Embedded Mode

If you're running this example from your laptop, you'll want to use Drill's embedded mode. Drill's embedded mode is the quickest way to get a single drillbit up and running for exploration on your computer. 

```
$DRILL_HOME/bin/drill-embedded
```

Once you're in sqlline (the shell that launches when you run `drill-embedded`), you can run the following to see that things are working:

```
select * from cp.`employees.json` limit 10;
```

That's just a simple query against some data that ships embedded in the Drill classpath (hence “cp”). If that works, you're ready to begin.

# Create a Drill Workspace

When you downloaded the data, you put it somewhere on your hard disk, and that somewhere has a path. When we create a workspace we are telling Drill how to access the data at that path, and also giving ourselves a shortcut to use in our queries, rather that specifying the entire path to the file or files we want to query. On my system, I created a directory called `/Users/vince/data/nyc/nypdmvc/raw` and placed the downloaded CSV files there.

Drill provides an HTTP UI and API. We can use these to update the storage configuration. Let's use the API to update our storage plugin configuration. 

But rather than download and editing the JSON file by hand, let's use the [amazing jq](https://stedolan.github.io/jq/) to do the editing for us in one awesome shell pipeline that is sure to impress your friends. Modify the path below to match where you put the data.

```
$ curl -s localhost:8047/storage.json |\
	jq '.[] | select(.name == "dfs") | .config.workspaces |= . + { "nypdmvc": { "location": "/Users/vince/data/nyc/nypdmvc", "writable": true, "defaultInputFormat": null}  }'
```

If that looks good (you should see a new `“nypdmvc”` workspace under the `dfs` plugin, and the `dfs` plugin should be all you see) then you can send the edited `dfs` plugin right back into Drill again using curl to update the configuration with the new workspace:

```
$ curl -s localhost:8047/storage.json |\
	jq '.[] | select(.name == "dfs") | .config.workspaces |= . + { "nypdmvc": { "location": "/Users/vince/data/nyc/nypdmvc", "writable": true, "defaultInputFormat": null}  }' |\
	curl -s -X POST -H "Content-Type: application/json" -d @- http://localhost:8047/storage/dfs.json
```

If you see the following output, we're good to proceed.

```json
{
  “result” : “success”
}
```

# See The Workspaces

Launch sqlline and use `show databases` to see the storage plugins and workspaces:


```
$DRILL_HOME/bin/drill-embedded
INFO: Initiating Jersey application, version Jersey: 2.8 2014-04-29 01:25:26...
apache drill 1.6.0
"a drill in the hand is better than two in the bush"
0: jdbc:drill:zk=local> show databases;
+---------------------+
|     SCHEMA_NAME     |
+---------------------+
| INFORMATION_SCHEMA  |
| cp.default          |
| dfs.default         |
| dfs.nypdmvc         |
| dfs.root            |
| dfs.tmp             |
| sys                 |
+---------------------+
7 rows selected (1.42 seconds)
0: jdbc:drill:zk=local>
```

So far so good. Let's use the dfs.nypdmvc schema:


```
0: jdbc:drill:zk=local> use dfs.nypdmvc;
+-------+------------------------------------------+
|  ok   |                 summary                  |
+-------+------------------------------------------+
| true  | Default schema changed to [dfs.nypdmvc]  |
+-------+------------------------------------------+
1 row selected (0.096 seconds)
0: jdbc:drill:zk=local> show tables;
+---------------+-------------+
| TABLE_SCHEMA  | TABLE_NAME  |
+---------------+-------------+
+---------------+-------------+
No rows selected (0.13 seconds)
```

Note though that when we `show tables` nothing appears. That's because there's just files there. Let's find out what files are there:

```
0: jdbc:drill:zk=local> show files;
+-------+--------------+---------+---------+--------+--------+--------------+------------------------+------------------------+
| name  | isDirectory  | isFile  | length  | owner  | group  | permissions  |       accessTime       |    modificationTime    |
+-------+--------------+---------+---------+--------+--------+--------------+------------------------+------------------------+
| raw   | true         | false   | 136     | vince  | staff  | rwxr-xr-x    | 1969-12-31 19:00:00.0  | 2016-04-19 12:14:44.0  |
+-------+--------------+---------+---------+--------+--------+--------------+------------------------+------------------------+
1 row selected (0.154 seconds)
```

There's a directory called `raw`. What's in it?

```
0: jdbc:drill:zk=local> show files in raw;
+--------------------------------------+--------------+---------+------------+--------+--------+--------------+------------------------+------------------------+
|                 name                 | isDirectory  | isFile  |   length   | owner  | group  | permissions  |       accessTime       |    modificationTime    |
+--------------------------------------+--------------+---------+------------+--------+--------+--------------+------------------------+------------------------+
| NYPD_Motor_Vehicle_Collisions-2.csv  | false        | true    | 149017380  | vince  | staff  | rw-r--r--    | 1969-12-31 19:00:00.0  | 2016-04-19 12:14:44.0  |
| NYPD_Motor_Vehicle_Collisions.csv    | false        | true    | 120319926  | vince  | staff  | rw-r-----    | 1969-12-31 19:00:00.0  | 2015-07-18 00:05:44.0  |
+--------------------------------------+--------------+---------+------------+--------+--------+--------------+------------------------+------------------------+
2 rows selected (0.123 seconds)
```

There's our CSV files. Now let's see what's in those.


# Query the Data

The data is in CSV format inside those files. First thing we usually try in Drill when exploring any data is a `select * from <foo> limit n` query. We do this to get an idea of what the data looks like, and if Drill can query it at all:


```
0: jdbc:drill:zk=local> select * from raw limit 2;
+---------+
| columns |
+---------+
| ["DATE","TIME","BOROUGH","ZIP CODE","LATITUDE","LONGITUDE","LOCATION","ON STREET NAME","CROSS STREET NAME","OFF STREET NAME","NUMBER OF PERSONS INJURED","NUMBER OF PERSONS KILLED","NUMBER OF PEDESTRIANS INJURED","NUMBER OF PEDESTRIANS KILLED","NUMBER OF CYCLIST INJURED","NUMBER OF CYCLIST KILLED","NUMBER OF MOTORIST INJURED","NUMBER OF MOTORIST KILLED","CONTRIBUTING FACTOR VEHICLE 1","CONTRIBUTING FACTOR VEHICLE 2","CONTRIBUTING FACTOR VEHICLE 3","CONTRIBUTING FACTOR VEHICLE 4","CONTRIBUTING FACTOR VEHICLE 5","UNIQUE KEY","VEHICLE TYPE CODE 1","VEHICLE TYPE CODE 2","VEHICLE TYPE CODE 3","VEHICLE TYPE CODE 4","VEHICLE TYPE CODE 5"] |
| ["07/14/2015","7:50","BROOKLYN","11207","40.6781627","-73.8974769","(40.6781627, -73.8974769)","JAMAICA AVENUE","PENNSYLVANIA AVENUE","","0","0","0","0","0","0","0","0","Physical Disability","Unspecified","","","","3258116","PASSENGER VEHICLE","PASSENGER VEHICLE","","",""] |
+---------+
2 rows selected (0.327 seconds)
```

So far so good! We queried the CSV and got back one column called "columns". That column contains an array of column names.

# Create Better Columns

So line 1 of the result set in the last query had the header of the CSV file. We can deal with this in two ways. First, we could use the "select with options" capability of Drill to extract the column names from the first line of the file:

```
0: jdbc:drill:zk=local> select * from table(`raw/NYPD_Motor_Vehicle_Collisions.csv`(type => 'text', extractHeader => true)) limit 2;
+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| DATE,TIME,BOROUGH,ZIP CODE,LATITUDE,LONGITUDE,LOCATION,ON STREET NAME,CROSS STREET NAME,OFF STREET NAME,NUMBER OF PERSONS INJURED,NUMBER OF PERSONS KILLED,NUMBER OF PEDESTRIANS INJURED,NUMBER OF PEDESTRIANS KILLED,NUMBER OF CYCLIST INJURED,NUMBER OF CYCLIST KILLED,NUMBER OF MOTORIST INJURED,NUMBER OF MOTORIST KILLED,CONTRIBUTING FACTOR VEHICLE 1,CONTRIBUTING FACTOR VEHICLE 2,CONTRIBUTING FACTOR VEHICLE 3,CONTRIBUTING FACTOR VEHICLE 4,CONTRIBUTING FACTOR VEHICLE 5,UNIQUE KEY,VEHICLE TYPE CODE 1,VEHICLE TYPE CODE 2,VEHICLE TYPE CODE 3,VEHICLE TYPE CODE 4,VEHICLE TYPE CODE 5 |
+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| 07/14/2015,7:50,BROOKLYN,11207,40.6781627,-73.8974769,"(40.6781627, -73.8974769)",JAMAICA AVENUE,PENNSYLVANIA AVENUE,,0,0,0,0,0,0,0,0,Physical Disability,Unspecified,,,,3258116,PASSENGER VEHICLE,PASSENGER VEHICLE,,, |
| 07/14/2015,7:50,,,40.7639012,-73.9532241,"(40.7639012, -73.9532241)",,,,1,0,0,0,0,0,1,0,Other Vehicular,Other Vehicular,Fatigued/Drowsy,,,3257780,PASSENGER VEHICLE,PASSENGER VEHICLE,PASSENGER VEHICLE,, |
+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
2 rows selected (0.24 seconds)
```

This is great because it saves us the work of having to label the columns ourselves. But what's not so great is that it only works when you name the file explicitly. See what happens when we query the directory in this way:

```
0: jdbc:drill:zk=local> select * from table(`raw`(type => 'text', extractHeader => true)) limit 2;
Error: SYSTEM ERROR: IllegalArgumentException: FileStatus work only works with files, not directories.


[Error Id: fca3d6d2-2529-4b92-9b00-bec5b172b377 on ip-192-168-100-120.ec2.internal:31010] (state=,code=0)
```

If your data is in a single file, you're good to go. If not, you'll need to label the columns yourself.


# Cast to Create Schema

So let's create the columns and also apply schema. Schema is important because if you want to do comparisons or joins, it's good to know what kind of data we're dealing with. Without providing the type information, Drill is forced to infer the types, and that may not get you what you want.

The good news is that we can apply type to a column with `cast` in our query. Let's generate good columns and cast the data in one shot:

```sql
    select 
        cast(to_date(columns[0], 'MM/dd/YYYY') as date) as `date`,
        cast(to_time(columns[1], 'HH:mm') as time) as `time`,
        cast(columns[2] as varchar(40)) as borough,
        cast(columns[3] as varchar(10)) as zipcode,
        CASE WHEN columns[4] in ('', 'LATITUDE') THEN NULL ELSE cast(columns[4] as double) END as latitude,
        CASE WHEN columns[5] in ('', 'LONGITUDE') THEN NULL ELSE cast(columns[5] as double) END as longitude,
        cast(columns[7] as varchar(80)) as on_street_name,
        cast(columns[8] as varchar(80)) as cross_street_name,
        cast(columns[9] as varchar(80)) as off_street_name,
        cast(columns[10] as int) as persons_injured,
        cast(columns[11] as int) as persons_killed,
        cast(columns[12] as int) as pedestrians_injured,
        cast(columns[13] as int) as pedestrians_killed,
        cast(columns[14] as int) as cyclist_injured,
        cast(columns[15] as int) as cyclist_killed,
        cast(columns[16] as int) as motorist_injured,
        cast(columns[17] as int) as motorist_killed,
        cast(columns[18] as varchar(80)) as factor_vehicle1,
        cast(columns[19] as varchar(80)) as factor_vehicle2,
        cast(columns[20] as varchar(80)) as factor_vehicle3,
        cast(columns[21] as varchar(80)) as factor_vehicle4,
        cast(columns[22] as varchar(80)) as factor_vehicle5,
        cast(columns[23] as bigint) as key,
        cast(columns[24] as varchar(80)) as vehicle1_type,
        cast(columns[25] as varchar(80)) as vehicle2_type,
        cast(columns[26] as varchar(80)) as vehicle3_type,
        cast(columns[27] as varchar(80)) as vehicle4_type,
        cast(columns[28] as varchar(80)) as vehicle5_type
    from raw;
```

But when we run this query, we get an error!

```
Error: SYSTEM ERROR: IllegalArgumentException: Invalid format: "DATE"
```

Remember our header column? It can't be parsed as a DATE type, so Drill throws an exception. What we need to do is skip the header line, since it doesn't contain data we care about.

```sql
    select 
        cast(to_date(columns[0], 'MM/dd/YYYY') as date) as `date`,
        cast(to_time(columns[1], 'HH:mm') as time) as `time`,
        cast(columns[2] as varchar(40)) as borough,
        cast(columns[3] as varchar(10)) as zipcode,
        CASE WHEN columns[4] in ('', 'LATITUDE') THEN NULL ELSE cast(columns[4] as double) END as latitude,
        CASE WHEN columns[5] in ('', 'LONGITUDE') THEN NULL ELSE cast(columns[5] as double) END as longitude,
        cast(columns[7] as varchar(80)) as on_street_name,
        cast(columns[8] as varchar(80)) as cross_street_name,
        cast(columns[9] as varchar(80)) as off_street_name,
        cast(columns[10] as int) as persons_injured,
        cast(columns[11] as int) as persons_killed,
        cast(columns[12] as int) as pedestrians_injured,
        cast(columns[13] as int) as pedestrians_killed,
        cast(columns[14] as int) as cyclist_injured,
        cast(columns[15] as int) as cyclist_killed,
        cast(columns[16] as int) as motorist_injured,
        cast(columns[17] as int) as motorist_killed,
        cast(columns[18] as varchar(80)) as factor_vehicle1,
        cast(columns[19] as varchar(80)) as factor_vehicle2,
        cast(columns[20] as varchar(80)) as factor_vehicle3,
        cast(columns[21] as varchar(80)) as factor_vehicle4,
        cast(columns[22] as varchar(80)) as factor_vehicle5,
        cast(columns[23] as bigint) as key,
        cast(columns[24] as varchar(80)) as vehicle1_type,
        cast(columns[25] as varchar(80)) as vehicle2_type,
        cast(columns[26] as varchar(80)) as vehicle3_type,
        cast(columns[27] as varchar(80)) as vehicle4_type,
        cast(columns[28] as varchar(80)) as vehicle5_type
    from raw where columns[0] not like '%DATE%';
```

This query is getting a bit long, so let's make it a bit easier to deal with now that we can apply some schema.

# Create a View

Creating a view will not only give us a shorthand to refer to the query, but it will also make the data visible as a table to traditional ODBC and JDBC tools.

Let's create the view:

```sql
create or replace view dfs.nypdmvc.nypdmvc_vw as 
    select 
        cast(to_date(columns[0], 'MM/dd/YYYY') as date) as `date`,
        cast(to_time(columns[1], 'HH:mm') as time) as `time`,
        cast(columns[2] as varchar(40)) as borough,
        cast(columns[3] as varchar(10)) as zipcode,
        CASE WHEN columns[4] in ('', 'LATITUDE') THEN NULL ELSE cast(columns[4] as double) END as latitude,
        CASE WHEN columns[5] in ('', 'LONGITUDE') THEN NULL ELSE cast(columns[5] as double) END as longitude,
        cast(columns[7] as varchar(80)) as on_street_name,
        cast(columns[8] as varchar(80)) as cross_street_name,
        cast(columns[9] as varchar(80)) as off_street_name,
        cast(columns[10] as int) as persons_injured,
        cast(columns[11] as int) as persons_killed,
        cast(columns[12] as int) as pedestrians_injured,
        cast(columns[13] as int) as pedestrians_killed,
        cast(columns[14] as int) as cyclist_injured,
        cast(columns[15] as int) as cyclist_killed,
        cast(columns[16] as int) as motorist_injured,
        cast(columns[17] as int) as motorist_killed,
        cast(columns[18] as varchar(80)) as factor_vehicle1,
        cast(columns[19] as varchar(80)) as factor_vehicle2,
        cast(columns[20] as varchar(80)) as factor_vehicle3,
        cast(columns[21] as varchar(80)) as factor_vehicle4,
        cast(columns[22] as varchar(80)) as factor_vehicle5,
        cast(columns[23] as bigint) as key,
        cast(columns[24] as varchar(80)) as vehicle1_type,
        cast(columns[25] as varchar(80)) as vehicle2_type,
        cast(columns[26] as varchar(80)) as vehicle3_type,
        cast(columns[27] as varchar(80)) as vehicle4_type,
        cast(columns[28] as varchar(80)) as vehicle5_type
    from dfs.nypdmvc.raw
        where columns[0] not like '%DATE%';
```



