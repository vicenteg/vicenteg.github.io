---
layout: post
title:  "Ansible Logs and High Resolution Times in Drill"
date:   2016-05-02 10:55:23 -0500
categories: ansible drill logs
---

Ansible is a tool for automation. I use it for lots of stuff, and sometimes it would be useful to compare runs of the same playbooks, particularly runtime of things like performance tests. So I need to make ansible log what it's doing, and then I want to be able to query the logs. Ideally, I want to do this with minimal fuss.

With a simple script called a callback plugin, Ansible can produce JSON logs. With the log files in JSON format, I can query the logs with Drill.

Here's what some of the raw data looks like:

```json
{
  "changed": true,
  "cmd": "hadoop jar /opt/mapr/lib/maprfs-diagnostic-tools-*.jar com.mapr.fs.RWSpeedTest /benchmarks/localhost.localdomain/rwspeedtest 5120 maprfs:/// > /tmp/rwspeedtest-output",
  "delta": "0:00:08.517780",
  "end": "2016-05-01 12:00:56.107679",
  "host": "maprsandbox",
  "invocation": {
    "module_args": "hadoop jar /opt/mapr/lib/maprfs-diagnostic-tools-*.jar com.mapr.fs.RWSpeedTest /benchmarks/localhost.localdomain/rwspeedtest 5120 maprfs:/// > /tmp/rwspeedtest-output",
    "module_complex_args": {},
    "module_name": "shell"
  },
  "item": 5120,
  "pid": 14962,
  "play": "vicenteg.mapr_post_install_test",
  "rc": 0,
  "start": "2016-05-01 12:00:47.589899",
  "stderr": "16/05/01 12:00:48 INFO Configuration.deprecation: fs.default.name is deprecated. Instead, use fs.defaultFS",
  "stdout": "",
  "task": "run RWSpeedTest, 5GB data per node",
  "warnings": []
}
```

An exploratory query of this data with Drill 1.6 throws an error:

```
0: jdbc:drill:zk=local> select * from dfs.`/Users/vince/src/logs` ;
Error: DATA_READ ERROR: Error parsing JSON - You tried to start when you are using a ValueWriter of type NullableVarCharWriterImpl.

File  /Users/vince/src/logs/vicenteg.mapr_post_install_test/2016/05/02/1462204256-18277.json
Record  7
Fragment 0:0

[Error Id: 9e9efe35-1c5f-4367-9688-de7b48105073 on 172.30.1.146:31010] (state=,code=0)
```

We can enable experimental union type support and try again:

```
0: jdbc:drill:zk=local> ALTER SESSION SET `exec.enable_union_type` = true;
+-------+----------------------------------+
|  ok   |             summary              |
+-------+----------------------------------+
| true  | exec.enable_union_type updated.  |
+-------+----------------------------------+
1 row selected (0.07 seconds)
0: jdbc:drill:zk=local> select * from dfs.`/Users/vince/src/logs` ;
+------+------+------+------+---------+------+------------+-----+-----+------+----+---------+------+-------+-----+--------+--------------------+-------+--------+--------+--------------+----------+------+---------+------+------+-----+
| dir0 | dir1 | dir2 | dir3 | changed | host | invocation | msg | pid | play | rc | results | task | delta | end | failed | failed_when_result | start | stderr | stdout | stdout_lines | warnings | name | volumes | item | stat | cmd |
+------+------+------+------+---------+------+------------+-----+-----+------+----+---------+------+-------+-----+--------+--------------------+-------+--------+--------+--------------+----------+------+---------+------+------+-----+
| vicenteg.mapr_post_install_test | 2016 | 05 | 02 | false | maprsandbox | {"module_args":"name=python-requests state=present","module_complex_args":{},"module_name":"yum"} |  | 18277 | vicenteg.mapr_post_install_test | 0 | ["python-requests-2.6.0-3.el6.noarch providing python-requests is already installed"] | install python-requests | null | null | null | null | null | null | null | [] | [] | null | [] | null | {} | null |
| vicenteg.mapr_post_install_test | 2016 | 05 | 02 | false | maprsandbox | {"module_args":"hadoop mfs -lsd /benchmarks | egrep /benchmarks | egrep \"^v\"","module_complex_args":{},"module_name":"shell"} | null | 18277 | vicenteg.mapr_post_install_test | 0 | [] | check whether /benchmarks is a volume | 0:00:02.322755 | 2016-05-01 14:17:45.849982 | false | false | 2016-05-01 14:17:43.527227 |  | vrwxr-xr-x  Z U U   - mapr shadow          1 2016-05-01 10:40  268435456 /benchmarks | ["vrwxr-xr-x  Z U U   - mapr shadow          1 2016-05-01 10:40  268435456 /benchmarks"] | [] | null | null | null | {} | hadoop mfs -lsd /benchmarks | egrep /benchmarks | egrep "^v" |
| vicenteg.mapr_post_install_test | 2016 | 05 | 02 | false | maprsandbox | {"module_args":"name=benchmarks path=/benchmarks replication=1 minreplication=1 state=present username=mapr password=maprmapr mapr_webserver=https://localhost:8443","module_complex_args":{},"module_name":"mapr_volume"} | null | 18277 | vicenteg.mapr_post_install_test | null | [] | create benchmarks volume | null | null | null | null | null | null | null | null | null | benchmarks | [{"ReplTypeConversionInProgress":"0","accesstime":"May 1, 2016","acl":{"Allowed actions":["dump","restore","m","a","d","fc"],"Principal":"User mapr"},"actualreplication":[0,100,0,0,0,0,0,0,0,0,0],"advisoryquota":"0","aename":"mapr","aetype":0,"allowGrant":"false","auditVolume":0,"audited":0,"coalesceInterval":60,"creator":"mapr","creatorcontainerid":2567,"creatorvolumeuuid":"-5875528307949895707:5385798119526666980","dbrepllagsecalarmthresh":"0","disableddataauditoperations":"","enableddataauditoperations":"getattr,setattr,chown,chperm,chgrp,getxattr,listxattr,setxattr,removexattr,read,write,create,delete,mkdir,readdir,rmdir,createsym,lookup,rename,createdev,truncate,tablecfcreate,tablecfdelete,tablecfmodify,tablecfScan,tableget,tableput,tablescan,tablecreate,tableinfo,tablemodify,getperm","fixCreatorId":"false","limitspread":"true","logicalUsed":"5125","maxinodesalarmthreshold":"0","minreplicas":"1","mirrorscheduleid":0,"mirrorthrottle":"1","mirrortype":3,"mountdir":"/benchmarks","mounted":1,"nameContainerId":2567,"nameContainerSizeMB":0,"needsGfsck":false,"nsMinReplicas":"1","nsNumReplicas":"1","numreplicas":"1","partlyOutOfTopology":0,"quota":"0","rackpath":"/data","reReplTimeOutSec":"0","readonly":"0","replicationtype":"high_throughput","scheduleid":0,"schedulename":"","snapshotcount":"0","snapshotused":"0","totalused":"650","used":"650","volumeAces":{"readAce":"p","writeAce":"p"},"volumeid":189093514,"volumename":"benchmarks","volumetype":0}] | null | {} | null |
```

We're interested in seeing how long commands take to run. There's start and end columns in the data, but not every command has timing metrics, so some are null:

```
0: jdbc:drill:zk=local>  select t.invocation.module_name module_name, t.`start`,t.`end` from dfs.`/Users/vince/src/logs` t;
+--------------+-----------------------------+-----------------------------+
| module_name  |            start            |             end             |
+--------------+-----------------------------+-----------------------------+
| yum          | null                        | null                        |
| shell        | 2016-05-01 14:17:43.527227  | 2016-05-01 14:17:45.849982  |
| mapr_volume  | null                        | null                        |
| shell        | 2016-05-01 14:17:46.849579  | 2016-05-01 14:18:00.089092  |
| shell        | 2016-05-01 14:18:00.393699  | 2016-05-01 14:18:16.937748  |
| stat         | null                        | null                        |
| command      | 2016-05-01 14:18:17.405104  | 2016-05-01 14:18:19.642868  |
| command      | 2016-05-01 14:18:19.915907  | 2016-05-01 14:18:48.636978  |
| command      | 2016-05-01 14:18:48.900256  | 2016-05-01 14:18:51.157188  |
| command      | 2016-05-01 14:18:51.439254  | 2016-05-01 14:19:21.704467  |
| command      | 2016-05-01 14:19:21.984261  | 2016-05-01 14:19:24.492170  |
+--------------+-----------------------------+-----------------------------+
11 rows selected (0.148 seconds)
```

These are strings, in those columns, so to do math on the data, we can cast it to a timestamp. But the timestamp type doesn't support the level of precision that is in the source data. How do we avoid losing this extra precision?

No date type in Drill supports finer than milliseconds. So what we do is convert to a `unix_timestamp` (seconds since the epoch) and store the additional precision converting the nanos to a floating point number:

```sql
select
  t.invocation.module_name,
  case
    when
      t.`start` is not null
    then
      unix_timestamp(t.`start`, 'yyyy-MM-dd HH:mm:ss.SSSSSS') 
  end as `start`,
  case
    when
      t.`end` is not null
    then
      unix_timestamp(t.`end`, 'yyyy-MM-dd HH:mm:ss.SSSSSS')
  end as `end`,
  cast(substr(`start`, position('.' in t.`start`)+1) as INT) as start_nanos,
  cast(substr(`end`, position('.' in t.`end`)+1) as INT) as end_nanos
from dfs.`/Users/vince/src/logs` t;
```

And then we can add the floating point number to the integral timestamp, and do the arithmetic to get the delta:

```sql
select u.module_name, (u.`end` + u.end_nanos) - (u.`start` + u.start_nanos) as delta from 
  (select
    t.invocation.module_name module_name,
    case
      when
        t.`start` is not null
      then
        unix_timestamp(t.`start`, 'yyyy-MM-dd HH:mm:ss.SSSSSS') 
    end as `start`,
    case
      when
        t.`end` is not null
      then
        unix_timestamp(t.`end`, 'yyyy-MM-dd HH:mm:ss.SSSSSS')
    end as `end`,
    cast(substr(`start`, position('.' in t.`start`)+1) as INT) / POW(10,6) as start_nanos,
    cast(substr(`end`, position('.' in t.`end`)+1) as INT) / POW(10,6) as end_nanos
  from dfs.`/Users/vince/src/logs` t) u;
```

Now I can create a view:

```
create or replace view dfs.tmp.ansible_task_times as
  select u.pid,u.play,u.task,u.module_name, (u.`end` + u.end_nanos) - (u.`start` + u.start_nanos) as delta from 
    (select
      t.pid,
      t.play,
      t.task,
      t.invocation.module_name module_name,
      case
        when
          t.`start` is not null
        then
          unix_timestamp(t.`start`, 'yyyy-MM-dd HH:mm:ss.SSSSSS') 
      end as `start`,
      case
        when
          t.`end` is not null
        then
          unix_timestamp(t.`end`, 'yyyy-MM-dd HH:mm:ss.SSSSSS')
      end as `end`,
      cast(substr(`start`, position('.' in t.`start`)+1) as INT) / POW(10,6) as start_nanos,
      cast(substr(`end`, position('.' in t.`end`)+1) as INT) / POW(10,6) as end_nanos
    from dfs.`/Users/vince/src/logs` t) u;
```

Now I can query my view much more simply:

```
0: jdbc:drill:zk=local> select
     task,
     count(1) runs,
     min(delta) mindelta,
     avg(delta) avgdelta,
     max(delta) maxdelta
   from ansible_task_times
   where delta is not null
   group by task;
+---------------------------------------------------+-------+---------------------+---------------------+---------------------+
|                       task                        | runs  |      mindelta       |      avgdelta       |      maxdelta       |
+---------------------------------------------------+-------+---------------------+---------------------+---------------------+
| check whether /benchmarks is a volume             | 5     | 1.744338035583496   | 2.3756442070007324  | 4.0512919425964355  |
| run RWSpeedTest, 5GB data per node                | 4     | 12.751620054244995  | 14.994079291820526  | 17.44113516807556   |
| clean up the teragen                              | 6     | 2.2377641201019287  | 2.8072693745295205  | 3.3901071548461914  |
| try to submit a small job (teragen - MRv2)        | 2     | 28.721071004867554  | 29.579795956611633  | 30.438520908355713  |
| try to submit a small job (default MR framework)  | 2     | 30.020743131637573  | 30.142978072166443  | 30.265213012695312  |
| RWSpeedTest write                                 | 2     | 8.541120052337646   | 8.740686535835266   | 8.940253019332886   |
| RWSpeedTest read                                  | 2     | 12.471827030181885  | 13.445463418960571  | 14.419099807739258  |
| clean up teragen                                  | 6     | 2.0737199783325195  | 2.26914648214976    | 2.6462981700897217  |
| teragen MRv2                                      | 2     | 29.630738019943237  | 29.948596954345703  | 30.26645588874817   |
| teragen defaultMR                                 | 2     | 23.943012952804565  | 25.599283456802368  | 27.25555396080017   |
+---------------------------------------------------+-------+---------------------+---------------------+---------------------+
10 rows selected (1.096 seconds)
```


