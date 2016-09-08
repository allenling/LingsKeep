Ubuntu下pydev调试多进程问题
=============================

环境: Ubuntu 14.04 server, pydev4.4.0, python2.7

在调试gunicorn里面, gracefully kill就是设置一个标志为,如alive=True, 收到TERM, INT, ABORT, QUIT等信号的时候, 设置alive=False,然后在While True循环里面检查alive是False, 则break/return, 结束while, 之后回到主进程里面, 直接sys.exit,     raise一个SystemExit异常, 在主进程master里面捕获, 然后自己sys.exit, 这样的话, 一旦kill -s SIGTERM 一个子进程, 整个master-worker就会全部关闭. 这样的话, 若其他worker还在处理任务, 则会被强制终止.
    
好吧, 上面说子进程死了之后, 主进程也会死, 这是不对的, 这是因为我在使用pydev的debug启动gunicorn的时候主进程才会死掉, 而run不会.

而pydev的debug/run模式启动区别在于, debug运行的时候, 有一堆前缀

/path/to/your/env/bin/python2.7 -u /home/allen/Downloads/eclipse/dropins/PyDev 4.4.0/plugins/org.python.pydev_4.4.0.201510052309/pysrc/pydevd.py --multiprocess --print-in-debugger-startup --vm_type python --client 127.0.0.1 --port 55262 --file /path/to/your/env/bin/gunicorn

而run就是跟正常的执行文件一样了.

