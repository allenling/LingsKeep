amqp的一些了解
===============

amqp协议版本是0-9-1

主要是rabbitmq的实现, 因为rabbitmq对于amqp的实现会有一点点小差别


connection channel
--------------------

connection是真正的tcp连接, 而channel是为了multiplexing一个connection而实现的一个虚拟连接

跟rabbitmq的绝大部分都是用channel, 比如publish, consume等等.

可以类比进程和线程, 一个进程就是一个connection, 然后channel就是线程, 进程至少有一个线程, 并且有一个主线程, 只有一个线程的进程的主线程就唯一的那个线程, 线程是计算逻辑, 所以

进程必须有一个线程

然后connecion至少需要有一个channel来consume, consume是channel来consume.


channel consumer queue
------------------------

.. raw:

                ----start consume---> consumer1 ---> queue1
               |
    channel ---
               |
                ----start consume---> consumer2 ---> queue2
               |
                ----start consume---> consumer3 ---> queue2

每次channel发起start consume的命令, 就是在某个queue上指定出consumer

所以 **channel可以生成很多很多的consumer, 然后每个consumer对应一个queue, 也可以多个consumer消费同一个queue, 同时一个channel可以开启对个consumer来消费多个queue.**

关于consumer的说明: amqp协议-3.1.6

We use the term "consumer" to mean both the client application and the entity that controls how a specific client application receives messages off a message queue.

**When the client "starts a consumer" it creates a consumer entity in the server. When the client "cancels a consumer" it destroys a consumer entity in the

server**. Consumers belong to a single client channel and cause the message queue to send messages asynchronously to the client.

就是说, channel发起starts a consumer这个命令会在server端(这里只的是rabbitmq)创建一个consumer entity, 一个在server中的consumer实例.


basic-consume的实现

.. code-block:: erlang

    consume(short reserved-1, queue-name queue, consumer-tag consumer-tag, no-local no-local, no-ack no-ack, bit exclusive, no-wait no-wait, table arguments)

其中参数必须传入queue-name, 所以consume的时候就是只能指定一个queue进行consume

qos
-----

global
~~~~~~~~

amqp协议中, qos有一个参数global(pika接口中参数名字是all_channels), 可以指定设置当前channel或者设置当前connecion的所有channel, 但是rabbitmq自己

重新解释了这个参数, 这个参数影响的是consumer而不是channel了, 也就是说这个参数为true的时候, 设置的是当前channel的qos, 也就是channel最多接收指定个数

如果global=false, 那么设置的是每一个consumer的qos, 

rabbitmq的qos:

https://www.rabbitmq.com/consumer-prefetch.html, 

Furthermore for many uses it is simply more natural to specify a prefetch count that applies to each consumer.

比如qos=2, global=true的时候, 整个channel只会收到最多2个msg, 如果global=false, 那么如果channel有两个consumer, 那么每一个consumer都会收到2个msg, channel一共收到4个.

下面一个channel发起了两个consumer, 然后分别消费queue1, queue2, 然后queue1的数据格式是1, 2, 3整数, 然后queue2的消息格式是r1, r2, r3这样的格式, qos=2.

.. raw:: 

                ----start consume---> consumer1 ---> queue1(1, 2, 3, 4, ...)
               |
    channel ---
               |
                ----start consume---> consumer2 ---> queue2(r1, r2, r3, r4, ...)

当global=true的时候

输出:

.. code-block:: 

    [*] Waiting for logs. To exit press CTRL+C
    [x] b'1'
    [x] b'2'

当global=false的时候

.. code-block:: 

    [*] Waiting for logs. To exit press CTRL+C
    [x] b'1'
    [x] b'2'
    [x] b'r1'
    [x] b'r2'


区别就是如果global=true的话, rabbitmq只会发送两个msg给channel, 如果global=false的话, 那么每个consumer的prefetch count都是2, 那么一共就有4个

batch or one-by-one
~~~~~~~~~~~~~~~~~~~~~

一旦有msg被ack之后, 那么rabbitmq会发送下一个, 如果是设置了qos, 那么一开始就发送qos个, 然后一旦有一个msg被ack, 那么rabbitmq就接着发下一个, 而不是batch send,

判断条件就是delivery_tag的数字.

https://www.rabbitmq.com/confirms.html

For example, given that there are delivery tags 5, 6, 7, and 8 unacknowledged on channel Ch and channel Ch's prefetch count is set to 4,

RabbitMQ will not push any more deliveries on Ch unless at least one of the outstanding deliveries is acknowledged. **When an acknowledgement frame arrives

on that channel with delivery_tag set to 8, RabbitMQ will notice and deliver one more message.**


round robin?
~~~~~~~~~~~~~

在上面的例子中, qos=2, global=true, 那么channel一次能拿到2个mgs, 这两个msg是都属于一个queue吗?还是一个queue1一个queue2的? **都不一定**

如果ack掉一个msg, 那么下一个msg是优先从同一个queue的msg? 比如假设queue1和queue2都有足够的消息, 那么ack掉一个msg之后, 下一个msg是属于queue1呢还是queue2呢? **都不一定**

下一个msg的规律? 与global有关? 或者没有规律?


close
======

关闭connection之前必须先关闭channel, 否则rabbitmq会报invalid command错误


