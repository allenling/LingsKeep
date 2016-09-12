Gunicorn Prefork
=======================
master是一个Arbiter类, 由它来孵化子进程, 管理子进程的.
worker在Prefork模式下, 是gunicorn.workers.sync.SyncWorker类.
下面master就是指Arbiter类, worker就是指gunicorn.workers.sync.SyncWorker


信号处理以及关闭master
----------------------------

gunicorn中, 信号处理串行化, 把接收到的信号放在一个列表中, 每次循环就去信号列表中pop出信号名称, 然后唤醒master, 唤醒master是用pipe, 然后select监听pipe.


.. code-block:: python

   # gunicorn.arbiter

   class Arbiter(object):
       def init_signals(self):
           # close old PIPE
           if self.PIPE:
               [os.close(p) for p in self.PIPE]

           # initialize the pipe
           self.PIPE = pair = os.pipe()
           for p in pair:
               util.set_non_blocking(p)
               util.close_on_exec(p)

           self.log.close_on_exec()
           '''
           上面是初始化pipe的过程
           这里, 所有的信号都是加入的列表串行化处理, 但是唯独SIGCHLD信号是用call back的形式去处理, 这是因为在关闭master的时候, 必须等待所有的worker都关闭.
           一旦一个worker exit, 则内核会发出SIGCHLD信号, 我们需要在该call back中调用waitpid去清理进程, pop掉worker
           '''
           [signal.signal(s, self.signal) for s in self.SIGNALS]
           signal.signal(signal.SIGCHLD, self.handle_chld)
       
        def stop(self, graceful=True):
            if self.reexec_pid == 0 and self.master_pid == 0:
                for l in self.LISTENERS:
                    l.close()

            self.LISTENERS = []
            sig = signal.SIGTERM
            if not graceful:
                sig = signal.SIGQUIT
            limit = time.time() + self.cfg.graceful_timeout
            # 这里先发一个TERM/QUIT信号, 然后worker会自己sys.exit, 这个时候需要我们去waitpid
            self.kill_workers(sig)
            # 这里等待SIGCHLD的call back去清理worker进程, pop掉worker
            while self.WORKERS and time.time() < limit:
                time.sleep(0.1)

            self.kill_workers(signal.SIGKILL)




当master被杀死之后, worker自己shutodown
------------------------------------------
当master被kill -9之后, worker在run中, 会每个循环去检查父进程是否变化, 变化的话就shutdown自己.

.. code-block:: python

   class SyncWorker(base.Worker):
       def run_for_one(self, timeout):
           listener = self.sockets[0]
           while self.alive:
               try:
                   self.accept(listener)
                   continue

               except EnvironmentError as e:
                   if e.errno not in (errno.EAGAIN, errno.ECONNABORTED,
                           errno.EWOULDBLOCK):
                       raise

               # 这里检查父进程是否有变化
               # 有变化就返回, shutdown gracefully
               if not self.is_parent_alive():
                   return

               try:
                   self.wait(timeout)
               except StopWaiting:
                   return

        def is_parent_alive(self):
            # 检查ppid和自己记录的ppid
            if self.ppid != os.getppid():
                self.log.info("Parent changed, shutting down: %s", self)
                return False
            return True
