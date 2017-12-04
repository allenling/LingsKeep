
debounce，throttle, requestAnimationFrame
===========================================

http://hackll.com/2015/11/19/debounce-and-throttle/

https://jinlong.github.io/2016/04/24/Debouncing-and-Throttling-Explained-Through-Examples/


关于像素和距离
================

http://weizhifeng.net/viewports.html


循环发送ajax
===============


如要一直发送ajax直到返回满足某些条件的时候，用回调，不要用定时器.


比如一直发送ajax直到status_code = 200, 用定时器的话就很不对:


.. code-block:: 

    let timer =setInterval(function () {
        sendAjax(url, function(resp) {
                          if (resp.status_code == 200) {
                              clearInterval(timer);
                          }
                      }
                );
    }, 1000);
    

这样的话, 如果ajax的时间小于1s, 那么看起来这样是对的，因为一旦ajax返回，定时器会继续发送请求，看起来是定时器等待ajax返回，执行判断条件，然后不满足再发送ajax.

其实不是的，不管ajax什么时候返回，定时器总是固定的每隔1s发送一个请求，因为ajax是异步的，不会阻塞定时器的, 如果ajax花了2s, 那么就会看到同时有两个请求在pending,
因为前一个还没有返回就继续发送ajax了.


应该是使用callback(是的，我也很不喜欢callback)

.. code-block::

    let sender = () => {
        sendAjax(url, function(resp) {
            if (resp.status_code != 200) { 
                sender();
            }
        });
    }
    sender();


**callback和递归区别?**


