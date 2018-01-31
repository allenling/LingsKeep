set
======

1. 和dict一样, 散列表实现, 但是没有key字典.

2. 有序性是hash顺序, 因为遍历的时候是遍历hash表.

3. 混合线性探测和开放地址法来解决冲突

4. 如果插入一个dummy的槽位, 那么不需要扩容, 而插入一个空槽位, 则需要考虑是否扩容

5. 如果dummy和active的总数大于hash表的2/3, 那么resize.

6. 减少元素并不会缩减hash表. 这个和dict一样, 虽然dict的resize目标是key数组.

7. pop操作是按hash顺序找到第一个可用的key, 然后弹出.


实现思路
==========


.. code-block:: c

    /* set object implementation
    
       Written and maintained by Raymond D. Hettinger <python@rcn.com>
       Derived from Lib/sets.py and Objects/dictobject.c.
    
       The basic lookup function used by all operations.
       This is based on Algorithm D from Knuth Vol. 3, Sec. 6.4.
    
       The initial probe index is computed as hash mod the table size.
       Subsequent probe indices are computed as explained in Objects/dictobject.c.
    
       To improve cache locality, each probe inspects a series of consecutive
       nearby entries before moving on to probes elsewhere in memory.  This leaves
       us with a hybrid of linear probing and open addressing.  The linear probing
       reduces the cost of hash collisions because consecutive memory accesses
       tend to be much cheaper than scattered probes.  After LINEAR_PROBES steps,
       we then use open addressing with the upper bits from the hash value.  This
       helps break-up long chains of collisions.
    
       All arithmetic on hash should ignore overflow.
    
       Unlike the dictionary implementation, the lookkey function can return
       NULL if the rich comparison returns an error.
    */

1. 散列表实现

2. 解决冲突是先线性探测n个槽位, 然后再使用开放地址法


----


PySetObject
================

.. code-block:: c

    typedef struct {
        PyObject_HEAD
    
        // 已用的和dummy的总数, 用于计算是否resize
        Py_ssize_t fill;            /* Number active and dummy entries*/

        // 已用的个数
        Py_ssize_t used;            /* Number active entries */
    
        /* The table contains mask + 1 slots, and that's a power of 2.
         * We store the mask instead of the size because the mask is more
         * frequently needed.
         */
        // hash表的掩码
        Py_ssize_t mask;
    
        // table就是has表
        // 然后小set的table会指向smalltable
        setentry *table;
        Py_hash_t hash;             /* Only used by frozenset objects */

        // 这个finger则是pop的时候使用的第一个位置
        // 一开始是0, 会变的, 看pop那一节
        Py_ssize_t finger;          /* Search finger for pop() */
    
        setentry smalltable[PySet_MINSIZE];
        PyObject *weakreflist;      /* List of weak references */
    } PySetObject;


创建set
=========

.. code-block:: c

    static PyObject *
    make_new_set(PyTypeObject *type, PyObject *iterable)
    {
        PySetObject *so;
    
        // 分配内存大小
        so = (PySetObject *)type->tp_alloc(type, 0);
        if (so == NULL)
            return NULL;
    
        // 各种初始化
        so->fill = 0;
        so->used = 0;
        so->mask = PySet_MINSIZE - 1;
        // 这里初始化为小hash表
        so->table = so->smalltable;
        so->hash = -1;
        so->finger = 0;
        so->weakreflist = NULL;
    
        if (iterable != NULL) {
            // 这里会更新set结构
            if (set_update_internal(so, iterable)) {
                Py_DECREF(so);
                return NULL;
            }
        }
    
        return (PyObject *)so;
    }


set_update_internal
========================

set更新操作

.. code-block:: c

    static int
    set_update_internal(PySetObject *so, PyObject *other)
    {
        PyObject *key, *it;
    
        if (PyAnySet_Check(other))
            return set_merge(so, other);
    
        if (PyDict_CheckExact(other)) {
            PyObject *value;
            Py_ssize_t pos = 0;
            Py_hash_t hash;
            Py_ssize_t dictsize = PyDict_Size(other);
    
            /* Do one big resize at the start, rather than
            * incrementally resizing as we insert new keys.  Expect
            * that there will be no (or few) overlapping keys.
            */
            // 如果是dict, 那么会拿dict的key来作为set的元素
            // 这里会可能直接一次
            // 增长固定大小而不是插入一个key而扩张一次
            if (dictsize < 0)
                return -1;
            // 这里会根据dict的大小去resize
            if ((so->fill + dictsize)*3 >= so->mask*2) {
                if (set_table_resize(so, (so->used + dictsize)*2) != 0)
                    return -1;
            }
            while (_PyDict_Next(other, &pos, &key, &value, &hash)) {
                // 一个个插入
                if (set_add_entry(so, key, hash))
                    return -1;
            }
            return 0;
        }
    
        it = PyObject_GetIter(other);
        if (it == NULL)
            return -1;
    
        // 迭代一下
        while ((key = PyIter_Next(it)) != NULL) {
            // 然后插入
            if (set_add_key(so, key)) {
                Py_DECREF(it);
                Py_DECREF(key);
                return -1;
            }
            Py_DECREF(key);
        }
        Py_DECREF(it);
        if (PyErr_Occurred())
            return -1;
        return 0;
    }


set_add_entry
==================

逐个添加元素到set


.. code-block:: c

    static int
    set_add_entry(PySetObject *so, PyObject *key, Py_hash_t hash)
    {
        restart:

          mask = so->mask;
          // 拿到第一个位置
          i = (size_t)hash & mask;
          
          // 拿到第一个位置的槽位
          entry = &so->table[i];
          if (entry->key == NULL)
              goto found_unused;

          freeslot = NULL;
          perturb = hash;

          // 下面就是查找过程
          while (1) {
           // 好的, hash值相同
           if (entry->hash == hash) {
               PyObject *startkey = entry->key;
               /* startkey cannot be a dummy because the dummy hash field is -1 */
               assert(startkey != dummy);

               // 并且key的地址也相等
               // 这里直接==的话是比较内存地址
               if (startkey == key)
                   // 说明已经存在set了, 直接退出
                   goto found_active;
               // 一个unicode类型的key, 那么调用unicode的比较函数比较一下
               if (PyUnicode_CheckExact(startkey)
                   && PyUnicode_CheckExact(key)
                   && _PyUnicode_EQ(startkey, key))
                   // 是一样的, 退出
                   goto found_active;
               
               // 需要更详细的比较
               table = so->table;
               Py_INCREF(startkey);

               // 调用一般性比较函数
               cmp = PyObject_RichCompareBool(startkey, key, Py_EQ);
               Py_DECREF(startkey);
               // 这个说明两者"很像"?
               if (cmp > 0)                                          /* likely */
                   // 说明两者是同一个, 退出
                   goto found_active;
               if (cmp < 0)
                   goto comparison_error;
               /* Continuing the search from the current entry only makes
                  sense if the table and entry are unchanged; otherwise,
                  we have to restart from the beginning */

               // 这里需要重新开始, 没太明白
               if (table != so->table || entry->key != startkey)
                   goto restart;
               mask = so->mask;                 /* help avoid a register spill */
           }
           else if (entry->hash == -1 && freeslot == NULL)
               // hash == -1, 说明是一个dummy的槽位
               freeslot = entry;

           // 下面是探测的过程
           if (i + LINEAR_PROBES <= mask) {

             // 这个是线性探测的过程
             // 也是重复上面的比较过程了
            for (j = 0 ; j < LINEAR_PROBES ; j++) {
                entry++;
                if (entry->hash == 0 && entry->key == NULL)
                    goto found_unused_or_dummy;
                if (entry->hash == hash) {
                    PyObject *startkey = entry->key;
                    assert(startkey != dummy);
                    if (startkey == key)
                        goto found_active;
                    if (PyUnicode_CheckExact(startkey)
                        && PyUnicode_CheckExact(key)
                        && _PyUnicode_EQ(startkey, key))
                        goto found_active;
                    table = so->table;
                    Py_INCREF(startkey);
                    cmp = PyObject_RichCompareBool(startkey, key, Py_EQ);
                    Py_DECREF(startkey);
                    if (cmp > 0)
                        goto found_active;
                    if (cmp < 0)
                        goto comparison_error;
                    if (table != so->table || entry->key != startkey)
                        goto restart;
                    mask = so->mask;
                }
                else if (entry->hash == -1 && freeslot == NULL)
                    // hash == -1, 说明是一个dummy的槽位
                    freeslot = entry;
              }
           }

           // 下面是开放地址法获得下一个位置
           perturb >>= PERTURB_SHIFT;
           i = (i * 5 + 1 + perturb) & mask;

           entry = &so->table[i];
           // 一个空槽位
           if (entry->key == NULL)
               goto found_unused_or_dummy;

        // 获得可用的处理
        found_unused_or_dummy:
          // freeslot是空, 说明是一个dummy的
          if (freeslot == NULL)
              goto found_unused;

          // 插入已经删除过的位置的话, 不需要扩容
          so->used++;
          freeslot->key = key;
          freeslot->hash = hash;
          return 0;

        // 空槽位, 并且是没有删除过的
        found_unused:
          so->fill++;
          so->used++;
          entry->key = key;
          entry->hash = hash;
          // 这个时候的插入需要考虑扩容
          if ((size_t)so->fill*3 < mask*2)
              return 0;
          return set_table_resize(so, so->used>50000 ? so->used*2 : so->used*4);
    }



1. freeslot是一个dummy的槽位, 这样插入的时候不需要resize, 所以分unused和dummy两种情况


pop
=====


pop只是把槽位设置为dummy, 然后并不缩减hash大小


.. code-block:: c

    static PyObject *
    set_pop(PySetObject *so)
    {
        /* Make sure the search finger is in bounds */
        // finger初始化是0
        Py_ssize_t i = so->finger & so->mask;
        setentry *entry;
        PyObject *key;
    
        assert (PyAnySet_Check(so));
        if (so->used == 0) {
            PyErr_SetString(PyExc_KeyError, "pop from an empty set");
            return NULL;
        }
    
        // 找到第一个不为dummy的key
        // 弹出去
        while ((entry = &so->table[i])->key == NULL || entry->key==dummy) {
            i++;
            if (i > so->mask)
                i = 0;
        }
        // 找到了一个可用的key
        key = entry->key;
        entry->key = dummy;
        entry->hash = -1;
        so->used--;

        // finger是可用位置的下一个位置
        so->finger = i + 1;         /* next place to start */
        return key;
    }





