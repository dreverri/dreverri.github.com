---
layout: post
title: Riak Basics
---

[riak]: http://riak.basho.com/
[erlang]: http://www.erlang.org/download.html
[consult]: http://erldocs.com/otp_src_R13B/kernel/file.html#consult/1
[basic_setup]: http://hg.basho.com/riak/src/tip/doc/basic-setup.txt
[heart]: http://erldocs.com/otp_src_R13B/kernel/heart.html
[erl]: http://www.erlang.org/doc/man/erl.html
[cast]: http://erldocs.com/otp_src_R13B/kernel/rpc.html#cast/4
[stop]: http://erldocs.com/otp_src_R13B/erts/init.html#stop/0
[join_cluster]: http://hg.basho.com/riak/src/tip/src/riak_startup.erl#cl-18

# {{ page.title }}

## Clone and compile

The first step to getting started with [Riak][riak] is to grab the source and compile it. You will need [Erlang][erlang] installed on your system.

{% highlight bash %}
mkdir getting_started
cd getting_started
hg clone http://hg.basho.com/riak/
cd riak
make
{% endhighlight %}

We created a new project directory, "getting_started", to contain our work.

## Prepare node configuration file

Next, we'll prepare a configuration for our first node. Configuration files are plain text files that contain Erlang terms separated by '.'. These types of files can be read directly by Erlang with [file:consult/1][consult].

We'll create a new folder to house our nodes.

{% highlight bash %}
getting_started/
-riak/
-nodes/
--node1/
---config.erlenv
{% endhighlight %}

Within "config.erlenv" we input the following:

{% highlight erlang %}
{cluster_name, "my_cluster"}.
{ring_state_dir, "priv/ringstate"}.
{ring_creation_size, 64}.
{gossip_interval, 60000}.
{storage_backend, riak_ets_backend}.
{riak_cookie, riak_demo_cookie}.
{riak_nodename, node1}.
{riak_hostname, "127.0.0.1"}.
{riak_web_ip, "127.0.0.1"}.
{riak_web_port, 8098}.
{jiak_name, "jiak"}.
{% endhighlight %}

Configuration parameters expected by Riak are described [here][basic_setup]. I've been reviewing the docs in the source rather than those on the website since the published docs tend to lag behind a bit.

In the above config you will notice I left out the "riak_heart_command". This option is only needed when using [heart][heart] and is only parsed by the Riak shell scripts (e.g. start-fresh.sh). For this post we are going to start our nodes directly with the erl command.

## Start the node

With our config file in place we can start our node. For this post we are going to start the node directly with erl and from within the node directory. Since we are starting from within the node directory the log and ringstate information for this node will be output into the node directory.

Here is the command to start the node from within "nodes/node1":

{% highlight bash %}
erl -connect_all false \
-detached \
-pa ../../riak/deps/*/ebin \
-pa ../../riak/ebin \
-name node1@127.0.0.1 \
-setcookie riak_demo_cookie \
-run riak start config.erlenv
{% endhighlight %}

The options passed to erl are described [here][erl]. The above command tells erl to start a new Erlang runtime system with the name "node1@127.0.0.1", the cookie "riak_demo_cookie", and the ebin directories of Riak and it's dependencies (webmachine, mochiweb) included in the code path. The command also tells erl to run "riak:start\[config.erlenv\]".

## Query the node

Querying Riak will be covered in a separate post.

## Stop the node

To stop the node we'll use [rpc:cast/4][cast] to issue an [init:stop/0][stop] on the running node.

{% highlight bash %}
erl -noshell \
-name node_stopper@127.0.0.1 \
-setcookie riak_demo_cookie \
-eval "net_adm:ping('node1@127.0.0.1'), 
rpc:cast('node1@127.0.0.1', init, stop, [])" \
-run init stop
{% endhighlight %}

This method will also work for nodes with a corresponding [heart][heart].

## Start and cluster two nodes

So far we know how to setup a node configuration file, start the node, and stop the node. Now we want to setup two nodes and cluster them together. Let's copy the "node1" folder to "node2" so was end up with the following directories:

{% highlight bash %}
nodes/
    node1/
        config.erlenv
    node2/
        config.erlenv
{% endhighlight %}

Edit "nodes/node2/config.erlenv" and change "riak_nodename" and "riak_web_port".

{% highlight erlang %}
{cluster_name, "my_cluster"}.
{ring_state_dir, "priv/ringstate"}.
{ring_creation_size, 64}.
{gossip_interval, 60000}.
{storage_backend, riak_ets_backend}.
{riak_cookie, riak_demo_cookie}.
{riak_nodename, node2}.
{riak_hostname, "127.0.0.1"}.
{riak_web_ip, "127.0.0.1"}.
{riak_web_port, 8099}.
{jiak_name, "jiak"}.
{% endhighlight %}

Updating "riak_nodename" will ensure the Erlang node names will not collide. Updating "riak_web_port" will ensure the RESTful web interface for each node will not conflict with one another.

Using the start command from above we'll start both nodes.

From within "nodes/node1":
{% highlight bash %}
erl -connect_all false \
-detached \
-pa ../../riak/deps/*/ebin \
-pa ../../riak/ebin \
-name node1@127.0.0.1 \
-setcookie riak_demo_cookie \
-run riak start config.erlenv
{% endhighlight %}

From within "nodes/node2":
{% highlight bash %}
erl -connect_all false \
-detached \
-pa ../../riak/deps/*/ebin \
-pa ../../riak/ebin \
-name node2@127.0.0.1 \
-setcookie riak_demo_cookie \
-run riak start config.erlenv
{% endhighlight %}

Now we can join "node2" to "node1" by issuing [riak_startup:join_cluster/1][join_cluster] with [rpc:cast/4][cast].

{% highlight bash %}
erl -noshell \
-name node_join@127.0.0.1 \
-setcookie riak_demo_cookie \
-eval "net_adm:ping('node2@127.0.0.1'), 
rpc:cast('node2@127.0.0.1', riak_startup, join_cluster, ['node1@127.0.0.1'])" \
-run init stop
{% endhighlight %}
<br />

## Confirm cluster from a third node

To ensure we've successfully clustered the nodes we'll start a third node, join the cluster, and list the available nodes.

{% highlight bash %}
cp -r nodes/node1 nodes/node3
vi nodes/node3/config.erlenv
{% endhighlight %}

<br />

{% highlight erlang %}
{cluster_name, "my_cluster"}.
{ring_state_dir, "priv/ringstate"}.
{ring_creation_size, 64}.
{gossip_interval, 60000}.
{storage_backend, riak_ets_backend}.
{riak_cookie, riak_demo_cookie}.
{riak_nodename, node3}.
{riak_hostname, "127.0.0.1"}.
{riak_web_ip, "127.0.0.1"}.
{riak_web_port, 8100}.
{jiak_name, "jiak"}.
{% endhighlight %}

<br />

{% highlight bash %}
cd nodes/node3
erl -connect_all false \
-pa ../../riak/deps/*/ebin \
-pa ../../riak/ebin \
-name node3@127.0.0.1 \
-setcookie riak_demo_cookie \
-run riak start config.erlenv
{% endhighlight %}

If you look closely at the erl command for "node3" you'll notice that we've removed the "-detached" flag. This means the node will be attached to your console. Once the node finishes loading you can press enter to gain access to the erlang shell. From the erlang shell we can connect to node1 and list the nodes within the cluster.

{% highlight erlang %}
=PROGRESS REPORT==== 9-Nov-2009::03:15:38 ===
         application: riak
          started_at: 'node3@127.0.0.1'

(node3@127.0.0.1)1> riak_startup:join_cluster('node1@127.0.0.1').

=PROGRESS REPORT==== 9-Nov-2009::03:15:52 ===
          supervisor: {local,inet_gethost_native_sup}
             started: [{pid,<0.271.0>},{mfa,{inet_gethost_native,init,[[]]}}]

=PROGRESS REPORT==== 9-Nov-2009::03:15:52 ===
          supervisor: {local,kernel_safe_sup}
             started: [{pid,<0.270.0>},
                       {name,inet_gethost_native_sup},
                       {mfa,{inet_gethost_native,start_link,[]}},
                       {restart_type,temporary},
                       {shutdown,1000},
                       {child_type,worker}]

ok
(node3@127.0.0.1)2> nodes().
['node2@127.0.0.1','node1@127.0.0.1']
(node3@127.0.0.1)3>
{% endhighlight %}
<br />

## Conclusion

Above, we successfully configured three Riak nodes and clustered them together. Soon, we'll look at creating buckets, storing keys, and walking links.

I've implemented the steps above as rake tasks and made them available on [github](http://github.com/dreverri/riak_basics).