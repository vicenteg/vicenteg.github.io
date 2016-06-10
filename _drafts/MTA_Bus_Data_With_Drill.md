
# Querying NYC MTA Bus Data with Drill

Using Apache Drill 1.6.0 embedded mode on Mac OS, let's get a data set prepared to query.

We will:

* Do some initial exploration of the data
* Create a workspace
* Provide some type information for the columns
* Filter out rows with invalid data

# Get the Data

Original MTA Data set (compressed with xz):

```
$ aws s3 sync s3://MTABusTime/AppQuest3 .
```




# First Query

The file contains pretty clean, tab-delimited data. So we should be able to start up Drill and query it. 

```
$ /opt/apache-drill-1.6.0/bin/drill-embedded
apache drill 1.6.0
"a drill in the hand is better than two in the bush"
0: jdbc:drill:zk=local> select * from dfs.`/Users/vince/Downloads/MTA-Bus-Time_.2014-10-31.txt` limit 5;
May 06, 2016 8:10:06 AM org.apache.calcite.sql.validate.SqlValidatorException <init>
SEVERE: org.apache.calcite.sql.validate.SqlValidatorException: Table 'dfs./Users/vince/Downloads/MTA-Bus-Time_.2014-10-31.txt' not found
May 06, 2016 8:10:06 AM org.apache.calcite.runtime.CalciteException <init>
SEVERE: org.apache.calcite.runtime.CalciteContextException: From line 1, column 15 to line 1, column 17: Table 'dfs./Users/vince/Downloads/MTA-Bus-Time_.2014-10-31.txt' not found
Error: VALIDATION ERROR: From line 1, column 15 to line 1, column 17: Table 'dfs./Users/vince/Downloads/MTA-Bus-Time_.2014-10-31.txt' not found

SQL Query null

[Error Id: 8ffeb904-f301-4756-81b7-3330d2e2f023 on ip-192-168-56-1.ec2.internal:31010] (state=,code=0)
```

But the first query fails because the file has an extension that Drill does not understand. The error message is perhaps a little deceiving, since it leads you to believe your file does not exist. But it does in my case. The way around this one would be to use a table function. 

```
0: jdbc:drill:zk=local> select * from table(dfs.`/Users/vince/Downloads/MTA-Bus-Time_.2014-10-31.txt`(type => 'text', fieldDelimiter => '\t')) limit 5;
Error: PARSE ERROR: Expected single character but was String: \t

table /Users/vince/Downloads/MTA-Bus-Time_.2014-10-31.txt
parameter fieldDelimiter
SQL Query null

[Error Id: f2145855-19e5-456e-8f71-62fa201b9547 on ip-192-168-100-120.ec2.internal:31010] (state=,code=0)
```

This one fails too! This appears to be a bug. So I filed [DRILL-4658](https://issues.apache.org/jira/browse/DRILL-4658).


# Workarounds!

So we have a few issues. We can't use a table function because DRILL-4658 doesn't allow us to specify a tab delimiter. The simplest thing to do here is rename the file so it has an extension that Drill can deal with. In the storage plugin configuration for `dfs` we can look at the supported formats and find `tsv` in the list, which looks like this (obtained with `$ curl -s localhost:8047/storage.json | jq '.[] | select(.name=="dfs") | .config.formats.tsv'`):

```json
{
  "type": "text",
  "extensions": [
    "tsv"
  ],
  "delimiter": "\t"
}
```

So after changing the extension of the file to `.tsv` let's see what happens:

```
0: jdbc:drill:zk=local> select * from dfs.`/Users/vince/Downloads/MTA-Bus-Time_.2014-10-31.tsv` limit 5;
+---------+
| columns |
+---------+
| ["latitude","longitude","time_received","vehicle_id","distance_along_trip","inferred_direction_id","inferred_phase","inferred_route_id","inferred_trip_id","next_scheduled_stop_distance","next_scheduled_stop_id"] |
| ["40.640498","-73.978939","2014-10-31 04:00:01","468","325.451653058215","0","IN_PROGRESS","MTA NYCT_B67","MTA NYCT_JG_D4-Weekday-SDon-145600_B68_135","20.996880842467363","MTA_306549"] |
| ["40.640011","-73.967389","2014-10-31 04:00:01","576","9909.15874003493","1","IN_PROGRESS","MTABC_B103","MTABC_7010793-SCPD4-SC_D4-Weekday-99-SDon","33.85347446514788","MTA_350076"] |
| ["40.748098","-73.987561","2014-10-31 04:00:01","4212","325.92143512069975","0","IN_PROGRESS","MTA NYCT_Q32","MTA NYCT_CS_D4-Weekday-SDon-143200_MISC_831","220.33496140421818","MTA_400555"] |
| ["40.645810","-73.902253","2014-10-31 04:00:01","4847","2205.762613639745","0","IN_PROGRESS","MTA NYCT_B42","MTA NYCT_EN_D4-Weekday-SDon-001000_B42_1","14.173806664634867","MTA_303345"] |
+---------+
5 rows selected (1.558 seconds)
```

Good, now we have some data!

# Create columns

When we `select *` against a delimited file, by default we get one column back called `columns`. We want a table. We have a couple of options to get there.

Option #1 would be to use a table function to add the `extractHeader` option:

```
0: jdbc:drill:zk=local> select * from table(dfs.`/Users/vince/Downloads/MTA-Bus-Time_.2014-10-31.tsv`(type => 'text', extractHeader => true)) limit 5;
+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| latitude  longitude   time_received   vehicle_id  distance_along_trip inferred_direction_id   inferred_phase  inferred_route_id   inferred_trip_id    next_scheduled_stop_distance    next_scheduled_stop_id |
+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| 40.640498 -73.978939  2014-10-31 04:00:01 468 325.451653058215    0   IN_PROGRESS MTA NYCT_B67    MTA NYCT_JG_D4-Weekday-SDon-145600_B68_135  20.996880842467363  MTA_306549 |
| 40.640011 -73.967389  2014-10-31 04:00:01 576 9909.15874003493    1   IN_PROGRESS MTABC_B103  MTABC_7010793-SCPD4-SC_D4-Weekday-99-SDon   33.85347446514788   MTA_350076 |
| 40.748098 -73.987561  2014-10-31 04:00:01 4212    325.92143512069975  0   IN_PROGRESS MTA NYCT_Q32    MTA NYCT_CS_D4-Weekday-SDon-143200_MISC_831 220.33496140421818  MTA_400555 |
| 40.645810 -73.902253  2014-10-31 04:00:01 4847    2205.762613639745   0   IN_PROGRESS MTA NYCT_B42    MTA NYCT_EN_D4-Weekday-SDon-001000_B42_1    14.173806664634867  MTA_303345 |
| 40.734429 -73.989743  2014-10-31 04:00:01 5301    2983.9132735649764  1   IN_PROGRESS MTA NYCT_M14    MTA NYCT_MQ_D4-Weekday-SDon-141500_M14AD_69 232.24727241309301  MTA_403893 |
+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
5 rows selected (0.291 seconds)
```

At first glance, this looks like what we want, but looking closer, we see that not specifying a fieldDelimiter gave us just one column - clearly Drill needs the fieldDelimiter in this case. But we know specifying it will cause the query to fail, due to DRILL-4658. So we move on to Option #2; naming the columns ourselves by extracting each of the columns.

This is not a very wide table, so this isn't that painful:

```
0: jdbc:drill:zk=local> select columns[0] latitude, columns[1] longitude, columns[2] time_received, columns[3] vehicle_id, columns[4] distance_along_trip, columns[5] inferred_direction_id, columns[6] inferred_phase, columns[7] inferred_route_id, columns[8] inferred_trip_id, columns[9] next_scheduled_stop_distance, columns[10] next_scheduled_stop_id from dfs.`/Users/vince/Downloads/MTA-Bus-Time_.2014-10-31.tsv` limit 5;
+------------+-------------+----------------------+-------------+----------------------+------------------------+-----------------+--------------------+----------------------------------------------+-------------------------------+-------------------------+
|  latitude  |  longitude  |    time_received     | vehicle_id  | distance_along_trip  | inferred_direction_id  | inferred_phase  | inferred_route_id  |               inferred_trip_id               | next_scheduled_stop_distance  | next_scheduled_stop_id  |
+------------+-------------+----------------------+-------------+----------------------+------------------------+-----------------+--------------------+----------------------------------------------+-------------------------------+-------------------------+
| latitude   | longitude   | time_received        | vehicle_id  | distance_along_trip  | inferred_direction_id  | inferred_phase  | inferred_route_id  | inferred_trip_id                             | next_scheduled_stop_distance  | next_scheduled_stop_id  |
| 40.640498  | -73.978939  | 2014-10-31 04:00:01  | 468         | 325.451653058215     | 0                      | IN_PROGRESS     | MTA NYCT_B67       | MTA NYCT_JG_D4-Weekday-SDon-145600_B68_135   | 20.996880842467363            | MTA_306549              |
| 40.640011  | -73.967389  | 2014-10-31 04:00:01  | 576         | 9909.15874003493     | 1                      | IN_PROGRESS     | MTABC_B103         | MTABC_7010793-SCPD4-SC_D4-Weekday-99-SDon    | 33.85347446514788             | MTA_350076              |
| 40.748098  | -73.987561  | 2014-10-31 04:00:01  | 4212        | 325.92143512069975   | 0                      | IN_PROGRESS     | MTA NYCT_Q32       | MTA NYCT_CS_D4-Weekday-SDon-143200_MISC_831  | 220.33496140421818            | MTA_400555              |
| 40.645810  | -73.902253  | 2014-10-31 04:00:01  | 4847        | 2205.762613639745    | 0                      | IN_PROGRESS     | MTA NYCT_B42       | MTA NYCT_EN_D4-Weekday-SDon-001000_B42_1     | 14.173806664634867            | MTA_303345              |
+------------+-------------+----------------------+-------------+----------------------+------------------------+-----------------+--------------------+----------------------------------------------+-------------------------------+-------------------------+
5 rows selected (0.627 seconds)
```

Okay, good enough. Now we have nice columns with nice names. But we still have the header row in there with the data (see the first line of the output above).

Here again I'd like to use a table function with the `skipFirstLine` but DRILL-4658 haunts me.

Time to punt, and create a new workspace for this dataset. Maybe this is what I should have done from the start.

# A New Workspace

One downside of this approach is that if you're a regular user on a distributed Drill setup, you may not have the privileges to do what follows. But if you're running in embedded mode, or can get your Drill admins to do this for you, you can create a new workspace. We'll call this one "mtabus".

Under the "workspaces" dictionary, add a new map that looks like this:

```
    "mta": {
      "location": "/Users/vince/data/nyc/mta",
      "writable": true,
      "defaultInputFormat": "tsvh"
    }
```

Note the `defaultInputFormat`. `tsvh` isn't a thing yet. So similar to the `csvh` input format, let's create a `tsvh` one that extracts headers for us automatically. Under the `formats` dictionary, add:
 
```
    "tsvh": {
      "type": "text",
      "extensions": [
        "tsv"
      ],
      "extractHeader": true,
      "delimiter": "\t"
    }
```

Now, move the MTA file to a directory in your new workspace, creating directories as necessary. Also, rename the file to have our new `.tsvh` extension. My tree looks like this after I moved the renamed data file to `~/data/nyc/mta/bustime`:

```
$ tree /Users/vince/data/nyc/mta/
/Users/vince/data/nyc/mta/
└── bustime
    └── MTA-Bus-Time_.2014-10-31.tsvh
```

Now we have a workspace where we can group related datasets - for example, if we had other data from the MTA, we could create a directory alongside `bustime` and put that data in there.

The mta workspace is configured with the `tsvh` input format, so it knows how to handle the data. Now, we can switch to that workspace (or schema) and list the files:
 
```
0: jdbc:drill:zk=local> use dfs.mta;
+-------+--------------------------------------+
|  ok   |               summary                |
+-------+--------------------------------------+
| true  | Default schema changed to [dfs.mta]  |
+-------+--------------------------------------+
1 row selected (0.102 seconds)
0: jdbc:drill:zk=local> show files;
+----------+--------------+---------+---------+--------+--------+--------------+------------------------+------------------------+
|   name   | isDirectory  | isFile  | length  | owner  | group  | permissions  |       accessTime       |    modificationTime    |
+----------+--------------+---------+---------+--------+--------+--------------+------------------------+------------------------+
| bustime  | true         | false   | 102     | vince  | staff  | rwxr-xr-x    | 1969-12-31 19:00:00.0  | 2016-05-06 10:57:15.0  |
+----------+--------------+---------+---------+--------+--------+--------------+------------------------+------------------------+
1 row selected (0.145 seconds)
0: jdbc:drill:zk=local> show files in bustime;
+--------------------------------+--------------+---------+------------+--------+--------+--------------+------------------------+------------------------+
|              name              | isDirectory  | isFile  |   length   | owner  | group  | permissions  |       accessTime       |    modificationTime    |
+--------------------------------+--------------+---------+------------+--------+--------+--------------+------------------------+------------------------+
| MTA-Bus-Time_.2014-10-31.tsvh  | false        | true    | 878127732  | vince  | staff  | rw-r--r--    | 1969-12-31 19:00:00.0  | 2016-05-05 17:02:17.0  |
+--------------------------------+--------------+---------+------------+--------+--------+--------------+------------------------+------------------------+
1 row selected (0.142 seconds)
```

And we don't need to query the individual file anymore; we can query the top level directory that holds the file:

```
0: jdbc:drill:zk=local> select * from bustime limit 2;
+------------+-------------+----------------------+-------------+----------------------+------------------------+-----------------+--------------------+---------------------------------------------+-------------------------------+-------------------------+
|  latitude  |  longitude  |    time_received     | vehicle_id  | distance_along_trip  | inferred_direction_id  | inferred_phase  | inferred_route_id  |              inferred_trip_id               | next_scheduled_stop_distance  | next_scheduled_stop_id  |
+------------+-------------+----------------------+-------------+----------------------+------------------------+-----------------+--------------------+---------------------------------------------+-------------------------------+-------------------------+
| 40.640498  | -73.978939  | 2014-10-31 04:00:01  | 468         | 325.451653058215     | 0                      | IN_PROGRESS     | MTA NYCT_B67       | MTA NYCT_JG_D4-Weekday-SDon-145600_B68_135  | 20.996880842467363            | MTA_306549              |
| 40.640011  | -73.967389  | 2014-10-31 04:00:01  | 576         | 9909.15874003493     | 1                      | IN_PROGRESS     | MTABC_B103         | MTABC_7010793-SCPD4-SC_D4-Weekday-99-SDon   | 33.85347446514788             | MTA_350076              |
+------------+-------------+----------------------+-------------+----------------------+------------------------+-----------------+--------------------+---------------------------------------------+-------------------------------+-------------------------+
2 rows selected (0.203 seconds)
```

Notice that the inputFormat configuration we provided before now has Drill extracting the headers for us.

# Add Type Data

Schema in Drill is optional. That is, you don't need to have explicit type information in order to query your data. But schema can be very helpful for performance, storage and comparisons. Providing Drill information about the types of data in the columns of your data gives it the ability to store the data more efficiently in memory and allows it to do the most correct comparisons.

As good as our data looks now, Drill doesn't really know what kind of data each column holds:

```
0: jdbc:drill:schema=dfs.mta> describe bustime;
+--------------+------------+--------------+
| COLUMN_NAME  | DATA_TYPE  | IS_NULLABLE  |
+--------------+------------+--------------+
+--------------+------------+--------------+
No rows selected (0.156 seconds)
```

This is not a problem for basic exploration, but once we start to really query the data we can see the problem:

With no type on the `vehicle_id`  column:

```
0: jdbc:drill:schema=dfs.mta> select min(vehicle_id) from bustime;
+---------+
| EXPR$0  |
+---------+
| 1001    |
+---------+
1 row selected (3.238 seconds)
```

If we cast the column to an INT though:

```
0: jdbc:drill:schema=dfs.mta> select min(cast(vehicle_id as INT)) from bustime;
+---------+
| EXPR$0  |
+---------+
| 101     |
+---------+
1 row selected (3.248 seconds)
```

This might make a difference if we're comparing buses based on vehicle IDs and not expecting lexicographic sorting.

So let's go ahead and apply some types to our columns using the `CAST` function:

```sql
select
    cast(latitude as float) latitude,
    cast(longitude as float) longitude,
    cast(to_timestamp(time_received, 'YYYY-MM-dd HH:mm:ss') as timestamp) time_received,
    cast(vehicle_id as int) as vehicle_id,
    cast(distance_along_trip as float) distance_along_trip,
    case when inferred_direction_id not in ('NULL', '') then cast(inferred_direction_id as int) end as inferred_direction_id,
    cast(inferred_phase as VARCHAR(20)) inferred_phase,
    cast(inferred_route_id as VARCHAR(20)) inferred_route_id,
    cast(inferred_trip_id as VARCHAR(30)) inferred_trip_id,
    case when next_scheduled_stop_distance not in ('NULL', '') then cast(next_scheduled_stop_distance as float) end next_scheduled_stop_distance,
    cast(next_scheduled_stop_id as VARCHAR(20)) as next_scheduled_stop_id
  from dfs.mta.bustime limit 5;
```

 When casting a number of columns, it's a good idea to test each column individually. Text formats often have dirty data, and these can cause casts to fail deep within your dataset. As a result, casting the data can be a very iterative process. I'd suggest you try a cast over a one column at a time in a sample of the dataset, observe the results, and proceed. If you hit exceptions with a cast, you may need to use case statements to work around bad values, as in the cast of the `next_scheduled_stop_distance` column above.

Having typed the data, let's create a view.

# Creating a View

```sql
create or replace view dfs.mta.bustime_vw as
    select
        cast(latitude as float) latitude,
        cast(longitude as float) longitude,
        cast(to_timestamp(time_received, 'YYYY-MM-dd HH:mm:ss') as timestamp) time_received,
        cast(vehicle_id as int) as vehicle_id,
        cast(distance_along_trip as float) distance_along_trip,
        case when inferred_direction_id not in ('NULL', '') then cast(inferred_direction_id as int) end as inferred_direction_id,
        cast(inferred_phase as VARCHAR(20)) inferred_phase,
        cast(inferred_route_id as VARCHAR(20)) inferred_route_id,
        cast(inferred_trip_id as VARCHAR(30)) inferred_trip_id,
        case when next_scheduled_stop_distance not in ('NULL', '') then cast(next_scheduled_stop_distance as float) end next_scheduled_stop_distance,
        cast(next_scheduled_stop_id as VARCHAR(20)) as next_scheduled_stop_id
      from dfs.mta.bustime where latitude > 0.0;
```


# Count the Data with Valid Lat/Long

# Find the buses with the most distinct trips

Top ten buses by the number of different trip IDs.

```
select vehicle_id,count(distinct inferred_trip_id) trips from bustime_vw group by vehicle_id order by trips desc limit 10;
```

# Create a view of the above

# Convert to Parquet

This is really easy. Just make sure your workspace is writable.

```
create table bustime_parquet as select * from bustime_vw;
```

Check out the difference in size:

```
$ du -hs bustime bustime_parquet/
838M    bustime
182M    bustime_parquet/
```

And the difference in performance:

```
$ sqlline -u jdbc:drill:schema=dfs.mta
apache drill 1.6.0
"got drill?"
0: jdbc:drill:schema=dfs.mta> select vehicle_id,count(distinct inferred_trip_id) trips from bustime_vw group by vehicle_id order by trips desc limit 10;
+-------------+--------+
| vehicle_id  | trips  |
+-------------+--------+
| 184         | 67     |
| 3521        | 61     |
| 164         | 61     |
| 3556        | 48     |
| 174         | 48     |
| 5543        | 45     |
| 599         | 44     |
| 3622        | 43     |
| 180         | 42     |
| 3616        | 40     |
+-------------+--------+
10 rows selected (6.836 seconds)
0: jdbc:drill:schema=dfs.mta> select vehicle_id,count(distinct inferred_trip_id) trips from bustime_parquet group by vehicle_id order by trips desc limit 10;
+-------------+--------+
| vehicle_id  | trips  |
+-------------+--------+
| 184         | 67     |
| 3521        | 61     |
| 164         | 61     |
| 174         | 48     |
| 3556        | 48     |
| 5543        | 45     |
| 599         | 44     |
| 3622        | 43     |
| 180         | 42     |
| 3616        | 40     |
+-------------+--------+
10 rows selected (1.543 seconds)
```


