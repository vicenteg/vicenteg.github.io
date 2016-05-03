---
layout: post
title:  "How To Use Logstash with MapR Streams"
date:   2016-04-02 16:15:27 -0500
categories: logstash maprstreams
---

# How to use logstash with MapR Streams

Logstash is a great tool for transport and ingest of events from a large variety of sources to a wide variety of destinations. At very high ingest rates, Logstash can be scaled by adding a messaging layer in between the inputs and outputs, as detailed [here](https://www.elastic.co/guide/en/logstash/current/deploying-and-scaling.html). 

Perhaps most commonly, logstash sends data to Elasticsearch for indexing. Before indexing events, it's often the case that a logstash filter plugin transforms or enriches the data, which can add some latency to each event's processing time. The messaging layer helps when the filtering/indexing rate cannot keep up with the ingest rate. In these situations, it's useful to have a scalable buffer so that the indexing layer can work through the event backlog without creating backpressure on the ingest layer.

The messaging layer is also useful for testing. For instance, you can try out new filter plugins on events in the message queue, without affecting other consumers.

This style buffer can be implemented using redis, rabbitmq, kafka, Amazon SQS or zeromq. In this post, I'll tell you how you can implement this buffer with MapR Streams.

Now, why would you do this instead of using another of the messaging systems? With MapR, we can solve for a couple of parts of the log pipeline; the important buffer part as well as a Hadoop compatible archive. Since MapR Streams are API compatible with Kafka 0.9, you can consume from the raw stream with another instance of logstash. That other instance of logstash (the output tier) can then store filtered/transformed logs in MapR FS for query by Hadoop ecosystem tools as well as a full text indexing tool like Elasticsearch.

Now here's how to do it.

First install logstash. I used the logstash 2.2.2 RPMs on a CentOS 6.7 machine, but you can install logstash any way you like.

I assume that you're installing logstash either on a cluster node or a client node that's already configured to talk to your MapR cluster. Your MapR cluster can be secure or insecure; in this guide, I'm working with a cluster running with MapR Security.

You will also need to install the mapr-kafka package to make the kafka-clients library available, which we'll need later.

# Installing the plugins

To use MapR Streams with logstash, we need to use the 3.0 version of the input and output plugins. These beta plugins use the 0.9 APIs. MapR Streams are API compatible with Kafka 0.9, so we can simply replace the jars that the input/output plugins ship with the MapR implementation. 

First, let's install the 3.0.0-beta1 of the logstash input and output plugins (per https://github.com/logstash-plugins/logstash-output-kafka/issues/32):

```
sudo /opt/logstash/bin/plugin install --version 3.0.0.beta1 logstash-input-kafka
sudo /opt/logstash/bin/plugin install --version 3.0.0.beta1 logstash-output-kafka
```

Next, let's update the jars needed at runtime. We need to replace the kafka jars with MapR implementations of the APIs:

```
cd /opt/logstash/vendor/bundle/jruby/1.9/gems/logstash-output-kafka-3.0.0.beta1/vendor/jar-dependencies/runtime-jars
sudo mv kafka-clients-0.9.0.1.jar{,.dist}
sudo ln -s \
  /opt/mapr/lib/mapr-streams-5.1.0-mapr.jar \
  /opt/mapr/lib/kafka-clients-0.9.0.0-mapr-*.jar \
  /opt/mapr/lib/maprfs-5.1.0-mapr.jar \
  /opt/mapr/lib/protobuf-java-*.jar .
  
cd /opt/logstash/vendor/bundle/jruby/1.9/gems/logstash-input-kafka-3.0.0.beta1/vendor/jar-dependencies/runtime-jars
sudo mv kafka-clients-0.9.0.0.jar{,.dist}
sudo ln -s \
  /opt/mapr/lib/mapr-streams-5.1.0-mapr.jar \
  /opt/mapr/lib/kafka-clients-0.9.0.0-mapr-*.jar \
  /opt/mapr/lib/maprfs-5.1.0-mapr.jar \
  /opt/mapr/lib/protobuf-java-*.jar .
```

That's it. Now let's test it.

# Using the output plugin

We'll use the Kafka output plugin to send events to a topic in a MapR stream.

If your cluster is secure, let's generate a service ticket for the logstash pseudo user.

Be sure you check that the UID for logstash is 498 on your install before copy-and-pasting this:

```
sudo -u mapr maprlogin generateticket -type service -user logstash -out /tmp/maprticket_498 -duration 999:0:0
sudo chown logstash:logstash /tmp/maprticket_498
```


Now start up logstash with a simple configuration. This logstash instance will take events you put on standard input and send them to the stream:topic you specify.

```
sudo -u logstash /opt/logstash/bin/logstash \
  -e 'input { stdin {} } output { kafka { topic_id => "/apps/logstash/stream:test" } }'
```

In another terminal on the same cluster (same node, different node, doesn’t matter) run the console consumer:

```
sudo -u logstash /opt/mapr/kafka/kafka-0.9.0/bin/kafka-console-consumer.sh \
  --topic /apps/logstash/stream:test \
  --new-consumer --bootstrap-server foo:9092 \
  --from-beginning
```

Type some stuff into the terminal window where logstash is running, and hit enter. You should see a message appear on the consumer window. If so, it's working! Let's also try the input plugin.

# Using the input plugin

Now, let's test the input plugin. The input plugin will consume messages from one or more topics in a stream, and send them to the output plugin you specifcy. In this case, we'll use stdout to print the events to the screen when they're consumed.

In a terminal window, run logstash as follows. This invocation will run logstash with the input plugin consuming from the stream:topic we created above, and the consumed events will be written to stdout:

```
sudo -u logstash /opt/logstash/bin/logstash -e 'input { kafka { topics => [ "/apps/logstash/stream:test" ] } } output { stdout { codec => "json" } }'
```

In another teminal window, use the kafka-console-producer to send events to the stream:

```
sudo -u logstash /opt/mapr/kafka/kafka-0.9.0/bin/kafka-console-producer.sh \
  --topic /apps/logstash/stream:test \
  --broker-list foo:9092
```

Type some text into the window running the kafka-console-producer and see that logstash prints the messages in JSON format.

If this works, you can go ahead and remove this stream:

```
sudo -u logstash maprcli stream delete -path /apps/logstash/stream
```

And that's it! You've successfully begun to use logstash and MapR Streams.
