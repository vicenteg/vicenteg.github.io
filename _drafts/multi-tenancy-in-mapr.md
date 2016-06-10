---
layout: post
title:  "Multi-Tenancy Features in MapR"
date:   2016-05-02 09:41:53 -0500
categories: mapr multi-tenancy
---

# Multi-Tenancy Features in MapR

## TODO

* ~~Multi-tenancy and Security~~
* Multi-tenancy and Resource Management
* Alerts
* Cluster permissions - delegating permissions to users/tenant
* Example scripts
* Multi-tenant Archetypes
	* Application Tenant (application with service account(s))
	* User Tenant (e.g. home directories)
	* Multi-user Tenants (e.g., Lines of Business)
* Multi-Tenant Data Organization

## What is Multi-Tenancy

Multi-tenancy can mean different things depending on the capabilities of the system being used. MapR provides multi-tenancy through data isolation, resource control and strong authentication and authorization.

## The Role of Authentication and Authorization in Multi-Tenancy

Usually in a multi-tenant system we will want to control access to the data and resources. We can do this with strong authentication and authorization. Strong authentication is essential to authorization. Without the ability to definitively verify the identity of a user, it is impossible to provide useful authorization.

Put another way, if a user can become any other user without the need to authenticate themselves as that other user, authorization features become meaningless. As such, MapR recommends that multi-tenant clusters use MapR security or Kerberos.


### Multi-Tenancy Features 

MapR provides the following features that underpin multi-tenancy:

* Authentication
* Authorization
* Volumes - for accounting and granular data protection
* ACEs at the volume, file, directory, table, column and stream level
* Topology - for data placement control
* Apache Myriad - for YARN and Mesos integration
* YARN labels and queues - for resource management of MapReduce jobs

Multi-tenancy can be implemented in many ways by combining some or all of the above features. But before we get to brass tacks, let's define some things.

## Isolation

MapR can provide some isolation between tenants. "Some" because complete isolation may not be possible when Hadoop ecosystem tools are involved. It's beyond the scope of this document to describe all the ways in which isolation can and cannot be achieved when ecosystem projects are used, and it's particularly hard to do when combinations of tools are used. So for our purposes here, we'll focus just on isolation provided for and by the MapR tools - volumes, files, tables, streams - and YARN.


## What is a Tenant?

In general, a tenant will comprise one or more users or groups from a central directory, such as Active Directory (AD).

A tenant can be a single developer, or it could be an entire department inside a large enterprise. A tenant could also span multiple departments.

In a large enterprise, a tenant may be charged back for the services they consume, so tenant resource consumption needs to be accounted for.

A tenant should get a volume that represents the "tenant-root", for example, a tenant representing the Wealth Management division at a bank might be provisioned a volume called `tenant-wealth` which is mounted in the cluster's filesystem namespace at `/tenants/wealth`.

## Identities

It is a must to have a solid identity management system for the MapR cluster to use for user and group information. It's essential for all nodes in the cluster to have the same idea about who a user is and what groups they belong to, and a centralized directory provides that capability. Examples of supported systems for authentication and identity management include Active Directory, OpenLDAP, FreeIPA and Kerberos.

It's usually a good idea to have tenants align to one or more groups in the user directory. We'll go into some detail on this for our first tenant example.


## What is a Tenant?

A tenant is a user or group of users that shares a MapR cluster with other users or groups of users. As in the real-estate analogy, tenants do not all have to be the same. In a multi-tenant apartment building, units may differ in their amenities, size, and cost. Certain units may allow different activities. For example, a building that is primarily residential may also have commercial units, such as doctors offices or retail stores with direct access to the street. Tenants will usually pay for their units in based on the amenities they consume, and other factors, such as square footage, number of rooms or features like outdoor space. Not all tenants have the same means or needs, so it's important to be able to provide multi-tenancy in a way that allows for tenants to choose units that are right for their needs.

In the real world example here, a tenant would not be able to dynamically expand their units, however. This is a desirable trait of a multi-tenant system; the ability to expand and contract the usage of the shared infrastructure. 

Having framed the concepts in something more real-world, let's now discuss the mechanisms we can use in MapR to provide multi-tenancy.

### Tenant sizing

We need to be able to provide tenants a portion of the cluster, sized to their need. Typically, tenants will care about getting the storage capacity they need, and the resource they need to carry out their work.

For storage capacity, we provision one or more volumes. A volume may have advisory and hard quotas associated with it. Advisory quotas exist to provide tenants an alert when they cross their advisory, or soft, quota.

Best practice is to set the advisory quota to be some fraction of the hard quota so that an alert can be generated and acted on before the hard quota is exceeded. At Hadoop scales, data can be generated very quickly, so the advisory quota could be set low - say, 65% of the hard quota. 

### Alerting

Email alerts ... How to.


## Types of Tenants

MapR provides a lot of features to implement multi-tenant clusters. These can be combined in interesting ways, and the possibilities are endless. Since this is not an endless document, I'll focus on a few common types of multi-tenancy.

### Multi-User Tenants

In a multi-user tenant, a set of one or more groups is isolated from other tenants. Each tenant has multiple users, and all users in the tenant share the resources allocated to the tenant. Accounts can be people, or non-people (e.g., service or application) accounts.

Within the tenant, users may have home directories that have individually managed quotas. Each user home directory may have files, MapR DB tables, or streams for pub/sub messaging.

Non-person users may be service accounts which exist to run applications.

Some examples of multi-user tenants include:

* A software development group within a line of business
** Person-users are software engineers, developing code in their home directories
** Non-person users are services, such as test-runners, or CI frameworks such as jenkins

* An analyst group
** Person users are business analysts who develop SQL queries
** Non-person users run SQL query engines to which analysts submit their queries

For YARN tasks, dedicated queues may be configured per group.

### Single-User Tenants

A single user tenant is essentially just a home for a user. The user can be, as above, a person or a non-person. In a single-tenant cluster, where just one group "owns" the cluster, individual users may be considered tenants themselves.

Each user may have a volume provisioned, and may or may not have permission to provision more volumes for themselves.

The user's capacity is constrained with quotas on their volume, and the user is the accountable entity for storage consumption cluster-wide.

For YARN tasks, it may be a little heavy-handed to create queues for each user. So instead, users might use a default queue.

### Examples

## A Multi-User Tenant Example

In most organizations, a central directory (such as Active Directory) organizes users into groups based on department or function. Usually, we want our tenants to map very closely to those structures. In this example, we'll consider a fictional IT organization.

The IT function in an organization might be comprised of multiple groups, perhaps, `storage` and `desktopsupport`.

Employees of the departments will have user accounts and be members of the appropriate group.

We want a tenant created for the IT group to consist of a group called `storage` and a group called `desktopsupport`. Within the tenant-root, there may be more volumes, perhaps one per sub-group, and toward the leaves, a volume per user. This yields a structure like the following in our IT example:

```
$ tree tenants/
tenants/
└── it <- volume
    ├── desktopsupport <- volume
    │   └── users
    │       └── rjones <- volume
    └── storage <- volume
        └── users
            └── jsmith <- volume

7 directories, 0 files
```

In this case, the directories `it`, `desktopsupport`, `storage`, `rjones` and `jsmith` are volumes, each of which can have quotas, ACEs, mirrors, snapshots and audit trails all their own. Note the nesting. Volumes can be mounted inside of other volumes.

Further, if a restrictive volume ACE is applied at `it`, no access is allowed at the lower levels. Access is restricted to all the contents of the volume, regardless of whether they are regular files, directories, tables, streams or other volumes. This helps provide isolation through authorization at the storage level.

When creating tenants, ideally, the users in the tenant contained within one or more groups in your identity system. Create a tenant-root volume with a volume-level ACE. This can be open, allowing other tenants access to the data, or can be restricted to only the members of the group or groups that make up the tenant.

Optionally, create a volume per tenant-user, and assign the volume a quota. You can also apply an ACE to the user-volume.

Using ACEs at just the volume level can provide a lot of access control flexibility, given that volumes can be created rather liberally. Tens of thousands of volumes can be created if you choose, each with their own ACEs. In this way, it's possible to avoid having to set ACEs on individual files and directories, which can become unwieldy at scale.

The example above could be implemented with the following commands:

```
# create the IT tenant, allowing only the IT groups access
maprcli volume create -name tenant-it -path /tenants/it \
	-readAce 'g:storage|g:desktopsupport' \
	-writeAce 'g:storage|g:desktopsupport' \
	-createparent true

# create the desktopsupport sub-tenant volume
maprcli volume create -name tenant-it-desktopsupport \
	-path /tenants/it/desktopsupport \
	-readAce 'g:desktopsupport' \
	-writeAce 'g:desktopsupport' \
	-createparent true

# create the storage sub-tenant volume
maprcli volume create -name tenant-it-storage \
	-path /tenants/it/storage \
	-readAce 'g:storage' \
	-writeAce 'g:storage' \
	-createparent true

# create the user home volumes
maprcli volume create -name tenant-it-desktopsupport.home.rjones \
	-path /tenants/it/desktopsupport/users/rjones \
	-createparent true

maprcli volume create -name tenant-it-storage.home.jsmith \
	-path /tenants/it/storage/users/jsmith \
	-createparent true

hadoop fs -chown rjones:it /tenants/it/desktopsupport/users/rjones
hadoop fs -chown jsmith:it /tenants/it/storage/users/jsmith
```

Once this structure is created, the users can authenticate to the cluster and access their home directories. Let's use jsmith to illustrate what the user can and cannot do.

First, jsmith authenticates herself, which generates a ticket on the node:

```
$ maprlogin password
[Password for user 'jsmith' at cluster 'localhost': ]
MapR credentials of user 'jsmith' for cluster 'localhost' are written to '/tmp/maprticket_503'
```

Now, jsmith can explore her home directory with hadoop tools and the POSIX client, and add some data:

```
$ hadoop fs -ls /tenants/it/storage/users/jsmith
$ hadoop fs -put mydata.csv /tenants/it/storage/users/jsmith
$ cd /mapr/localhost/tenants/it/storage/users/jsmith
$ mkdir data
$ ls -l
total 1700
drwxr-xr-x. 2 jsmith it       0 Apr 12 01:07 data
-rwxr-xr-x. 1 jsmith it 1740288 Apr 12 01:09 mydata.csv
$ mv mydata.csv data
```

Can `jsmith` do similar things to `rjones`' home?

```
$ cd /mapr/localhost/tenants/it/desktopsupport/users/rjones/
-bash: cd: /mapr/localhost/tenants/it/desktopsupport/users/rjones/: Permission denied
```

No. The volume ACE we applied to the desktopsupport sub-tenant volume prevents access to anyone not in the `desktopsupport` group.

### Sub-volumes

There's a lot users can do with just a single volume. But in some cases it can be useful to have more volumes. We don't always want to make this a task a cluster administrator needs to perform, so it would be nice to delegate this capability to users. We can delegate some administrative tasks to users using cluster ACLs.

#### Delegating Volume Creation Privileges

Let's say that jsmith wants to be able to create a volume under her home directory. Perhaps she's developing an application and wants to constrain it with a quota. Being a self-starter, she goes ahead and tries it:

```
$ maprcli volume create -path /tenants/it/storage/users/jsmith/myapp -name jsmith.myapp -quota 500G
ERROR (1) -  Volume Creation Failed: No privileges to create volume
```

But the attempt fails. Reporting this to a cluster admin, the cluster admin looks at the cluster ACLs to find the issue:

```
$ maprcli acl show -type cluster
Allowed actions         Principal
[login, ss, cv, a, fc]  User mapr
[login, ss, cv, a, fc]  User root
[login, ss, cv, fc]     Group wheel

$ id jsmith
uid=503(jsmith) gid=503(it) groups=503(it),506(storage)
```

User jsmith is not among the users and groups allowed to create volumes. Since jsmith is someone the cluster administrator can trust with this power, she is granted the `cv` privilege:

```
$ maprcli acl edit -type cluster  -user jsmith:cv
```

User jsmith now tries again, and after the volume create we can see that it is indeed mounted:

```
$ maprcli volume create -path /tenants/it/storage/users/jsmith/myapp -name jsmith.myapp -quota 500G
$ maprcli volume info -name jsmith.myapp -columns volumename,mounted
volumename    mounted
jsmith.myapp  1
```

Great! But what if, drunk with power, jsmith now tries to create a volume outside her home volume?

```
$ maprcli volume create -name test -path /tenants/it/storage/test
2016-04-28 20:59:39,7894 ERROR Client fs/client/fileclient/cc/client.cc:5290 Thread: 17197 Volume mount failed for volume test, Path /tenants/it/storage/test, error 13, for parent fid 2254.16.2
2016-04-28 20:59:39,7895 ERROR JniCommon fs/client/fileclient/cc/jni_MapRClient.cc:3071 Thread: 17197 : Could not mount volume test at /tenants/it/storage/test, error = 13

Successfully created volume: 'test'
ERROR (10003) -  Volume mount for /tenants/it/storage/test failed, Permission denied
```

She's able to create the volume, but the volume will not mount because jsmith lacks the permissions to write to the storage directory:

```
$ hadoop fs -ls -d /tenants/it/storage
drwxr-xr-x   - root root          1 2016-04-12 01:36 /tenants/it/storage
```

A couple of operational best practices fall out of this:

* Monitor your cluster for unmounted volumes. Users with `cv` may try to create volumes in the wrong place.
* Ensure that ACEs are set restrictively enough that users with `cv` cannot mount volumes in places they shouldn't.

To find unmounted volumes and the user who created them:

```
$ maprcli volume list -filter '[mounted==0] and [volumename!=mapr.cldb.internal]' \
	-columns volumename,mounted,creator
creator  volumename  mounted
jsmith   test        0
```

Note the `-filter`. We look for volumes that are not mounted (`mounted==0`) and are not named `mapr.cldb.internal`. `mapr.cldb.internal` is a special volume, and is not mounted, so we ignore it.


### Data Organization

In the book _Hadoop Application Architectures_ (published by O'Reilly, 2015), the authors put forth a recommendation 

## YARN

From a YARN perspective a tenant might share a YARN cluster with other tenants. This case is not different from other Hadoop distributions.

Tenants can be provided different shares of the resources through queues and scheduler configuration.

