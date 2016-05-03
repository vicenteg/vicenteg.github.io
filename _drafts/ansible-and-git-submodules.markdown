---
layout: post
title:  "Multiple Ansible roles and git"
date:   2015-12-04 17:13:23 -0500
categories: ansible git
---

Sometimes you want to build some complex automation, but maybe you want to
ensure reusability by creating very focused roles. When you do that, you might
end up using a lot of roles, composing them to build up to more and more complex functionality. How do you manage that in version control (git) while keeping the roles easy to for people to download and use?

There's a tool called repo that I was introduced to a while back.

Composing roles might be best done with submodules. The advantage of submodules
is that it doesn't involve any other tools, so sharing your roles and playbooks
remains simple and flexible; share the entire repo and all the submodules, or
share just an individual role. And you can do everything you need in git. When
you need to provide instructions on how to get your big, complicated roles
and the needed roles, you only need get someone a couple git commands.

Let's see if we can find a good, solid workflow.

- Start a repo for each role, scaffolded by `ansible-galaxy init`.

The top level of the role as created by `ansible-galaxy init` should be the top
level of the repo.

- Create a directory that will contain the playbook that references the roles. For example:


{% highlight bash %}
mkdir project
cd project; touch my_playbook.yml
{% endhighlight %}

- In the project directory, add your submodules created in step 1 above:

{% highlight bash %}
git submodule init git://.../role1.git roles/role1
git submodule init git://.../role2.git roles/role2
{% endhighlight %}