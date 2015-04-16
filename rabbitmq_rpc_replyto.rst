Rabbitmq: Direct reply-to
==========================

在celery中，如果使用了flower，总是会出现一个名为xxx.reply.celery.pidbox的queue。

我们知道celery中的monitor是基于event，远程命令也一样。当flower发送远程命令给worker之后，worker的回复就通过那个queue传递给flower，但是那个queue在rabbitmq的监控页面却看不到任何的income和outcome，

并且，肯定是flower消费该queue的message，但是同样的，在rabbitmq的监控页面也没显示consumer。

猜想reply-to这个功能应该是amqp或者rabbitmq自己实现的。

关于reply-to行为和特点
----------------------
http://www.rabbitmq.com/direct-reply-to.html

The direct reply-to feature allows RPC clients to receive replies directly from their RPC server, without going through a reply queue. ("Directly" here still means going through AMQP and the RabbitMQ server; there is no separate network connection between RPC client and RPC server.)

Usage:

* Consume from the **pseudo-queue** amq.rabbitmq.reply-to in no-ack mode. There is no need to declare this "queue" first, although the client can do so if it wants.

* Set the reply-to property in their request message to amq.rabbitmq.reply-to

The RPC server will then see a reply-to property with a generated name. It should publish to the default exchange ("") with the routing key set to this value (i.e. just as if it were sending to a reply queue as usual). **The message will then be sent straight to the client consumer.**

Rabbitmq的Direct Replyto是用于RPC（Remote procedure call）
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

https://www.rabbitmq.com/tutorials/tutorial-six-python.html
