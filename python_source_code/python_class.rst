##################
Python中的自定义类
##################

类就是类型, 类型就是类!!!!!

自定义类(也就是class关键字), 那么python就会生成一个PyTypeObject对象, 然后实例PyObject的ob_type就是这个PyTypeObject

显然PyTypeObject的ob_type就是PyType_Type


字节码
=========

先看看字节码

.. code-block:: python

    In [11]: dis.dis('''class A:\n    def __init__(self, data):\n        self.data=data\n        return''')
      1           0 LOAD_BUILD_CLASS
                  2 LOAD_CONST               0 (<code object A at 0x7fe6b8bebd20, file "<dis>", line 1>)
                  4 LOAD_CONST               1 ('A')
                  6 MAKE_FUNCTION            0
                  8 LOAD_CONST               1 ('A')
                 10 CALL_FUNCTION            2
                 12 STORE_NAME               0 (A)
                 14 LOAD_CONST               2 (None)
                 16 RETURN_VALUE

dis里面的内容就是:

.. code-block:: python

    class A:
        def __init__(self, data):
            self.data = data
            return

定义一个名字为A的类

builtins.builtin\_\_\_build\_class\_\_ -> type_prepare

PyType_Type.tp_call -> tp_new(type_new)
                    -> tp_init(type_init)


object_new

slot_tp_init


