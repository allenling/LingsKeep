Some thoughts on asynchronous API design in a post-async/await world
=====================================================================

出处: https://vorpus.org/blog/some-thoughts-on-asynchronous-api-design-in-a-post-asyncawait-world/

文中观点的小结是: asyncio是一种混用了call back为主, 添加了async/await支持的框架, 所以asyncio基本形式还是例如epoll处理过程.
而curio则是抛弃了所有的call back, 只支持async/await, 也就是event loop的主干是coro.send(None)的形式, 也就是只是对协程进行启动和再启动.

其中有三个例子

第一个是使用asyncio的protocl/transport的模式, 也就是最基本的基于回调的模式
第二个是curio
第三个个是使用asyncio的stream模式, 支持async/await.

其中asyncio有三个bug, 基本都是由于其callback的实现风格造成的.


bug1. backpressure, 后压
----------------------------

这里是写数据的时候, 局部缓冲膨胀导致内存消耗殆尽, 或者高延迟.

一旦read的速度大于write的速度, 要么消耗完内存, 要么有大量的延迟. 这是因为transport.write不会等待数据发送完, 相反, 若遇到数据一次性发送不完, 则将剩余的数据
存储在自己的一个buffer中, 同时向event loop注册一个write事件, 时间的call back就是将自己buffer中的数据发送出去, 一直等到buffer中没有数据. 而curio中, 读的时候都是必须发送才结束, 不存在
局部buffer来存储待发送数据.

内存膨胀是因为累计在自己的buffer中的数据会越来越多, 也正由于buffer越多, 后面接收的数据需要等待很长一段时间才能发送, 造成大规模延迟.

文中例子假设接受数据是3MB/s, 发送是1MB/s, 则10秒后, 有20MB存储在我们的buffer中, 若此时有数据发送过来, 这些发送过来的数据必须等待这20MB数据发送完, 也就是20s.
时间越长, buffer越到, 然后延迟越大

asyncio中socket使用的_SelectSocketTransport代码如下

.. code-block:: python

  class _SelectorSocketTransport(_SelectorTransport):

      def write(self, data):
          # 省略了代码
          # 如果待发送的buffer还有数据, 则将数据放到待发送buffer里面
          if not self._buffer:
              try:
                  # 直接发送
                  n = self._sock.send(data)
              except (BlockingIOError, InterruptedError):
                  pass
              except Exception as exc:
                  self._fatal_error(exc, 'Fatal write error on socket transport')
                  return
              else:
                  data = data[n:]
                  if not data:
                      return
              # 若还有数据, 注册一个write事件, handler是self._write_ready
              self._loop.add_writer(self._sock_fd, self._write_ready)
  
          self._buffer.extend(data)
          self._maybe_pause_protocol()
  
      def _write_ready(self):
          assert self._buffer, 'Data should not be empty'
  
          if self._conn_lost:
              return
          try:
              # 继续发送待发送buffer中的数据
              n = self._sock.send(self._buffer)
          except (BlockingIOError, InterruptedError):
              pass
          except Exception as exc:
              self._loop.remove_writer(self._sock_fd)
              self._buffer.clear()
              self._fatal_error(exc, 'Fatal write error on socket transport')
          else:
              if n:
                  # 缩减待发送buffer的长度
                  del self._buffer[:n]
              self._maybe_resume_protocol()  # May append to buffer.
              # 直到待发送buffer没有数据
              if not self._buffer:
                  self._loop.remove_writer(self._sock_fd)
                  if self._closing:
                      self._call_connection_lost(None)
                  elif self._eof:
                      self._sock.shutdown(socket.SHUT_WR)




文中作者给出了修复这个bug的思路, 也就是先等dest的transport发送完数据, source的transport再继续读数据, 或者, 等待dest的transport数据发送完了
, source transport再读取数据.

文中作者给出了asyncio的async/await的修复方式, 思路是后一种, 调用des_transport.drain, 把数据发送完, source_transport再读取数据.
drain调用其实不会把所有的数据都发送出去, 只是当数据长度高于高水位的时候, 发送数据, 直到数据长度低于低水位.

下面是asyncio的protocol模式的修复方式, 思路是第一种, dest发送数据的时候, 停止source读取数据, 直到待发送buffer为0, 重新启动source的读操作.


.. code-block:: python


  class OneWayProxyDest(asyncio.Protocol):
      def send_data(self, data):
          self.transport.write(data)
          # 这里先暂停sour_transport的读
          self.source_transport.pause_reading()
  
      def resume_writing(self):
          # 这里, 若待发送的buffer为空, 则再启动source_transport的读操作
          if not self._buffer:
              self.source_transport.resume_reading()


bug3. 关闭event loop导致数据丢失
---------------------------------

这个bug也是因为存在write buffer, write操作并没有真正的去发送数据, 导致关闭event loop的时候, 若buffer中还有数据, 则会丢失这部分数据.

在关闭loop的时候，asyncio并没有等write操作完全完成才关闭loop，这导致会有一些数据未被发送。

可以在关闭loop之前，调用transport中的drain方法，等待write尽可能的发送数据，之所以是尽可能而不是完全是因为drain方法会在数据达到高水位的时候，阻塞直到数据量低于低水位.

所以，仍然有一些在低水位之下的数据未被发送而loop却关闭了

我们可以调用transport.set_write_buffer_limits(0)把高低水位设置都设置位0，这样调用drain的时候就会阻塞到完全发送完毕。但是我们要访问transport对象，就必须把asyncio.open_connection的实现复制到我们代码中，
才能调用set_write_buffer_limits方法.

但是，文档说将高低水位设为0并不是最优的选择，因为高低水位的设置是不浪费带宽所设置的缓冲值.



.. code-block:: python

  # 例子中问题所在
  async def proxy(loop, connect_event, server_closed_event,
                  dest_host, dest_port,
                  source_reader, source_writer):
      connect_event.set()
      try:
          with closing(source_writer):
              tmp = await asyncio.open_connection(dest_host, dest_port, loop=loop)
              dest_reader, dest_writer = tmp
              # 这里，当copy_all返回的时候，会调用dest_writer.close
              # 但是，此时数据并没有发送完毕
              with closing(dest_writer):
                  await copy_all(source_reader, dest_writer)
      finally:
          await server_done_event.wait()
          # 然后我们就直接关闭loop
          loop.stop()

  # 在关闭loop之前先调用drain方法
  async def proxy(loop, connect_event, server_closed_event,
                  dest_host, dest_port,
                  source_reader, source_writer):
      connect_event.set()
      try:
          with closing(source_writer):
              tmp = await asyncio.open_connection(dest_host, dest_port, loop=loop)
              dest_reader, dest_writer = tmp
              try:
                  await copy_all(source_reader, dest_writer)
              finally:
                  # 在关闭dest_writer之前，调用drain
                  await dest_writer.drain()
                  dest_writer.close()
      finally:
          await server_done_event.wait()
          loop.stop()

高低水位的默认值

.. code-block:: python

  # asyncio.transports._FlowControlMixin
  class _FlowControlMixin(Transport):
      def _set_write_buffer_limits(self, high=None, low=None):
          if high is None:
              if low is None:
                  high = 64*1024
              else:
                  high = 4*low
          if low is None:
              low = high // 4
          if not high >= low >= 0:
              raise ValueError('high (%r) must be >= low (%r) must be >= 0' %
                               (high, low))
          self._high_water = high
          self._low_water = low



作者提出, 由于os的send调用也并不是直接发送数据, 而是把数据加入到内核中的待发送buffer中, 而select调用通过自己实现了高低水位的逻辑, 这样就不必等
内核中的发送缓冲区满了才标记socket为可发送, 也就是不需要我们实现高低水位逻辑, 而且内核中的待发送buffer通常是足够用的, 所以内核帮我们做了一切事情. 所以,在用户空间(程序)
设置一个发送缓冲区完全是多余的.

而作者提出, asyncio的缓冲区的并不能提升性能, 缓冲区和高低水位设自己完全只是为了让transport.wirte的调用是异步的. 所以, 文档上关于高低水位的说法是错误的, 我们应该让高低水位的值为0, 才能避免内存
膨胀带来的很多问题.


关闭event loop前先关闭资源
~~~~~~~~~~~~~~~~~~~~~~~~~~~~


在文中asyncio例子中, 关闭event loop的时候, 有可能writer的socket并没有被关闭, 这对于示例程序来来说倒是无所谓, 但是在一般情况下, 这种情况并不好.

比如在测试中, 每个功能都单独使用不同的event loop来运行, 若每个event loop关闭的时候都遗留有额外资源未关闭, 就很可能出现问题.

asyncio.stream.StreamWriter的代码

.. code-block:: python

  class StreamWriter:
      def close(self):
          # 调用self._transport.close
          # 也就是asyncio.selector_events._SelectorTransport.close
          return self._transport.close()

  class _SelectorTransport(transports._FlowControlMixin, transports.Transport):

      def close(self):
          if self._closing:
              return
          self._closing = True
          self._loop.remove_reader(self._sock_fd)
          if not self._buffer:
              self._conn_lost += 1
              self._loop.remove_writer(self._sock_fd)
              # 调用call_soon, 下一次event loop迭代的是调用self._call_connection
              self._loop.call_soon(self._call_connection_lost, None)

      def _call_connection_lost(self, exc):
          try:
              if self._protocol_connected:
                  self._protocol.connection_lost(exc)
          finally:
              # 这里真正的关闭socket
              self._sock.close()
              self._sock = None
              self._protocol = None
              self._loop = None
              server = self._server
              if server is not None:
                  server._detach()
                  self._server = None

  # asyncio.base_events.BaseEventLoop
  class BaseEventLoop(events.AbstractEventLoop):

      def run_forever(self):
          self._check_closed()
          if self.is_running():
              raise RuntimeError('Event loop is running.')
          self._set_coroutine_wrapper(self._debug)
          self._thread_id = threading.get_ident()
          try:
              while True:
                  self._run_once()
                  # 完成self.scheduled中的task之后, 判断是否要停止
                  if self._stopping:
                      break
          finally:
              self._stopping = False
              self._thread_id = None
              self._set_coroutine_wrapper(False)
      def stop(self):
          self_stopping = True

在asyncio的async/await例子中, 关闭dest_writer之后, 调用loop.close关闭loop, 这个时候有可能event loop中的scheduled的任务为空, 然后跳出run_once, 遇到stopping=True, 直接break, 没有再次
去执行_SelectorTransport._call_connection_lost, 即使_SelectorTransport._call_connection_lost已经被加入到scheduled任务列表中


所以, 在event loop关闭必须等待完全关闭socket, 我们可以在关闭dest writer和loop.stop之前yield一次到event loop, yield出去之后, _SelectorTransport._call_connection_lost在scheduled任务
列表中, event loop必定会执行完scheduled任务列表才回去判断stopping, 所以socket会被完全关闭.


最后, 作者给出了asyncio的async/await模式的完整代码, 主要是transport.set_write_buffer_limits(0)设置write buffer为0, 

.. code-block:: python

  # 下面主要是设置高低水位
  @asyncio.coroutine
  def fixed_open_connection(host=None, port=None, *,
                            loop=None, limit=65536, **kwds):
      if loop is None:
          loop = asyncio.get_event_loop()
      reader = asyncio.StreamReader(limit=limit, loop=loop)
      protocol = asyncio.StreamReaderProtocol(reader, loop=loop)
      transport, _ = yield from loop.create_connection(
          lambda: protocol, host, port, **kwds)
      ###### Following line added to fix buffering issues:
      # 这里设置了高低水位都是0
      transport.set_write_buffer_limits(0)
      ######
      writer = asyncio.StreamWriter(transport, protocol, reader, loop)
      return reader, writer


  try:
      await copy_all(source_reader, dest_writer)
  finally:
      # 尽可能地发送数据
      await dest_writer.drain()
      dest_writer.close()
      # yield 一个task到scheduled任务列表
      # 这样下event loop也就会处理到_SelectorTransport._call_connection_lost任务
      await asyncio.sleep(0, loop=loop)


Causality
-------------

文中提出Causality这个概念, 个人理解为程序应该遵循执行先后的逻辑顺序. 若f();g(), 则意味着f()执行完, 才会去执行g().

常规同步模式下, 确实是Causality的, 若是callback模式, f中注册了一个call back, 然后执行到g, 这时执行到g的时候, f并没有执行完, 就发生了f和g的执行是交叉在一个的情况.

这个时候, f和g哪个先执行完是不确定的, 若我们在g执行完之后马上退出, f有可能没有执行完.

文中根据Unyielding(https://glyph.twistedmatrix.com/2014/02/unyielding.html)这篇博文中一个程序的执行复杂度的描述:

`When you’re looking at a routine that manipulates some state, in a single-tasking, nonconcurrent system, you only have to imagine the state at the beginning of the routine, and the state at the end of the routine. To imagine the different states, you need only to read the routine and imagine executing its instructions in order from top to bottom. This means that the number of instructions you must consider is n, where n is the number of instructions in the routine. By contrast, in a system with arbitrary concurrent execution – one where multiple threads might concurrently execute this routine with the same state – you have to read the method in every possible order, making the complexity n**n.`

一个单任务, 非并发系统, 一个程序的执行逻辑是顺序的, 从头到尾执行. 若一个程序有n条指令, 你只需要顺序去理解这n条指令.

而在一个执行顺序是不确定的, 也就是任何指令都有可能在其他指令执行前执行, 你需要去理解n**n个情况.


文中指出, 若有N个线程并发执行有Y个挂起点(Y yield points)的程序, 会有N**Y个可能执行的顺序(个人理解: 每个挂起点有N个线程执行的可能, 所有是Y个N相乘). 原生线程的Y很大, 而回调方式的
协程或者async/await的协程的Y很小.


但是, 在回调模式中, 每次注册一个回调函数, 都产生一个新的线程, 所有回调模型下虽然有小Y, 但是有大N, 换句话说, 违反了Causality, 这也影响了asyncio中的async/await模式. 在上述文中中, 有大多数
是因为违反调用的逻辑先后顺序, 也就是调用f();g()的时候, 当f看起来结束了, 实际上还没结束, 我们就开始调用g.

curio都遵循Causality, 包括curio.run_in_{thread,process,executor}, 因为curio.run_in_{thread,process,executor}都是挂起, 等待结果返回的.


* curio.spawn是一个asyn函数, 同步函数无法调用它, 所以保证了同步函数的Causality. 在callback模式下, 同步函数注册一个callback为f, 然后继续执行之后的代码g, 这样f和g的调用顺序并不会.

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

* 在asyncio中, 你可以生成很多不同的对象, 并且将该对象加入到event loop执行的列表中, 比如loop.add_reader, 创建一个reader对象, asyncio.Task回调, Future回调. 在curio中, 只有所以的程序
  都必须被包装成curi.task.Task对象.

* curio记录了哪个函数spawn了哪个函数,并且将父程序的task_local_storage变量复制到子程序中, 或许这让我们可以进行这样的操作: 取消当前任务已经其spawn的子任务. 也就是说, curio目前还不支持这么做.

curio的最主要的优势还是在于我们写的代码直接明了, 是强制顺序的.


Who needs causality, really?
------------------------------

两个例子

1. HTTP servers
~~~~~~~~~~~~~~~~~

例子是一个客户端, 其不断发起请求, 但是并不会读取返回. 当这个客户端向一个twisted的server发起这样一个压力测试的时候, 服务端会因为内存耗尽而崩溃.

这个是因为twisted的server是不断将要发送的response保存到自己的发送缓存区里面, 接着继续处理请求.

由于客户端并不读取返回值, server端的发送缓冲区就越来越大, 耗尽内存. twisted的内存使用增幅非常大, 基本上, 如果上一个response大小为100KB, 则客户端每请求1MB的数据, server会吃掉2GB的内存.

关于twisted耗尽内存的issue: https://twistedmatrix.com/trac/ticket/8868

这是一个典型的DOS攻击.


如果把server换成aiohttp, 鲁棒性更好, aiohttp的server在处理request之后, 调用了StreamWriter.drain, 确保自己的发送缓冲区不会无限制增长.

aiohttp最终也会崩溃, 这是asyncio会继续处理下一个请求, 即使当前请求还在处理中, 也就是asyncio会不断地获取请求, 生成response, 虽然aiohttp已经调用了StreamWriter.drain, 但是
发送缓存区依然有无限增长的可能.

https://github.com/KeepSafe/aiohttp/issues/1368中说明了期望的情况, 若还有一定数量的response没有被读取的时候(发送缓存区大小达到限制), 则停止处理下一个请求.

aiohttp自己的一些关于发送缓冲区的实现, 把发送缓存去的增幅控制在一倍, 对比起来小很多. 若一个客户端想要dos一个aiohttp, 可能需要发送GB的数据, 但是发送GB可能需要上千个连接, 所以, 还是限制住的.


aiohttp的graceful shutdown, 最后是调用StreamWriter.close, 依然会丢失数据, 这是asyncio机制决定的.


2. Websocket servers
~~~~~~~~~~~~~~~~~~~~~~

websocket一旦建立连接, 服务端会一直发送msg给客户端.

例子是一个客户端, 建立websocket连接, 然后就挂起了. 然后就像http servers例子中的服务端一样, 这里依然由于客户端没有真正的读取返回, 导致服务端的发送缓冲区膨胀.

这种情况也很正常正常, 因为有可能客户端建立连接之后就崩溃或者离线了.


大多数websocket的项目都是使用类似与asyncio的方式取写数据，即将要发送的数据放在程序的发送缓存区， 所以，这很容易导致发送缓存区膨胀的问题.


* aiohttp默认是没办法避免这个问题. 在aiohttp中使用await来接受数据，应该能将后压应用给发送太快的客户端. 这里的意思应该是一旦发送缓存区到达现在， 就await等待发送缓存区为空
  在接受数据，await作用就是挂起接收操作知道发送缓存区为空.

* autobahn默认也没办法避免这个问题. 当autobahn在twisted模式下，可以通过注册一个twisted producer来通知客户端发送太快了， 而在asyncio模式下， 没有实现这个注册producer的功能.

* tornado默认也不能避免这个问题，但是它有一些api可以让开发者取自己实现解决后压.


关于tornado，他们在4.3之后的WebSocketHandler.write_message将返回一个future对象，然后我们可以在这个future对象上使用await，例如: await websock.write_message(...)，强制等待future对象完成，也就是write_message完成，这样也可以
将后压应用到客户端. 但是，在write_message之前不使用awiat也可以运行，这样就没办法将后压应用给客户端了. 



Other challenges for hybrid APIs
----------------------------------

Timeouts and cancellation
~~~~~~~~~~~~~~~~~~~~~~~~~~~

超时和取消操作是很常见的, curio提供了curio.timeout_after函数, 并且可以使用context manager的形式使用它, 很方便.

而在asyncio中, 由于asyncio是混合了callback和async/await, 设置超时的代价相对curio来说很大, 有很多不必要的操作.

asyncio中超时是基于future的，并没有一个callback级别的timeout, 所以必须由我们自己实现. 在curio中，不需要取适应两个风格的timeou，所以更简单方便.

文中的例子是，两个任务await同一个future对象，当你cancel第一个任务(或者说任意一个任务)的时候，两个任务都会被cancel.



Cancellation
~~~~~~~~~~~~~~~

在文中关于asyncio的cancel例子中, 调用了task1.cancel()导致了task2也被cancel了, 并且注释掉event1.wait或者将task1.cancel放到event1.wait之后，都会将task2取消掉.

关于文中说注释调event1.wait会导致程序挂起，但是我实验了下，并没有挂起, 并且task1和task2都被cancel了.(ubuntu16.04.1, python3.5, asyncio3.3).

注释掉event1.wait理应不会挂起的，因为这只是等待event在其他地方被set而已，注释掉只是不等待了，应该是保留event1.wait而注释掉event.set()才会挂起.

所以
**expected in the first place – but we might not have expected that line to affect the result). Note also also that if we move the cancellation to just after the call to event1.wait(), before spawning task2, then the program does not hang – so we can't avoid this by checking for multiple waiters when propagating cancellations from tasks->futures.)**
这个没看懂


我实验下来还有一个情况是，若task1.cancel缓存task2.cancel，await task2换成await task1, 输出依然是task 1 cancelled/task 2 cancelled.


上述现象的原因是一般一个future有一个唯一的consumer，但是确实可以有任意多个, 比如文中的例子，task2和task2都是同一个future的consumer. 当调用Task.cancel, cancel中不能区分
当前的future是否可以作为当前的task的"一部分". curio中没有Future和回调链，所以不会有这种现象.

对应到例子，也就是说调用task1.cancel后，asyncio.tasks.Task.cancel中会把调用future.cancel， 而future.cancel会把所有的callback都加入到event loop中，也就是强制执行所有的task
这样task1.cancel导致task2也被强制执行，当执行task1和task2的时候，调用Future.result，Future.result中发现当前的状态是_CANCELLED, 则抛出异常, 所以task2也会被强制cancel.

所以说task和future都没辨别当前的cancel是否需要强制执行task2，或者说不需要执行task2.

.. code-block:: python

  # asyncio.tasks.Task.cancel
  def cancel(self):
      if self.done():
          return False
      if self._fut_waiter is not None:
          # 这里取消所有的future waiter
          # 对应到例子，就是task1和task2公用的future
          if self._fut_waiter.cancel():
              return True
      self._must_cancel = True
      return True

  # asyncio.futures.Future.cancel
  def cancel(self):
      if self._state != _PENDING:
          return False
      # 这里future的状态被置为_CANCELLED
      self._state = _CANCELLED
      # 这里会执行所有的task
      self._schedule_callbacks()
      return True

  # asyncui.futures.Future._schedule_callbacks

  def _schedule_callbacks(self):
      callbacks = self._callbacks[:]
      if not callbacks:
          return

      self._callbacks[:] = []
      # 将callback加入到event lop中，也就是强制执行task1和task2
      for callback in callbacks:
          self._loop.call_soon(callback, self)

  # asyncio.futures.Future.result
  def result(self):
      # 这里状态是_CANCELLED, 引发异常，所以task2也取消了
      if self._state == _CANCELLED:
          raise CancelledError
      if self._state != _FINISHED:
          raise InvalidStateError('Result is not ready.')
      self._log_traceback = False
      if self._tb_logger is not None:
          self._tb_logger.clear()
          self._tb_logger = None
      if self._exception is not None:
          raise self._exception
      return self._result

Timeout
~~~~~~~~~~

在超时例子中，即使我们在write的时候调用了drain, 超时之后，由于write本身是回调的，所有即使引发超时异常，write操作还是会继续.

任何调用cancellation-unsafe的函数都是cancellation-unsafe, 由于在asyncio中，write本身是cancellation-unsafe的，所有似乎没有办法取使得调用write的函数变为cancellation-unsafe的.

而curio所有的基本操作都是cancellation-safe，在此基础上，我们的代码只需要try/with到cancellation异常，然后做出合适的清理工作就好了.




Review and summing up: what is "async/await-native" anyway?
===============================================================


1. 一个纯async/await模式的application是由一系列的协程构成的，并且所有的逻辑都在这些协程中完成.

2. 这些协程是可监督的，也就是说这些协程必须是有停止的条件或者可以被显式地cancel

3. 所有的spawn都是显式的，而非隐式的.

4. 程序调用栈的每一帧都是常规的sync或者async函数，并且是执行顺序一定是从上到下的(也就是说上一个函数开始执行了，下一个函数才能执行).

5. 错误，取消操作以及超时都是通过异常来引发.

6. 清理资源已经错误捕获处理都是被with或者try管理的.

