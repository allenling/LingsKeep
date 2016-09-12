声明consumer
=============

发送basic.on_consumer_ok只是向server声明一个consumer, 消费message还是需要一个loop来监听socket之类的, 而server会将message发给socket, 一般用epoll去socket把message给pull下来.


* celery.component.Hub: 创建poll, 循环

* kombu.connection.Connection.drain_event: 拉取socket中的message

rabbitmq文档中又一段关于获取message的方式, 有push/poll两者.

https://www.rabbitmq.com/tutorials/amqp-concepts.html
 
`Consumers

Storing messages in queues is useless unless applications can consume them. In the AMQP 0-9-1 Model, there are two ways for applications to do this:

Have messages delivered to them ("push API")
Fetch messages as needed ("pull API")

With the "push API", applications have to indicate interest in consuming messages from a particular queue. When they do so, we say that they register a consumer or, simply put, subscribe to a queue. It is possible to have more than one consumer per queue or to register an exclusive consumer (excludes all other consumers from the queue while it is consuming).

Each consumer (subscription) has an identifier called a consumer tag. It can be used to unsubscribe from messages. Consumer tags are just strings.`_

push方式是发送一个consume的amqp frame向server声明了一个consumer, 则server会将message发送给这个consumer(确切的说是这个connection, socket方式), 但是client还是自己去socket拿数据(那是自然)


epoll拉取消息
=============

从reply来查看如何拉取amqp发送过来的消息

查看reply的过程, 可以, reply会建立一个临时的queue, 然后注册call_back, 然后调用drain_event来获取amqp发送过来的message.

.. code-block:: python

   # kombu.pidbox
   class Mailbox(object):
       def _broadcast(self, command, arguments=None, destination=None,
                  reply=False, timeout=1, limit=None,
                  callback=None, channel=None, serializer=None):
           # 省略代码
           # self._collect就是搜集reply, 建立一个consumer监听reply的queue
           if reply_ticket:
               return self._collect(reply_ticket, limit=limit,
                                        timeout=timeout,
                                        callback=callback,
                                        channel=chan)

       def _collect(self, ticket,
                    limit=None, timeout=1, callback=None,
                    channel=None, accept=None):
           if accept is None:
               accept = self.accept
           chan = channel or self.connection.default_channel
           queue = self.reply_queue
           # 建立一个临时的consume
           consumer = Consumer(channel, [queue], accept=accept, no_ack=True)
           responses = []
           unclaimed = self.unclaimed
           adjust_clock = self.clock.adjust

           try:
               return unclaimed.pop(ticket)
           except KeyError:
               pass
           
           # call_back 函数
           def on_message(body, message):
               # ticket header added in kombu 2.5
               header = message.headers.get
               adjust_clock(header('clock') or 0)
               expires = header('expires')
               if expires and time() > expires:
                   return
               this_id = header('ticket', ticket)
               if this_id == ticket:
                   if callback:
                       callback(body)
                   responses.append(body)
               else:
                   unclaimed[this_id].append(body)
           # consumer注册call_back
           consumer.register_callback(on_message)
           try:
               // 循环调用drain_event来获取数据
               with consumer:
                   for i in limit and range(limit) or count():
                       try:
                           # 这里主动去拉取消息
                           self.connection.drain_events(timeout=timeout)
                       except socket.timeout:
                           break
                   return responses
           finally:
               chan.after_reply_message_received(queue.name)

