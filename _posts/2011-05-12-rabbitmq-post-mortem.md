---
layout: post
title: RabbitMQ Chat Post Mortem
---

# {{page.title}} #

<span class="meta">May 12 2011</span>

## Background ##

Last week I started preparing a chapter for the [book](http://bit.ly/rabbitmq+) I'm writting so I had to come up with an example on how to extend RabbitMQ. I was looking for something fairly simple to implement that at the same time proved of value for the reader. Since recently [some](https://github.com/jbrisbin/riak-exchange) [custom](https://github.com/squaremo/rabbitmq-lvc-plugin) [exchanges](https://github.com/jbrisbin/random-exchange) had appeared on the mailing list, I saw this could be a nice example to implemenet, so I went for it.

In case you don't know what [RabbitMQ](http://www.rabbitmq.com/) is, I'll explain it briefly here. RabbitMQ is a messaging server that as such you use it to pass messages around in your applications. You can send event notifications through it; enqueue tasks for background processing; collect logs from many sources and filter them; etc. Considering all that, creating a Chat where RabbitMQ is used as the message router doesn't sound so _crazy_.

For that purpose I created the [Recent History Exchange](https://github.com/videlalvaro/rabbitmq-recent-history-exchange). The purpose of having such custom exchange was to provide the user that joined the chat room with a minimal context of the last 20 messages that passed through the chat room.

So I sat down and started hacking, in about 4 hours I've got the [Chat Repository](https://github.com/videlalvaro/rabbitmq-chat) ready and up on Github. It was very very basic. The frontend part was based on [YakRiak](https://github.com/seancribbs/yakriak), a web chat with a [Riak](https://github.com/basho/riak_kv) backend.

## The Architecture ##

OK, let's assume there's an _architecture_ as such behind this little book example.

First as some one said, I was [cheating](http://www.reddit.com/r/programming/comments/h6aai/hey_reddit_i_wrote_a_chat_app_using_rabbitmq_and/c1swq7w), I wrote all this in Erlang.

The HTTP bits of the chat: the HTML page, the CSS and Javascript was all served by [Misultin](https://github.com/ostinelli/misultin), an Erlang web server. Also I used Misultin as a Websockets server to send the messages to the connected clients.

What happened when a user connects to the chat is this:

1) The server starts a Websockets connection to the user browser.

2) The Websockets [loop](https://github.com/videlalvaro/rabbitmq-chat/blob/master/src/rabbitmq_chat_rest.erl#L69) starts a RabbitMQ consumer. This consumer takes care of creating a _private auto delete queue on the server_. This means __each user has its own queue__ on the server. By making the queues `auto delete`, I've made sure that if the users closed his chat window, then his queue will be automatically deleted by RabbitMQ.

3) All queues where bound to the custom Recent History Exchange. As I said before, every user that connected to the chat got the last 20 messages thanks to this exchange. This exchange acted like a single chat room where every user was logged in.

4) Every time a user sent a message to the server, that message got __fanout'ed to every queue bound to the exchange__. Depending on the amount of queues created on the server, this pattern of message distribution can get very very slow. When the chat got __500~ users online__ and __10~__ messages published per second, that mean the server was routing each of those then messages to each of the __500~ queues__.

5) Once the broker delivered the message to the private queue, the consumer took care of sending it back to the Websocket process that started it, and from there it was sent to the browser.

And that was basically it.

Some people raised valid complains about the chat interface having some CSS problems. Also I was too lazy to implement a rooster kind of feature. The goal of the chat was to test the proof of concept that was the Recent History Exchange. Just that, nothing more. It wasn't meant to become a full blown chat. Perhaps in the future I could try to build a more interesting chat, but not in this case.

## The Setup ##

To deploy the server I've got a __vps200__ from [Servergrove](http://www.servergrove.com/vps): 1 GB of RAM and 2 cores. I asked for a bare bones installation of Ubuntu 10.4 on it. The account set up was really easy and their admin panel is pretty slick.

Once I've got the login details I ssh'ed into the server and compiled Erlang from source. From there it was a matter of __20 seconds to get RabbitMQ up and running__. I used the latest [Generic Unix Distribution](http://www.rabbitmq.com/releases/rabbitmq-server/v2.4.1/rabbitmq-server-generic-unix-2.4.1.tar.gz) to get it installed.

<iframe src="http://player.vimeo.com/video/10254034?title=0&amp;byline=0&amp;portrait=0" width="400" height="300" frameborder="0"></iframe><p><a href="http://vimeo.com/10254034">Installing RabbitMQ</a> from <a href="http://vimeo.com/user1169087">Alvaro Videla</a> on <a href="http://vimeo.com">Vimeo</a>.</p>

After I've got the server installed, I've added the Recent History Exchange and the [Management Plugin](http://www.rabbitmq.com/plugins.html#rabbitmq-management). The latter so I could easily monitor the status of the server directly from the web.

## Deployment Method ##

To deploy the code I used Github to get the source code for the application. Whenever there was a bug, or a new feature I wanted to add, like logging, what I did was to develop in my machine, pushed to Github, pulled on the server, compiled and restarted the Chat.

## Going Live ##

To get the Chat up and running I didn't do anything special. Erlang compiled with default options. RabbitMQ started with default options. Nothing fancy. There was no clustering set up or any kind of failover.

Once I've got the chat up and running. I've showed it to some friends on Twitter to be sure it wasn't broken. The next day I decided to ask [Reddit](http://www.reddit.com/r/programming/comments/h6aai/hey_reddit_i_wrote_a_chat_app_using_rabbitmq_and/) to try to take down the chat. There the fun started.

Then Reddit users started to do some nasty things against the poor ol' little server. Some guy ran Apache Benchmark against it. Here are the [results](http://pastebin.com/ecVX7xtv): 100000 requests in 43~ seconds. I would say that's not bad for such a simple and _smalish_ setup.

Besides from all the obscenity that you can get when you leave the internet without moderation, some funny messages passed around, like:

> whats rabbitmq anyway, some weird language bindings to zmq?

Not happy with that another developer –[@danopia](http://twitter.com/#!/danopia)– created some Ruby [gists](https://gist.github.com/5c4769f21486a3c34d6a/442139759a20ed79ec2f9d805de156f2d2216cec) to act as bots that were constantly posting messages to chat.

[@michaelklishin](http://twitter.com/#!/michaelklishin) further developed the concept and created some bots that opened hundreds of idle websockets connections to the server. They weren't sending messages, but they triggered the creation of their own queues on the server which added more workload to RabbitMQ. All these scripts now have their own [Github repository](https://github.com/michaelklishin/rmq-chat-load-testing-scripts).

After some hours it reached __350~__ users and the server crashed. There was no flow control… no throttle, no nothing… the internet… free to take down a poor web chat baby.

I took a look at the Erlang crash dump and found out that the server somehow ran out of memory. [@michaelklishin](http://twitter.com/#!/michaelklishin) suggested to change the RabbitMQ `memory_high_watermark` settings. The server was using __400MB__ top. The new settings allowed the server to go up to __800MB__ if needed.

From that point it kept running pretty smoothly and I went to sleep when there were 530~ users online. How did I knew that… pretty simple. I used a simple shell command to count the number of private queues on the broker:

{% highlight sh %}
while true; do ./rabbitmqctl list_queues | grep amq. | wc -l; uptime; sleep 5s; done
{% endhighlight %}

It uses the `rabbitmqctl` script to list the queues, it `greps` the queues whose name starts with `amq.` and pipes that to the `wc` command to get the number of lines. As simple as it can get. Poor man stats.

![RabbitMQ routing 5000~ msgs/sec](/images/5000_msgs.png)

As you can see there, the server was routing more than __5000 messages__ per second using __86MB of RAM__.

As the time passed I started to get tired since it was around 4:00AM, Sunday morning. I took some screenshots of the current broker status, made a small screen cast out of what was happening and went to bed.

This was the video showcasing the server _load average_, number of routed messages, channels opened to the broker and queues:

<iframe src="http://player.vimeo.com/video/23424752?title=0&amp;byline=0&amp;portrait=0" width="400" height="300" frameborder="0"></iframe><p><a href="http://vimeo.com/23424752">RabbitMQ Chat</a> from <a href="http://vimeo.com/user1169087">Alvaro Videla</a> on <a href="http://vimeo.com">Vimeo</a>.</p>

The load average was pretty low. The same can be said of the memory used by the broker. It seems I was really cheating by using Erlang :D

After a day I took a look at google analytics, the response from Reddit was quite impressive:

![More than 7000~ visits in one day](/images/analytics.png)

The chat got more than 7000 visits in one day. You can see that nearly a thousand of visits happened between 0:00 and 1:00 AM.

On Monday I decided to take the chat offline since the test was finished and people started to abuse the Gravatars, impersonating other persons. The chat didn't have any kind of authentication, it didn't store any messages nor user information. So it was not possible to check that the people using Gravatar was legitimate.

## What I've learned ##

Apart that I've used RabbitMQ in production before, this scenario showed again that is a pretty robust software. Used with the default settings it crashed only because the memory allocated to it was reduced. It's worth noting here that is the Erlang Virtual Machine what crashed when it runs out of memory.

Another point is that Erlang has some serious tools to keep a system alive. Apart from all the cool OTP behaviors like `supervisors` and `gen_servers`, you get for free tools like:

- Standard report logger
- Report browser, with filters by date, regex. etc. That makes it simple to browse through logs.
- Etop. Is like the unix top command but for Erlang Virtual Machines.

Also it has things like and embedded database: [Mnesia](http://www.erlang.org/doc/apps/mnesia/index.html). Having a database that understand the language you are building your applications with simplifies the process a lot.

Then, thanks to the Erlang Distribution System, is pretty easy to hook into a running Erlang Virtual Machine and send messages to the running processes there. It's also possible to perform RPC invocations. That simplifies the process of restarting some parts of the application while keeping others alive.

Finally a big thanks to the reddit/programming folks for joining the _destroy the chat_ party last Saturday.