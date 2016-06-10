---
layout: post
title:  "Scaling POSIX Reads on MapR FS with Replicas"
date:   2016-05-02 12:00:23 -0500
categories: maprfs posix performance
---

## Background

Some customers would like to consider MapR for some read-dominant scale out NFS use cases. As such, they need to be sure that reads via NFS gateways can be scaled through replication.

## Test Method

There's a topology path called /data/repl3 in a 6 node cluster (larger so that CLDB volumes are not affected by topology moves), and a volume with replication = 1 placed in that topology. One node is placed in the /data/repl3 topology to start. Finally, we'll turn off compression so we can get an accurate read of throughput exclusive of client-side compression.

```
maprcli node topo -path /data/repl3

maprcli node move -serverids 5764333826204443948 -topology /data/repl3

maprcli volume create -name repl3 \
-path /benchmarks/repl3 \
-replication 1 -minreplication 1 \
-topology /data/repl3 \
-createparent true

hadoop mfs -setcompression off /benchmarks/repl3
```

We use an edge node (i.e., not a cluster node) for the test. I create three mount points on the client. Then I mount the volume via NFSv3. In other words, the client performs a standard linux NFS mount of the NFSv3 server on the cluster node in the above topology.

```
sudo mkdir -p /mnt/repl3/node{1,2,3}

sudo mount 172.16.2.25:/mapr/vgonzalez.replicatest/benchmarks/repl3 /mnt/repl3/node1

sudo mount 172.16.2.45:/mapr/vgonzalez.replicatest/benchmarks/repl3 /mnt/repl3/node2

sudo mount 172.16.2.68:/mapr/vgonzalez.replicatest/benchmarks/repl3 /mnt/repl3/node3
```

The client now has three mounts, one for each node in the topology. This ensures that I/O I send to those mount points passes through the NFS server on those nodes, which enables us to see exactly how the data is moving.

Now let's do the tests.

### Replication = 1

I run fio against this single mount point, reading from a single 10GB file from one job. A picture of what this looks like:

[https://docs.google.com/drawings/d/1THRsaokoWPrTOw6FzbobZVWdD3dNIQpnyvHW6uMVdw0](https://docs.google.com/drawings/d/1THRsaokoWPrTOw6FzbobZVWdD3dNIQpnyvHW6uMVdw0)

fio will create a single file, and that file will end up being written to just one node - all chunks of the file are written locally where the NFS server is running. So the output of the check_chunks.sh script looks like this:

```
$ ./check_chunks.sh /benchmarks/repl3/testfiles.0
     40 ip-172-16-2-25:5660
```

This indicates that 100% of chunks that make up testfiles.0 are on the node ip-172-16-2-25:5660.

On my AWS setup, I'm seeing about 235MB/s, and I confirm through MCS Nodes>Performance that all the IO is coming from the one node - all network and disk reads come from the node holding all the chunks.

#### Result: Fio job one thread to one mount, replication = 1

```
fio one-nfs-repl1.job
...
Run status group 0 (all jobs):
   READ: io=10240MB, aggrb=247504KB/s, minb=247504KB/s, maxb=247504KB/s, mint=42366msec, maxt=42366msec
```

Since we already have the other two NFS mounts on the client, we can also see what happens when we run the fio job against two mounts, then against three.

#### Result: two Fio jobs, one thread to one mount, two mounts, replication = 1

fio against two mounts - note how little the aggregate throughput changes, and how the min/max bandwidth is halved, indicating that each job is getting half of the throughput the single node did:

```
$ fio two-nfs-repl2.job
...
Run status group 0 (all jobs):
   READ: io=14378MB, aggrb=245347KB/s, minb=122673KB/s, maxb=122684KB/s, mint=60004msec, maxt=60009msec
```

#### Result: three Fio jobs, one thread to one mount, three mounts, replication = 1

fio against three mounts - again, aggregate is unchanged, but the minimum and maximum suggest even more sharing of the total single node bandwidth. The bottleneck here is probably total network bandwidth on the node holding the data, since the result does not change whether the data is read from disk or MFS cache.

```
$ fio three-nfs-repl3.job
...
Run status group 0 (all jobs):
   READ: io=14394MB, aggrb=245624KB/s, minb=72182KB/s, maxb=89064KB/s, mint=60003msec, maxt=60008msec
```

### Replication = 2

Now I increase replication to 2. I set minreplication to 2 so that the cluster will immediately re-replicate, so we don't need to wait.

```
maprcli volume modify -name repl3 -replication 2 -minreplication 2
```

The "volume under-replicated" alarm will fire when the replication factor is increased. Wait for that to clear.

Now, we have a replica for each chunk. Note that on each of the bold lines, we have two MFS hostnames. The first is the head of the replication chain, and the second is the tail:

```
$ hadoop mfs -ls /benchmarks/repl3/testfiles.0

Found 1 items

-rw-r--r-- U U U 2 ec2-user ec2-user 10737418240 2016-02-09 01:15 268435456 /benchmarks/repl3/testfiles.0
p 2154.80.262710 ip-172-16-2-25:5660 ip-172-16-2-68:5660
0 2197.114.131490 ip-172-16-2-25:5660 ip-172-16-2-68:5660
1 2184.100.262594 ip-172-16-2-25:5660 ip-172-16-2-45:5660
2 2196.115.131346 ip-172-16-2-25:5660 ip-172-16-2-68:5660
3 2182.115.262646 ip-172-16-2-25:5660 ip-172-16-2-68:5660
4 2186.101.262774 ip-172-16-2-25:5660 ip-172-16-2-68:5660
5 2197.115.131492 ip-172-16-2-25:5660 ip-172-16-2-68:5660
6 2184.101.262596 ip-172-16-2-25:5660 ip-172-16-2-45:5660
7 2196.116.131348 ip-172-16-2-25:5660 ip-172-16-2-68:5660
8 2186.102.262776 ip-172-16-2-25:5660 ip-172-16-2-68:5660
9 2182.116.262648 ip-172-16-2-25:5660 ip-172-16-2-68:5660
10 2197.116.131494 ip-172-16-2-25:5660 ip-172-16-2-68:5660
11 2184.102.262598 ip-172-16-2-25:5660 ip-172-16-2-45:5660
12 2196.117.131350 ip-172-16-2-25:5660 ip-172-16-2-68:5660
13 2186.103.262778 ip-172-16-2-25:5660 ip-172-16-2-68:5660
14 2182.117.262650 ip-172-16-2-25:5660 ip-172-16-2-68:5660
15 2197.117.131496 ip-172-16-2-25:5660 ip-172-16-2-68:5660
16 2196.118.131352 ip-172-16-2-25:5660 ip-172-16-2-68:5660
17 2184.103.262600 ip-172-16-2-25:5660 ip-172-16-2-45:5660
18 2186.104.262780 ip-172-16-2-25:5660 ip-172-16-2-68:5660
19 2191.78.262688 ip-172-16-2-25:5660 ip-172-16-2-45:5660
20 2195.81.262632 ip-172-16-2-25:5660 ip-172-16-2-45:5660
21 2190.91.262846 ip-172-16-2-25:5660 ip-172-16-2-45:5660
22 2185.81.393632 ip-172-16-2-25:5660 ip-172-16-2-68:5660
23 2188.92.262702 ip-172-16-2-25:5660 ip-172-16-2-45:5660
24 2191.79.262690 ip-172-16-2-25:5660 ip-172-16-2-45:5660
25 2190.92.262848 ip-172-16-2-25:5660 ip-172-16-2-45:5660
26 2185.82.393634 ip-172-16-2-25:5660 ip-172-16-2-68:5660
27 2195.82.262634 ip-172-16-2-25:5660 ip-172-16-2-45:5660
28 2188.93.262704 ip-172-16-2-25:5660 ip-172-16-2-45:5660
29 2191.80.262692 ip-172-16-2-25:5660 ip-172-16-2-45:5660
30 2190.93.262850 ip-172-16-2-25:5660 ip-172-16-2-45:5660
31 2185.83.393636 ip-172-16-2-25:5660 ip-172-16-2-68:5660
32 2195.83.262636 ip-172-16-2-25:5660 ip-172-16-2-45:5660
33 2188.94.262706 ip-172-16-2-25:5660 ip-172-16-2-45:5660
34 2195.84.262638 ip-172-16-2-25:5660 ip-172-16-2-45:5660
35 2191.81.262694 ip-172-16-2-25:5660 ip-172-16-2-45:5660
36 2190.94.262852 ip-172-16-2-25:5660 ip-172-16-2-45:5660
37 2185.84.393638 ip-172-16-2-25:5660 ip-172-16-2-68:5660
38 2197.118.131498 ip-172-16-2-25:5660 ip-172-16-2-68:5660
39 2184.104.262602 ip-172-16-2-25:5660 ip-172-16-2-45:5660
```

Data is now replicated.

#### Result: two Fio jobs, one thread to one mount, two mounts, replication = 2

$ fio two-nfs-repl2.job

Run status group 0 (all jobs):

   READ: io=14396MB, aggrb=245675KB/s, minb=122837KB/s, maxb=122837KB/s, mint=60004msec, maxt=60004msec

Same two-nfs mount job as before. Interesting that we still see only one node's worth of aggregate bandwidth. 

```
$ fio two-nfs-repl2.job

Run status group 0 (all jobs):

   READ: io=20480MB, aggrb=402725KB/s, minb=201362KB/s, maxb=203705KB/s, mint=51475msec, maxt=52074msec
```

## Replication = 3

#### Result: three Fio jobs, one thread to one mount, three mounts, replication = 3

Finally, run the fio with one thread for each of three mounts soon after increasing replication to 3:

$ fio three-nfs-repl3.job

...

Run status group 0 (all jobs):

   READ: io=30720MB, **aggrb=573044KB/s**, minb=191014KB/s, maxb=209715KB/s, mint=50000msec, maxt=54895msec

[ec2-user@ip-172-16-2-206 ~]$ fio three-nfs-repl3.job

Throughput is pretty even across nodes, but overall throughput is less than we expect, probably because of the remote reads:

:-)

About an hour after the replication factor was changed, the results change:

$ fio three-nfs-repl3.job

...

Run status group 0 (all jobs):

   READ: io=30720MB, **aggrb=735309KB/s**, minb=245103KB/s, maxb=247049KB/s, mint=42444msec, maxt=42781msec

Total throughput gets up to 735MB/s, and the min and max bandwidth show very even contribution from all the nodes. Also, the reads are 100% from local disks:


## Commands

```
maprcli volume create -name repl3 -path /benchmarks/repl3 -replication 3 -createparent true

maprcli node topo -path /data/repl3

maprcli node move -serverids 7406425749999142246,1679206658172211655,6329351963962791882 -topology /data/repl3

maprcli volume move -name repl3 -topology /data/repl3

hadoop mfs -setcompression off /benchmarks/repl3
```

Create mount points on a client and mount nodes:

```
 sudo mkdir -p /mnt/repl1 /mnt/repl2/node1 /mnt/repl2/node2 /mnt/repl3/node1 /mnt/repl3/node2 /mnt/repl3/node3

mount 172.16.2.140:/mapr/vgonzalez.replicatest/benchmarks/repl1 /mnt/repl1

mount 172.16.2.186:/mapr/vgonzalez.replicatest/benchmarks/repl2 /mnt/repl2/node1

mount 172.16.2.241:/mapr/vgonzalez.replicatest/benchmarks/repl2 /mnt/repl2/node2

mount 172.16.2.25:/mapr/vgonzalez.replicatest/benchmarks/repl3 /mnt/repl3/node1

mount 172.16.2.45:/mapr/vgonzalez.replicatest/benchmarks/repl3 /mnt/repl3/node2

mount 172.16.2.68:/mapr/vgonzalez.replicatest/benchmarks/repl3 /mnt/repl3/node3
```

## Fio Job files

[https://gist.github.com/vicenteg/60bf3b8bd07534755caa](https://gist.github.com/vicenteg/60bf3b8bd07534755caa)