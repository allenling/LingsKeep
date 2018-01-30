list
=========

1. 连续数组来存储元素

2. 每当元素增减, 会resize调整数组长度, 而dict是不会的.

3. pop最后一个元素的操作很快, 调整list长度就好了, 甚至不需要把list槽位置空, 其他位置触发slice操作

4. append也很快, 只需数组槽位赋值而已

5. insert需要把目标位置之后的元素一个个往后移动, 从而腾出目标位置

----


PyListObject
===============

.. code-block:: c

    typedef struct {
        // 这个头包含了长度值
        PyObject_VAR_HEAD

        // 元素数组
        PyObject **ob_item;
    
        /* ob_item contains space for 'allocated' elements.  The number
         * currently in use is ob_size.
         * Invariants:
         *     0 <= ob_size <= allocated
         *     len(list) == ob_size
         *     ob_item == NULL implies ob_size == allocated == 0
         * list.sort() temporarily sets allocated to -1 to detect mutations.
         *
         * Items must normally not be NULL, except during construction when
         * the list is not yet visible outside the function that builds it.
         */
        // 已分配的大小t, 其总是保证: t/2 > 实际元素个数
        Py_ssize_t allocated;
    } PyListObject;



append
===========

.. code-block:: c

    // append的操作的时候i就是最后一个下标
    #define PyList_SET_ITEM(op, i, v) (((PyListObject *)(op))->ob_item[i] = (v))

    // append实际逻辑
    static int
    app1(PyListObject *self, PyObject *v)
    {
        // 获取list已经分配大大小
        Py_ssize_t n = PyList_GET_SIZE(self);
    
        assert (v != NULL);
        if (n == PY_SSIZE_T_MAX) {
            PyErr_SetString(PyExc_OverflowError,
                "cannot add more objects to list");
            return -1;
        }
    
        // 直接去resize, 反正如果需要resize会resize
        // 不需要resize, 就直接返回的
        if (list_resize(self, n+1) < 0)
            return -1;
    
        Py_INCREF(v);
        PyList_SET_ITEM(self, n, v);
        return 0;
    }
    
    // 这个是append的函数
    // 调用上面那个app1
    int
    PyList_Append(PyObject *op, PyObject *newitem)
    {
        if (PyList_Check(op) && (newitem != NULL))
            return app1((PyListObject *)op, newitem);
        PyErr_BadInternalCall();
        return -1;
    }



resize
========

.. code-block:: c		

    static int list_resize(PyListObject *self, Py_ssize_t newsize)		
    {		
        PyObject **items;		
        size_t new_allocated;		
        Py_ssize_t allocated = self->allocated;		
    		
        /* Bypass realloc() when a previous overallocation is large enough		
           to accommodate the newsize.  If the newsize falls lower than half		
           the allocated size, then proceed with the realloc() to shrink the list.		
        */		
        // allocated >> 1这个是allocated / 2, 这样计算二分之一, 可以可以		
        // 这里的判断条件中前一个是如果是append, 并且列表本身已经分配的内存足够, 则不需要额外分配内存		
        // 第二个判断条件是新的大小, 有可能是长度变小了, 如果还是大于已分配内存的一半, 也不需要缩减内存		
        // 所以, 换句话说:
        // 1. 需要扩容: newsize大于已分配的内存
        // 或者
        // 2. 需要缩减: newsize的大小小于已分配的一半		
        if (allocated >= newsize && newsize >= (allocated >> 1)) {		
            assert(self->ob_item != NULL || newsize == 0);		
            // 这里说明新长度也没有满足条件, 改变一下list的长度就好了
            Py_SIZE(self) = newsize;		
            return 0;		
        }		
    		
        /* This over-allocates proportional to the list size, making room		
         * for additional growth.  The over-allocation is mild, but is		
         * enough to give linear-time amortized behavior over a long		
         * sequence of appends() in the presence of a poorly-performing		
         * system realloc().		
         * The growth pattern is:  0, 4, 8, 16, 25, 35, 46, 58, 72, 88, ...		
         */		
        // newsize >> 3是newsize往右移３位, 也就是newsize / 8	
        // new_allocated是多分配的大小, new_allocated加上newsize才是上面注释写的步长		
        // 比如newsize = 1, 然后 1 >> 3 = 0, 1 < 9, 所以是new_allocated = 0 + 3 =3, newsize = 1		
        // 所以是allocated = new_allocated + newsize = 3 +1 = 4		
        // 如果是pop等操作的话, allocated会减少, 比如allocated = 8, newsize = 3		
        // 则new_allocated = 0 + 3 = 3, 所以最后allocated = new_allocated + newsize = 3 + 3 = 6		
        new_allocated = (newsize >> 3) + (newsize < 9 ? 3 : 6);		
    		
        /* check for integer overflow */		
        if (new_allocated > SIZE_MAX - newsize) {		
            PyErr_NoMemory();		
            return -1;		
        } else {		
            new_allocated += newsize;		
        }		
    		
        if (newsize == 0)		
            new_allocated = 0;		
        items = self->ob_item;		
        // 这里的PyMem_RESIZE才是真正的去改变内存大小		
        // 也就是移动数组元素填补空位
        if (new_allocated <= (SIZE_MAX / sizeof(PyObject *)))		
            PyMem_RESIZE(items, PyObject *, new_allocated);		
        else		
            items = NULL;		
        if (items == NULL) {		
            PyErr_NoMemory();		
            return -1;		
        }		
        self->ob_item = items;		
        // 这里self是列表对象, PySIZE(self)是self的长度, 然后这里就赋值为newsize		
        Py_SIZE(self) = newsize;		
        // 这里赋值列表对象的已分配内存为new_allocated		
        self->allocated = new_allocated;		
        return 0;		
    }

insert
==========


.. code-block:: c

    // insert的逻辑
    static int
    ins1(PyListObject *self, Py_ssize_t where, PyObject *v)
    {
        Py_ssize_t i, n = Py_SIZE(self);
        PyObject **items;
        if (v == NULL) {
            PyErr_BadInternalCall();
            return -1;
        }
        if (n == PY_SSIZE_T_MAX) {
            PyErr_SetString(PyExc_OverflowError,
                "cannot add more objects to list");
            return -1;
        }
    
        // 看看需不需要resize
        if (list_resize(self, n+1) < 0)
            return -1;
    
        // 插入是负位置, 计算一下
        if (where < 0) {
            where += n;
            if (where < 0)
                where = 0;
        }

        // 插入的位置大于长度, 只能在最后插入
        if (where > n)
            where = n;
        items = self->ob_item;

        // 一个个移动元素
        for (i = n; --i >= where; )
            items[i+1] = items[i];
        Py_INCREF(v);
        // 空位置插入
        items[where] = v;
        return 0;
    }

pop
====



.. code-block:: c

    // 这个是pop
    static PyObject *
    listpop(PyListObject *self, PyObject *args)
    {

        Py_ssize_t i = -1;
        PyObject *v;
        int status;

        // 这里是把参数赋值为i, 如果没有arg, 那么i就是默认的-1
        if (!PyArg_ParseTuple(args, "|n:pop", &i))
            return NULL;

        // 如果i是负号的下标, 那么其真正的位置就是加上list长度
        if (i < 0)
            i += Py_SIZE(self);

        // 如果是pop最后一个, 直接改变list长度就可以了~~~
        // 所以最后一个的pop是很快的
        if (i == Py_SIZE(self) - 1) {
            status = list_resize(self, Py_SIZE(self) - 1);
            if (status >= 0)
                return v; /* and v now owns the reference the list had */
            else
                return NULL;
        }
 
        // 其他位置的pop则是要当做slice来操作
        // 这里的增加和减少引用计数没看懂
        Py_INCREF(v);
        status = list_ass_slice(self, i, i+1, (PyObject *)NULL);
        if (status < 0) {
            Py_DECREF(v);
            return NULL;
        }
        return v;
    }

list_ass_slice
=================

注释上说明了, slice的赋值和删除操作


.. code-block:: c

    /* a[ilow:ihigh] = v if v != NULL.
     * del a[ilow:ihigh] if v == NULL.
     *
     * Special speed gimmick:  when v is NULL and ihigh - ilow <= 8, it's
     * guaranteed the call cannot fail.
     */
    static int
    list_ass_slice(PyListObject *a, Py_ssize_t ilow, Py_ssize_t ihigh, PyObject *v)
    {
        /* Because [X]DECREF can recursively invoke list operations on
           this list, we must postpone all [X]DECREF activity until
           after the list is back in its canonical shape.  Therefore
           we must allocate an additional array, 'recycle', into which
           we temporarily copy the items that are deleted from the
           list. :-( */
        // 上面这个注释就是说删除需要延迟减少计数的原因
        // 是因为直接减少引用计数的话, 会引发list的引用计数减少操作

        // result默认是失败的
        int result = -1;            /* guilty until proved innocent */
    #define b ((PyListObject *)v)
        // v是null, 则代表删除
        if (v == NULL)
            n = 0;
        else {
            // 下面是获取slice大小的
            // 主要是没看懂, 就先省略了
        }
        // 下面都是计算slice的左右边界的

        // slice的左右边界的大小
        norig = ihigh - ilow;
        assert(norig >= 0);
        d = n - norig;

        // 如果是直接让list长度变0, 直接清空list
        if (Py_SIZE(a) + d == 0) {
            Py_XDECREF(v_as_SF);
            return list_clear(a);
        }
 
        // 拿到元素数组
        item = a->ob_item;
        /* recycle the items that we are about to remove */
        // 复制item
        // s是n个要赋值元素的大小
        s = norig * sizeof(PyObject *);
        /* If norig == 0, item might be NULL, in which case we may not memcpy from it. */
        if (s) {
            // 之前预分配了8个, 可能不够大
            if (s > sizeof(recycle_on_stack)) {
                recycle = (PyObject **)PyMem_MALLOC(s);
                if (recycle == NULL) {
                    PyErr_NoMemory();
                    goto Error;
                }
            }
            // 复制元素
            memcpy(recycle, &item[ilow], s);
        }
    
        if (d < 0) { /* Delete -d items */
            Py_ssize_t tail;
            tail = (Py_SIZE(a) - ihigh) * sizeof(PyObject *);
            memmove(&item[ihigh+d], &item[ihigh], tail);

            // 这个list_resize是变小大小, a+d, d<0
            if (list_resize(a, Py_SIZE(a) + d) < 0) {
                memmove(&item[ihigh], &item[ihigh+d], tail);
                memcpy(&item[ilow], recycle, s);
                goto Error;
            }
            item = a->ob_item;
        }
        // 这里是说明赋值slice操作的
        else if (d > 0) { /* Insert d items */
            k = Py_SIZE(a);
            if (list_resize(a, k+d) < 0)
                goto Error;
            item = a->ob_item;
            // 依然要复制元素
            memmove(&item[ihigh+d], &item[ihigh],
                (k - ihigh)*sizeof(PyObject *));
        }
        for (k = 0; k < n; k++, ilow++) {
            // 这里是要赋值的slice的值, 所以要增加引用计数
            PyObject *w = vitem[k];
            Py_XINCREF(w);
            item[ilow] = w;
        }
        for (k = norig - 1; k >= 0; --k)
            // 这里是减少要删除的元素的引用计数
            Py_XDECREF(recycle[k]);
        result = 0;
     Error:
        if (recycle != recycle_on_stack)
            PyMem_FREE(recycle);
        Py_XDECREF(v_as_SF);
        return result;
    #undef b
    }


