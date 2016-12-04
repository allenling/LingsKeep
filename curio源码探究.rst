curio.open_connection例子
=========================

一个协程await一个future给event loop之后，就暂停了，IO任务是在Thread Pool或者Process Pool中运行，运行完成之后，调用future.set_result.

一旦future对象被调用set_result之后，怎么能告诉event loop，future完成了，赢把结果返回给协程？

在curio，包括asyncio中，都是设置一个set_result的对调函数，在future.set_result的是，调用回调函数，回调函数的作用将协程放入待处理的任务列表，唤醒event loop去处理待处理任务。


下面一个fetch data的例子， 主协程是打开连接，发送GET，然后打印

.. code-block:: python

  async def main():
      sock = await curio.open_connection('www.baidu.com', 80,
                                         )
      async with sock:
          await sock.sendall(b'GET / HTTP/1.0\r\nHost: www.baidu.com\r\n\r\n')
          chunks = []
          while True:
              chunk = await sock.recv(10000)
              if not chunk:
                  break
              chunks.append(chunk)
  
      response = b''.join(chunks)
      print(response.decode('utf-8'))


下面的curio中open_connection的代码

1. curio.open_connection将io操作放入线程池中运行之后，yield一个event loop的处理函数.
------------------------------------------------------------------------------------

.. code-block:: python

  # curio.workers.ThreadWorker

  class ThreadWorker(object):

      async def apply(self, func, args=(), kwargs={}):
          if self.thread is None:
              # _launch是启动工作线程
              self._launch()

          # 这里生成一个future对象
          future = _FutureLess()
          self.request = (func, args, kwargs, future)
          # 这里就是将event loop中的函数名称和参数yield会event loop
          # _future_wait=curio.traps._future_wait
          await _future_wait(future, self.start_evt)
          return future.result()


  # curio.traps._future_wait
  @coroutine
  def _future_wait(future, event=None):
      # 这里_trap_future_wait是一个event loop一个处理函数，future和event就是上面apply中的传入的future, self.start_evt
      yield (_trap_future_wait, future, event)
      

2. event loop得到yield出来的函数名称和参数，调用它.
---------------------------------------------------------

上面的例子返回的event loop应该执行的方法是trap_future_wait


.. code-block:: python

  # curio.kernel.Kernel就是asyncio中的event loop

  class Kernel(object):

      def run(self, coro=None, *, shutdown=False):

          _wake = self._wake

          # 在run的时候，初始化event loop的跟踪方法，都是_trap开头
          # 这个_trap_future_wait上面open_connection中yield出来
          # current就是当前的协程
          def _trap_future_wait(_, future, event):
              if self._kernel_task_id is None:
                  _init_loopback_task()

              current.state = 'FUTURE_WAIT'
              current.cancel_func = (lambda task=current:
                                     setattr(task, 'future', future.cancel() and None))
              current.future = future

              # 这里就是为future对象添加一个set_result的call back
              # 也就是说，future.set_result的时候调用回调函数，这里是_wake.
              # _wake在run方法开始的时候，是赋值为self._wake
              future.add_done_callback(lambda fut, task=current: _wake(task, fut))
              if event:
                  event.set()


      # 这里就是当future.set_result调用的回调函数
      def _wake(self, task=None, future=None):
          if task:
              # 就是把task加入到wake队列
              self._wake_queue.append((task, future))
          # 然后，唤醒event loop
          self._notify_sock.send(b'\x00')

          
3. 协程暂停之后，even loop如何将结果send到协程呢
--------------------------------------------------


open_connection通过yield想event loop返回_trap_future_wait方法，_trap_future_wait将一个kernel_task放入ready队列中，然后注册set_result


.. code-block:: python

  def _trap_future_wait(_, future, event):
      if self._kernel_task_id is None:
          # 这个_init_loopback_task主要的目的是, 将kernel_task放入ready队列中
          # kernel_task的作用主要是wait_sock注册到epoll中
          _init_loopback_task()
  
      current.state = 'FUTURE_WAIT'
      current.cancel_func = (lambda task=current:
                             setattr(task, 'future', future.cancel() and None))
      current.future = future
      # 设置回调函数
      future.add_done_callback(lambda fut, task=current: _wake(task, fut))
      if event:
          event.set()

  def _init_loopback_task():
      # 设置wait_sock
      self._init_loopback()
      # 将kernel_task放入ready队列
      # kernel_task就是将设置好的wait_sock注册到epoll中
      task = Task(_kernel_task(), taskid=0, daemon=True)
      # 将kernel_task加入到ready队列
      _reschedule_task(task)
      self._kernel_task_id = task.id
      self._tasks[task.id] = task

    def _init_loopback(self):
        # 生成一对unix sock domain
        # 这个_wait_sock是当future.set_result的是，调用self._wait, self._wait就是通过self._notify_sock发送数据，激活self._wait_sock
        self._notify_sock, self._wait_sock = socket.socketpair()
        self._wait_sock.setblocking(False)
        self._notify_sock.setblocking(False)
        return self._wait_sock.fileno()

    async def _kernel_task():
        wake_queue_popleft = wake_queue.popleft
        wait_sock = self._wait_sock

        while True:
            # _read_wait是yield返回一个_trap_io给event loop执行，暂停
            # _trap_io是将传入的wait_sock, 也就是self._wait_sock注册到epoll
            await _read_wait(wait_sock)
            data = wait_sock.recv(1000)

            while wake_queue:
                task, future = wake_queue_popleft()
                if future and task.future is not future:
                    continue
                task.future = None
                task.state = 'READY'
                task.cancel_func = None
                ready_append(task)

            # 将待处理的信号加入ready队列
            if not self._signal_sets:
                continue

            sigs = (n for n in data if n in self._signal_sets)
            for signo in sigs:
                for sigset in self._signal_sets[signo]:
                    sigset.pending.append(signo)
                    if sigset.waiting:
                        _reschedule_task(sigset.waiting, value=signo)
                        sigset.waiting = None

    @coroutine
    def _read_wait(fileobj):
        # yield返回_trap_io返回给event loop
        yield (_trap_io, fileobj, EVENT_READ, 'READ_WAIT')


    def _trap_io(_, fileobj, event, state):
        if current._last_io != (state, fileobj):
            if current._last_io:
                selector_unregister(current._last_io[1])
            # 对应上面的例子，fileobj就是self._wait_sock, event就是EVENT_READ, data就是current，也就是kernel_task
            selector_register(fileobj, event, current)
        current._last_io = None
        current.state = state
        current.cancel_func = lambda: selector_unregister(fileobj)
 
以上，就是_trap_future_wait之后的流程，执行玩_trap_io之后，就是在epoll中等待self._wait_sock被激活

self._wait_sock被激活是在future.set_result的回调函数Kernel._wake中被激活的


.. code-block:: python

  class Kernel(object):

      def _wake(self, task=None, future=None):
          if task:
              # task就是我们的主协程
              # 加入到_wake_queue中
              self._wake_queue.append((task, future))
          # 激活self._wait_sock
          self._notify_sock.send(b'\x00')


将task加入到等待唤醒的队列中(_wake_queue), 激活self._wait_sock


.. code-block:: python

  def run(self, coro=None, *, shutdown=False):
    while njobs > 0:
        # 上述_trap_io之后，ready队列就为空
        # 之后就是进入epoll的处理中
        while not ready:
            timeout = (sleeping[0][0] - time_monotonic()) if sleeping else None
            events = selector_select(timeout)
            for key, mask in events:
                # 对应上面的例子，data就是注册的时候的current, 也就是kernel_task
                task = key.data
                task._last_io = (task.state, key.fileobj)
                task.state = 'READY'
                task.cancel_func = None
                # 将kernel_task加入ready列表
                ready_append(task)

        while ready:
            # 这里将kernel_task从ready队列pop出来
            current = ready_popleft()
            try:
                current.state = 'RUNNING'
                current.cycles += 1
                with _enable_tasklocal_for(current):
                    if current.next_exc is None:
                        # 重新启动kernel_task
                        trap = current._send(current.next_value)
                    else:
                        trap = current._throw(current.next_exc)
                        current.next_exc = None


kernel_task重新启动之后，因为在self._wake中将我们的主协程加入到了_wake_queue了, 所以从_wake_queue中将我们的主协程pop出来

加入到ready队列中


.. code-block:: python

  async def _kernel_task():
      wake_queue_popleft = wake_queue.popleft
      wait_sock = self._wait_sock
  
      while True:
          await _read_wait(wait_sock)
          # self._wait_sock被激活
          data = wait_sock.recv(1000)
          while wake_queue:
              # 将我们的主协程从_wake_queue中pop出来
              task, future = wake_queue_popleft()
              if future and task.future is not future:
                  continue
              task.future = None
              task.state = 'READY'
              task.cancel_func = None
              # 将主协程加入到ready队列中
              ready_append(task)

接下来, event loop调用send重新激活主线程

.. code-block:: python

  while ready:
      current = ready_popleft()
      try:
          current.state = 'RUNNING'
          current.cycles += 1
          with _enable_tasklocal_for(current):
              if current.next_exc is None:
                  # 激活主协程
                  trap = current._send(current.next_value)
              else:
                  trap = current._throw(current.next_exc)
                  current.next_exc = None

主协程之前暂停在open_connection中的ThreadWorker.apply中

.. code-block:: python

  class ThreadWorker(object):
  
      async def apply(self, func, args=(), kwargs={}):
          if self.thread is None:
              self._launch()
  
          future = _FutureLess()
          self.request = (func, args, kwargs, future)
  
          # 这里重新启动主协程
          await _future_wait(future, self.start_evt)
          # 返回一个future.result, 也就是返回得到的socket对象
          return future.result()

之后，将结果一直返回给run_in_thread, 之后返回给create_connection

.. code-block:: python

  # 返回socket给run_in_thread
  async def run_in_thread(callable, *args, **kwargs):
      kernel = await _get_kernel()
      if not kernel._thread_pool:
          kernel._thread_pool = WorkerPool(ThreadWorker, MAX_WORKER_THREADS)
      # 得到socket
      return await kernel._thread_pool.apply(callable, args, kwargs)


  # 返回到create_connection
  @wraps(_socket.create_connection)
  async def create_connection(*args, **kwargs):
      # 这里拿到socket
      sock = await workers.run_in_thread(_socket.create_connection, *args, **kwargs)
      return io.Socket(sock)



run in pool
=============

curio将IO操作放到pool里面执行，可以是线程模式，也可以是进程模式.

下面是线程池的入口，进程池也一样


.. code-block:: python

  # curio.workers.run_in_thread
  async def run_in_thread(callable, *args, **kwargs):
      kernel = await _get_kernel()
      # 获取线程池，然后分配执行io函数
      # 线程或者进程，区别是传入WorkPool的是ThreadWorker还是ProcessWorker
      if not kernel._thread_pool:
          kernel._thread_pool = WorkerPool(ThreadWorker, MAX_WORKER_THREADS)
      return await kernel._thread_pool.apply(callable, args, kwargs)

  # curio.workers.WorkerPool.apply
  # pool将任务分配到worker中，先创建最大worker数的worker，然后一个一个的pop，分配func到pool
  class WorkerPool(object):
      async def apply(self, func, args=(), kwargs={}):
          async with self.nworkers:
              # pop出worker
              worker = self.workers.pop()
              try:
                  # 调用worker.apply, 取执行任务
                  return await worker.apply(func, args, kwargs)
              except CancelledError:
                  worker.shutdown()
                  worker = self.workercls()
                  raise
              finally:
                  if self.workers is not None:
                      self.workers.append(worker)


ThreadPool(线程池)
=====================


以curio.open_connection为例子

.. code-block:: python
  
  # curio.socks.create_connection
  @wraps(_socket.create_connection)
  async def create_connection(*args, **kwargs):
      # 启动一个worker，默认是thread worker，取执行io.
      sock = await workers.run_in_thread(_socket.create_connection, *args, **kwargs)

  # curio.workers.ThreadWorker.apply
  class ThreadWorker(object):


      def _launch(self):
          self.start_evt = threading.Event()
          # 启动一个daemon thread，也就是守护线程取执行IO
          self.thread = threading.Thread(target=self.run_worker, daemon=True)
          self.thread.start()

      def run_worker(self):
          while True:
              self.start_evt.wait()
              self.start_evt.clear()
              # If there is no pending request, but we were signalled to
              # start, it means terminate.
              if not self.request:
                  return

              func, args, kwargs, future = self.request

              try:
                  # 执行io函数，然后调用future.set_result, 异常的话, set_exception
                  result = func(*args, **kwargs)
                  future.set_result(result)
              except Exception as e:
                  future.set_exception(e)

      async def apply(self, func, args=(), kwargs={}):
          if self.thread is None:
              self._launch()

ProcessPool(进程池)
=========================

进程池是用multiprocessing.Process来启动子进程, 父子进程使用multiprocessing.Pipe(一个使用socket实现的双工pipe)来传递数据

.. code-block:: python

  # curio.workers.ProcessWorker
  class ProcessWorker(object):


      def _launch(self):
          # 启动子进程， 运行任务
          client_ch, server_ch = multiprocessing.Pipe()
          self.process = multiprocessing.Process(target=self.run_server, args=(server_ch,), daemon=True)
          self.process.start()
          server_ch.close()
          self.client_ch = Channel.from_Connection(client_ch)

      def run_server(self, ch):
          # 这里真正运行任务
          # 注册信号回调
          # 这里不调用future.set_result是因为两个进程不能共享一个对象呀
          signal.signal(signal.SIGINT, signal.SIG_IGN)
          while True:
              func, args, kwargs = ch.recv()
              try:
                  # 执行任务，成功，通过multiprocessing.Pipe发送True和result给父进程
                  result = func(*args, **kwargs)
                  ch.send((True, result))
              except Exception as e:
                  e = ExceptionWithTraceback(e, e.__traceback__)                
                  # 执行任务，异常，通过multiprocessing.Pipe发送False和exc给父进程
                  ch.send((False, e))
              func = args = kwargs = None
        
      async def apply(self, func, args=(), kwargs={}):
          if self.process is None or not self.process.is_alive():
              # self._launch是执行方法
              self._launch()
      
          msg = (func, args, kwargs)
          await self.client_ch.send(msg)
          success, result = await self.client_ch.recv()
          if success:
              return result
          else:
              raise result



守护线程
==========

daemon标识位是True的时候，表示该线程是守护线程，默认是False, 并且会继承自创建它的线程.

非守护线程会阻塞主线程的退出
-------------------------------

.. code-block:: python

  import time
  import threading
  
  
  def fun():
      print "start fun"
      time.sleep(2)
      print "end fun"
  
  
  print "main thread"
  t1 = threading.Thread(target=fun,args=())
  #t1.setDaemon(True)
  t1.start()
  time.sleep(1)
  print "main thread end"


输出是

main thread 
start fun 
# 这里main thread end的时候，程序并没有终止，而是等待非daemon都结束才终止
main thread end 
# 这里非daemon线程终止，整个程序终止
end fun

主进程等待所以非守护线程终止之后，退出
---------------------------------------

也就是，当所有的非守护进程退出的时候，主进程退出，而不会等待守护进程退出。

守护线程不会阻塞主线程退出。主进程退出会粗暴地终止守护线程

.. code-block:: python

  import time
  import threading
  
  
  def fun():
      print "start fun"
      time.sleep(2)
      print "end fun"
  
  
  print "main thread"
  t1 = threading.Thread(target=fun, args=())
  t1.setDaemon(True)
  #t1.setDaemon(True)
  t1.start()
  time.sleep(1)
  print "main thread end"
  
得到的输出是

 main thread
 start fun
 # 这里主线程退出了，直接终止了守护线程
 main thread end

Threading文档中说明，当守护线程被意外终止(主线程退出, 关机等等)，其资源可能无法被合理的释放。因此如果需要线程被优雅的结束，请设置为非Daemon线程，并使用合理的信号方法，如事件Event。

Daemon threads are abruptly stopped at shutdown. Their resources (such as open files, database transactions, etc.) may not be released properly. If you want your threads to stop gracefully, make them non-daemonic and use a suitable signalling mechanism such as an Event.

关于守护线程的用途

Daemon线程当且仅当主线程运行时有效，当其他非Daemon线程结束时可自动杀死所有Daemon线程。如果没有Daemon线程的定义，则必须手动的跟踪这些线程，在程序结束前手动结束这些线程。通过设置线程为Daemon线程，则可以放任它们运行，并遗忘它们，当主程序结束时这些Daemon线程将自动被杀死。

Daemons are only useful when the main program is running, and it's okay to kill them off once the other non-daemon threads have exited. Without daemon threads, we have to keep track of them, and tell them to exit, before our program can completely quit. By setting them as daemon threads, we can let them run and forget about them, and when our program quits, any daemon threads are killed automatically.


所以，curio中将一些io操作放置在守护线程中也是合理的.


curio.spawn流程
=================

curio.spawn是一个asyn函数, 同步函数无法调用它, 所以保证了同步函数的Causality. 在callback模式下, 同步函数注册一个callback为f, 然后继续执行之后的代码g, 这样f和g的调用顺序并不会.

而在curio中, 当你在main中调用await curio.spawn(coro)孵化一个协程的时候, curio.kernel将会把coro加入到curio.kernel自己的ready列表中, 然后再将main加入到curio.kernel的列表中, 所以coro一旦挂起,

接下来就马上启动main, 这个时候再执行main之后的代码, 我们直到coro已经启动并且挂起了. 而在callback模式下, 执行到coro后面的代码的时候, 有可能coro并没有执行挂起.

curio.spawn执行就是kernel调用kernel.run._trap_spawn

.. code-block:: python

   # curio.kernel.run._trap_spawn
   def _trap_spawn(coro, daemon):
       # _new_task就是直接把coro加入到kernel._ready列表中
       task = _new_task(coro, daemon)
       # 此时kernel._ready列表为[..., Task(coro)]
       # 将current的本地变量赋值到task中
       _copy_tasklocal(current, task)
       # 这里再把curren加入到_ready列表中
       _reschedule_task(current, value=task
       # 此时kernel._ready列表为[..., Task(coro), current]
   
   # curio.kernel.run._new_task
   def _new_task(coro, daemon=False):
       nonlocal njobs
       # 用Task包装coro
       task = Task(coro, daemon)
       tasks[task.id] = task
       if not daemon:
           njobs += 1
       # 将task加入到kernel._ready列表中
       _reschedule_task(task)
       return task
   # curio.kernel.run._reschedule_task
   def _reschedule_task(task, value=None, exc=None):
       ready_append(task)
       # 在_trap_spawn最后调用_reschedule_task中, task就是current, value就是coro
       # 这里就保存了哪个函数spawn了哪个函数
       task.next_value = value
       task.next_exc = exc
       task.state = 'READY'
       task.cancel_func = None

  所以, 一旦coro挂起, 接下来就恢复current, 也就是调用curio.spawn(coro)的函数.

