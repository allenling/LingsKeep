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

Each consumer (subscription) has an identifier called a consumer tag. It can be used to unsubscribe from messages. Consumer tags are just strings.`

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

注册call back, 调用call back
=============================

amqp协议通信中是以frame为单位, 并且会有一个method声明, 类似与信号码, 下面类似与basic.xxx, 其中xxx就是方法名, basic表示该方法是basic方法.

amqp中的methods可以在rabbitmq中查找到, 版本是0.9, http://www.rabbitmq.com/amqp-0-9-1-quickref.html

建立连接之后, client发送一个basic.consume的frame, 请求开启一个consumer, 若成功了, server会返回一个basic.consum_ok的frame

.. code-block:: python

    # amqp.channel.Channel
    
    def basic_consume(...):
 
        # 发送basic.consume
        self._send_method((60, 20), args)

        # 等待服务器返回basic.consume_ok
        if not nowait:
            consumer_tag = self.wait(allowed_methods=[
                (60, 21),  # Channel.basic_consume_ok
            ])

        # 注册一个call_back, 当有message过来的时候调用call_back. 这里的call_back是kombu.messaging.Consumer._receive_callback
        self.callbacks[consumer_tag] = callback

rabbitmq文档

basic.consume(short reserved-1, queue-name queue, consumer-tag consumer-tag, no-local no-local, no-ack no-ack, bit exclusive, no-wait no-wait, table arguments) ➔ consume-ok

Support: partial
Start a queue consumer.

This method asks the server to start a "consumer", which is a transient request for messages from a specific queue. Consumers last as long as the channel they were declared on, or until the client cancels them.

之后client不断epoll拉取amqp的frame, 解析看看server向自己发送了什么frame

.. code-block:: python

    # amqp.connection

    class Connection(AbstractChannel):

        def drain_events(self, timeout=None):
            chanmap = self.channels
            # 读取frame
            chanid, method_sig, args, content = self._wait_multiple(
                chanmap, None, timeout=timeout,
            )

            channel = chanmap[chanid]

            if (content and
                    channel.auto_decode and
                    hasattr(content, 'content_encoding')):
                try:
                    content.body = content.body.decode(content.content_encoding)
                except Exception:
                    pass

            # 解析出该frame中是上面amqp方法
            amqp_method = (self._method_override.get(method_sig) or
                           channel._METHOD_MAP.get(method_sig, None))

            if amqp_method is None:
                raise AMQPNotImplementedError(
                    'Unknown AMQP method {0!r}'.format(method_sig))

            # 调用自己已经接口的amqp方法
            if content is None:
                return amqp_method(channel, args)
            else:
                return amqp_method(channel, args, content)

当frame中method为basic.deliver的时候, 证明server在发送message给client, 解包(decode), 然后调用channel中的call_back, 就是上面的kombu.messaging.Consumer._receive_callback
    
.. code-block:: python

    # amqp.channel.Channel._basic_deliver
    class Channel(AbstractChannel):
        def _basic_deliver(self, args, msg):
            consumer_tag = args.read_shortstr()
            delivery_tag = args.read_longlong()
            redelivered = args.read_bit()
            exchange = args.read_shortstr()
            routing_key = args.read_shortstr()

            msg.channel = self
            msg.delivery_info = {
                'consumer_tag': consumer_tag,
                'delivery_tag': delivery_tag,
                'redelivered': redelivered,
                'exchange': exchange,
                'routing_key': routing_key,
            }

            # 获取consumer对应的call_back, 上面的例子就是kombu.messaging.Consumer._receive_callback
            try:
                fun = self.callbacks[consumer_tag]
            except KeyError:
                pass
            else:
                fun(msg)

rabbitmq文档
basic.deliver(consumer-tag consumer-tag, delivery-tag delivery-tag, redelivered redelivered, exchange-name exchange, shortstr routing-key)

Support: full
Notify the client of a consumer message.

This method delivers a message to the client, via a consumer. In the asynchronous message delivery model, the client starts a consumer using the Consume method, then the server responds with Deliver methods as and when messages arrive for that consumer.

最终, kombu.messaging.Consumer._receive_callback会decode数据, 调用receive方法, receive我们定义的on_message方法

而对于task message, 则会调用到celery.worker.__init__.WorkController._process_task去将task分发到pool中


.. code-block:: python

  class WorkerController(object):

      def _process_task(self, req):
          """Process task by sending it to the pool of workers."""
          try:
              # req = celery.worker.job.Request
              # 将任务分发到pool中
              req.execute_using_pool(self.pool)
          except TaskRevokedError:
              try:
                  self._quick_release()   # Issue 877
              except AttributeError:
                  pass
          except Exception as exc:
              logger.critical('Internal error: %r\n%s',
                              exc, traceback.format_exc(), exc_info=True)


而celery.worker.job.Reuqest.execute_using_pool之间简单地调用celery.concurrency.prefork.TaskPool的apply_async方法

.. code-block:: python

  # celery.worker.job.Request

  class Request(object):

      def execute_using_pool(self, pool, **kwargs):
          # pool = celery.concurrency.prefork.TaskPool
          # 将任务分发给workers
          result = pool.apply_async(
            trace_task_ret,
            args=(self.name, uuid, self.args, kwargs, request),
            accept_callback=self.on_accepted,
            timeout_callback=self.on_timeout,
            callback=self.on_success,
            error_callback=self.on_failure,
            soft_timeout=soft_timeout,
            timeout=timeout,
            correlation_id=uuid,
        )


而celery.concurrency.prefork.TaskPool.apply_async会调用celery.concurrency.asynpool.AsynPool.on_apply, 也就是billiard.pool.Pool.apply_async


然后billiard.pool.Pool.apply_async将task添加到一个queue中,

.. code-block:: python

  # billiard.pool.Pool

  class Pool(object):

      def apply_async():
          # 是否是线程模型
          if self.threads:
              self._taskqueue.put()
          else:
              # 这里self.quick_put就是celery.concurrency.asynpool.AsynPool.send_job
              self._quick_put()


celery.concurrency.asynpool.AsynPool.send_job将task加入到一个自己维护的deque中


.. code-block:: python

  # celery.concurrency.asynpool.AsynPool
  class AsynPool(_pool.Pool):
  
      def __init__():
          self.outbound_buffer = deque()
  
      def _create_write_handlers():
          outbound = self.outbound_buffer
          # append_message = deque.append
          append_message = outbound.append
          def send_job(tup):
              # Schedule writing job request for when one of the process
              # inqueues are writable.
              body = dumps(tup, protocol=protocol)
              body_size = len(body)
              header = pack('>I', body_size)
              # index 1,0 is the job ID.
              job = get_job(tup[1][0])
              job._payload = buf_t(header), buf_t(body), body_size
              # append_message(job) = deque.append(job)
              append_message(job)



Worker Comsumer
===============

启动event loop的顺序

.. code-block:: python

  # celery.worker.components
  # Hub功能是获取event loop, 若没有, 则设置当前event loop, 并赋值给worker
  class Hub(bootsteps.StartStopStep):
      def create(self, w):
          # get_event_loop = kombu.async.get_event_loop
          w.hub = get_event_loop()
          # w.hub = None
          if w.hub is None:
              # _Hub = kombu.async.hub.Hub
              # 这里基本上设置event loop就是epoll, 并赋值给worker
              w.hub = set_event_loop(_Hub(w.timer))
          self._patch_thread_primitives(w)
          return self

  # celery.worker.consumer
  class Consumer(object):
      # 初始化传入worker的hub和pool
      def __init__(hub, pool):
          # hub = kombu.async.hub.Hub
          self.hub = hub
          # pool = celery.concurrency.prefork.TaskPool
          self.pool = pool
          # hub = celery.worker.components.Hub
          if not hasattr(self, 'loop'):
              self.loop = loops.asynloop if hub else loops.synloop

      def loop_args(self):
          return (self, self.connection, self.task_consumer,
                  self.blueprint, self.hub, self.qos, self.amqheartbeat,
                  self.app.clock, self.amqheartbeat_rate)

  class Evloop(bootsteps.StartStopStep):
      label = 'event loop'
      last = True
  
      def start(self, c):
          self.patch_all(c)
          # c = celery.worker.consumer.Consumer
          # c.loop = celery.worker.loops.asynloop
          c.loop(*c.loop_args())
  
      def patch_all(self, c):
          c.qos._mutex = DummyLock()


在celery.worker.loops.asynloop, 就是不断地去循环event loop

.. code-block:: python

   # celery.worker.loops.asynloop

  def asynloop(obj, connection, consumer, blueprint, hub, qos,
               heartbeat, clock, hbrate=2.0, RUN=RUN):
  
      # obj = celery.worker.consumer.Consumer
      # on_task_received就是获取到task之后的handler
      on_task_received = obj.create_task_handler()
  
      if heartbeat and connection.supports_heartbeats:
          hub.call_repeatedly(heartbeat / hbrate, hbtick, hbrate)
  
      # 注册call back
      consumer.callbacks = [on_task_received]
      # consumer = celery.app.amqp.TaskConsumer, 代表一个amqp的consumer
      # consumer.consume就是发送一个basic_consume的amqp frame
      consumer.consume()
      # 这里create_loop就是启动一次event loop
      # 由于create_loop是一个生成器, 所以这里是先启动生成器
      loop = hub.create_loop()
  
      try:
          while blueprint.state == RUN and obj.connection:
              if state.should_stop:
                  raise WorkerShutdown()
              elif state.should_terminate:
                  raise WorkerTerminate()
              if qos.prev != qos.value:
                  update_qos()
  
              try:
                  # 不断loop
                  next(loop)
              except StopIteration:
                  loop = hub.create_loop()
      finally:
          try:
              hub.reset()
          except Exception as exc:
              error(
                  'Error cleaning up after event loop: %r', exc, exc_info=1,
              )


而kombu.async.hub.Hub是真正地去读取fd, 解析rabbitmq发送回来的frame, 然后调用fd对应的call back

而task frame的call back就是之前的amqp.connection.Connection.drain_event


.. code-block:: python
   
  # kombu.async.hub.Hub

  class Hub(object):

      def create_loop(self,
                      generator=generator, sleep=sleep, min=min, next=next,
                      Empty=Empty, StopIteration=StopIteration,
                      KeyError=KeyError, READ=READ, WRITE=WRITE, ERR=ERR):
          # readers和writers基本上监听的fd和call back的字典, 例如: {1: call_back}
          readers, writers = self.readers, self.writers
          # 基本上是epoll
          poll = self.poller.poll
      
          while 1:
              # poll超时时间
              poll_timeout = fire_timers(propagate=propagate) if scheduled else 1
              if readers or writers:
                  to_consolidate = []
                  try:
                      # 这里拉取frame
                      events = poll(poll_timeout)
                  except ValueError:  # Issue 882
                      raise StopIteration()
      
                  for fd, event in events or ():
                      cb = cbargs = None
      
                      if event & READ:
                          try:
                              # 拿到fd对应的call back
                              cb, cbargs = readers[fd]
                          except KeyError:
                              self.remove_reader(fd)
                              continue
                      elif event & WRITE:
                          try:
                              cb, cbargs = writers[fd]
                          except KeyError:
                              self.remove_writer(fd)
                              continue
                      elif event & ERR:
                          general_error = True
                      else:
                          logger.info(W_UNKNOWN_EVENT, event, fd)
                          general_error = True
                      
                      if isinstance(cb, generator):
                          try:
                              # 若call back是生成器, 不断去loop
                              next(cb)
                          except OSError as exc:
                              if get_errno(exc) != errno.EBADF:
                                  raise
                              hub_remove(fd)
                          except StopIteration:
                              pass
                          except Exception:
                              hub_remove(fd)
                              raise
                      else:
                          try:
                              # 调用call back
                              cb(*cbargs)
                          except Empty:
                              pass
              else:
                  # no sockets yet, startup is probably not done.
                  sleep(min(poll_timeout, 0.1))
              yield


