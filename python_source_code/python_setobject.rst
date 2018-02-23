set
======

1. 和dict一样, 散列表实现, 但是没有key字典.

2. 有序性是hash顺序, 因为遍历的时候是遍历hash表.

3. 混合线性探测和开放地址法来解决冲突, 先线性遍历n个槽位, 如果找到空槽位, 则返回, 否则开放地址法探测, 若没有dummy或者空槽位置, 继续线性探测.

4. 如果插入一个dummy的槽位, 那么不需要扩容, 也就是fill不变, 而插入一个空槽位, 则需要考虑是否扩容

5. 如果dummy和used的总数, 也就是fill的值, 大于hash表的2/3, 那么resize, 和dict一样, set也是优先占据dummy槽位.

6. resize之后, 如果原来的fill包含了dummy的个数, 那么新的set的fill会变小. 也就是新的set只会包含used的元素.

7. 删除元素并不会缩减hash表. 这个和dict一样

8. pop操作是按hash顺序找到第一个可用的key, 然后弹出.


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


resize流程
===============

1. add的时候会占用dummy的槽位, 遇到空槽位和dummy, 优先插入dummy.

2. 线性探测只寻找空槽位, 遇到dummy则记录下来继续, 而二次探测遇到dummy则会返回, 最后综合, 如果之前记录到dummy了, 优先插入dummy槽位.

3. resize的条件是: 3 * fill > 2 * mask

4. used小于50000之前, resize的最小大小是used * 4, 否则是used * 2

5. 查找的时候先线性查找9个位置, 若没有发现空槽位, 那么继续开放地址法, 

.. code-block:: python

    '''
    
    x = {1, 2, 3, 4}, 此时fill = used = 4, hash_size = 8, mask = hash_size - 1 = 7
    
    xh是x的hash表, -1表示dummy, NULl表示空槽位, 所以hash表:
    
    xh = {NULL, 1, 2, 3, 4, NULL, NULL, NULL}, 长度是8
    
    1. 然后x.add(5)
    
    由于插入后, fill = 3 * 5 > 2 * 7, 所以扩容,要求 new_hash_size > 20 (used * 4), 所以new_hash_size = 32, mask=31, new_fill = used = 5

    xh = {NULL, 1, 2, 3, 4, 5, NULL, ...}, 长度是32
    
    2. 然后一直add, 有x={1, 2, 3, 4, 5, 6, 7, 8, 9, 10}, xh不会扩容, fill = used = 10

    xh = {NULL, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, NULL, ...}, 长度是32
    
    3. 然后x.pop(), pop出1, 此时used = 9(used-=1), fill=10(不变)

    xh = {NULL, -1, 2, 3, 4, 5, 6, 7, 8, 9, 10, NULL, ...}
    
    4. 继续pop, pop出2, used = 8, fill = 10
    
    xh = {NULL, -1, -1, 3, 4, 5, 6, 7, 8, 9, 10, NULL, ...}

    5. 然后我们add(1)
    
    5.1 发现槽位1是dummy, 记录下来
   
    5.2 然后先线性查找9个位置, 2, 3, 4, 5, 6, 7, 8, 9, 10, 都没有空槽位置
    
    5.3 那么二次探测, 得到槽位6, 还是不为空或者dummy, 那么继续从槽位6开始线性探测
    
    5.4 线性探测有: 7, 8, 9, 10, 11, 11是空槽位, 但是我们也找到了一个dummy槽位, 也就是1位置, 那么优先插入dummy槽位, fill=10(不变), used = 9(used += 1)

    xh = {NULL, 1, -1, 3, 4, 5, 6, 7, 8, 9, 10, NULL, ...}
    

    6. 继续x.add(2), 那么根据5的步骤, 一开始2槽位是dummy, 记录下来, 线性查找到槽位11为空, 然后判断, 发现我们记录有dummy槽位置, 那么2插入到xh下标2的位置而不是11

    xh = {NULL, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, NULL, ...}
    
    '''


跟dict不太一样, dict不会占用dummy的槽位的.


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
              // 第一个槽位是空的, 直接返回
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
           // 一个可用槽位
           if (entry->key == NULL)
               goto found_unused_or_dummy;

        // 获得可用槽位置
        found_unused_or_dummy:
          // freeslot是空, 说明是一个空槽位
          if (freeslot == NULL)
              goto found_unused;

          // 插入已经删除过的, dummy, 位置的话, 不需要扩容
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
          // 已用的和dummy的总大小大于hash的2/3, 扩容
          return set_table_resize(so, so->used>50000 ? so->used*2 : so->used*4);
    }



1. freeslot是一个dummy的槽位, 判断条件是该位置的entry.hash == -1. 这样插入的时候不需要resize, 所以分unused和dummy两种情况

2. 扩容的时候, 如果已用槽位大于50000, 那么扩容的时候至少要比used的两部大, 否则是4倍大. 也就是小于50000的set, 扩容会很快.

寻址dummy或者empty
=====================


寻址的时候, 线性探测总是要寻找一个空槽位置, 二次探测对于dummy也会返回

1. 一开始槽位i, 不为empty, 那么是dummy吗, 是dummy的话, 记录到free_slot, 继续.

2. 线性探测, 连续9次i+1, 但是i不变, 期间如果没有记录free_slot, 记录下来

3. 2找不到empty槽位, 那么进行一次开放地址法, 此时i变为开放地址法的下一个下标ii

4. 3拿到的元素如果是可用的(包括dummy), 记为entry. 因为此时判断条件是key == NULL, 而不是hash == - 1, 则返回, 否则继续1, 此时i=ii

5. 4中拿到的是可用的元素, 那么先查看free_slot是否有值, 也就是是否记录了一个dummy槽位, 如果记录了, 则优先插入dummy, free_slot赋值, 否则4中的entry

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

1. pop的时候不去resize

2. pop的时候不会减少fill, 而是只减少used

resize
=============


insert的时候传入的minused可能是used的两倍(used大于50000), 或者used的四倍(used小于50000).


.. code-block:: c

    static int
    set_table_resize(PySetObject *so, Py_ssize_t minused)
    {
        Py_ssize_t newsize;
        setentry *oldtable, *newtable, *entry;
        Py_ssize_t oldfill = so->fill;
        Py_ssize_t oldused = so->used;
        Py_ssize_t oldmask = so->mask;
        size_t newmask;
        int is_oldtable_malloced;
        setentry small_copy[PySet_MINSIZE];
    
        assert(minused >= 0);
    
        /* Find the smallest table size > minused. */
        /* XXX speed-up with intrinsics */

        // 最小大小不断乘以2, 得到新大小
        // 新大小一定要大于最小大小, 不算dummy的
        for (newsize = PySet_MINSIZE;
             newsize <= minused && newsize > 0;
             newsize <<= 1)
            ;
        if (newsize <= 0) {
            PyErr_NoMemory();
            return -1;
        }
    
        /* Get space for a new table. */

        oldtable = so->table;
        assert(oldtable != NULL);
        is_oldtable_malloced = oldtable != so->smalltable;
    
        // 新大小是最小hash表
        if (newsize == PySet_MINSIZE) {
            /* A large table is shrinking, or we can't get any smaller. */
            newtable = so->smalltable;
            if (newtable == oldtable) {
                if (so->fill == so->used) {
                    /* No dummies, so no point doing anything. */
                    return 0;
                }
                /* We're not going to resize it, but rebuild the
                   table anyway to purge old dummy entries.
                   Subtle:  This is *necessary* if fill==size,
                   as set_lookkey needs at least one virgin slot to
                   terminate failing searches.  If fill < size, it's
                   merely desirable, as dummies slow searches. */
                assert(so->fill > so->used);
                memcpy(small_copy, oldtable, sizeof(small_copy));
                oldtable = small_copy;
            }
        }
        else {
            // 否则分配一个新大小的hash表
            newtable = PyMem_NEW(setentry, newsize);
            if (newtable == NULL) {
                PyErr_NoMemory();
                return -1;
            }
        }
    
        /* Make the set empty, using the new table. */
        assert(newtable != oldtable);
        // hash表初始化空
        memset(newtable, 0, sizeof(setentry) * newsize);

        // 这里fill会赋值为used, 所以
        // 新大小的fill会比原来的小
        so->fill = oldused;
        so->used = oldused;
        so->mask = newsize - 1;
        so->table = newtable;
    
        /* Copy the data over; this is refcount-neutral for active entries;
           dummy entries aren't copied over, of course */
        // 下面是根据是否有dummy来考虑是否加入dummy的判断
        // if和else的代码差不多, 只是else多了一个dummy判断
        newmask = (size_t)so->mask;
        if (oldfill == oldused) {
            for (entry = oldtable; entry <= oldtable + oldmask; entry++) {
                if (entry->key != NULL) {
                    set_insert_clean(newtable, newmask, entry->key, entry->hash);
                }
            }
        } else {
            for (entry = oldtable; entry <= oldtable + oldmask; entry++) {
                if (entry->key != NULL && entry->key != dummy) {
                    set_insert_clean(newtable, newmask, entry->key, entry->hash);
                }
            }
        }
    
        if (is_oldtable_malloced)
            PyMem_DEL(oldtable);
        return 0;
    }


1. new_size满足2**n, 并且2**n一定要大于传入的minused大

2. fill和原来的fill相比, 可能变小, 因为原来的fill包含了dummy和used, 新的fill值包含used


remove
==========

remove的操作和pop一样, 只是pop是python自己找key而remove是用户指定的


.. code-block:: c

    static int
    set_discard_entry(PySetObject *so, PyObject *key, Py_hash_t hash)
    {
        setentry *entry;
        PyObject *old_key;
    
        entry = set_lookkey(so, key, hash);
        if (entry == NULL)
            return -1;
        if (entry->key == NULL)
            return DISCARD_NOTFOUND;
        old_key = entry->key;
        entry->key = dummy;
        entry->hash = -1;
        so->used--;
        Py_DECREF(old_key);
        return DISCARD_FOUND;
    }

