Celery Startup And Monitoring
================================

Celery version 3.1.17

Flower version 0.8.3

celery监控机制
---------------

celery中提供一个\ **EventReciver(celery.events.__init__.EventReciver)**\ 来捕获发出的event, 客户端只需要不断循环使用这个reciver来捕获event, 解析event就可以了.

而发送event的handler是一个\ **EventDispatcher(celery.events.__init__.EventDispatcher)**\ , 在worker初始化的时候是初始化一个event dispatcher, 根据需要来发送对应的event信息, 比如task
的信息, worker的信息等.

客户端(Flower)
----------------

flower是一个tornado的app, 是异步的.

flower启动, 初始化一个名为Flower(flower.app.Flower)的app, 调用start方法, 运行flower.

1. 初始化的过程是绑定url和views, 初始化io_loop, 绑定celery app, 生成一个event的thread线程.

.. code-block:: python

    class Flower(tornado.web.Application):
        pool_executor_cls = ThreadPoolExecutor  # 线程池
        max_workers = 4

        def __init__(self, options=None, capp=None, events=None,
                     io_loop=None, **kwargs):
            kwargs.update(handlers=handlers)  # 绑定url和views
            super(Flower, self).__init__(**kwargs)
            self.options = options or default_options
            self.io_loop = io_loop or ioloop.IOLoop.instance()
            self.ssl_options = kwargs.get('ssl_options', None)

            self.capp = capp or celery.Celery()  # 绑定celery app
            # 生成event线程
            self.events = events or Events(self.capp, db=self.options.db,
                                           persistent=self.options.persistent,
                                           enable_events=self.options.enable_events,
                                           io_loop=self.io_loop,
                                           max_tasks_in_memory=self.options.max_tasks)


2. 运行的过程

激活线程池, 运行ioloop, 运行event线程

.. code-block:: python

    class Flower(tornado.web.Application):
    # 省略了很多代码

    def start(self):
        self.pool = self.pool_executor_cls(max_workers=self.max_workers)
        # 循环发送enable event, 让worker发送task的event
        self.events.start()
        self.listen(self.options.port, address=self.options.address,
                    ssl_options=self.ssl_options, xheaders=self.options.xheaders)
        self.io_loop.add_future(
            control.ControlHandler.update_workers(app=self),
            callback=lambda x: logger.debug('Successfully updated worker cache'))
        self.started = True
        self.io_loop.start()

所以, event的主要处理是在self.event中, self.event是一个Flower.events.Events类.

3. event线程处理过程

* __init__方法绑定celery app, 若持久化, 则初始化持久化文件, 初始化tornado定时任务, 绑定call back函数on_enable_events.
* on_enable_events则会定时广播worker一个enbale_event的消息, 让worker们发送task状态的消息.
* start方法启动定时任务.
* 同时, 线程池会不断阻塞并使用EventReciver来捕获worker发送来的message.

注意的是, 这里有两个循环操作, tornado的定时任务以及Event本身是一个线程池. 定时任务是定时发送enable_events的消息, 而线程池负责捕获消息.


.. code-block:: python

    class Events(threading.Thread):
        events_enable_interval = 5000  # 每五秒发送一个enable_event消息

        def __init__(self, capp, db=None, persistent=False,
                     enable_events=True, io_loop=None, **kwargs):
            threading.Thread.__init__(self)
            self.daemon = True

            self.io_loop = io_loop or IOLoop.instance()  # 自己的ioloop
            self.capp = capp  # 绑定celery app

            self.db = db
            self.persistent = persistent
            self.enable_events = enable_events
            self.state = None

            if self.persistent and celery.__version__ < '3.0.15':
                logger.warning('Persistent mode is available with '
                               'Celery 3.0.15 and later')
                self.persistent = False

            # 若持久化, 则使用shelve来初始化持久化文件
            if self.persistent:
                logger.debug("Loading state from '%s'...", self.db)
                state = shelve.open(self.db)
                if state:
                    self.state = state['events']
                state.close()
            # 保存到flwoer的内存对象中
            if not self.state:
                self.state = EventsState(**kwargs)
            # 初始化定时任务!!!!!!
            self.timer = PeriodicCallback(self.on_enable_events,
                                          self.events_enable_interval)

    def on_enable_events(self):
        # 定时任务的cll back函数, 定时广播enable_events的message
        try:
            # 广播消息
            self.capp.control.enable_events()
        except Exception as e:
            logger.debug("Failed to enable events: '%s'", e)

    # start方法激活定时任务.
    def start(self):
        threading.Thread.start(self)
        if self.enable_events and celery.VERSION[0] > 2:
            self.timer.start()


    def run(self):
        try_interval = 1
        while True:
            try:
                try_interval *= 2

                with self.capp.connection() as conn:
                    # 捕获worker发送回来的消息, 并解析.
                    recv = EventReceiver(conn,
                                         handlers={"*": self.on_event},
                                         app=self.capp)
                    try_interval = 1
                    recv.capture(limit=None, timeout=None, wakeup=True)

            except (KeyboardInterrupt, SystemExit):
                try:
                    import _thread as thread
                except ImportError:
                    import thread
                thread.interrupt_main()
            except Exception as e:
                logger.error("Failed to capture events: '%s', "
                             "trying again in %s seconds.",
                             e, try_interval)
                logger.debug(e, exc_info=True)
                time.sleep(try_interval)

服务端(Celery)
=================

Celery Startup流程
--------------------

一般我们是使用celery worker的命令来启动worker, 启动之前或导入相应的module, 这个时候会初始化Celery对象, 然后找到celery.bin.worker命令执行.

1. 生成Celery对象

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

而worker会根据启动参数和配置来决定需要哪些步骤, worker一般有Hub, Pool, Consumer

其中Consumer的启动步骤又有
[<step: Connection>, <step: Events>, <step: Heart>, <step: Mingle>, <step: Gossip>, <step: Tasks>, <step: Control>, <step: event loop>]


