Inspect in Celery
===================

celery中, 每发送一个inspect命令都是会回复一个reply的, 而一些control命令是不需要reply的, 需不需要reply是通过mailbox._broadcast中的reply参数指定.

发送命令都是使用celery.app.control.Control对应的方法来发送, 例如celery.app.control.Control.enable_events(不需要reply), celery.app.control.Control.state(需要reply).
    
1. flower是通过注册成为一个celery的命令来启动,  定义在flower.command.FlowerCommand(Command), celery会传入一个celery app, 之后会生成一个flower.app.Flower tornado对象, 启动这个对象. flower的所有功能都在该对象中.

.. code-block:: python

   class FlowerCommand(Command):
       # self.app是Command初始化的时候生成的celery app.
       # Flower对象是flower.app.Flower
       lower = Flower(capp=self.app, ...)
       # 省略代码
       # 启动flower
       try:
           flower.start()
       except (KeyboardInterrupt, SystemExit):
           pass

2. flower.app.Flower启动的时候, 先发送enable_events命令, 以及一些inspect命令, 包括state, conf等.enable_event是没隔一段时间发一次, 而inspect命令则是启动的时候发一次, 之后就不发了.

.. code-block:: python

   class Flower(tornado.web.Application):
       
       def start(self):
           # 开始event循环, 每隔一段时间发送一次.
           self.events.start()
           # 发送inspect命令
           self.io_loop.add_future(
               control.ControlHandler.update_workers(app=self),
               callback=lambda x: logger.debug('Successfully updated worker cache'))

3. 循环发送enable_events是flower.events.Events

.. code-block:: python

   class Events(threading.Thread):

       def __init__(...):
           # 注册一个定时任务, 执行self.no_enable_events
           self.timer = PeriodicCallback(self.on_enable_events,
                                 self.events_enable_interval)

       def start(self):
           threading.Thread.start(self)
           # Celery versions prior to 2 don't support enable_events
           # 启动定时任务
           if self.enable_events and celery.VERSION[0] > 2:
               self.timer.start()
   
       def on_enable_events(self):
           # 定时任务就是循环发送enable_events命令
           # Periodically enable events for workers
           # launched after flower
           try:
               self.capp.control.enable_events()
           except Exception as e:
               logger.debug("Failed to enable events: '%s'", e)

4. flower.control.ControlHandler.update_workers发送inspec 命令
    
.. code-block:: python

     class ControlHandler(BaseHandler):
                     
       @classmethod
       @gen.coroutine
       def update_workers(cls, app, workername=None):

       # 检测所有的inspect命令, 并发送(delay)
       inspect = app.capp.control.inspect(
       timeout=timeout, destination=destination)
       for method in cls.INSPECT_METHODS:
           futures.append(app.delay(getattr(inspect, method)))

5. reply or not

不管是enable_events或者inspect命令,都是用celery.control.Control来发送的

.. code-block:: python

     class Inspect(object):
         def _prepare(self, reply):
             if not reply:
                 return
             by_node = flatten_reply(reply)
             if self.destination and \
                     not isinstance(self.destination, (list, tuple)):
                 return by_node.get(self.destination)
             return by_node

         def _request(self, command, ...):
             # 这里也是使用广播, 但是手动设置reply=True
             return self._prepare(self.app.control.broadcast(
                 command,
                 arguments=kwargs,
                 destination=self.destination,
                 callback=self.callback,
                 connection=self.connection,
                 limit=self.limit,
                 timeout=self.timeout, reply=True,
             ))

     class Control(object):
         Mailbox = Mailbox


         @cached_property
         def inspect(self):
             # 这里的Inspect是celery.app.control.Inspect类
             return self.app.subclass_with_self(Inspect, reverse='control.inspect')

         def disable_events(self, destination=None, ...:
             """Tell all (or specific) workers to disable events."""
             #  这里直接广播, reply默认是False
             # 广播在下面
             return self.broadcast('disable_events', {}, destination, ...)

         def broadcast(self, command, arguments=None, destination=None,
             connection=None, reply=False, timeout=1, limit=None,
             callback=None, channel=None, ...):

             # 这里广播调用的是kombu.pidbox.Mailbox._broadcast
             with self.app.connection_or_acquire(connection) as conn:
                 arguments = dict(arguments or {}, ...)
                 return self.mailbox(conn)._broadcast(
                     command, arguments, destination, reply, timeout,
                     limit, callback, channel=channel,
                 )

6. broadcast中需要reply, 则在message中加入reply ticket, kombu.pidbox.Mailbox
 
.. code-block:: python

   class Mailbox(object):
       def _broadcast(self, command, arguments=None, destination=None,
              reply=False, timeout=1, limit=None,
              callback=None, channel=None, serializer=None):
           if destination is not None and not isinstance(destination, (list, tuple)):
               raise ValueError(
               'destination must be a list/tuple not {0}'.format(
                   type(destination)))

           arguments = arguments or {}
           # reply_ticket是一个uuid
           reply_ticket = reply and uuid() or None

7. collect reply

广播之后就直接搜集reply了
   
.. code-block:: python

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


8. 执行命令并且reply
   
   上面是flower这些monitor广播一些命令, 并且搜集reply. 而在worker端, 收到命令, 执行命令, 发送reply. 命令的实现是在celery.worker.control中.

   8.1. worker启动pidbox来消费command message.

   .. code-block:: python

       # celery.worker.pidbox

       class Pidbox(object):

           def on_message(self, body, message):
               # 收到命令之后, 执行命令
               self._forward_clock()
               try:
                   self.node.handle_message(body, message)
               except KeyError as exc:
                   error('No such control command: %s', exc)
               except Exception as exc:
                   error('Control command error: %r', exc, exc_info=True)
                   self.reset()

           def start(self, c):
               self.node.channel = c.connection.channel()
               self.consumer = self.node.listen(callback=self.on_message)
               self.consumer.on_decode_error = c.on_decode_error


   8.2. node去寻找命令, 并且发送reply

   .. code-block:: python

       # kombu.pidbox.Node

       class Node(object):

           def handle_message(self, body, message=None):
               destination = body.get('destination')
               if message:
                   self.adjust_clock(message.headers.get('clock') or 0)
               if not destination or self.hostname in destination:
                   # dispatch方法就是负责寻找命令并reply
                   return self.dispatch(...)

           def dispatch(self, method, arguments=None,
                        reply_to=None, ticket=None, ...):
               # 省略代码
               # 这里handle就是celery.worker.control中的命令
               try:
                   reply = handle(method, kwdict(arguments))
               except SystemExit:
                   raise
               except Exception as exc:
                   error('pidbox command error: %r', exc, exc_info=1)
                   reply = {'error': repr(exc)}
               
               # 上面说过, inspect的命令都是需要reply的, 而一些events命令都是不需要reply的.
               if reply_to:
                   self.reply({self.hostname: reply},
                              exchange=reply_to['exchange'],
                              routing_key=reply_to['routing_key'],
                              ticket=ticket)
               return reply    def reply(self, data, exchange, routing_key, ticket, **kwargs):

           # 调用mailbox的reply
           def reply(self, data, exchange, routing_key, ticket, **kwargs):
               self.mailbox._publish_reply(data, exchange, routing_key, ticket,
                                    channel=self.channel,
                                    serializer=self.mailbox.serializer)

       class Mailbox(object):
           # reply发送到目标queue
           def _publish_reply(self, reply, exchange, routing_key, ticket,
                       channel=None, **opts):
               chan = channel or self.connection.default_channel
               exchange = Exchange(exchange, exchange_type='direct',
                                   delivery_mode='transient',
                                   durable=False)
               producer = Producer(chan, auto_declare=False)
               try:
                   producer.publish(
                       reply, exchange=exchange, routing_key=routing_key,
                       declare=[exchange], headers={
                           'ticket': ticket, 'clock': self.clock.forward(),
                       },
                       **opts
                   )
               except InconsistencyError:
                   pass   # queue probably deleted and no one is expecting a reply.
