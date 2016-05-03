---
layout: post
title:  "Multiple Ansible roles and repo"
date:   2015-12-20 13:42:23 -0500
categories: ansible git repo
---

Freeze the repos at specific commits:

{% highlight bash %}
 repo manifest -r > /tmp/mapr-5.0.0-drill-1.2.0-cluster.xml
{% endhighlight %}

{% highlight bash %}
 gist -d "repo manifest for MapR v5.0.0 with Drill 1.2.0" -u https://gist.github.com/vicenteg/ab51673c1e29dd202a04 /tmp/mapr-5.0.0-drill-1.2.0-cluster.xml
{% endhighlight %}
