Celery Startup
===============

一般我们是使用celery worker的命令来启动worker, 启动之前或导入相应的module, 这个时候会初始化Celery对象, 然后找到celery.bin.worker命令执行.

1. 生成Celery对象
------------------

生成Celery, 配置当前线程的_state, 包括设置当前app(TLS类型: thread.local)等.

celery.app.base.Celery

.. code-block:: python

    class Celery(object):

        def __init__(# 省略了很多代码):
        # 省略了很多代码
            if self.set_as_current:
                self.set_current()

            self.on_init()
            _register_app(self)

其中self.set_as_current是调用celery._state._set_default_app

.. code-block:: python

    class _TLS(threading.local):
        #: Apps with the :attr:`~celery.app.base.BaseApp.set_as_current` attribute
        #: sets this, so it will always contain the last instantiated app,
        #: and is the default app returned by :func:`app_or_default`.
        current_app = None
    _tls = _TLS()

    def _set_default_app(app):
        _tls.current_app = app


2. 初始化celery.bin.worker命令, 执行命令
-----------------------------------------

celery.bin.worker的run方法最终会初始化一个celery.apps.worker.Worker, 并调用其start方法

.. code-block:: python

    class worker(Command):
    # celery.bin.worker
    # 省略了很多代码

        def run(self, hostname=None, pool_cls=None, app=None, uid=None, gid=None,
                loglevel=None, logfile=None, pidfile=None, state_db=None,
                **kwargs):
            # 省略了很多代码
            # self.app.Worker = celery.app.worker.Worker
            return self.app.Worker(
                hostname=hostname, pool_cls=pool_cls, loglevel=loglevel,
                logfile=logfile,  # node format handled by celery.app.log.setup
                pidfile=self.node_format(pidfile, hostname),
                state_db=self.node_format(state_db, hostname), **kwargs
            ).start()

3. celery.app.worker.Worker
-------------------------------

按Blueprint中的步骤启动, 默认的worker blueprint在celery.worker.__init__.WorkerControl中.

.. code-block:: python

    class WorkController(object):
        # 省略了代码
        # 默认的blueprint
        class Blueprint(bootsteps.Blueprint):
            """Worker bootstep blueprint."""
            name = 'Worker'
            default_steps = set([
                'celery.worker.components:Hub',
                'celery.worker.components:Queues',
                'celery.worker.components:Pool',
                'celery.worker.components:Beat',
                'celery.worker.components:Timer',
                'celery.worker.components:StateDB',
                'celery.worker.components:Consumer',
                'celery.worker.autoscale:WorkerComponent',
                'celery.worker.autoreload:WorkerComponent',

            ])

        def __init__(self, app=None, hostname=None, **kwargs):
        self.app = app or self.app
        self.hostname = default_nodename(hostname)
        self.app.loader.init_worker()
        self.on_before_init(**kwargs)
        self.setup_defaults(**kwargs)
        self.on_after_init(**kwargs)

        # setup_instance中会根据配置来决定真正的启动步骤
        self.setup_instance(**self.prepare_args(**kwargs))
        self._finalize = [
            Finalize(self, self._send_worker_shutdown, exitpriority=10),
        ]

        # 按blueprint启动
        def start(self):
            """Starts the workers main loop."""
            try:
                self.blueprint.start(self)
            except WorkerTerminate:
                self.terminate()
            except Exception as exc:
                logger.error('Unrecoverable error: %r', exc, exc_info=True)
                self.stop()
            except (KeyboardInterrupt, SystemExit):
                self.stop()

其中celery.worker.components.Queues就是concurrency中worker为子进程分配任务所使用的queue.

而celery.worker.components.Consumer则有自己的启动步骤.

celery.worker.consumer.Consumer

.. code-block:: python

    class Consumer(object):
        # 省略了代码
        # Strategies会在绑定和发送event的时候用到
        Strategies = dict

        class Blueprint(bootsteps.Blueprint):
            name = 'Consumer'
            default_steps = [
                'celery.worker.consumer:Connection',
                'celery.worker.consumer:Mingle',
                'celery.worker.consumer:Events',
                'celery.worker.consumer:Gossip',
                'celery.worker.consumer:Heart',
                'celery.worker.consumer:Control',
                'celery.worker.consumer:Tasks',
                'celery.worker.consumer:Evloop',
                'celery.worker.consumer:Agent',
            ]

启动顺序为

[<step: Connection>, <step: Events>, <step: Mingle>, <step: Gossip>, <step: Tasks>, <step: Control>, <step: Heart>, <step: event loop>]

4. Event
----------

.. _add_task_to_group:

Event是绑定consumer发送event所使用的event dispatcher的, 默认是使用整个app(定义的Celery对象)初始化时候绑定的event dispatcher类.

**其中groups是指定发送的event类型, 默认是['worker'], 也可以在配置中指定是否要发送其他类型的event, 比如flower会发送enable_event的event, 将task加入groups中, consumer就会发送\
task状态的任务, 比如task receive, task failed等.** 

.. code-block:: python

    class Events(bootsteps.StartStopStep):
        requires = (Connection, )
        # 省略了代码

        def __init__(self, c, send_events=None, **kwargs):
            self.send_events = True
            # self.groups就是指定要发送的event类型
            self.groups = None if send_events else ['worker']
            c.event_dispatcher = None

        def start(self, c):
            # flush events sent while connection was down.
            # 其中group就是指定要发送的event类型.
            prev = self._close(c)
            dis = c.event_dispatcher = c.app.events.Dispatcher(
                c.connect(), hostname=c.hostname,
                enabled=self.send_events, groups=self.groups,
            )
            if prev:
                dis.extend_buffer(prev)
                dis.flush()

5. Mingle
----------

Mingle步骤是像其他的worker同步revoke task和时钟的.

celery.worker.consumer.Mingle

.. code-block:: python

    class Mingle(bootsteps.StartStopStep):

        # 省略代码
        def start(self, c):
            info('mingle: searching for neighbors')
            I = c.app.control.inspect(timeout=1.0, connection=c.connection)
            # hello命令返回的是{'revoked': worker_state.revoked._data, 'clock': state.app.clock.forward()}
            replies = I.hello(c.hostname, revoked._data) or {}
            replies.pop(c.hostname, None)
            if replies:
                info('mingle: sync with %s nodes',
                     len([reply for reply, value in items(replies) if value]))
                for reply in values(replies):
                    if reply:
                        try:
                            other_clock, other_revoked = MINGLE_GET_FIELDS(reply)
                        except KeyError:  # reply from pre-3.1 worker
                            pass
                        else:
                            c.app.clock.adjust(other_clock)
                            revoked.update(other_revoked)
                info('mingle: sync complete')
            else:
                info('mingle: all alone')

6. Gossip
-------------

Gossip的作用是记录集群worker信息以及选举, 详情在这: http://celery.readthedocs.org/en/latest/whatsnew-3.1.html#gossip-worker-worker-communication

7. Task
----------

Task主要是设置consumer qos以及配置task event发送策略的.

celery.worker.consumer.Tasks

.. code-block:: python

    class Tasks(bootsteps.StartStopStep):
        requires = (Mingle, )

        def __init__(self, c, **kwargs):
            c.task_consumer = c.qos = None

        def start(self, c):
            # 调用consumer.update_strategies方法
            c.update_strategies()
            # 下面省略了很多代码

consumer.update_strategies方法则会初始化strategy, 

.. code-block:: python

    class Consumer(object):
        # 各种省略代码
        Strategy = 'celery.worker.strategy:default'

        def update_strategies(self):
            loader = self.app.loader
            for name, task in items(self.app.tasks):
                self.strategies[name] = task.start_strategy(self.app, self)
                task.__trace__ = build_tracer(name, task, loader, self.hostname,
                                              app=self.app)

        def start_strategy(self, app, consumer, **kwargs):
            return instantiate(self.Strategy, self, app, consumer, **kwargs)

在celery.worker.strategy:default中配置了task的什么状态发送什么message.

.. code-block:: python

    def default(task, app, consumer,
                info=logger.info, error=logger.error, task_reserved=task_reserved,
                to_system_tz=timezone.to_system):
        # 省略代码
        send_event = eventer.send

        def task_message_handler(message, body, ack, reject, callbacks,
                                 to_timestamp=to_timestamp):
            req = Req(body, on_ack=ack, on_reject=reject,
                      app=app, hostname=hostname,
                      eventer=eventer, task=task,
                      connection_errors=connection_errors,
                      message=message)
            if req.revoked():
                return

            if _does_info:
                info('Received task: %s', req)

            if events:
                send_event(
                    'task-received',
                    uuid=req.id, name=req.name,
                    args=safe_repr(req.args), kwargs=safe_repr(req.kwargs),
                    retries=req.request_dict.get('retries', 0),
                    eta=req.eta and req.eta.isoformat(),
                    expires=req.expires and req.expires.isoformat(),
                )

            if req.eta:
                try:
                    if req.utc:
                        eta = to_timestamp(to_system_tz(req.eta))
                    else:
                        eta = to_timestamp(req.eta, timezone.local)
                except OverflowError as exc:
                    error("Couldn't convert eta %s to timestamp: %r. Task: %r",
                          req.eta, exc, req.info(safe=True), exc_info=True)
                    req.acknowledge()
                else:
                    consumer.qos.increment_eventually()
                    call_at(eta, apply_eta_task, (req, ), priority=6)
            else:
                if rate_limits_enabled:
                    bucket = get_bucket(task.name)
                    if bucket:
                        return limit_task(req, bucket, 1)
                task_reserved(req)
                if callbacks:
                    [callback() for callback in callbacks]
                handle(req)

        return task_message_handler

其中send方法则是consumer.event_dispatcher = celery.events.EventDispatcher, 只有该类型的event是在groups里面才会发送该event. 具体请看\ add_task_to_group_

8. Control
--------------

设置pidbox, 绑定channel和call back函数.

pidbox主要是用来处理发送过来的control命令, control命令定义celery.app.control中, 而命令具体的调用是在celery.worker.contol中. 比如, 发送celery inspect active命令定义为

.. code-block:: python

    class Inspect(object):
        # 你懂的, 省略代码
        def active(self, safe=False):
            return self._request('dump_active', safe=safe)

具体调用

.. code-block:: python

    @Panel.register
    def dump_active(state, safe=False, **kwargs):
        return [request.info(safe=safe)
                for request in worker_state.active_requests]

调用关系使用Panel的register来设置的, 其实就是一个字典对应名字和调用函数

celery.worker.pidbox.Pidbox

.. code-block:: python

    class Pidbox(object):
        consumer = None

        def __init__(self, c):
            self.c = c
            self.hostname = c.hostname
            # 命令和调用绑定, 使用Panel
            self.node = c.app.control.mailbox.Node(
                safe_str(c.hostname),
                handlers=control.Panel.data,
                state=AttributeDict(app=c.app, hostname=c.hostname, consumer=c),
            )
            self._forward_clock = self.c.app.clock.forward

        def start(self, c):
            self.node.channel = c.connection.channel()
            # 监听channel
            self.consumer = self.node.listen(callback=self.on_message)
            self.consumer.on_decode_error = c.on_decode_error


celery.worker.consumer.Control

.. code-block:: python

    class Control(bootsteps.StartStopStep):
        requires = (Tasks, )

        def __init__(self, c, **kwargs):
            self.is_green = c.pool is not None and c.pool.is_green
            self.box = (pidbox.gPidbox if self.is_green else pidbox.Pidbox)(c)
            self.start = self.box.start
            self.stop = self.box.stop
            self.shutdown = self.box.shutdown

        def include_if(self, c):
            return c.app.conf.CELERY_ENABLE_REMOTE_CONTROL

**之所以要求启用CELERY_ENABLE_REMOTE_CONTROL, 是因为有些contro 命令需要reply, reply是使用rabbitmq的RCP(remote procedure call: 远程程序调用)来实现的.**

9. Heart
------------

这里就是配置发送worker-heartbeat

celery.worker.heartbeat.Heart

.. code-block:: python

    class Heart(object):
        # 省略代码
        def __init__(self, timer, eventer, interval=None):
            # 省略代码
            # 不设置heartbeat频率的话, 默认代码写死是2.0
            self.interval = float(interval or 2.0)

        def start(self):
            # 使用定时器来发送worker-hearbeat
            # start的时候先发送work-online, 然后周期性发送worker-hearbeat
            if self.eventer.enabled:
                self._send('worker-online')
                self.tref = self.timer.call_repeatedly(
                    self.interval, self._send, ('worker-heartbeat', ),
                )

        def stop(self):
            if self.tref is not None:
                self.timer.cancel(self.tref)
                self.tref = None
            if self.eventer.enabled:
                self._send('worker-offline')

