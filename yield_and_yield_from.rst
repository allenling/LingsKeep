yield和yield from的区别
===========================

yield from在pep380中，称为代理子生成器. 可以看成获取子生成器返回的数据和将上层数据发送到子生成器的通道. http://stackoverflow.com/questions/9708902/in-practice-what-are-the-main-uses-for-the-new-yield-from-syntax-in-python-3


自己实现一个简单的yield from

.. code-block:: python


    def test():
        data = None
        while True:
            data = yield 'test got data : %s' % data
    
    
    def wrap(t):
        value = None
        t.send(value)
        value = yield
        while True:
            value = yield t.send(value)
    
    w = wrap(test())
    
    w.send(None)
    
    for i in range(4):
        w.send(i)
 
    def test_yield_from(t):
        yield from t

    tt=test_yield_from(test())
    
    tt.send(None)

    for i in range(4):
        print(tt.send(i))


        
    
上面的例子中，我们定义的wrap和test_yield_from(yield from)的行为是一样的


