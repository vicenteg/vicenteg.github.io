---
layout: "post"
title:  "Bootstrapping MapR Cluster Nodes in EC2"
date:   2016-07-15 09:33:33 -0500
categories: "mapr"
tags: "mapr,amazon,ec2,cloud,ansible"
---

There's a bunch of ways to spin up MapR in AWS. Primarily, I use ansible to do it.
Why spin up instances and install them myself? Mainly it's because I'm not always
using AWS, and I need a repeatable way to install clusters under any circumstance,
whether on-prem or cloud.

The lowest common demoninator in this case is to think of this as getting handed
a bunch of machines, and installing MapR on them. Originally I built ansible
roles to install MapR as a way of avoiding what was at the time a fussy procedure.
Over time, it's become complete enough that I can use the roles to deploy clusters
of any size in any environment (sometimes with some tweaking).

The playbooks and roles do a few useful things, and they can be used together or separately. This post is intended to help you make sense of the repo should you
find your way to it on my github.

# About the Roles

They can set up AWS EC2 instances, execute a minimal pre-install cluster validation,
install MapR with or without security (with security is the default), configure
some ecosystem projects to work together, and it can run some smoke tests of the
cluster after the installation and configuration completes.

Extensions and fixes are welcomed!

At the moment, I'm waffling between using Ansible 2.x's `json` callback plugin
and the default `skippy`. I really like the idea of getting JSON output that I
can store and query later with Apache Drill, but I don't love the lack of feedback
as the plays execute - the `json` plugin doesn't say anything at all until all the
plays complete or they fail.

So you might not see any output for minutes at a time if the `json` plugin is
enabled.  You can change this by switching `skippy` in for `json` in the
`ansible.cfg` provided.

# Bootstrapping

The bootstrap playbook is simple as its only job is to spin up some Amazon EC2
instances. There's nothing special about this process or these instances - it's
just the method I have settled on as the quickest way to get instances up to
install MapR on. The instances are usually either Amazon Linux (not officially
supported by MapR, but my experience has been that it works great) or CentOS.
I don't use Ubuntu much, so the playbooks don't work with Ubuntu. I'd like to
change that at some point, but I have not had the need to date.

I could use the Marketplace option for deploying MapR AMIs, but that costs more,
 and we end up in the odd situation where MapR is paying MapR to run MapR. Also,
by spinning up generic OS AMIs, I get to test the installation process on a
pretty frequent basis.

So that's why this spins up vanilla CentOS or RHEL or Amazon Linux.

The bootstrap playbook needs configuration. You copy `group_vars/all.example` to
`group_vars/all` and edit it. You need to make sure you set your security groups,
keypair, regions, and so on correctly. You also should have installed `awscli` and
run `aws configure` to establish your credentials.

Also note that you can decide how many instances to create, and whether to assign
them public IPs. Public IPs are not required as long as you have a way to access
your instance's public IPs, such as via a VPN or ssh tunnel.

Usually, if you get some of these configuration parameters wrong, AWS will yell
at you with an error message that helps you figure out what's wrong.

To bootstrap some nodes:

```
ansible-playbook mapr_aws_bootstrap.yml > ansible-mapr_bootstrap_result.json
```

If this succeeds, you'll have some new instances running, and you'll have a new
inventory file called `inventory.py`. This is a python script, which generates
JSON that is used by ansible to populate the inventory. The inventory will have
a number of groups that contain the hosts that will run services.

You can confirm that the instances are up and reachable by ansible after a few
minutes (change the username as needed - AMZ linux uses `ec2-user`, CentOS uses
`centos`):

```
ansible -i inventory.py -u centos -m ping all
```

Hopefully you have all green output from this.

# Prerequisites

MapR requires some things to be in place before you install and run the cluster.
They're well documented, and actually automated in the latest installer.

You can run these prereqs by executing the tasks tagged with `preinstall` in the
mapr_install playbook.

```
ansible-playbook -i inventory.py -u centos --tags preinstall mapr_install.yml >\
  ansible-mapr_preinstall_result.json
```

# Pre-Install validation

My colleague John Benninghoff has a set of scripts that are useful for validating
a cluster hardware and performance. The tasks under the `validation` tag will
execute a subset of these:

- Triad memory test (memory throughput)
- fio for sequential disk throughput
- rpctest for MapR RPC throughput

```
ansible-playbook -i inventory.py -u centos --tags validation mapr_install.yml >\
  ansible-mapr_validation_result.json
```

# MapR Installation and Configuration

Now for the main event - installation. The `mapr_install.yml` playbook will also
do the whole installation. Now we can invoke `ansible-playbook` with some options
to skip the above steps and carry on with the installation:

```
ansible-playbook -i inventory.py -u centos \
  --skip-tags preinstall,validation mapr_install.yml >\
    ansible-mapr_install_result.json
```

Once that's done, you can print out some of the URLs for your cluster using the
included `printurls.sh` script, which is a simple wrapper around `jq` that reads
the inventory output:

```
$ ./printurls.sh
[]
[
  "MCS: https://172.16.2.153:8443",
  "MCS: https://172.16.2.151:8443"
]
[
  "Solr Admin: http://172.16.2.152:8983/solr",
  "Solr Admin: http://172.16.2.153:8983/solr",
  "Solr Admin: http://172.16.2.151:8983/solr"
]
[
  "OpenTSDB: http://172.16.2.151:4242"
]
[
  "Hue: http://172.16.2.153:8888"
]
[
  "Mesos Master: http://172.16.2.153:8080",
  "Mesos Master: http://172.16.2.151:8080"
]
[
  "Resource Manager: https://172.16.2.153:8090",
  "Resource Manager: https://172.16.2.151:8090"
]
```

# Powering Off

If you want to power off the instances when they're not needed, you can use the
following invocation:

```
ansible-playbook -i inventory.py mapr_aws_poweroff.yml
```

# Powering On

When you want them back on, use:

```
ansible-playbook -i inventory.py mapr_aws_poweron.yml
```

# Instance Termination

If you're going to be using this cluster, go right ahead! It should be ready to go.

When you want to get rid of the cluster, you can terminate the instances as follows:

```
ansible-playbook -i inventory.py mapr_aws_terminate.yml
```

Be aware that if you've modified the volume configuration and you left out the
`delete_on_termation` flag, the volumes will remain even when the instances
are terminated. In this case, you'll have to clean them up yourself. You'll continue
to be charged for the use of the storage, whether they're attached to instances
or not, so this is something to remain vigilant about.

# Reading the JSON output

I'll write post at some point where I'll use Drill to query the output.

But for now, the main thing to be looking for in the output is that there are no
 failures.

If you have `jq` installed and you're using the JSON output, you can do the
following to check for failures:

```
jq '.stats[].failures' < ansible-mapr_preinstall_result.json
```

You should see a bunch of zeros - one for each host in the inventory.
