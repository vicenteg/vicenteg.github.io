---
layout: post
title:  "Submitting Drill queries from Sublime Text 3"
date:   2016-05-28 09:17:59 -0500
categories: drill
---

When we're writing queries for Drill, we can spend a lot of time jumping back and forth between a text editor and sqlline. The text editor is used to write the query, and sqlline to run the query.

So we edit, copy/paste, inspect, edit the query, copy/paste, ad infinitum. Works well enough.

But if you use Sublime Text, you can try an easier way to iterate, and it doesn't involve any other tools.

You can use the Sublime Text build system functionality to launch a query from sublime and view the output without having to leave sublime.

So first, launch sublime. Then navigate to Tools > Build System > New Build System. You should get a new file tab called `untitled.sublime-build`.

My build system looks like the following. You may want to make some adjustments in your environment:

```json
{
	"cmd": [ "/opt/apache-drill-1.6.0/bin/sqlline", "--color=false", "-u", "jdbc:drill:drillbit=localhost",  "--run='$file'" ],
	"working_dir": "$file_path",
	"selector": "source.sql"
}
```

You could also use something like this if you're targeting a Drill cluster:

```json
{
	"cmd": [ "/opt/apache-drill-1.6.0/bin/sqlline", "--color=false", "-u", "jdbc:drill:zk=localhost:2181",  "--run='$file'" ],
	"working_dir": "$file_path",
	"selector": "source.sql"
}
```

Note the changed JDBC string, with the zookeeper specification.

Save it. You'll be prompted for a filename - leave the extension as-is. I changed `untitled` to `localDrillbitQuery`, but you can call it whatever you want.

Now create a new tab, and write a simple query to test:

```
select * from cp.`employee.json` limit 1;
```

Go to Tools > Build System and select the just-created build system.

Now in your tab where you added your query, save, then hit command-B to run the build system.


_Inspired by https://www.youtube.com/watch?v=tPd4m3PLVqU_
