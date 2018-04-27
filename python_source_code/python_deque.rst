#########
deque实现
#########

deque
========

快速pop(pop/pop_left)和append(append/appendleft)操作, 使用固定大小的block来存储, 存储左右中间值, append/pop都是从中间往两边操作

每一个block有一个数据数组, 存储数据, 还有双链表的prev/next


.. code-block:: python

    '''
    
    block +-----------+ leftlink  (block类型指针, 相当于双链表的prev)
                      |
                      |
                      + rightlink (block类型指针, 相当于双链表的next)
                      |
                      |
                      + data      (存储数据的数组, 默认大小64)
    
    
    
    '''


下面是append/pop的流程图示

.. code-block:: python

    '''
    
    全局定义有BLOCK_LEN = 64, 然后中间值自然为CENTER = (64 - 1) / 2 = 31
    
    1. 初始化x = deque(), 分配一个block, 假设是b0, 有leftblock = rightblock = b0, leftindex = 31, rightindex = 32

       所以初始化的左右指针是从中间开始, 分别往左右走(中间开花 :)
    
    
    deque +---------------------+ leftblock +--->>>>-+
                                |                    |
                                |                    |
                                + rightblock +--->>>-+
                                |                    |
                                |                    |
                                + leftindex  = 31    |
                                |                    |
                                |                    |
                                + rightindex = 32    |
                                                     |
                                                     |
                                                     +-->->--> b0 +----------+ leftlink  = NULL
                                                                             |
                                                                             + rightlink = NULL
                                                                             |
                                                                             + data      = [](长度64, 中间值就是leftindex=31, rightindex=32)
    
    2. append(a)
    
    deque +---------------------+ leftblock +--->>>>-+
                                |                    |
                                |                    |
                                + rightblock +--->>>-+
                                |                    |
                                |                    |
                                + leftindex  = 31    |
                                |                    |
                                |                    |
                                + rightindex = 33    |
                                                     |
                                                     |
                                                     +-->->--> b0 +----------+ leftlink  = NULL
                                                                             |
                                                                             + rightlink = NULL
                                                                             |
                                                                             + data      = [..., ..., a, ..., ...]
                                                                                                rightindex先自增为33
                                                                                                a的位置是rightindex=33
    
    
    
    如果append(a), 那么会先移动rigindex += 1, 然后设置下标rightblock->data[rightindex] = a, 如果append(b), 发现rightblock用完了, 那么
    
    2.1 新建一个block=b1
    
    2.2 rightblock指向b1, 然后b0和b1通过自己的leftlink和rightlink做成两链表, b0  <----> b1
    
    2.3 然后deque->rightblock = b1, 自然此时rightindex不能是63了, 应该是0, 所以deque->rightindex=0

    deque +---------------------+ leftblock +--->>>>------+
                                |                         |
                                |                         |
                                + rightblock +--->>>-+    +
                                |                    |    |
                                |                    |    |
                                + leftindex  = 31    |    |
                                |                    |    |
                                |                    |    |
                                + rightindex = 0     |    |
                                                     |    |
                                                     |    |
                                                     |    +--> b0 +----------+ leftlink  = NULL
                                                     |                       |
                                                     |                       + rightlink = b1
                                                     |                       |
                                                     |                       + data      = [..., ..., ..., ..., ...]
                                                     |
                                                     +-->->--> b1 +----------+ leftlink  = b0
                                                                             |
                                                                             + rightlink = NULL
                                                                             |
                                                                             + data      = [b, ..., ..., ..., .....]
    
    3. pop
    
    pop则出了设置rightblock和rightlink之外, 还是会回收block的

    比如上面一直pop, b1完全没有使用了, 那么直接释放掉b1, 然后如果pop之后deque为空, 那么把左右index重新设置为中心开始!!!

           rightindex 
    |      leftindex  |            |
    +------------------------------+

    pop的时候, 使得rightindex--, 然后判断deque.size = 0

    那么设置leftindex和right从中间开始

    |      leftindex=31|rightindex=33             |
    +---------------------------------------------+
    
    '''


append
==========

cpython/Modules/_collectionsmodule.c

.. code-block:: c

    static int
    deque_append_internal(dequeobject *deque, PyObject *item, Py_ssize_t maxlen)
    {
        // 下面的if说明block的右边已经满了
        if (deque->rightindex == BLOCKLEN - 1) {
            // 一个新的block
            block *b = newblock();
            if (b == NULL)
                return -1;
            // 下面是双链表的链接
            b->leftlink = deque->rightblock;
            CHECK_END(deque->rightblock->rightlink);
            deque->rightblock->rightlink = b;
            deque->rightblock = b;
            MARK_END(b->rightlink);
            // 这里rightindex设置为-1, 因为下面会rightindex++操作
            deque->rightindex = -1;
        }
        // 下面就是设置data[++rightindex] = item
        Py_SIZE(deque)++;
        deque->rightindex++;
        deque->rightblock->data[deque->rightindex] = item;
        if (NEEDS_TRIM(deque, maxlen)) {
            PyObject *olditem = deque_popleft(deque, NULL);
            Py_DECREF(olditem);
        } else {
            deque->state++;
        }
        return 0;
    }


