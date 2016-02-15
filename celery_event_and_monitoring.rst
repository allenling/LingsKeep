Celery Event And Monitoring
==============================

Celery version 3.1.17

Flower version 0.8.3

celery监控机制
---------------

celery中的监控是使用event机制来实现的. 有些event是worker发送的, 比如worker的hearbeat, 有些是consumer发送的, 比如task的event.

celery中提供一个\ **EventReciver(celery.events.__init__.EventReciver)**\ 来捕获发出的event, 客户端只需要不断循环使用这个reciver来捕获event, 解析event就可以了.

而发送event的handler是一个\ **EventDispatcher(celery.events.__init__.EventDispatcher)**\ , 在worker初始化的时候是初始化一个event dispatcher, 根据需要来发送对应的event信息, 比如task
的信息, worker的信息等.

服务端(Celery)
-----------------

客户端(Flower)
----------------

flower是一个tornado的app, 是异步的.

flower启动, 初始化一个名为Flower(flower.app.Flower)的app, 调用start方法, 运行flower.

1. 初始化的过程
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

初始化过程是绑定url和views, 初始化io_loop, 绑定celery app, 生成一个event的thread线程.

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
~~~~~~~~~~~~~~~~

Flower.start会启动flower, 激活ioloop, 运行event线程

.. code-block:: python

    class Flower(tornado.web.Application):
    # 省略了很多代码

    def start(self):
        self.pool = self.pool_executor_cls(max_workers=self.max_workers)
        # self.events.start主要是循环发送enable event, 让worker发送task的event
        self.events.start()
        self.listen(self.options.port, address=self.options.address,
                    ssl_options=self.ssl_options, xheaders=self.options.xheaders)
        self.io_loop.add_future(
            control.ControlHandler.update_workers(app=self),
            callback=lambda x: logger.debug('Successfully updated worker cache'))
        self.started = True
        self.io_loop.start()

所以, event的主要处理是在self.event中, self.event是一个Flower.events.Events类.

3. event捕获和处理
~~~~~~~~~~~~~~~~~~~~~~

flower.events.Events是一个threading.Thread子类, 在start方法中启动定时器, 循环发送event_enable消息, 而本身的run方法会循环去捕获event.

* __init__方法绑定celery app, 若持久化, 则初始化持久化文件, 初始化tornado定时任务, 绑定call back函数on_enable_events将task类型加入到服务端event dispatcher中的groups中, 也就是
  让consumer发送task

* on_enable_events则会定时广播worker一个enbale_event的消息, 让consumer发送task状态的消息.

* start方法启动定时任务.

* 同时, 线程池会不断阻塞并使用EventReciver来捕获worker发送来的message.

注意的是, 这里有两个循环操作, tornado的定时任务以及Events本身. 定时任务是定时发送enable_events的消息, 而线程池负责捕获消息.


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

    # 捕获到event之后的call back
    def on_event(self, event):
        # Call EventsState.event in ioloop thread to avoid synchronization
        self.io_loop.add_callback(partial(self.state.event, event))

4. 处理event过程
~~~~~~~~~~~~~~~~~~~~~~~~~

捕获到event之后, 会调用对应的call back函数: self.state.event

self.state.event=flower.events.EventsState, 其中其实没做什么, 只是调用celery.events.state.State来存储这个worker的所有event统计数据.

flower.events.EventsState

.. code-block:: python

    class EventsState(State):

        def __init__(self, *args, **kwargs):
            super(EventsState, self).__init__(*args, **kwargs)
            self.counter = collections.defaultdict(Counter)

        def event(self, event):
            worker_name = event['hostname']
            event_type = event['type']

            self.counter[worker_name][event_type] += 1

            # Send event to api subscribers (via websockets)
            # 这里的classname只有task开头的event才有
            classname = api.events.getClassName(event_type)
            cls = getattr(api.events, classname, None)
            if cls:
                cls.send_message(event)

            # Save the event
            # 调用celery.events.state.State.event来统计worker, task的状态.
            super(EventsState, self).event(event)

若event_type=task, 则会获取到一个cls, 继承于flower.events.EventsApiHandler, 但是send_message的时候, 其实也没都没做

.. code-block:: python

    class EventsApiHandler(BaseWebSocketHandler):
        def open(self, task_id=None):
            BaseWebSocketHandler.open(self)
            self.task_id = task_id

        @classmethod
        def send_message(cls, event):
            # 如何处理task类型的event取决于这里有多少个listeners在监听.
            for l in cls.listeners:
                if not l.task_id or l.task_id == event['uuid']:
                    l.write_message(event)

    EVENTS = ('task-sent', 'task-received', 'task-started', 'task-succeeded',
              'task-failed', 'task-revoked', 'task-retried')


    def getClassName(eventname):
        return ''.join(map(lambda x: x[0].upper() + x[1:], eventname.split('-')))


    # Dynamically generates handler classes
    thismodule = sys.modules[__name__]
    for event in EVENTS:
        classname = getClassName(event)
        setattr(thismodule, classname,
                # 每一个task的handler都会设置listeners为空列表, 所以send_message的时候, 什么也没做.
                type(classname, (EventsApiHandler, ), {'listeners': []}))

5. 检索过程
~~~~~~~~~~~~~~~

之后在flower的页面上看到的一些数据统计, 都是调用celery.urls中map的对应的接口.

比如根据时间戳获取所有的task的列表, 在celery.urls中

.. code-block:: python

    # handlers中省略了其他api
    handlers = [
                (r"/tasks", TasksView),
    ]

TasksView是在flower.views.tasks中, 过程基本上是调用celery.events.state.tasks_by_timestamp

.. code-block:: python

    class TasksView(BaseHandler):

        def get(self):
            # 省略代码
            tasks = iter_tasks(
                app.events,
                limit=limit,
                type=type,
                worker=worker,
                state=state,
                sort_by=sort_by,
                received_start=received_start,
                received_end=received_end,
                started_start=started_start,
                started_end=started_end,
                search_terms=parse_search_terms(search),
            )

在iter_tasks中, 获取tasks则是调用celery.events.state.tasks_by_timestamp

flower.utils.tasks

.. code-block:: python

    def iter_tasks(events, limit=None, type=None, worker=None, state=None,
                   sort_by=None, received_start=None, received_end=None,
                   started_start=None, started_end=None, search_terms=None):
        i = 0
        # celery.events.state的query方法
        tasks = events.state.tasks_by_timestamp()
        # 省略代码

类似的worker的统计等等.

6. flower命令
~~~~~~~~~~~~~~

flower也可以发命令, 基本上跟检索的流程类似, 都是直接掉celery.app.control中的命令.

7. 最后
~~~~~~~~~~~

celery中已经提供了很多统计集群状态的数据, 我们可以直接拿到. 比如

.. raw:: html

    <p style="font-size: 50px;">flower中获取的统计数据基本上也是celery.events统计好的.</p>

