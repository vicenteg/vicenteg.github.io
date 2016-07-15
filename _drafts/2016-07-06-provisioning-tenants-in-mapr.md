---
layout: post
title:  "Provisioning Tenants in MapR"
date:   2016-07-06 14:26:22 -0500
categories: mapr multi-tenancy
---

For these examples we assume that any users and groups already exist. You can follow along on the sandbox using the following groups and users:

```
[mapr@maprdemo ~]$ sudo groupadd tenant1
[mapr@maprdemo ~]$ sudo groupadd tenant2
[mapr@maprdemo ~]$ sudo groupadd tenant3

[mapr@maprdemo ~]$ sudo useradd -g tenant1 alice
[mapr@maprdemo ~]$ sudo useradd -g tenant2 bob
[mapr@maprdemo ~]$ sudo useradd -g tenant3 charlie
```

Tenants = primary groups

Users = members of groups

best practice - use user primary group as AE, particularly if your tenants map 1:1 to user primary groups. In general, you probably don't care about accounting at the user level in most cases.

best practice - put a volume quota on everything to make sure runaway processes don't fill up your cluster. Or, at least use an advisory quota so that your tenant and the administrators can be notified when thresholds are crossed.

best practice - if using the HDFS schema suggested in Hadoop Application Architectures, create a "tenant root" for each tenant. E.g., `/etl/tenant1`, `/user/tenant1`.

tip - quotas have no relationship across volumes and entities. IOW, setting the quota for a volume does not impact the entity quota, or vice versa. You can use this to oversubscribe the cluster from a provisioning standpoint, such that the sum of the quotas is larger than the total capacity of the cluster. This approach needs to be undertaken only after a plan is in place to expand capacity as needed.

What happens when a user is a member of multiple tenant groups?

How do you create a tenant? Create the group(s) then make the users members of that group. When you create a new tenant for a cluster, you might want to set some default quotas. Do this with the `maprcli config` command. We use the config command to set default parameters as follows:

```
[mapr@maprdemo ~]$ maprcli config save -values \
  '{ "mapr.quota.group.advisorydefault": "60G", "mapr.quota.group.default": "100G" }'
```

So we set the default advisory quota to 60GB and the default hard quota to 100GB.

Create a tenant root volume in `/user`:

```
[mapr@maprdemo ~]$ maprcli volume create \
  -name home.tenant1 -path /user/tenant1 \
  -ae tenant1 \
  -readAce 'g:tenant1 | g:mapr' \
  -writeAce 'g:tenant1 | g:mapr' \
  -aetype 1
```

And see the result in the entity list. Entity `tenant1` has a single volume and no usage.

```
[mapr@maprdemo ~]$ maprcli entity list
EntityType  EntityId  EntityName  EntityAdvisoryquota  DiskUsage  VolumeCount  EntityQuota
0           2000      mapr        0                    339        16           0
1           2003      tenant1     0                    0          1            0
```

If we adopt the HDFS schema put forth in Hadoop Application Architectures, we can also create an ETL area for tenant1 (note the new parameter `-createparent 1` - this will create the `/etl/` directory if it does not already exist):

```
[mapr@maprdemo ~]$ maprcli volume create \
  -name etl.tenant1 -path /etl/tenant1 \
  -ae tenant1 \
  -readAce 'g:tenant1 | g:mapr' \
  -writeAce 'g:tenant1 | g:mapr' \
  -aetype 1 -createparent 1
```

If we list the entities again, we'll see now that tenant1 has two volumes:

```
[mapr@maprdemo ~]$ maprcli entity list
EntityType  EntityId  EntityName  EntityAdvisoryquota  DiskUsage  VolumeCount  EntityQuota
0           2000      mapr        0                    339        16           0
1           2003      tenant1     0                    0          2            0
```

How do you create a user in a tenant? Add the user (if not already done). Then create a volume for that user. You might want to create the volume with the -ae flag. Also remember to set the permissions and ownership. Setting ACEs do not also set POSIX ownership:

```
[mapr@maprdemo ~]$ maprcli volume create \
  -path /user/tenant1/alice \
  -name tenant1.home.alice -ae alice \
  -quota 30g -advisoryquota 20g \
  -readAce 'u:alice | g:tenant1' -writeAce 'u:alice'

[mapr@maprdemo ~]$ hadoop fs -chown alice:tenant1 /user/tenant1/alice

[mapr@maprdemo ~]$ maprcli entity list
EntityType  EntityId  EntityName  EntityAdvisoryquota  DiskUsage  VolumeCount  EntityQuota
0           2002      alice       20480                0          1            30720
0           2000      mapr        0                    339        16           0
1           2003      tenant1     61440                0          2            102400
```

So let's test this by creating a 20MB file in Alice's home, and seeing the space count against Alice's quota.

```
[alice@maprdemo alice]$ dd if=/dev/zero of=file1 count=20 bs=1M
20+0 records in
20+0 records out
20971520 bytes (21 MB) copied, 0.0651742 s, 322 MB/s

[alice@maprdemo alice]$ ls -lh
total 20M
-rw-r--r-- 1 alice tenant1 20M Jul  4 21:18 file1
```

Ok, Alice just created a 20MB file. Let's see this against her quota.

```
[mapr@maprdemo ~]$ maprcli entity info -name alice
EntityType  EntityId  EntityName  EntityAdvisoryquota  DiskUsage  VolumeCount  EntityQuota
0           2002      alice       150                  2          1            204800
```

Usage info comes back in MB. But it only shows 2MB used. What gives? We just created a 20MB file?

Answer is that MapR FS compresses by default. So only the compressed data size counts against your quota. So Alice gets 18MB for free! (tip - if for whatever reason you'd prefer not to compress Alice's data, you can turn it off on her home directory's root using `hadoop mfs -setcompression off /user/tenant1/alice`)

Ok, mystery solved. But when we list all the entities, we see that Alice's data counts against Alice as an accountable entity, but the tenant she is a member of:

```
[mapr@maprdemo ~]$ maprcli entity list
EntityType  EntityId  EntityName  EntityAdvisoryquota  DiskUsage  VolumeCount  EntityQuota
0           2002      alice       20480                2          1            30720
0           2000      mapr        0                    339        16           0
1           2003      tenant1     61440                0          2            102400
```

So this means if we want to account for user space consumption in this way and we wanted to track it against tenant consumption, we'd have to come up with some way to "roll up" all the AEs that are members of tenants. This is not overly difficult, but is perhaps not the kind of complexity we want out of the box.

Also, in most cases end users will not be the entity responsible for the costs incurred by a tenant. So it doesn't make much sense to have the end user be the accountable entity. Usually what we want is for the user to be constrained by a quota, but for the usage to count against the *tenant* rather than the user. So let's modify Alice's volume to have an accountable entity of her primary group:

```
[mapr@maprdemo ~]$ maprcli volume modify \
  -name tenant1.home.alice -ae tenant1 -aetype 1
```

As an aside, we can get clever here too:

```
[mapr@maprdemo ~]$ maprcli volume modify \
  -name tenant1.home.alice -ae `id -ng alice` -aetype 1
```

Another convenience of using primary groups to organize tenants.

Now listing the entities shows us something that makes more sense, if our organization uses primary groups to denote a tenant. Note how alice now has no `DiskUsage` as far as her entity is concerned, and the DiskUsage counts against tenant1's quota:

```
[mapr@maprdemo ~]$ maprcli entity list
EntityType  EntityId  EntityName  EntityAdvisoryquota  DiskUsage  VolumeCount  EntityQuota
0           2002      alice       20480                0          0            30720
0           2000      mapr        0                    339        16           0
1           2003      tenant1     61440                2          3            102400
```

Let's create a second tenant with a user `bob` as above, in just a few commands:

```
[mapr@maprdemo ~]$ maprcli volume create \
  -path /user/tenant2 -name home.tenant2 \
  -ae tenant2 -aetype 1

[mapr@maprdemo ~]$ maprcli volume create \
  -path /user/tenant2/bob -name tenant2.home.bob \
  -ae `id -ng bob` -aetype 1 \
  -quota 10g -advisoryquota 6g \
  -readAce 'g:tenant2' -writeAce 'g:tenant2'

[mapr@maprdemo ~]$ hadoop fs -chown bob:tenant2 /user/tenant2/bob
```
