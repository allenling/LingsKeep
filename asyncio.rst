async/await和asyncio.coroutine(types.coroutine)/yield from
=============================================================

async定义一个函数为协程是将该函数作为collections.abc.Coroutine的实例, 属于严格定义协程.

由于python中协程是一开始是用强化生成器的方式(加入send, close, throw方法, 主要是send方法可以将值传递给生成器, 使其接收传入值并重新启动), 所以
协程在python之前都是基于生成器的协程.

而async的出现, 可以直接使用async def定义一个协程, 协程要求函数内部必须有await一个awaitable对象, await作用是
调用awaitable对象的__await__方法, 该方法返回一个可迭代对象, 并且暂停函数运行.

asyncio.coroutine(types.coroutine)装饰器是为了区分一个生成器是用来作为一般性生成器(用来迭代)还是用来作为协程(基于生成器的协程). 一个生成器
不带asyncio.coroutine(types.coroutine)装饰器也可以在asyncio的event loop中运行.

.. code-block:: python

  import asyncio
  
  
  def test():
      yield from asyncio.sleep(1)
      print ('test done')
  
  
  def main():
      t = test()
      print (t)
      loop = asyncio.get_event_loop()
      tasks = [t]
      loop.run_until_complete(asyncio.wait(tasks))
      loop.close()
  
  main()


test不带asyncio.coroutine(types.coroutine)装饰器, 一样可以正常在event loop中执行. 所以一个函数, 主要能暂停, 可以传入一个值, 并且重新启动, 就是协程了.

asyncio.coroutine和async只是定义上的区别, 并不是能否执行的关键. 关键是暂停之后, 传出来的参数是否是一个awaitable对象, 因为event loop是观察一个awaitable对象来判断
一个协程是否暂停或者需要重新启动.

yield from 后面能接一个协程的条件是本身就是一个协程(用asyncio.coroutine装饰或者async def定义)


def test():
    # co是协程
    # 由于test本身不是协程,所以yield from后面不能跟协程
    yield from co()

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

可以调用drain, 但是由于高低水位配置,


关于用户控件的缓冲区
-------------------------

.. code-block:: python



作者提出, 由于os的send调用也并不是直接发送数据, 而是把数据加入到内核中的待发送buffer中, 而select调用通过自己实现了高低水位的逻辑, 这样就不必等
内核中的发送缓冲区满了才标记socket为可发送, 也就是不需要我们实现高低水位逻辑, 而且内核中的待发送buffer通常是足够用的, 所以内核帮我们做了一切事情. 所以,在用户空间(程序)
设置一个发送缓冲区完全是多余的.

而作者提出, asyncio的缓冲区的并不能提升性能, 缓冲区和高低水位设自己完全只是为了让transport.wirte的调用是异步的. 所以, 文档上关于高低水位的说法是错误的, 我们应该让高低水位的值为0, 才能避免内存
膨胀带来的很多问题.


关闭event loop前先关闭资源
----------------------------


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





Other challenges for hybrid APIs
----------------------------------

Timeouts and cancellation
~~~~~~~~~~~~~~~~~~~~~~~~~~~

超时和取消操作是很常见的, curio提供了curio.timeout_after函数, 并且可以使用context manager的形式使用它, 很方便.

而在asyncio中, 由于asyncio是混合了callback和async/await, 设置超时的代价相对curio来说很大, 有很多不必要的操作.

asyncio中并没有一个callback级别的timeout, 所以必须由我们自己实现.

**没太看懂

First, since we can't assume that everyone is using async/await, our hybrid system needs to have some alternative, redundant system for handling timeouts and cancellations in callback-using code – in asyncio this is the Future cancellation system, and there isn't really a callback-level timeout system so you have to roll your own. In curio, there are no callbacks, so there's no need for a second system. In fact, in curio there's only the one way to express timeouts – timeout= kwargs simply don't exist. So we can focus our energies on making this one system as awesome as possible.**


