celery的分布式设计
=======================

去中心化的模式而不是带有中心节点的设计

恩~~去中心化, 共识~~~嗯~~~是不是有点区块链的味道~~~~:)

所以, 这里梳理一下celery中分布式的思路, 看看去中心化的架构一般是怎么弄的

整体设计
=========


1. celery中, 并没有一个master去管理监控其他worker, 每一个worker都保存了集群中的信息(**state**), 比如已被撤销的任务(**invoked tasks**)和时钟(**clock**)等等

   并且信息是保存到内存的, 所以重启就会消失

2. celery中worker加入集群之后, 会寻找其他worker(**neighbor**), 然后发送消息, 通知其他worker自己加入, 同时其他worker会发送

   需要同步的数据给新加入的worker

3. worker之间并不会监控其他worker, 只是说一旦worker退出(正常退出), 那么退出的worker会发送一个退出的消息(**worker-offline**)给集群中其他worker

   而其他监控, 需要其他应用去监控, 比如flower会去监控worker的task等等, 同时, flower会让worker发送心跳(**heartbeat**)信息, 这样flower可以监控worker是否掉线了


4. celery中的监控是celery自己的特性, 而flower则是搜集celery提供的统计信息, 做一个数据可视化

   当然, 控制命令也是celery自己实现了, flower中的发送命令也是通过celery提供的功能

5. mingle是同步附近节点的组件


.. code-block:: python


        集群
    ------------------------+
                            |
    +---------+             |
    |         |             |
    | worker1 |             |
    |         |             |
    +---------+             |
                            |
                            |
    +---------+             |
    |         |             |
    | worker2 |             |
    |         |             |
    +---------+             |



广播命令
=============






mingle
========

直接看代码

.. code-block:: python


    class Mingle(bootsteps.StartStopStep):
        """Bootstep syncing state with neighbor workers.
    
        At startup, or upon consumer restart, this will:
    
        - Sync logical clocks.
        - Sync revoked tasks.
    
        """
    
        label = 'Mingle'
        requires = (Events,)
        compatible_transports = {'amqp', 'redis'}
    
        def __init__(self, c, without_mingle=False, **kwargs):
            self.enabled = not without_mingle and self.compatible_transport(c.app)
            super(Mingle, self).__init__(
                c, without_mingle=without_mingle, **kwargs)
    
        def compatible_transport(self, app):
            with app.connection_for_read() as conn:
                return conn.transport.driver_type in self.compatible_transports
    
        def start(self, c):
            self.sync(c)
    
        def sync(self, c):
            info('mingle: searching for neighbors')
            replies = self.send_hello(c)
            if replies:
                info('mingle: sync with %s nodes',
                     len([reply for reply, value in items(replies) if value]))
                [self.on_node_reply(c, nodename, reply)
                 for nodename, reply in items(replies) if reply]
                info('mingle: sync complete')
            else:
                info('mingle: all alone')
    
        def send_hello(self, c):
            inspect = c.app.control.inspect(timeout=1.0, connection=c.connection)
            our_revoked = c.controller.state.revoked
            replies = inspect.hello(c.hostname, our_revoked._data) or {}
            replies.pop(c.hostname, None)  # delete my own response
            return replies
    
        def on_node_reply(self, c, nodename, reply):
            debug('mingle: processing reply from %s', nodename)
            try:
                self.sync_with_node(c, **reply)
            except MemoryError:
                raise
            except Exception as exc:  # pylint: disable=broad-except
                exception('mingle: sync with %s failed: %r', nodename, exc)
    
        def sync_with_node(self, c, clock=None, revoked=None, **kwargs):
            self.on_clock_event(c, clock)
            self.on_revoked_received(c, revoked)
    
        def on_clock_event(self, c, clock):
            c.app.clock.adjust(clock) if clock else c.app.clock.forward()
    
        def on_revoked_received(self, c, revoked):
            if revoked:
                c.controller.state.revoked.update(revoked)

启动
---------

当mingle启动的时候, 调用self.sync, 而self.sync则是发送hello消息给其他worker, 这是一个广播消息, 其中带有自己的revoked的数据

.. code-block:: python

        def send_hello(self, c):
            # 广播命令的对象
            inspect = c.app.control.inspect(timeout=1.0, connection=c.connection)
            # 获取自己的revoke信息
            our_revoked = c.controller.state.revoked
            # 广播hello, 带上revoke信息
            replies = inspect.hello(c.hostname, our_revoked._data) or {}
            # 拿到reply, 删除自己的消息
            replies.pop(c.hostname, None)  # delete my own response
            # 返回replies
            return replies


同步
-------

收到同步信息之后, 调用self.on_node_reply开始同步返回的消息内容

.. code-block:: python

    def on_node_reply(self, c, nodename, reply):
        debug('mingle: processing reply from %s', nodename)
        try:
            self.sync_with_node(c, **reply)
        except MemoryError:
            raise
        except Exception as exc:  # pylint: disable=broad-except
            exception('mingle: sync with %s failed: %r', nodename, exc)


要同步的是时钟和revoked tasks

.. code-block:: python

    def sync_with_node(self, c, clock=None, revoked=None, **kwargs):
        # 同步时钟
        self.on_clock_event(c, clock)
        # 同步revoked tasks
        self.on_revoked_received(c, revoked)
    
    def on_clock_event(self, c, clock):
        # 教研时钟
        c.app.clock.adjust(clock) if clock else c.app.clock.forward()
    
    def on_revoked_received(self, c, revoked):
        # update revoked tasks
        if revoked:
            c.controller.state.revoked.update(revoked)

gossip
=======

集群选举的组件, **但是, 为什么需要选举?**


heartbeat
===========


