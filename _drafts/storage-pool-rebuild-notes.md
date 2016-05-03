---
layout: post
title:  "Rebuilding Storage Pools on a Running Cluster on MapR v4.1"
date:   2015-12-2 17:22:09 -0500
categories: maprv4
---


By default, we choose SP widths of 3 on MapR v4.1:

```
# /opt/mapr/server/mrconfig sp list -v
ListSPs resp: status 0:8
No. of SPs (8), totalsize 25540354 MB, totalfree 25535805 MB

SP 0: name SP1, Online, size 3192544 MB, free 3191975 MB, path /dev/sdt, log 200 MB, guid a3720dfcb7b4036b00564d84860752b6, clusterUuid -5615337016734347730--608579758950429034, disks /dev/sdt /dev/sdv /dev/sdw
SP 1: name SP2, Online, size 3192544 MB, free 3191976 MB, path /dev/sdx, log 200 MB, guid 01186bda201081b000564d84880cb8d2, clusterUuid -5615337016734347730--608579758950429034, disks /dev/sdx /dev/sdb /dev/sdn
SP 2: name SP3, Online, size 3192544 MB, free 3191975 MB, path /dev/sdg, log 200 MB, guid 81e5ade3f8dafdd500564d848b01cdb8, clusterUuid -5615337016734347730--608579758950429034, disks /dev/sdg /dev/sdj /dev/sdq
SP 3: name SP4, Online, size 3192544 MB, free 3191974 MB, path /dev/sdr, log 200 MB, guid 64b47a8b229e13d100564d848d062f64, clusterUuid -5615337016734347730--608579758950429034, disks /dev/sdr /dev/sds /dev/sdp
SP 4: name SP5, Online, size 3192544 MB, free 3191977 MB, path /dev/sdu, log 200 MB, guid 0b696cc097e58ad000564d848f0b71c1, clusterUuid -5615337016734347730--608579758950429034, disks /dev/sdu /dev/sdh /dev/sdm
SP 5: name SP6, Online, size 3192544 MB, free 3191975 MB, path /dev/sda, log 200 MB, guid 771c17c64f642aa400564d849201ce49, clusterUuid -5615337016734347730--608579758950429034, disks /dev/sda /dev/sdi /dev/sdd
SP 6: name SP7, Online, size 3192544 MB, free 3191975 MB, path /dev/sdo, log 200 MB, guid 17a3a4606af200af00564d849406e545, clusterUuid -5615337016734347730--608579758950429034, disks /dev/sdo /dev/sdc /dev/sdf
SP 7: name SP8, Online, size 3192544 MB, free 3191976 MB, path /dev/sdk, log 200 MB, guid 58aa761295acc02b00564d84960bcf8c, clusterUuid -5615337016734347730--608579758950429034, disks /dev/sdk /dev/sdl /dev/sde
```

This results in lackluster performance for a node with 24 disks:

```
[root@rhel5 ~]# sudo -u mapr hadoop jar /opt/mapr/lib/maprfs-diagnostic-tools-4.1.0-mapr.jar com.mapr.fs.RWSpeedTest /benchmarks/`hostname`/f-`date +%s` $((20*1024)) maprfs:///
Write rate: 813.1943532262951 M/s
[root@rhel5 ~]# sudo -u mapr hadoop jar /opt/mapr/lib/maprfs-diagnostic-tools-4.1.0-mapr.jar com.mapr.fs.RWSpeedTest /benchmarks/`hostname`/f-`date +%s` $((20*1024)) maprfs:///
Write rate: 853.0223030784508 M/s
```

We can do better! Let’s use a width of 6 instead. Do this for each node in the cluster. Check that alarms clear before proceeding to the next node. It may take some time for the under-replicated alarms to clear.

```
## On a secure cluster, begin copy/paste here ##
## HEY YOU! If you don't wait for the data under-replicated alarms to clear,
## YOU WILL LOSE DATA! So go one node at a time, and WAIT FOR THE ALARMS TO CLEAR
## BEFORE MOVING ON TO THE NEXT NODE.
echo -n "Press enter to proceed: "; read
sudo -u mapr maprcli node topo -path /decommissioned
ID=$(sudo -u mapr maprcli node list -filter hostname==$(hostname) -columns id -noheader | awk '{ print $1 }')
sudo -u mapr maprcli node move -serverids $ID -topology /decommissioned
sudo -u mapr maprcli volume remove -name mapr.$(hostname).local.metrics
sudo -u mapr maprcli volume remove -name mapr.$(hostname).local.logs

/opt/mapr/server/mrconfig sp offline all
for disk in $(lsblk | egrep "^sd" | awk '{ print $1 }'); do /opt/mapr/server/mrconfig disk remove /dev/$disk; done
service mapr-warden stop
rm -f /opt/mapr/conf/disktab
/opt/mapr/server/disksetup -F -W 6 /tmp/disks.txt
service mapr-warden start
sleep 30
sudo -u mapr maprcli alarm list | grep -v SERVICE_
## Stop copy/paste here ##

sudo -u mapr maprcli node move -serverids $ID -topology /data
```

#### Now test performance again

```
sudo -u mapr maprcli volume create -name benchmark.local.$(hostname) -path /benchmarks/$(hostname) -replication 1 -minreplication 1 -localvolumehost $(hostname) -createparent true
sudo -u mapr hadoop mfs -setcompression off /benchmarks/$(hostname)
sudo -u mapr hadoop jar /opt/mapr/lib/maprfs-diagnostic-tools-4.1.0-mapr.jar com.mapr.fs.RWSpeedTest /benchmarks/$(hostname)/f-`date +%s` $((20*1024)) maprfs:///

[root@rhel5 ~]# sudo -u mapr hadoop jar /opt/mapr/lib/maprfs-diagnostic-tools-4.1.0-mapr.jar com.mapr.fs.RWSpeedTest /benchmarks/$(hostname)/f-`date +%s` $((20*1024)) maprfs:///
Write rate: 1387.6218971310416 M/s
```

Better!

## Doing the benchmarks on all nodes via ansible

```
ansible -i inventory.py -s --sudo-user mapr -m shell -a 'hadoop fs -mkdir /benchmarks/$(hostname)' cluster

ansible -i inventory.py -s --sudo-user mapr -m shell -a 'hadoop mfs -setcompression off /benchmarks/$(hostname)' cluster

ansible -i inventory.py -s --sudo-user mapr -m shell -a 'hadoop jar /opt/mapr/lib/maprfs-diagnostic-tools-5.0.0-mapr.jar com.mapr.fs.RWSpeedTest /benchmarks/$(hostname)/f-`hostname` $((20*1024)) maprfs:///' cluster

ansible -i inventory.py -s --sudo-user mapr -m shell -a 'hadoop jar /opt/mapr/lib/maprfs-diagnostic-tools-5.0.0-mapr.jar com.mapr.fs.RWSpeedTest /benchmarks/$(hostname)/f-`hostname` -$((20*1024)) maprfs:///' cluster
```
