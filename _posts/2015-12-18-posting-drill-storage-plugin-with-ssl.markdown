---
layout: post
title:  "POSTing a storage plugin with an SSL-enabled drillbit"
date:   2015-12-18 11:27:33 -0500
categories: drill
---

I spent a couple hours trying to get a drill storage plugin into a drillbit with SSL enabled. I finally solved it so I thought I'd share here in case anyone else has the same need. Here's the scenario.

I built a drill cluster with security features enabled (SSL, impersonation, etc). I use ansible to deploy Drill, and during post-install configuration I had been using curl in an ansible command to POST a hive storage plugin config. I found that my curl invocation to add the plugin was no longer working once SSL was enabled. After trying a few different things that did not work, here's what worked for me:


{% highlight bash %}
curl -c ~/.drill_cookies \
	-X POST -k \
	-d j_username=username -d j_password=password \
	https://localhost:8047/j_security_check
{% endhighlight %}

{% highlight bash %}
curl -b ~/.drill_cookies \
	-k -H "Content-Type: application/json" \
	-X POST -d @/tmp/hive-storage-plugin.json \
	https://localhost:8047/storage/hive.json
{% endhighlight %}


The first curl command sends authentication credentials to the drill bit, and the `-c` option stores the cookies in `~/.drill_cookies`. What ends up here is a session ID, and the session ID appears to be all that's necessary to perform further operations against the drillbit (unsure whether this will work on other drillbits, didn't bother to try). Be aware that, at least on my machine, the cookie-jar file ends up world readable, which on a shared system might be a good way for someone to hijack your session! So be careful with that.

The second command actually POSTs the plugin config, and the `-b` option reads back the aforementioned session cookie.

I'm not sure if there's a better way to do this, but it works fairly cleanly for my needs. Unsure whether this can be done in a more concise one-liner.
