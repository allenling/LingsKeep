list
=========

1. 使用数组来存储元素

2. **任何添加删除元素**, 会先修改当前list对象的size为newsize, 然后根据新的size的值去调整数组长度, 增加或者减少数组长度, 而dict是不会的缩减存储的长度的. resize会将老数组元素复制到新数组上.

3. 已分配大小allocated, 元素个数size, resize的条件是: allocated > newsize的时候, 增加数组大小, newsize < (allocated / 2)的时候, 缩减数组长度

4. resize之后的allocated长度, 是基于newsize的, 也就是如果是增加数组长度, allocated = newsize + 增加步长, 如果是缩减, 则是allocated = newsize + 缩减步长

5. resize的时候, 增加和删除的步长是不一样的

6. 数组大小n字节, 对应pool的单位分配大小是k字节, 那么如果n/k > 3/4的时候, 数组并不会重新分配内存.

7. 插入slice(包括append)的时候, 把目标位置之后的元素往后移动, 然后在目标位置依次赋值

8. 删除slice(包括pop不是最后一个位置)的时候, 把要删除元素之后的元素往前移动覆盖删除元素, 删除slice的时候会有一个延迟减少引用计数的操作

9. **pop最后一个元素** 的操作很快, 只是将list对象的size减少1, 而不会真正操作数组的.

10. append也很快, 只需数组槽位赋值而已, pop和append的复杂度都是O(1)

11. insert是for循环一个个去移动, 而slice和resize则是调用memmove/memcpy系统调用去移动(复制)元素.

12. contains和index都是需要逐个去比较, 必去dict和set, 复杂度都是O(n)

增加数组长度的步长
=====================

.. code-block:: python

    '''

    1. x = []

    allocated=0, new_size=0

    2. x.append(1)
    
    new_size=1
    
    new_allocated = (new_size >> 3) + (new_size <9 ? 3: 6)
    
    new_allocated = (1 >> 3) + (1 <9 ? 3: 6) = 0 + 3 = 3
    
    new_allocated += new_size = 3 + 1 = 4
    
    3. x的长度为1, 已分配大小为4, 一直append直到x=[1, 2, 3, 4]都不会resize, 然后x.append(5):

    allocated=4, new_size=5
    
    new_allocated = (new_size >> 3) + (new_size <9 ? 3: 6)
    
    new_allocated = (5 >> 3) + (5 <9 ? 3: 6) = 0 + 3 = 3
    
    new_allocated += new_size = 3 + 5 = 8

    '''

删减数组长度的过程
=======================

x=[1, 2, 3, 4, 5], 调用x.pop(1), 此时allocated=8, new_size=4, 因为4>=8/2, 所以数组长度不变

.. code-block:: python

    '''

    x =  [1, 2, 3, 4, 5]

    allocated = 8, size = 5

    1. x.pop()
    
    allocated = 8, newsize = 4

    allocated > newsize = 8 > 4 && newsize >= (allocated >> 1) = 4 >= 8/2

    所以这一步并不会缩减数组长度, x = [1, 2, 3, 4]

    2. x.pop()
    
    allocated=8, newsize=3, 有newsize < (allocated >> 1) = 3 < 8 /2

    所以触发缩减数组操作
    
    new_allocated = (new_size >> 3) + (new_size <9 ? 3: 6)
    
    new_allocated = (3 >> 3) + (3 <9 ? 3: 6) = 0 + 3 = 3
    
    new_allocated += new_size = 3 + 3 = 6

    '''

x的数组长度变为6, 一直append知道x=[1, 2, 3, 4, 5, 6], 数组才会再次扩张.

所以注释中的步长只是一直append的时候的长度, 长度变化最主要要满足: allocated > new_size并且new_size>=allocated/2这两个条件



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


元素数组是定义为**, 也就是指针的指针, 这个可以看做数组, 具体请看: C_指针小结.rst


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
    
        // 调用resize函数去看看是否需要resize
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
        // 2. 需要缩减: newsize的小于已分配的一半		
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
        new_allocated = (newsize >> 3) + (newsize < 9 ? 3 : 6);		
    		
        /* check for integer overflow */		
        if (new_allocated > SIZE_MAX - newsize) {		
            PyErr_NoMemory();		
            return -1;		
        } else {		
            // 注意, 这里是+=
            new_allocated += newsize;		
        }		
    		
        if (newsize == 0)		
            new_allocated = 0;		
        items = self->ob_item;		
        if (new_allocated <= (SIZE_MAX / sizeof(PyObject *)))		
            // 这里的PyMem_RESIZE才是真正的去改变内存大小		
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

resize条件
==============

.. code-block:: c

        if (allocated >= newsize && newsize >= (allocated >> 1)) {
            // 不resize
        }        

1. 需要扩容: newsize大于已分配的内存, allocated < new_size

2. 需要缩减: newsize的小于已分配的一半, (allocated >> 1) > new_size

内存复制
===========

resize的时候是调用PyMem_RESIZE去新分配一个数组, 然后把元素复制过去

PyMem_RESIZE最后调用


.. code-block:: c

    static void *
    _PyObject_Realloc(void *ctx, void *p, size_t nbytes)
    {
        void *bp;
        poolp pool;
        size_t size;
    
        if (p == NULL)
            return _PyObject_Alloc(0, ctx, 1, nbytes);
    
        // 省略代码
    
        pool = POOL_ADDR(p);
        if (address_in_range(p, pool)) {
            /* We're in charge of this block */
            size = INDEX2SIZE(pool->szidx);
            if (nbytes <= size) {
                /* The block is staying the same or shrinking.  If
                 * it's shrinking, there's a tradeoff:  it costs
                 * cycles to copy the block to a smaller size class,
                 * but it wastes memory not to copy it.  The
                 * compromise here is to copy on shrink only if at
                 * least 25% of size can be shaved off.
                 */
                if (4 * nbytes > 3 * size) {
                    /* It's the same,
                     * or shrinking and new/old > 3/4.
                     */
                    // 这里并没有新分配内存, 而是返回
                    // 数组的原内存地址
                    return p;
                }
                size = nbytes;
            }
            // 新分配内存
            bp = _PyObject_Alloc(0, ctx, 1, nbytes);
            if (bp != NULL) {
                // 然后复制内存
                memcpy(bp, p, size);
                _PyObject_Free(ctx, p);
            }
            return bp;
        }
        
    }


从注释可以看出来, 如果pool的单位长度, 比如是32字节, 比缩减之后的长度, 比如25字节大, 

并且缩减之后的长度至少占pool单位长度的3/4, 那么不就会去新开辟内存空间. 估计是为了

充分利用数组原来所占的内存吧.


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

        // 拿到对应下标的对象
        v = self->ob_item[i];

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
        // 调用slice操作
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
            // 没看懂, 就省略了
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

            // 这里移动内存
            memmove(&item[ihigh+d], &item[ihigh], tail);

            // 这里缩小list的长度
            if (list_resize(a, Py_SIZE(a) + d) < 0) {
                // 这里list长度改变失败, 然后把老内容重新复制到list
                memmove(&item[ihigh], &item[ihigh+d], tail);
                memcpy(&item[ilow], recycle, s);
                goto Error;
            }
            item = a->ob_item;
        }
        // 这里是说明赋值slice操作的
        else if (d > 0) { /* Insert d items */
            k = Py_SIZE(a);
            // 增加list长度
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

memmove
===========

系统调用, 第一个参数是目标位置, 第二个参数是源位置, 第三个参数是内存大小

也就是把源位置开始, 之后的指定大小的内存, 复制到目标位置.


slice缩减移动元素
==================

.. code-block:: c


        if (d < 0) { /* Delete -d items */
            Py_ssize_t tail;
            tail = (Py_SIZE(a) - ihigh) * sizeof(PyObject *);

            // 这里移动内存
            memmove(&item[ihigh+d], &item[ihigh], tail);

            // 这里缩小list的长度
            if (list_resize(a, Py_SIZE(a) + d) < 0) {
                memmove(&item[ihigh], &item[ihigh+d], tail);
                memcpy(&item[ilow], recycle, s);
                goto Error;
            }
            item = a->ob_item;
        }

.. code-block:: python

    '''
    
    x = [1, 2, 3, 4, 5], x.pop(1)
    
    其中d=-1, ilow=1, ihight=2, 此时tail = 5 - 2 = 3, 也就是移动3个元素.
    
    调用memmove(&item[1], &item[2], 3), 也就是把3, 4, 5移动到2所在的位置, 变为:
    
    [1, 3, 4, 5, NULL, NULL, NULL, NULL], allocated = 8
    
    然后缩减x的长度为4, 但是数组长度依然是8, 因为4 >= 8/2, 然后继续pop(1), newsize=3, 3 < 8/2 有

    [1, 4, 5, NULL, NULL, NULL], allocated=6


    
    '''

增加长度移动元素
==================

.. code-block:: c

        else if (d > 0) { /* Insert d items */
            k = Py_SIZE(a);
            // 增加list长度
            if (list_resize(a, k+d) < 0)
                goto Error;
            item = a->ob_item;
            // 依然要复制元素
            memmove(&item[ihigh+d], &item[ihigh],
                (k - ihigh)*sizeof(PyObject *));
        }
       // n是slice的大小
       for (k = 0; k < n; k++, ilow++) {
        PyObject *w = vitem[k];
        Py_XINCREF(w);
        // 一个个插入到list中
        item[ilow] = w;
       }


.. code-block:: python

    '''
    
    x = [1, 2, 3, 4, 5], x[1:2] = [10: 11]
    
    其中k=5, d=1, ilow=1, ihight=2, n=2, 此时tail = 5 - 2 = 3, 也就是移动3个元素.
    
    调用memmove(&item[3], &item[2], 3), 也就是把3, 4, 5移动到4开始的位置
    
    [1, , , 3, 4, 5]
    
    然后一个个插入
    
    [1, 10, 11, 3, 4, 5]
    
    '''

