yield和yield from的区别
===========================

py2.x中的yield
----------------

.. code-block:: python

  def sub_yield():
      for i in range(3):
          yield i
  
  
  def test():
      yield sub_yield()
  
  
  def main():
      t = test()
      print t
      t.send(None)
      next(t)
  
  main()

next(t)的时候, 就引发StopIteration异常, 这是很正常的, t.send(None)的时候, t返回一个生成器, 也就是sub_yield这个生成器对象

然后next是对test进行求下一个值, 而test中只有一个yield, 所以return None, 而return None表示终止迭代, 也就是默认引发StopIteration(None)异常.

所以, 你不能调用send重新启动t.


py3.5中的yield from
----------------------

.. code-block:: python

  def sub_yield():
      i = 0
      while i < 10:
          num = yield i
          if num is not None:
              i += num
          else:
              i += 1
      return num
  
  
  def test():
      q = yield from sub_yield()
      return q
  
  
  def main():
      t = test()
      print (t)
      print (t.send(None))
      print (next(t))
      print (next(t))
      print (next(t))
      print (t.send(2))
      print (next(t))
      print (t.send(3))
      try:
          print (t.send(40))
      except StopIteration as e:
          print (e.value)

在try之前的send和next都正常打印出0, 1, 2, 3, 5, 6, 9.

你可以对t进行暂停, send重启, 本质上是对sub_yield进行暂停重启. yield from应该算是一个代理. 但是调用函数可以直接对代理进行iter或者send, 本质上就是对yiel from后面的对象迭代和send.

最后一个send(40), 是把40发送到sub_yield中, sub_yield 会return 40, return 40 就是raise StopIteration(40), 而q = yield from此时就是帮你

try:
  sub_yield().send(40)
except StopIteration as e:
  q = e.value

所以q = 40

然后在test中, 又return q, 也就是再次raise StopIteration(40), 被main中的try捕获, 然后打印.

另外, 若你不需要send, throw, close这些功能, 只是简单的迭代的话, yield from就等价于

for i in sub_yield():
    yield i

就像下面的例子:

.. code-block:: python

In [17]: def x():
    ...:     yield from [1,2,3]
    ...:     
In [18]: for i in x():
    ...:     print (i)
    ...:     
1
2
3

