---
layout: post
title:  "Exploring a JSON file with Apache Drill"
date:   2016-04-01 10:15:07 -0500
categories: json drill
---

So you have a JSON file, and you want to play with it in Drill. Where do you start?

## Will it blend?

First, get your file, and put it somewhere Drill can find it. If you're running on a cluster with MapR FS or HDFS, put it in a directory you can access. For my file, instances.json, I did something like this:

```
hadoop fs -put /tmp/instances.json /tmp/instances.json
```

Now that my file is in the distributed file system, I can use the dfs plugin. So start up Drill:

```
sqlline -u jdbc:drill:
```

And query the file. You can always try to just do a `select *` on the file to see what happens:

```
0: jdbc:drill:> select * from dfs.root.`/tmp/instances.json` ;
+--------------+
| Reservations |
+--------------+
| [{"OwnerId":"674241104242","ReservationId":"r-d46a077f","Instances":[{"Monitoring":{"State":"enabled"},"PublicDnsName":"","State":{"Code":16,"Name":"runn
ing"},"EbsOptimized":false,"LaunchTime":"2016-03-16T09:09:10.000Z","PrivateIpAddress":"172.16.2.119","VpcId":"vpc-18dc1a7d","StateTransitionReason":"","Ins
tanceId":"i-90225713","ImageId":"ami-1ecae776","PrivateDnsName":"ip-172-16-2-119.ec2.internal","KeyName":"vgonzalez_keypair","SecurityGroups":[{"GroupName"
:"vpc-internal","GroupId":"sg-0bd72a6f"},{"GroupName":"ssh-only","GroupId":"sg-cc723fa9"}],"ClientToken":"","SubnetId":"subnet-8f999ba7","InstanceType":"m4
.2xlarge","NetworkInterfaces":[{"Status":"in-use","MacAddress":"12:d6:17:0d:99:b3","SourceDestCheck":true,"VpcId":"vpc-18dc1a7d","Description":"","NetworkI
nterfaceId":"eni-b6e58993","PrivateIpAddresses":[{"P
...
upName":"vpc-internal","GroupId":"sg-0bd72a6f"},{"GroupName":"ssh-only","Group |
+--------------+
1 row selected (0.098 seconds)
```

Ok, we got something back without errors, so Drill can read the file. Yes, it blends!


## But it's a mess!

In my file, we get back a single column called `Reservations` and a single row. This is because the file I queried has a single JSON document in it. 

What we want is to get multiple rows and multiple columns. Looking at the document above, we see:

```
[{"OwnerId":"674241104242",...
```

The leading `[{` holds an important clue to how we'll proceed - it indicates we've probably got a list of maps. So maybe we need to `flatten` this list in order to turn the maps into individual rows?

```
0: jdbc:drill:> select flatten(Reservations) Reservations from dfs.root.`/tmp/instances.json` ;
+--------------+
| Reservations |
+--------------+
| {"OwnerId":"674241104242","ReservationId":"r-d46a077f","Instances":[{"Monitoring":{"State":"enabled"},"PublicDnsName":"","State":{"Code":16,"Name":"running"},"EbsOptimized":false,"LaunchTime":"2016-03-16T09:09:10.000Z","PrivateIpAddress":"172.16.2.119","VpcId":"vpc-18dc1a7d","StateTransitionReason":"","InstanceId":"i-90225713","ImageId":"ami-1ecae776","PrivateDnsName":"ip-172-16-2-119.ec2.internal","KeyName":"vgonzalez_keypair","SecurityGroups":[{"GroupName":"vpc-internal","GroupId":"sg-0bd72a6f"},{"GroupName":"ssh-only","GroupId":"sg-cc723fa9"}],"ClientToken":"","SubnetId":"subnet-8f999ba7","InstanceType":"m4.2xlarge","NetworkInterfaces":[{"Status":"in-use","MacAddress":"12:d6:17:0d:99:b3","SourceDestCheck":true,"VpcId":"vpc-18dc1a7d","Description":"","NetworkInterfaceId":"eni-b6e58993","PrivateIpAddresses"[{"PrivateDnsName":"ip-172-16-2-119.ec2.internal","Primary":true,"PrivateIpAddress":"172.16.2.119","Association":{}}],"PrivateDnsName":"i-172-16-2-119.ec2.inte
...
cture":"x86_64","RootDeviceType":"ebs","RootDeviceName":"/dev/sda1","VirtualizationType":"hvm","Tags":[{"Value":"vnaranammalpuram","Key":"Name"},{"Value":"vnaranammalpuram","Key":"user"}],"AmiLaunchIndex":2,"PublicIpAddress":"52.87.248.162","ProductCodes":[],"StateReason":{}}],"Groups":[]} |
+--------------+
41 rows selected (0.14 seconds)
```

The output is still not exactly a neat table, but now we have 41 rows instead of 1. Each row is a map, containing some of the things we want to turn into columns.

So now we need to get into those maps.

The maps are key/value pairs. Drill provides a nice simple syntax for accessing values in maps by key. Let's create a column for the reservationId. To do this we'll use the query above as a sub-select and alias the table name so we can access the key/value pairs in the map by name:

```
0: jdbc:drill:> select t.Reservations.ReservationId ReservationId from (select flatten(Reservations) Reservations from dfs.root.`/tmp/instances.json`) t;
+----------------+
| ReservationId  |
+----------------+
| r-d46a077f     |
| r-98a7d872     |
...
| r-a9349a7f     |
| r-1791adc4     |
| r-b7c2841c     |
+----------------+
41 rows selected (0.106 seconds)
```

Cool, so we got a column. But there are other key-value pairs as well. Let's look at



