GC
============

参考1: http://www.wklken.me/posts/2015/09/29/python-source-gc.html

参考2: https://docs.python.org/3/faq/design.html#how-does-python-manage-memory

*The standard implementation of Python, CPython, uses reference counting to detect inaccessible objects, and another mechanism to collect reference cycles, periodically
executing a cycle detection algorithm which looks for inaccessible cycles and deletes the objects involved*

CPython是引用计数为主, 当引用计数为0的时候, 触发回收(这里的回收统称, 不区分回收到os还是回收到内存池)， 标记清除为辅去清理循环引用, 这个过程也称为gc.

如果确定程序没有循环引用的情况或者不在乎, 那么可以关掉gc, gc.disbale, 即使禁用了gc, 引用计数也是可以正常工作的, 因为引用计数是数字减少到0的时候触发, 而不是周期性的运行.


循环引用检测
=====================

参考自: http://www.arctrix.com/nas/python/gc/

这个文档很老, 但是鉴于python的gc很久很久没改变过了, 那么思路也是一样的啦.

并且gc相关的信息都可以在cpython/Modules/gcmodule.c中找到, 注意看注释, 注释里面和注释包含的文档链接基本上就讲得"挺"清楚的(反正我看起来还是有点难懂)~~~


1. For each container object, set gc_refs equal to the object's reference count.

2. For each container object, find which container objects it references and decrement the referenced container's gc_refs field.

3. All container objects that now have a gc_refs field greater than one are referenced from outside the set of container objects. We cannot free these objects so we move them to a different set.

4. Any objects referenced from the objects moved also cannot be freed. We move them and all the objects reachable from them too.

5. Objects left in our original set are referenced only by objects within that set (ie. they are inaccessible from Python and are garbage). We can now go about freeing these objects.


几个概念
===========

gc对象
---------

gc只会对可能产生循环引用的对象进行收集和检测, 称为gc对象.

gc对象一般是一些容器对象, 比如list, dict等等, 当然包括对象, 但是最终觉得是否需要gc的还是由type中的flag来决定的


gc链表
---------

gc对象在开始创建的时候会被加入到gc链表(一个双链表)中, 这样每次gc开始的时候就从链表中拿到需要的对象, 然后进行标记清除.

这个链表有3个, 每一个代表"一代"(generation)对象, 新建立的对象自然放在比较新的代中, 最新的代称为0代, 最老的代称为2代.

数字越大, 表示gc的频率越低.


"外部引用"
------------

一个变量名a指向一个对象b, 那么称为对象b被a引用, 此时b的引用计数为1.

在gc的时候, 如果一个对象的引用不是在gc链表上的, 那么这个引用称为"外部引用", 源码注释中称为is referenced directly from outside the generation being collected.

如果改对象被gc链表上的其他对象引用, 那么, 可以(我觉得)称为"内部引用", 内部引用就是引发循环引用的原因了.

外部引用也就是被在栈上的变量引用:

.. code-block:: python

    class A:
        pass
    
    a = A()

此时对象A被a引用了, 并且a不是gc链表的对象, 那么此时A就是有1个外部引用, 引用计数是1.


.. code-block:: python

    class A:
        pass
   a = A()
   b = A()
   a.data = b
   b.data = a
   del b
   del a

一个循环引用的例子, 其中我们使用Aa表示a之前引用的对象, Ab表示b之前引用的对象.

那么调用del解除引用之后, Aa没有外部引用了, 但是它有一个内部引用来自Ab, Ab同理.

Aa和Ab此时形成了循环引用.

引用计数副本
---------------

gc的时候是stop the world的, 所以直接拿实际的引用计数是会影响到之后的运行, 所以gc的时候是复制一份对象引用计数来操作.


引用如何减1?
==============

打破循环引用是要把gc链表上的对象间的 **互相引用给减少1**. 网上很多文章都这样说, 但是说得不清不楚 :)~~~~

如果直接减1, 这样互相引用的对象的引用计数变为0了, 但是, 这样可能把只有外部引用的对象的引用计数也置为0~~~~

比如:

.. code-block:: python

    class A:
        pass
   a = A()
   b = A()
   a.data = b
   b.data = a
   del a
   del b
   c = A()

Ac也是一个gc对象, 也会被放在gc链表中. 当遍历gc链表, 如果直接引用减少1, 那么Ac的引用计数也是0, 但是明显Ac不能被gc掉的.

**所以引用减少1是遍历gc链表, 找到对象所引用的对象, 然后对其引用的对象的引用减1**

.. code-block:: python

    '''
    括号中为引用计数, Aa表示a指向的A, 以此类推:

    [gc头]-[Aa(1)]-[Ab(1)]-[Ac(1)]-[gc尾]    

    1. 遍历到Aa, 然后寻找Aa引用的gc对象, 也就是Ab, 然后b的引用计数减少1
    2. 遍历到Ab, 然后寻找Ab引用的gc对象, 也就是Aa, 然后a的引用计数减少1
    3. 遍历到Ac, Ac没有引用的gc对象, 继续.


    遍历完之后:

    [gc头]-[Aa(0)]-[Ab(0)]-[Ac(1)]-[gc尾]    
    '''

这样, Aa和Ab就是不可达对象, 严格来说是暂时不可达对象.


暂时不可达
=============

为什么是暂时不可达对象? 考虑这样的情况:

.. code-block:: python

    class A:
        pass
    a = A()
    b = A()
    c = b
    del a
    del b

那么按照之前的流程, Aa的引用计数是0, Ab, Ac是同一个对象, 引用计数是1, 那么, Aa可以被gc掉?

显然不行呀, 因为Ac(Ab)对象是一个可达对象, 它还引用这Aa.

看一下源码注释怎么说的:

.. code-block:: python

   '''
    GC_TENTATIVELY_UNREACHABLE
        move_unreachable() then moves objects not reachable (whether directly or
        indirectly) from outside the generation into an "unreachable" set.
        Objects that are found to be reachable have gc_refs set to GC_REACHABLE
        again.  Objects that are found to be unreachable have gc_refs set to
        GC_TENTATIVELY_UNREACHABLE.  It's "tentatively" because the pass doing
        this can't be sure until it ends, and GC_TENTATIVELY_UNREACHABLE may
        transition back to GC_REACHABLE.
    
        Only objects with GC_TENTATIVELY_UNREACHABLE still set are candidates
        for collection.  If it's decided not to collect such an object (e.g.,
        it has a __del__ method), its gc_refs is restored to GC_REACHABLE again.
   '''


所以引用计数为0也不一定是不可达对象. 所以gc原则: **可达对象的引用的对象一定也是可达的**


所以, 引用计数减1的遍历只是把引用计数为0的对象标识为暂时不可达, 之后需要从可达对象开始, 把其引用的对象也标记为可达.

.. code-block:: python

    '''

    [gc头]-[Aa(0)]-[Ac(1)]-[gc尾]    

    1. 遍历Aa, 是暂时不可达, 那么放到不可达链表
    
    [gc头]-[Ac(1)]-[gc尾] 
    
    [不可达链表头]-[Aa(0)]-[不可达链表尾] 
    
    2. 遍历到Ac, 是可达对象, 然后遍历Ac的引用, 把Ac的引用, 也就是Aa, 放到可达链表, 也就是gc链表

    [gc头]-[Ac(1)]-[Aa(0)]-[gc尾] 
    
    [不可达链表头]--[不可达链表尾] 

    '''

暂时不可达还有可能是, 对象的final过程没有执行完毕(比如__del__)方法, 所以还需要检测final过程.

所以, 最终不可达是在遍历可达对象完毕之后, 猜得到最终不可达对象, 对不可达对象进行gc.

----

引用计数
=============

PyObject中, 会带有一个每一个对象都有一个引用计数和一个链表结构:


.. code-block:: c

    #define _PyObject_HEAD_EXTRA            \
        struct _object *_ob_next;           \
        struct _object *_ob_prev;

    typedef struct _object {
        _PyObject_HEAD_EXTRA
        Py_ssize_t ob_refcnt;
        struct _typeobject *ob_type;
    } PyObject;


1. ob_refcnt就是引用计数的个数

2. _ob_next和ob_prev是标记清除/分代用到的链表结构


增加引用计数
==================

把ob_refcnt加1

.. code-block:: c

    #define Py_INCREF(op) (                         \
        _Py_INC_REFTOTAL  _Py_REF_DEBUG_COMMA       \
        ((PyObject *)(op))->ob_refcnt++)


减少引用计数
===============


.. code-block:: c

    #define Py_DECREF(op)                                   \
        do {                                                \
            PyObject *_py_decref_tmp = (PyObject *)(op);    \
            if (_Py_DEC_REFTOTAL  _Py_REF_DEBUG_COMMA       \
            --(_py_decref_tmp)->ob_refcnt != 0)             \
                _Py_CHECK_REFCNT(_py_decref_tmp)            \
            else                                            \
                _Py_Dealloc(_py_decref_tmp);                \
        } while (0)


如果引用计数等于=0, 那么调用_Py_Dealloc去回收内存.



gc对象标识
================

当一个对象需要进行循环引用的检测的时候, 则每次创建的时候加入到0代链表中等待检测.

PyObject包含了一个双链表头:

.. code-block:: c

    #define _PyObject_HEAD_EXTRA            \
        struct _object *_ob_next;           \
        struct _object *_ob_prev;

    typedef struct _object {
        _PyObject_HEAD_EXTRA
        Py_ssize_t ob_refcnt;
        struct _typeobject *ob_type;
    } PyObject;



但是, 一个对象是否需要加入到gc, 则是通过其tp_flags决定的, 比如dict和long:


.. code-block:: c

    // dict中包含了Py_TPFLAGS_HAVE_GC

     PyTypeObject PyDict_Type = {
         Py_TPFLAGS_DEFAULT | Py_TPFLAGS_HAVE_GC |
             Py_TPFLAGS_BASETYPE | Py_TPFLAGS_DICT_SUBCLASS,         /* tp_flags */
     }
    
    // long则没有包含这个gc头

    PyTypeObject PyLong_Type = {
        Py_TPFLAGS_DEFAULT | Py_TPFLAGS_BASETYPE |
            Py_TPFLAGS_LONG_SUBCLASS,               /* tp_flags */

    }

其中Py_TPFLAGS_HAVE_GC这个标识表示是一个gc对象.

新建对象
=================

例如dict, 调用dict.tp_alloc:

.. code-block:: c

    PyObject *
    PyType_GenericAlloc(PyTypeObject *type, Py_ssize_t nitems)
    {
        PyObject *obj;
        const size_t size = _PyObject_VAR_SIZE(type, nitems+1);
        /* note that we need to add one, for the sentinel */
    
        // 这里判断有没有gc flag
        if (PyType_IS_GC(type))
            // 这里分配对象的时候加入gc头部
            obj = _PyObject_GC_Malloc(size);
        else
            obj = (PyObject *)PyObject_MALLOC(size);
    
        if (obj == NULL)
            return PyErr_NoMemory();
    
        memset(obj, '\0', size);
    
        if (type->tp_flags & Py_TPFLAGS_HEAPTYPE)
            Py_INCREF(type);
    
        if (type->tp_itemsize == 0)
            (void)PyObject_INIT(obj, type);
        else
            (void) PyObject_INIT_VAR((PyVarObject *)obj, type, nitems);
    
        // 这里加入gc链表中
        if (PyType_IS_GC(type))
            _PyObject_GC_TRACK(obj);
        return obj;
    }


1. 生成对象的是加入gc头部: _PyObject_GC_Malloc

2. 加入gc链表: _PyObject_GC_TRACK



加入gc头结构
====================

cpython/Modules/gcmoduel.c


.. code-block:: c

    static PyObject *
    _PyObject_GC_Alloc(int use_calloc, size_t basicsize)
    {
        PyObject *op;
        PyGC_Head *g;
        size_t size;

        if (basicsize > PY_SSIZE_T_MAX - sizeof(PyGC_Head))
            return PyErr_NoMemory();
        // 这里的size要加上gc头部的大小
        size = sizeof(PyGC_Head) + basicsize;
        if (use_calloc)
            g = (PyGC_Head *)PyObject_Calloc(1, size);
        else
            // 分配一下
            g = (PyGC_Head *)PyObject_Malloc(size);
        if (g == NULL)
            return PyErr_NoMemory();

        // 注意看这个gc_refs
        g->gc.gc_refs = 0;
        // 设置该对象还未被加入到链表中
        _PyGCHead_SET_REFS(g, GC_UNTRACKED);
        // 0代链表对象个数加一个
        generations[0].count++; /* number of allocated GC objects */

        // 这里如果达到阀值, 则触发collect
        if (generations[0].count > generations[0].threshold &&
            enabled &&
            generations[0].threshold &&
            !collecting &&
            !PyErr_Occurred()) {
            collecting = 1;
            collect_generations();
            collecting = 0;
        }
        // FROM_GC是指针操作去获取
        // 对象内存的起始地址了
        op = FROM_GC(g);
        return op;
    }


关键数据是那个叫gc_refs的, 这个是标识是否是可达对象了.

加入链表
============

cpython/Include/objimpl.h

.. code-block:: c

    #define _PyObject_GC_TRACK(o) do { \
        // 拿到gc头
        PyGC_Head *g = _Py_AS_GC(o); \
        if (_PyGCHead_REFS(g) != _PyGC_REFS_UNTRACKED) \
            Py_FatalError("GC object already tracked"); \
        // 这里设置gc_ref为, 表示可达状态
        _PyGCHead_SET_REFS(g, _PyGC_REFS_REACHABLE); \
        // 下面就是链表操作了
        g->gc.gc_next = _PyGC_generation0; \
        g->gc.gc_prev = _PyGC_generation0->gc.gc_prev; \
        g->gc.gc_prev->gc.gc_next = g; \
        _PyGC_generation0->gc.gc_prev = g; \
        } while (0);


collect
============

该操作就是对链表进行一个检测删除的过程

cpython/Modules/gcmodule.c


.. code-block:: c

    static Py_ssize_t
    collect(int generation, Py_ssize_t *n_collected, Py_ssize_t *n_uncollectable,
            int nofail)
    {
    
        /* merge younger generations with one we are currently collecting */
        // 这里, 比如我们当前回收第一代, 那么把第0代的对象也一起和第一代对象
        // 连起来, 一起回收
        for (i = 0; i < generation; i++) {
            gc_list_merge(GEN_HEAD(i), GEN_HEAD(generation));
        }

        /* handy references */
        // 下面就是拿到上面的总的一个链表了
        young = GEN_HEAD(generation);
        if (generation < NUM_GENERATIONS-1)
            old = GEN_HEAD(generation+1);
        else
            old = young;
    
        // 这里是复制引用计数
        update_refs(young);
        // 引用计数减少1
        subtract_refs(young);

        /* Leave everything reachable from outside young in young, and move
         * everything else (in young) to unreachable.
         * NOTE:  This used to move the reachable objects into a reachable
         * set instead.  But most things usually turn out to be reachable,
         * so it's more efficient to move the unreachable things.
         */
        gc_list_init(&unreachable);

        // 标记可达不可达
        move_unreachable(young, &unreachable);
    
    
    }


update_refs
=============


.. code-block:: c

    static void
    update_refs(PyGC_Head *containers)
    {
        PyGC_Head *gc = containers->gc.gc_next;
        for (; gc != containers; gc = gc->gc.gc_next) {
            assert(_PyGCHead_REFS(gc) == GC_REACHABLE);
            _PyGCHead_SET_REFS(gc, Py_REFCNT(FROM_GC(gc)));
            assert(_PyGCHead_REFS(gc) != 0);
        }
    }

    // Py_REFCNT
    #define Py_REFCNT(ob)  (((PyObject*)(ob))->ob_refcnt)

把gc_ref设置为实际的应用计数的个数(ob_refcnt)

这一步就是复制引用计数, 从而不影响对象实际的引用计数了


subtract_refs
================

遍历每一个对象, 然后调用visit_reachable

.. code-block:: c

    static void
    subtract_refs(PyGC_Head *containers)
    {
        traverseproc traverse;
        PyGC_Head *gc = containers->gc.gc_next;
        for (; gc != containers; gc=gc->gc.gc_next) {
            // 获取对象的tp_traverse函数
            traverse = Py_TYPE(FROM_GC(gc))->tp_traverse;
            // 遍历的时候调用visit_decref
            (void) traverse(FROM_GC(gc),
                           (visitproc)visit_decref,
                           NULL);
        }
    }

这里遍历是使用tp_traverse, 是每个对象自己定义的遍历函数.

而visit_decref是将引用计数减少1.


**所以这一步就是遍历对象所引用的对象, 然后将其的引用计数减少1.**


visit_decref
===============

.. code-block:: c

    /* A traversal callback for subtract_refs. */
    static int
    visit_decref(PyObject *op, void *data)
    {
        assert(op != NULL);
        // 如果是gc对象
        if (PyObject_IS_GC(op)) {
            // 拿到gc头
            PyGC_Head *gc = AS_GC(op);
            assert(_PyGCHead_REFS(gc) != 0); /* else refcount was too small */
            if (_PyGCHead_REFS(gc) > 0)
                // 减少引用计数
                _PyGCHead_DECREF(gc);
        }
        return 0;
    }

move_unreachable
=====================

标记不可达对象


.. code-block:: c

    static void
    move_unreachable(PyGC_Head *young, PyGC_Head *unreachable)
    {
        PyGC_Head *gc = young->gc.gc_next;
    
        while (gc != young) {
            PyGC_Head *next;
    
            // 引用计数大于0, 是一个可达对象
            if (_PyGCHead_REFS(gc)) {

                PyObject *op = FROM_GC(gc);
                traverseproc traverse = Py_TYPE(op)->tp_traverse;
                assert(_PyGCHead_REFS(gc) > 0);
                _PyGCHead_SET_REFS(gc, GC_REACHABLE);
                (void) traverse(op,
                                (visitproc)visit_reachable,
                                (void *)young);
                next = gc->gc.gc_next;
                if (PyTuple_CheckExact(op)) {
                    _PyTuple_MaybeUntrack(op);
                }
            }
            else {
                next = gc->gc.gc_next;
                // 移动到不可达对象列表
                gc_list_move(gc, unreachable);
                // 设置暂时不可达状态
                _PyGCHead_SET_REFS(gc, GC_TENTATIVELY_UNREACHABLE);
            }
            gc = next;
        }
    }

注意的是:

1. 遇到不可达对象的时候, 先把其标记为暂时不可达, 然后移动到不可达链表

2. 遍历到对象是可达对象的时候, 会调用visit_reachable去把其引用的对象给放入可达链表中

3. 如果可达对象中包含了暂时不可达, 那么会把暂时不可达对象给放回可达对象链表


visit_reachable
====================


对可达对象中的引用对象进行处理


.. code-block:: c

    static int
    visit_reachable(PyObject *op, PyGC_Head *reachable)
    {
        if (PyObject_IS_GC(op)) {
            PyGC_Head *gc = AS_GC(op);
            const Py_ssize_t gc_refs = _PyGCHead_REFS(gc);
    
            if (gc_refs == 0) {
                /* This is in move_unreachable's 'young' list, but
                 * the traversal hasn't yet gotten to it.  All
                 * we need to do is tell move_unreachable that it's
                 * reachable.
                 */
                 
                _PyGCHead_SET_REFS(gc, 1);
            }
            else if (gc_refs == GC_TENTATIVELY_UNREACHABLE) {
                /* This had gc_refs = 0 when move_unreachable got
                 * to it, but turns out it's reachable after all.
                 * Move it back to move_unreachable's 'young' list,
                 * and move_unreachable will eventually get to it
                 * again.
                 */
                gc_list_move(gc, reachable);
                _PyGCHead_SET_REFS(gc, 1);
            }
            /* Else there's nothing to do.
             * If gc_refs > 0, it must be in move_unreachable's 'young'
             * list, and move_unreachable will eventually get to it.
             * If gc_refs == GC_REACHABLE, it's either in some other
             * generation so we don't care about it, or move_unreachable
             * already dealt with it.
             * If gc_refs == GC_UNTRACKED, it must be ignored.
             */

             else {
                assert(gc_refs > 0
                       || gc_refs == GC_REACHABLE
                       || gc_refs == GC_UNTRACKED);
             }
        }
        return 0;
    }

三个判断的作用

gc_refs == 0
----------------

这个对象是不可达, **但是还没有遍历到这个对象**. 这里这是对象为可达, 那么遍历到这个对象的是

就会把它放入到可达对象链表中, 然后继续这个流程.


gc_refs == GC_TENTATIVELY_UNREACHABLE
-----------------------------------------

这里, 可达对象中包含了(暂时)不可达对象, **说明之前已经被移动到不可达链表了**.

那么该对象也要移到可达对象链表中.


其他
------

1. 这里, gc_refs大于0, 那么说明这个可达对象还没被遍历到, 不用管他.

2. 如果gc_refs是GC_REACHABLE(-3), 那么说明这个对象是其他代(generation)的, 也不用管他.


\_\_del\_\_
=============

在collect的过程中, 处理完可达不可达之后, 后面还会处理带有\_\_del\_\_析构函数的对象

**注意, python3中, 定义了\_\_del\_\_函数的对象中, 在C代码级别不是tp_del, 而是tp_finalize!!!!而python2.7中__del__才是被映射到tp_del**

.. code-block:: c

    // 2.7中的tp_del, cpythonObjects/typeobject.c
    TPSLOT("__del__", tp_del, slot_tp_del, NULL, ""),
    
    // 3.6中的tp_finalize
    TPSLOT("__del__", tp_finalize, slot_tp_finalize, (wrapperfunc)wrap_del, ""),

其中, 如果定义了\_\_del\_\_方法, 那么tp_finalize会指向slot_tp_finalize函数, slot_tp_finalize这个函数是调用对象的\_\_del\_\_方法的

那么3.6中的tp_del是干嘛的呢? 根据 `python issue#4934 <https://bugs.python.org/issue4934>`_ 可知

tp_del只是内部使用, 并且已被废弃, tp_finalize替代了tp_del

*The reason tp_del has remained undocumented is that it's now obsolete.  You should use tp_finalize instead
tp_finalize is roughly the C equivalent of __del__ (tp_del was something else, despite the name).*

.. code-block:: c

    // 这个函数就是寻找__del__方法, 然后调用了
    static void
    slot_tp_finalize(PyObject *self)
    {
        _Py_IDENTIFIER(__del__);
        PyObject *del, *res;
        PyObject *error_type, *error_value, *error_traceback;
    
        /* Save the current exception, if any. */
        PyErr_Fetch(&error_type, &error_value, &error_traceback);
    
        /* Execute __del__ method, if any. */
        del = lookup_maybe(self, &PyId___del__);
        if (del != NULL) {
            // 调用__del__方法!!!!
            res = PyEval_CallObject(del, NULL);
            if (res == NULL)
                PyErr_WriteUnraisable(del);
            else
                Py_DECREF(res);
            Py_DECREF(del);
        }
    
        /* Restore the saved exception. */
        PyErr_Restore(error_type, error_value, error_traceback);
    }


来看看collect中接下来如何处理带有\_\_del\_\_方法的对象的
    
.. code-block:: c

    static Py_ssize_t
    collect(int generation, Py_ssize_t *n_collected, Py_ssize_t *n_uncollectable,
            int nofail)
    {
    
        update_refs(young);
        subtract_refs(young);
    
        /* Leave everything reachable from outside young in young, and move
         * everything else (in young) to unreachable.
         * NOTE:  This used to move the reachable objects into a reachable
         * set instead.  But most things usually turn out to be reachable,
         * so it's more efficient to move the unreachable things.
         */
        gc_list_init(&unreachable);
        move_unreachable(young, &unreachable);
    
    
        // 上面是处理可达不可达的情况, 下面是针对__del__
    
        /* All objects in unreachable are trash, but objects reachable from
        * legacy finalizers (e.g. tp_del) can't safely be deleted.
        */
        gc_list_init(&finalizers);
    
        // 针对tp_del的, 可以不用看
        move_legacy_finalizers(&unreachable, &finalizers);
    
        /* finalizers contains the unreachable objects with a legacy finalizer;
         * unreachable objects reachable *from* those are also uncollectable,
         * and we move those into the finalizers list too.
         */
        move_legacy_finalizer_reachable(&finalizers);

        // 这里会调用__del__方法
        /* Call tp_finalize on objects which have one. */
        finalize_garbage(&unreachable);

        if (check_garbage(&unreachable)) {
        }
        else {
            /* Call tp_clear on objects in the unreachable set.  This will cause
             * the reference cycles to be broken.  It may also cause some objects
             * in finalizers to be freed.
             */
            // 这里!!!!!会调用tp_clear函数去回收对象
            delete_garbage(&unreachable, old);
        }
    
        /* Append instances in the uncollectable set to a Python
         * reachable list of garbage.  The programmer has to deal with
         * this if they insist on creating this type of structure.
         */
         // 这个其实也不用看
        (void)handle_legacy_finalizers(&finalizers, old);
    
    }


所以, 主要是5个函数:

1. move_legacy_finalizers, move_legacy_finalizer_reachable, 以及
   
   handle_legacy_finalizers这是哪个函数主要是针对遗留的tp_del进行的处理

   前两个函数是把带有tp_del方法的对象加入到finalizers链表中, 同时把对象设置为可达对象, 也就是不回收

   然后最后一个函数是把对象放入到garbage这个list中, 也就是gc.garbage

2. finalize_garbage, 遍历uncollectable链表, 调用其中对象的tp_finalize函数, 也就是\_\_del\_\_方法

3. delete_garbage, 遍历unreachable链表, 调用其中对象的tp_clear函数去回收对象

4. 如果定义了DEBUG_SAVEALL, 那么只会把要回收的对象加入到gc.garbage而不是回收!!!!!!!!!!!!!!!!!
   

legacy(带有tp_del)处理
=========================

python2.7的情况的处理

move_legacy_finalizers是把unreachable链表中, 带有tp_del函数的对象移动到finalizers链表中, 并且移动到

finalizers链表的对象被设置为可达的(GC_REACHABLE)!

.. code-block:: c

    // 判断是否带有__del__方法
    static int
    has_legacy_finalizer(PyObject *op)
    {
        return op->ob_type->tp_del != NULL;
    }


    // 注释说明的功能
    /* Move the objects in unreachable with tp_del slots into `finalizers`.
     * Objects moved into `finalizers` have gc_refs set to GC_REACHABLE; the
     * objects remaining in unreachable are left at GC_TENTATIVELY_UNREACHABLE.
     */
    static void
    move_legacy_finalizers(PyGC_Head *unreachable, PyGC_Head *finalizers)
    {
        PyGC_Head *gc;
        PyGC_Head *next;
    
        /* March over unreachable.  Move objects with finalizers into
         * `finalizers`.
         */
        // 下面的for就是判断然后加入到finalizers, 同时设置gc状态为可达
        for (gc = unreachable->gc.gc_next; gc != unreachable; gc = next) {
            PyObject *op = FROM_GC(gc);
    
            assert(IS_TENTATIVELY_UNREACHABLE(op));
            next = gc->gc.gc_next;
    
            if (has_legacy_finalizer(op)) {
                gc_list_move(gc, finalizers);
                _PyGCHead_SET_REFS(gc, GC_REACHABLE);
            }
        }
    }


注意, 上一个函数是说把带有tp_del的对象移动到finalizers链表, 并且设置为可达状态

**所以, 下一步必须把finalizers中对象包含的对象也需要被设置为可达对象, 也就是说move_legacy_finalizers是移动

第一级对象, 而move_legacy_finalizer_reachable则是把第一级对象所引用的所有对象都设置为可达状态!!!!!**

.. code-block:: c

    // 这个就是遍历finalizers链表时候的处理函数
    static int
    visit_move(PyObject *op, PyGC_Head *tolist)
    {
        if (PyObject_IS_GC(op)) {
            // 也就是说, 在finalizers中的对象都是可达的
            // 那么必然, 其引用的对象也都是可达的
            // 如果不可达, 那么设置为可达对象, 然后从finalizers链表中移除
            if (IS_TENTATIVELY_UNREACHABLE(op)) {
                PyGC_Head *gc = AS_GC(op);
                gc_list_move(gc, tolist);
                _PyGCHead_SET_REFS(gc, GC_REACHABLE);
            }
        }
        return 0;
    }
    
    /* Move objects that are reachable from finalizers, from the unreachable set
     * into finalizers set.
     */
    static void
    move_legacy_finalizer_reachable(PyGC_Head *finalizers)
    {
        traverseproc traverse;
        PyGC_Head *gc = finalizers->gc.gc_next;
        for (; gc != finalizers; gc = gc->gc.gc_next) {
            /* Note that the finalizers list may grow during this. */
            traverse = Py_TYPE(FROM_GC(gc))->tp_traverse;
            (void) traverse(FROM_GC(gc),
                            (visitproc)visit_move,
                            (void *)finalizers);
        }
    }

所以, 经过move_legacy_finalizers和move_legacy_finalizer_reachable之后, 我们在finalizers

链表上就得到了所有带有tp_del函数的对象, 接下来就是handle_legacy_finalizers

可以看到, 把finalizers中的对象加入到gc.garbage这个list中

.. code-block:: c

    static int
    handle_legacy_finalizers(PyGC_Head *finalizers, PyGC_Head *old)
    {
        PyGC_Head *gc = finalizers->gc.gc_next;
    
        if (garbage == NULL) {
            garbage = PyList_New(0);
            if (garbage == NULL)
                Py_FatalError("gc couldn't create gc.garbage list");
        }
        for (; gc != finalizers; gc = gc->gc.gc_next) {
            PyObject *op = FROM_GC(gc);
    
            if ((debug & DEBUG_SAVEALL) || has_legacy_finalizer(op)) {
                // 加入链表
                if (PyList_Append(garbage, op) < 0)
                    return -1;
            }
        }
    
        gc_list_merge(finalizers, old);
        return 0;
    }


finalize_garbage
=====================

调用对象的tp_finalize函数, 也就是__del__方法

.. code-block:: c

    static void
    finalize_garbage(PyGC_Head *collectable)
    {
        destructor finalize;
        PyGC_Head seen;
    
        /* While we're going through the loop, `finalize(op)` may cause op, or
         * other objects, to be reclaimed via refcounts falling to zero.  So
         * there's little we can rely on about the structure of the input
         * `collectable` list across iterations.  For safety, we always take the
         * first object in that list and move it to a temporary `seen` list.
         * If objects vanish from the `collectable` and `seen` lists we don't
         * care.
         */
        gc_list_init(&seen);
    
        while (!gc_list_is_empty(collectable)) {
            PyGC_Head *gc = collectable->gc.gc_next;
            PyObject *op = FROM_GC(gc);
            gc_list_move(gc, &seen);
            if (!_PyGCHead_FINALIZED(gc) &&
                    PyType_HasFeature(Py_TYPE(op), Py_TPFLAGS_HAVE_FINALIZE) &&
                    (finalize = Py_TYPE(op)->tp_finalize) != NULL) {
                _PyGCHead_SET_FINALIZED(gc, 1);
                Py_INCREF(op);
                finalize(op);
                Py_DECREF(op);
            }
        }
        gc_list_merge(&seen, collectable);
    }

其中, *finalize = Py_TYPE(op)->tp_finalize*, 也就是__del__方法, 然后调用finalize()

check_garbage/delete_garbage
===============================

在collect中, delete_garbage的条件是调用check_garbage返回0

也就是check_garbage是在真正回收对象之前再次校验unreachable链表中, 是否都是可回收对象

因为如果存在tp_del的对象的话, unreachable中有可能存在可达对象, 这个时候就不会去回收对象

注意, 当存在tp_del对象的时候, unreachable链表上所有的对象都不会被回收, 也就是不会走到delete_garbage

.. code-block:: c

    // 如果check_garbage返回的不是0
    if (check_garbage(&unreachable)) {
        // revive_garbage则是把所有的对象都设置为可达
        revive_garbage(&unreachable);
        // 然后把所有对象都加入到gc链表中
        gc_list_merge(&unreachable, old);
    }
    else {
        // 也就是说, check_garbage返回0, 也就是说unreachable都是reachable的
        /* Call tp_clear on objects in the unreachable set.  This will cause
         * the reference cycles to be broken.  It may also cause some objects
         * in finalizers to be freed.
         */
        delete_garbage(&unreachable, old);
    }

check_garbage
----------------

一个个校验的过程

.. code-block:: c

    // 看注释, 注释说校验是否存储被finalizers, 也就是存在tp_del对象, 造成
    // unreachable链表中存在可达对象的情况
    /* Walk the collectable list and check that they are really unreachable
       from the outside (some objects could have been resurrected by a
       finalizer). */
    static int
    check_garbage(PyGC_Head *collectable)
    {
        PyGC_Head *gc;
        for (gc = collectable->gc.gc_next; gc != collectable;
             gc = gc->gc.gc_next) {
            _PyGCHead_SET_REFS(gc, Py_REFCNT(FROM_GC(gc)));
            assert(_PyGCHead_REFS(gc) != 0);
        }
        subtract_refs(collectable);
        for (gc = collectable->gc.gc_next; gc != collectable;
             gc = gc->gc.gc_next) {
            assert(_PyGCHead_REFS(gc) >= 0);
            if (_PyGCHead_REFS(gc) != 0)
                // 如果存在可达对象, 返回-1, 也就是不会走到
                // delete_garbage
                return -1;
        }
        return 0;
    }


delete_garbage
-----------------

回收的地方, 调用tp_clear函数, 但是, 如果定义了debug并且DEBUG_SAVEALL, 就不会去回收对象

只是把对象加入到gc.garbage这个list中, 来自 `文档 <https://docs.python.org/3/library/gc.html>`_ :

*If DEBUG_SAVEALL is set, then all unreachable objects will be added to this list rather than freed.*

.. code-block:: c

    static void
    delete_garbage(PyGC_Head *collectable, PyGC_Head *old)
    {
        inquiry clear;
    
        while (!gc_list_is_empty(collectable)) {
            PyGC_Head *gc = collectable->gc.gc_next;
            PyObject *op = FROM_GC(gc);
    
            if (debug & DEBUG_SAVEALL) {
                PyList_Append(garbage, op);
            }
            else {
                // 这里!!!!!!!!
                if ((clear = Py_TYPE(op)->tp_clear) != NULL) {
                    Py_INCREF(op);
                    clear(op);
                    Py_DECREF(op);
                }
            }
            if (collectable->gc.gc_next == gc) {
                /* object is still alive, move it, it may die later */
                gc_list_move(gc, old);
                _PyGCHead_SET_REFS(gc, GC_REACHABLE);
            }
        }
    }
    

gc线程安全的例子
=====================

参考: https://zhuanlan.zhihu.com/p/37665784

参考中貌似是python2.7的情况, 但是其实在python3中也会有问题

参考中的例子:

.. code-block:: python

    import gc
    counter = 0
    
    class MyObject(object):
        def __init__(self):
            global counter
            l = counter
            # Trigger GC
            a = list()
            for _ in range(10):
                a = [a]
                a.append(a)
            del a
            counter = l + 1
        def __del__(self):
            global counter
            l = counter
            counter = l - 1
    
    for _ in range(10000):
        a = list()
        b = list()
        a.append(b)
        b.append(a)
        a.append(MyObject())
        del a
        del b
        #gc.collect()
    
    gc.collect()
    print(counter)

如果将注释掉的gc.collect()恢复，就会正确输出0；否则会输出一个远远比0大的值。注意这是一个单线程的程序

评论中有指出: 

.. code-block:: python

    '''
    和原子操作有联系的，在程序员看来，init和del中的代码应该是顺序执行，即便其中会插入runtime的过程，但至少这两者是独立的
    
    然而py的gc机制可能使得其不独立，典型例子就是，在执行init的循环的时候（这时候counter的值被l缓存了）
    
    触发了gc过程，而gc过程中会回调其他MyObject对象的del，操作了counter，结果counter的加减操作被穿插了，简单点就是：
    
    il = counter #init的代码
    
    #被gc机制强制跳到了del
    
    dl = counter #del的代码
    
    count = dl - 1 #del中counter减一了
    
    #del执行完，假设这时候gc结束，跳回被中断的init
    
    count = il + 1 #你以为init中加一了，其实这里相对上面这句，是加2了
    
    而且，中间dl的两句在gc过程中会被重复N次，偏差更大，最终的结果就是counter远大于0
    
    '''

理解上面评论的关键点在于, 不管是init还是del, 都是使用l来保存counter, 也就是说, 即使global的counter被修改了

因为init和del已经用l来保存counter旧值, 那么不管是+1还是-1, 都是老值去执行. 比如l = counter = 10, 然后counter被

-1, 也就是counter = 9, 但是因为init和del中是counter = l + 1, 也就是counter = 10 + 1 = 11, 所以说counter的值就变化了

如果我们不用l去保存counter的值, 而是counter += 1, counter -= 1, 那么结果就是正确的

然后参考中也指出python的pep556的提案, 说会提供线程安全的gc

    
Finalizers
=============

再说吧


循环引用的问题和__del__
==========================

https://www.holger-peters.de/an-interesting-fact-about-the-python-garbage-collector.html

如果定义了__del__方法, 那么循环引用则不会被gc掉

python2.7中

.. code-block:: python

    In [1]: import gc
    
    In [2]: class T:
       ...:     def __del__(self):
       ...:         print('in T __del__')
       ...:         
    
    In [3]: a=T()
    
    In [4]: b=T()
    
    In [5]: a.other = b
    
    In [6]: b.other = a
    
    In [7]: a=None
    
    In [8]: b=None
    
    In [9]: gc.collect()
    Out[9]: 67
    
    In [10]: gc.garbage
    Out[10]: 
    [<__main__.T instance at 0x7f80798a3518>,
     <__main__.T instance at 0x7f80798ff8c0>]

gc不能回收a和b之前指向的对象, 因为__del__方法被定义了, 如果没有定义__del__方法呢

.. code-block:: python

    In [1]: import gc
    
    In [2]: class T:
       ...:     pass
       ...: 
    
    In [3]: a=T()
    
    In [4]: b=None
    
    In [5]: b=T()
    
    In [6]: a.other = b
    
    In [7]: b.other = a
    
    In [8]: a=None
    
    In [9]: b=None
    
    In [10]: gc.collect()
    Out[10]: 53
    
    In [11]: gc.garbage
    Out[11]: []

gc.garbage为空表示即使出现循环引用, 对象也会被回收掉了

大概的原因是当出现循环引用的时候, 比如上面的a, b, python并不知道应该先调用谁的__del__, 如果调用顺序错了怎么办, 所以决定上面都不做(do nothing)~~


python3.4之后这个问题就解决了, 下面的代码是python3.6

.. code-block:: python

    In [2]: import gc
    
    In [3]: class T:
       ...:     def __del__(self):
       ...:         print('in T __del__%s' % self.name)
       ...:         
    
    In [4]: a=T()
    
    In [5]: b=T()
    
    In [6]: a.name='a'
    
    In [7]: b.name='b'
    
    In [8]: a.other = b
    
    In [9]: b.other = a
    
    In [10]: a=b=None
    
    In [11]: gc.collect()
    in T __del__a
    in T __del__b
    Out[11]: 60

https://www.python.org/dev/peps/pep-0442/ 中有具体细节


python weakref
===============

https://docs.python.org/3/library/weakref.html

https://segmentfault.com/a/1190000005729873

In the following, the term referent means the object which is referred to by a weak reference.

下面提到的 **引用对象** 是指的是一个 **被弱引用对象引用** 的对象

A weak reference to an object is not enough to keep the object alive: when the only remaining references to a referent are weak references, garbage collection is free to destroy the referent and reuse its memory for something else.

However, until the object is actually destroyed the weak reference may return the object even if there are no strong references to it.

一旦一个对象只有弱引用指向它, 那么它随时可以被回收

weakref是创建一个到目标对象的弱引用, 创建的弱引用不会增加对象的引用计数:

.. code-block:: python

    In [5]: import weakref
    
    In [6]: class M:
       ...:     def __init__(self, name):
       ...:         self.name = name
       ...:         
    
    In [7]: x=M('x')
    
    In [8]: import sys
    
    In [9]: sys.getrefcount(x)
    Out[9]: 2
    
    In [10]: r=weakref.ref(x)
    
    In [11]: r
    Out[11]: <weakref at 0x7fc279f1ad08; to 'instance' at 0x7fc279e9d7e8>
    
    In [12]: sys.getrefcount(x)
    Out[12]: 2


