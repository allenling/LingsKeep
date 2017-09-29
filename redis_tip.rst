list
========


如果一个list只有一个元素, 然后rpop/lpop之后，这个list就会被删除. 

然后如果你之前在list上有设置了过期时间的话，rpop/lpop之后，这个list就会被删除

然后再push的话，此时虽然key同名，但是其实是一个新的list, 所以就会发现某个名字的list的过期时间莫名其妙就没了~~~~~


pipeline
===========


根据redis的文档, 一个连接内如果执行了pipeline, 那么当前连接就不能做其他事情了, 也就是比如多线程下

a线程pipeline了, b线程执行get, b就会阻塞住, 这么也可以理解.


然后不要滥用pipeline.

只有命令比较多的时候, pipeline才有明显的优势, 比较多的话一般得上百.


.. code-block:: python

  import redis
  import time
  
  
  r = redis.StrictRedis()
  
  keys = range(3000)
  
  
  def test_pipeline():
      start = time.time()
      with r.pipeline() as p:
          for i in keys:
              p.get(i)
          p.execute()
      print('+++with pipeline\n', time.time() - start)
  
  
  def test_no_pipeline():
      start = time.time()
      for i in keys:
          r.get(i)
      print('---no pipeline\n', time.time() - start)
      return
  
  
  def main():
      test_pipeline()
      test_no_pipeline()
      return
  
  
  if __name__ == '__main__':
      main()

简单的测试下, 命令比较少(<100)的时候，用pipeline会比较慢, 命令多的时候, pipeline一般是没有pipeline的1/2.

