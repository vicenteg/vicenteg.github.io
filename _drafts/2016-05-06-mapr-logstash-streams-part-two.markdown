---
layout: post
title:  "How To Use Logstash with MapR Streams Part 2"
date:   2016-05-06 18:41:33 -0500
categories: logstash maprstreams
---

In part one, we set up the kafka input and output plugins to work with MapR Streams. In part 2, we'll continue to build out our logstash pipeline with an input tier.

This setup will be a little bit more involved and will take advantage of some other MapR features to manage the input data tier. First, let's create a volume for logstash. We can constrain the volume using quotas to keep the stream from filling up the cluster.

As root or mapr:

```
maprcli volume create -path /apps/logstash -name apps.logstash -quota 1T -advisoryquota 750G -ae logstash -readAce 'g:logstash' -writeAce 'g:logstash'
```

Change ownership to the logstash user:

```
sudo -u mapr hadoop fs -chown -R logstash /apps/logstash
```

If you’re running on a cluster with MapR security, generate a service ticket for the logstash user (check to make sure that the UID is 498 on your install before copy-and-pasting this). Do this on each node that will run an instance of logstash. Note that you'll need to run this as a user that's got the ability to generate tickets - such as the cluster superuser, usually `mapr`.

```
maprlogin generateticket -type service -user logstash -out /tmp/maprticket_498 -duration 999:0:0
sudo chown logstash:logstash /tmp/maprticket_498
```

Next, as the logstash user, make a stream in a MapR FS directory:

```
sudo -u logstash maprcli stream create -path /apps/logstash/stream 
```

If you run the command exactly as above, you probably got a warning message. The message is telling you that the logstash user will be allowed to create, produce to or consume from the stream.

Let’s test that logstash can write to the stream without editing configuration files.

```
sudo -u logstash /opt/logstash/bin/logstash -e 'input { file { path => "/opt/mapr/logs/warden.log" } } output { kafka { topic_id => "/apps/logstash/stream:warden" } stdout { codec => "rubydebug" } }'
```

This will read events from /opt/mapr/logs/warden.log, then output them to a new topic called `warden` on our stream. You can also choose to monitor a different log file, just change the paths and desination topic as you like.

It’ll also output each event to stdout using the rubydebug codec so that we can see the events that flow through. If this works for you, you can create /etc/logstash/conf.d/mapr-warden.conf with the following contents to make this permanent:

```
input {
  file {
    path => [ "/opt/mapr/logs/warden*.log" ]
    type => “warden"
  }
}

output {
  if [type] == “warden" {
        kafka {
            codec => "json"
            topic_id => "/apps/logstash/stream:warden"
        }
    }
}
```

And start or restart logstash. After a little while, assuming the file you chose has some activity, you can fire up the kafka-console-consumer and look at the events that are hitting the stream:

```
sudo -u logstash /opt/mapr/kafka/kafka-0.9.0/bin/kafka-console-consumer.sh \
  --topic /apps/logstash/stream:warden \
  --new-consumer --bootstrap-server foo:1 \
  --from-beginning
```

All we're doing here is reading files from the disk, and putting them on the stream. This is deliberately simple; we're going to avoid doing much processing of the data at this layer so that we can optimize for speed. We want to handle lots of events on ingest, and we'll let the output tier worry about filtering and transforming the data.

How much data is in the stream? Run `maprcli stream info -path /apps/logstash/stream -json` and you should see something like the following:

```json
{
	"timestamp":1461870579209,
	"timeofday":"2016-04-28 03:09:39.209 GMT-0400",
	"status":"OK",
	"total":1,
	"data":[
		{
			"path":"/apps/logstash/stream",
			"physicalsize":26001408,
			"logicalsize":9797632,
			"numtopics":1,
			"defaultpartitions":1,
			"ttl":604800,
			"compression":"lz4",
			"clientcompression":true,
			"autocreate":true
		}
	]
}
```

The fields are probably fairly self-explanatory. You can see the effect of the compression on the stream in the difference between the physicalsize and logicalsize values. logicalsize is the space used after compression.


