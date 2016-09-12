Gunicorn Prefork
=======================
master是一个Arbiter类, 由它来孵化子进程, 管理子进程的.
worker在Prefork模式下, 是gunicorn.workers.sync.SyncWorker类.
下面master就是指Arbiter类, worker就是指gunicorn.workers.sync.SyncWorker


master.run的过程
-----------------

孵化worker是使用os.fork, 然后将worker的pid保存起来, 在run循环中, 每次都去查看num_workers是否和当前保存的worker数量相等, 不想等则增加/减少worker数目

.. code-block:: python

   # gunicron.arbiter

   class Arbiter(object):
       def run(self):
           # start方法是初始化资源
           self.start()
           util._setproctitle("master [%s]" % self.proc_name)

           try:
               # 增加/减少worker数目
               self.manage_workers()
               while True:
                   self.maybe_promote_master()

                   # 获取信号, 处理信号
                   sig = self.SIG_QUEUE.pop(0) if len(self.SIG_QUEUE) else None
                   if sig is None:
                       # 如果不sleep, 则会吃满CPU
                       self.sleep()
                       # murder_workers主要是杀死超时的worker
                       self.murder_workers()
                       # 每次都去看worker数目是否异常
                       self.manage_workers()
                       continue

                   if sig not in self.SIG_NAMES:
                       self.log.info("Ignoring unknown signal: %s", sig)
                       continue

                   signame = self.SIG_NAMES.get(sig)
                   handler = getattr(self, "handle_%s" % signame, None)
                   if not handler:
                       self.log.error("Unhandled signal: %s", signame)
                       continue
                   self.log.info("Handling signal: %s", signame)
                   handler()
                   self.wakeup()
           except StopIteration:
               # 这里一般是TERM/QUIT信号停掉master
               self.halt()
           except KeyboardInterrupt:
               # 这里一般是CTRL-C停掉master
               self.halt()
           except HaltServer as inst:
               self.halt(reason=inst.reason, exit_status=inst.exit_status)
           except SystemExit:
               # 这里一般是worker从自己的while循环return出来或者worker调用sys.exit
               raise
           except Exception:
               self.log.info("Unhandled exception in main loop",
                             exc_info=True)
               self.stop(False)
               if self.pidfile is not None:
                   self.pidfile.unlink()
               sys.exit(-1)


管理worker
-------------

.. code-block:: python

   # gunicorn.arbiter

   class Arbiter(object):
       def manage_workers(self):
           """\
           Maintain the number of workers by spawning or killing
           as required.
           """
           # 如果worker数量少了, 重新孵化一个worker
           if len(self.WORKERS.keys()) < self.num_workers:
               self.spawn_workers()

           workers = self.WORKERS.items()
           workers = sorted(workers, key=lambda w: w[1].age)
           # worker多了, 则删掉最老的worker
           while len(workers) > self.num_workers:
               (pid, _) = workers.pop(0)
               self.kill_worker(pid, signal.SIGTERM)


孵化worker
-----------

孵化worker在spawn_worker方法中, 使用os.fork

.. code-block:: python

   # gunicorn.arbiter

   class Arbiter(object):
       def spawn_worker(self):
           # 记录worker的age, 每次有新的worker生成, 则age加1, 当reload的时候, 会先生成新的worker, 然后删除老的worker
           # age越小越老
           self.worker_age += 1
           # worker类
           worker = self.worker_class(self.worker_age, self.pid, self.LISTENERS,
                                      self.app, self.timeout / 2.0,
                                      self.cfg, self.log)
           self.cfg.pre_fork(self, worker)
           # fork子进程
           pid = os.fork()
           if pid != 0:
               self.WORKERS[pid] = worker
               return pid

           # Process Child
           worker_pid = os.getpid()
           try:
               util._setproctitle("worker [%s]" % self.proc_name)
               self.log.info("Booting worker with pid: %s", worker_pid)
               self.cfg.post_fork(self, worker)
               worker.init_process()
               # 这里若是子进程自己sys.exit, 会被下面的SystemExit捕获,
               # 或者子进程从while循环return出来, 下面的sys.exit保证也会被下面的SystemExit捕获到
               # 所以子进程sys.exit或者return都表示子进程结束
               sys.exit(0)
           except SystemExit:
               raise
           except AppImportError as e:
               self.log.debug("Exception while loading the application",
                              exc_info=True)
               print("%s" % e, file=sys.stderr)
               sys.stderr.flush()
               sys.exit(self.APP_LOAD_ERROR)
           except:
               self.log.exception("Exception in worker process"),
               if not worker.booted:
                   sys.exit(self.WORKER_BOOT_ERROR)
               sys.exit(-1)
           finally:
               self.log.info("Worker exiting (pid: %s)", worker_pid)
               try:
                   worker.tmp.close()
                   self.cfg.worker_exit(self, worker)
               except:
                   self.log.warning("Exception during worker exit:\n%s",
                                     traceback.format_exc())


信号处理以及关闭master
----------------------------

gunicorn中, 信号处理串行化而不是使用call back的形式, 把接收到的信号放在一个列表中, 每次循环就去信号列表中pop出信号名称, 然后唤醒master, 唤醒master是用pipe, 然后select监听pipe.

而SIGCHLD信号必须要单独使用call back. 这是因为关闭master的时候, 必须同时调用waitpid去清理已经sys.exit的worker.


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


reload
--------

reload的过程就是很直接了

重置环境变量, reload app, 启动(setup)app, 重新打开log文件, 孵化新的worker, 最后manager_workers.

最后先孵化新的worker, 会有2x(x为worker的数量)的worker进程, 然后manager_workers的时候, 会把老的worker给删掉

reload wsgi是在worker孵化的时候, 调用顺序为worker.spawn_worker->worker.init_process->base.Worker.load_wsgi->base.Worker.wsgi->wsgiapp.load->wsgiapp.load_wsgiapp->util.import_app


.. code-block:: python

   # gunciron.app.wsgiapp

   class WSGIApplication(Application):

       def load_wsgiapp(self):
           self.chdir()

           # load the app
           # 导入app, 如util.import_app('my.app.module')
           return util.import_app(self.app_uri)
