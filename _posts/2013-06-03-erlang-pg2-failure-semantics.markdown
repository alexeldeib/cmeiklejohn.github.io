---
layout: post
title:  "Erlang pg2 Failure Semantics"
date:   2013-06-03 17:20:16
categories: erlang
---

I recently spent some time writing an Erlang/OTP application where I
needed to broadcast messages to a series of blocking
[`webmachine`][webmachine] resources, which would return multipart
responses to the user derived from the messages each process received.
I implemented this prototype using the built in [`pg2`][pg2]
distributed named process group registry provided by OTP, as the
webmachine resources would be split up amongst a series of nodes.

However, unsure of how `pg2` actually performs during node
unavailability and cluster topology changes, as well as reports of
problems due to [race conditions][races] and [partitions][partitions], I
set out to figure out exactly what the failure semantics of `pg2`
actually are.

# What is `pg2`?

`pg2` is a distributed named process group registry, which has replaced
the experimental process group registry [`pg`][pg].  One of the major
departures from `pg` is that rather than being able to message a group
of processes, `pg2` will only return a process listing, and messaging is
left up to the user.  This drastically simplifies the domain, as the
library no longer needs to worry about partial writes to the group, or
failed message sends due to processes being located across a partition,
before the Erlang distribution protocol determines the node is
unreachable and times out the connection.

# So, how does `pg2` work?

First, we can register a process group on one node, and see the results
on the clustered node.

We'll run these operations in a [`riak_core`][riak_core] application,
with two clustered nodes `riak_pg1`, `riak_pg2`.

{% highlight erlang %}
(riak_pg1@127.0.0.1)3> pg2:start().
{ok,<0.496.0>}
(riak_pg1@127.0.0.1)1> pg2:create(group).
ok
(riak_pg1@127.0.0.1)2>
{% endhighlight %}

Getting the members, then returns the membership list containing no
processes, rather than throwing `{error, no_such_group}`.

{% highlight erlang %}
(riak_pg2@127.0.0.1)3> pg2:start().
{ok,<0.463.0>}
(riak_pg2@127.0.0.1)2> pg2:get_members(group).
[]
(riak_pg2@127.0.0.1)3>
{% endhighlight %}

However, you can see interesting behaviour if you attempt to use `pg2`,
prior to starting the `pg2` application, as it's started on demand.

{% highlight erlang %}
(riak_pg1@127.0.0.1)1> pg2:create(group).
ok
(riak_pg1@127.0.0.1)2>
{% endhighlight %}

{% highlight erlang %}
(riak_pg2@127.0.0.1)1> pg2:get_members(group).
{error,{no_such_group,group}}
(riak_pg2@127.0.0.1)2> pg2:get_members(group).
[]
(riak_pg2@127.0.0.1)3>
{% endhighlight %}

`pg2` also, maybe counterintuitively, allows processes to join the group
multiple times, which is something to be aware of when leveraging `pg2`
as a publish/subscribe mechanism.

{% highlight erlang %}
(riak_pg1@127.0.0.1)26> pg2:create(group2).
ok
(riak_pg1@127.0.0.1)27> pg2:join(group2, self()).
ok
(riak_pg1@127.0.0.1)28> pg2:join(group2, self()).
ok
(riak_pg1@127.0.0.1)29> pg2:get_members(group2).
[<0.2933.0>,<0.2933.0>]
(riak_pg1@127.0.0.1)30>
{% endhighlight %}

# First, how does `pg2` handle a node addition?

First, register a group and add a process to it.

{% highlight erlang %}
(riak_pg1@127.0.0.1)2> pg2:create(group).
ok
(riak_pg1@127.0.0.1)3> pg2:join(group, self()).
ok
(riak_pg1@127.0.0.1)4> pg2:get_members(group).
[<0.494.0>]
{% endhighlight %}

{% highlight erlang %}
(riak_pg2@127.0.0.1)2> pg2:get_members(group).
[<12230.494.0>]
{% endhighlight %}

Then, start a third node and register two groups, one with the same
name, and one with a different name.

{% highlight erlang %}
(riak_pg3@127.0.0.1)3> pg2:create(group).
ok
(riak_pg3@127.0.0.1)4> pg2:create(group2).
ok
(riak_pg3@127.0.0.1)5> pg2:join(group, self()).
ok
(riak_pg3@127.0.0.1)6> pg2:join(group2, self()).
ok
(riak_pg3@127.0.0.1)7> pg2:get_members(group).
[<0.445.0>]
(riak_pg3@127.0.0.1)8> pg2:get_members(group2).
[<0.445.0>]
(riak_pg3@127.0.0.1)9>
{% endhighlight %}

Now, join the node to the cluster.

{% highlight erlang %}
(riak_pg1@127.0.0.1)5> pg2:get_members(group).
[<12231.445.0>,<0.494.0>]
(riak_pg1@127.0.0.1)6> pg2:get_members(group2).
[<12231.445.0>]
{% endhighlight %}

{% highlight erlang %}
(riak_pg2@127.0.0.1)3> pg2:get_members(group).
[<12231.445.0>,<12230.494.0>]
(riak_pg2@127.0.0.1)4> pg2:get_members(group2).
[<12231.445.0>]
{% endhighlight %}

{% highlight erlang %}
(riak_pg3@127.0.0.1)9> pg2:get_members(group).
[<0.445.0>,<14219.494.0>]
(riak_pg3@127.0.0.1)10> pg2:get_members(group2).
[<0.445.0>]
{% endhighlight %}

Everything resolves nicely.

# So, how does `pg2` handle a partition?

{% highlight erlang %}
(riak_pg1@127.0.0.1)12> pg2:join(group, self()).
ok
(riak_pg1@127.0.0.1)13> pg2:get_members(group).
[<0.2933.0>]
{% endhighlight %}

{% highlight erlang %}
(riak_pg2@127.0.0.1)6> pg2:join(group, self()).
ok
(riak_pg2@127.0.0.1)7> pg2:get_members(group).
[<0.2915.0>,<12230.2933.0>]
{% endhighlight %}

{% highlight erlang %}
(riak_pg1@127.0.0.1)14> pg2:get_members(group).
[<12270.2915.0>,<0.2933.0>]
{% endhighlight %}

At this point, we suspend the `riak_pg1@127.0.0.1` node, and wait the
net tick time interval for the node to be detected as down.

{% highlight erlang %}
[error] ** Node 'riak_pg1@127.0.0.1' not responding **
** Removing (timedout) connection **
(riak_pg2@127.0.0.1)18> pg2:get_members(group).
[<0.2915.0>]
(riak_pg2@127.0.0.1)19>)
{% endhighlight %}

And, once the node is restored the process returns.

{% highlight erlang %}
(riak_pg2@127.0.0.1)19> pg2:get_members(group).
[<0.2915.0>,<12230.2933.0>]
(riak_pg2@127.0.0.1)20>
{% endhighlight %}

# So, how does this work?

When a remote `pid` is registered, and its state transferred to another
node, the `pg2` registration system sets up an Erlang monitor to that
node. When that node happens to become unavailable, processes located on
remote nodes are removed from the process group.  When the node returns,
it transfers its state, repopulating the process list with the processes
which are now available.

# What happens when you attempt to delete a group during a partition?

First, the `delete` request will block until the unavailable node times
out.  At that point the group will be considered deleted.

{% highlight erlang %}
(riak_pg1@127.0.0.1)4> pg2:delete(group).
21:39:09.898 [error] ** Node 'riak_pg2@127.0.0.1' not responding **
** Removing (timedout) connection **
ok
(riak_pg1@127.0.0.1)5>
(riak_pg1@127.0.0.1)6> pg2:get_members(group).
{error,{no_such_group,group}}
{% endhighlight %}

However, when the partition heals, the second node will re-create the
group on the primary.

{% highlight erlang %}
(riak_pg1@127.0.0.1)7> pg2:get_members(group).
[]
{% endhighlight %}

{% highlight erlang %}
(riak_pg2@127.0.0.1)17> pg2:get_members(group).
[]
{% endhighlight %}

# Additionally...

We also observe the same timeout behaviour when performing `leave` and
`create` operations, however, once the disconnected node is reconnected
the correct state is propagated out.

When performing a `create` and `join` on either side of a partition, we
can also observe that when the partition heals, the updates are
performed correctly.

# What about `create` with a three node cluster?

Here's the state before the partition is healed.

{% highlight erlang %}
(riak_pg3@127.0.0.1)17> pg2:create(group3).
22:12:10.669 [error] ** Node 'riak_pg2@127.0.0.1' not responding **
** Removing (timedout) connection **
22:12:10.669 [error] ** Node 'riak_pg1@127.0.0.1' not responding **
** Removing (timedout) connection **
ok
(riak_pg3@127.0.0.1)18> pg2:join(group3, self()).
ok
(riak_pg3@127.0.0.1)19> pg2:get_members(group3).
[<0.445.0>]
{% endhighlight %}

At this point node 2 is completely disconnected from the rest of the
nodes.

{% highlight erlang %}
(riak_pg2@127.0.0.1)23> pg2:create(group3).
ok
(riak_pg2@127.0.0.1)24> pg2:get_members(group3).
[]
(riak_pg2@127.0.0.1)25> pg2:join(group3, self()).
ok
(riak_pg2@127.0.0.1)26> pg2:get_members(group3).
[<0.451.0>]
{% endhighlight %}

Here's the state after the partition is healed.

{% highlight erlang %}
(riak_pg1@127.0.0.1)19> pg2:get_members(group3).
[<14116.445.0>,<12230.451.0>]
{% endhighlight %}

{% highlight erlang %}
(riak_pg2@127.0.0.1)27> pg2:get_members(group3).
[<14254.445.0>,<0.451.0>]
{% endhighlight %}

{% highlight erlang %}
(riak_pg3@127.0.0.1)20> pg2:get_members(group3).
[<0.445.0>,<13986.451.0>]
{% endhighlight %}

# What about `leave` with a three node cluster?

{% highlight erlang %}
(riak_pg1@127.0.0.1)3> pg2:join(group, self()).
ok
(riak_pg1@127.0.0.1)4> pg2:get_members(group).
[<0.456.0>]
{% endhighlight %}

{% highlight erlang %}
(riak_pg2@127.0.0.1)2> pg2:get_members(group).
[<12230.456.0>]
{% endhighlight %}

{% highlight erlang %}
(riak_pg3@127.0.0.1)2> pg2:get_members(group).
[<12230.456.0>]
{% endhighlight %}

Then we leave during the partition, which can only happen on the node
running the process.

{% highlight erlang %}
(riak_pg1@127.0.0.1)5> pg2:leave(group, self()).
14:33:08.769 [error] ** Node 'riak_pg3@127.0.0.1' not responding **
** Removing (timedout) connection **
14:33:08.769 [error] ** Node 'riak_pg2@127.0.0.1' not responding **
** Removing (timedout) connection **
ok
(riak_pg1@127.0.0.1)6> pg2:get_members(group).
[]
{% endhighlight %}

And, we see that the updated state is propagated.

{% highlight erlang %}
(riak_pg2@127.0.0.1)3> pg2:get_members(group).
[]
(riak_pg2@127.0.0.1)4>
{% endhighlight %}

{% highlight erlang %}
(riak_pg3@127.0.0.1)3> pg2:get_members(group).
[]
(riak_pg3@127.0.0.1)4>
{% endhighlight %}

# Conclusion

While `pg2` has some interesting semantics regarding processes able to
register themselves in the process group, and with groups resurrecting
themselves during the healing of a partition, the failure conditions
during cluster membership and partitions appears to be straightforward,
and deterministic.

I'm hopeful that I didn't miss any obvious failure scenarios.  Feedback
welcome and greatly appreciated!

__Updated 2013-06-04:__ Added details on how node addition is handled in
`pg2`.

_View this story on [Hacker News][hn]._

[webmachine]: http://github.com/basho/webmachine
[riak_core]:  http://github.com/basho/riak_core
[pg]:         http://erlang.org/doc/man/pg.html
[pg2]:        http://erlang.org/doc/man/pg2.html
[races]:      http://erlang.org/pipermail/erlang-questions/2008-April/034161.html
[partitions]: http://erlang.org/pipermail/erlang-questions/2010-April/050992.html
[hn]:         https://news.ycombinator.com/item?id=5885987
