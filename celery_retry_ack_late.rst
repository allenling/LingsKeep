Celery Retry and Late acknowledgements acknowledgements
=======================================================



Retry
-----

retry是Worker代码中自己捕获指定错误并且重试

`But this is not possible without enabling late acknowledgements acknowledgements; A task that has been started, will be retried if the worker crashes mid execution so the task must be idempotent`\ :sup:`[1]`

使用例子\ :sup:`[2]`

.. code-block:: python

    >>> from imaginary_twitter_lib import Twitter
    >>> from proj.celery import app

    >>> @app.task()
    ... def tweet(auth, message):
    ...     twitter = Twitter(oauth=auth)
    ...     try:
    ...         twitter.post_status_update(message)
    ...     except twitter.FailWhale as exc:
    ...         # Retry in 5 minutes.
    ...         raise tweet.retry(countdown=60 * 5, exc=exc)

Late acknowledgements acknowledgements
---------------------------------------

Acknowledgements是AMQP中的行为，当consumer接收message之后，会对message进行acknowledgement, 可选择一旦consumer收到message，直接ack，或者让任务执行完之后进行ack，后者成为Late acknowledgements。

若选择Late acknowledgements，则当task(message)在执行的过程中worker崩溃，则这个message会回到queue中。

Celery中设置CELERY_ACKS_LATE = True(default True)，开启Late acknowledgements。

Late acknowledgements和Retry功能上都是为了能够在出现异常的时候重试task，不同的是Retry是应用于代码级别的异常，需要自己捕获，而Late acknowledgements则是针对worker崩溃的情况，将message(task)重新回到queue中再次等待执行。

两者可以一起使用，Celery中建议
`So use retry for Python errors, and if your task is idempotent combine that with acks_late if that level of reliability is required.`\ :sup:`[1]`


.. [1] http://docs.celeryproject.org/en/latest/faq.html#faq-acks-late-vs-retry 

.. [2] 来自源码celery.app.task

