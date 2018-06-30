###########################
这里主要解析curio的源码实现
###########################


curio版本: >= 0.9, <=1.0


**对curio源码进行一个比较系统性的分析, 主要目的是理解async api的原理和正确的使用方式**

相似的项目有asyncio, trio:

* asyncio: 是标准库, 但是代码太难看了, 代码难看程度和celery有一拼, 所以不建议直接看asyncio的代码

           如果是想理解async-io, 那么curio是更好看的库

* trio   : **Nathaniel Smith** (这个作者对async有很大的参与度, 有博客和github, 值得看看)实现的async框架, 代码不是很难看

           其中参考了一些curio的设计, 也有其他自己不同于curio的设计的地方


下面简单聊一下curio中的概念, curio整个设计上借鉴了很多linux内核的概念(比如kernel, trap等等), 所以

如果了解linux内核的话, 那么理解起来就比较轻松一点

kernel
=========

curio中执行的主要执行代码的对象叫kernel, 这里可以类比linux中的kenerl, 只不过

curio的kernel是yield模式的调度, 和linux中的调度器(cfs, O(1))一样, 也是负责选择下一个任务, 以及维护任务的状态

比如curio中也有task, task也有状态, 比如状态有RUNNING, SLEEP等等

kernel也有各种队列, 比如ready(就绪队列), sleepq(休眠队列)等等, 当然, 具体实现可能不太一样

kernel也就是遍历ready, sleepq等等, 去找到下一个被处理的task

当然, curio中的kernel调度方法不是linux中的cfs, 更接近于O(1), curio中调度的模式一般是FIFO


task
======

curio中调度的单位(是不是很熟悉, linux中调度单位也叫task), 每一个协程都会包装成task(准确的说是Task类)

也就是把coro保存到Task.coro, 然后执行task就是执行Task.coro

之所以包装是为了保存task的额外信息, 执行cancel等操作

并且当前执行的task是用current来表示, 对的, linux中当前执行的task也叫current!!!


系统调用和trap(中断)
======================

curio中, 也有系统调用, 中断的概念, 当然不是linux那种完备的中断机制

比如, curio中, 一个task如果执行sleep, 那么概念上会进入一个系统调用, 从而就是引发一个中断, 也就是通过yield

把sleep中断的执行函数名(_sleep)返回给kernel, 那么kernel拿到trap执行函数名之后, 找到中断的

执行函数, 然后执行该trap handler

其他"系统调用", 比如说_read_wait, _write_wait等等也是一样, coro把系统调用函数通过yield返回给

kernel, kernel找到中断的handler, 然后执行handler

几个队列
=============

* task:   所有的task, 包括的所有的task

* ready:  表示就绪, 需要执行的task

* active: 表示被激活的task

* sleepq: 休眠队列, 包括sleep, 需要timeout, 已经等待IO的task



