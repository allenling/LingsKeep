python的容器对象
====================

容器, 指dict/list/tuple/set

加一个queue

dict
=========

https://docs.python.org/3/faq/design.html#how-are-dictionaries-implemented

使用开放地址法的变长的hash map

Python’s dictionaries are implemented as resizable hash tables. Compared to B-trees, this gives better performance for lookup (the most common operation by far) under most circumstances, and the implementation is simpler.

插入一个key,value之后的hash map的长度大于总长度的2/3, 则hash map会扩容, 并且把老的hash map的元素使用新的掩码(新的hash map的长度)复制到新的hash map上

hash key的算法参考: http://effbot.org/zone/python-hash.htm

https://stackoverflow.com/questions/327311/how-are-pythons-built-in-dictionaries-implemented

3.6已经重新实现了一个结构更紧凑的dict, 参考了pypy的实现: https://docs.python.org/3/whatsnew/3.6.html#new-dict-implementation, https://morepypy.blogspot.com/2015/01/faster-more-memory-efficient-and-more.html

比起3.5, 节省了20%-25%的内存, 并且现在keys返回是有序的，和key插入的顺序一样, 而3.6之前是ascii顺序, 但是ipython中你回车出来看到的依然是ascii顺序的:

.. code-block:: python

    In [44]: x={'b': 1, 'a': 2}
    
    In [45]: x
    Out[45]: {'a': 2, 'b': 1}
    
    In [46]: list(x.keys())
    Out[46]: ['b', 'a']

dict的key是由对象的hash值决定的, 然后所有的不可变类型都是可以hash的, 然后用户定义的类型的默认hash值是其的id, 比如类T, t=T(), t的hash值就是id(t), 你可以定义一个__hash__方法来决定对象的hash值

https://docs.python.org/3/glossary.html#term-hashable


如何压缩
--------------------

源码在: https://github.com/python/cpython/blob/v3.6.3/Objects/dictobject.c

首先源码注释中说明了新的dict的结构

.. code-block:: python

    '''
    +---------------+
    | dk_refcnt     |
    | dk_size       |
    | dk_lookup     |
    | dk_usable     |
    | dk_nentries   |
    +---------------+
    | dk_indices    |
    |               |
    +---------------+
    | dk_entries    |
    |               |
    +---------------+
    '''

并且dk_indices是哈希表, 然后dk_entries是key对象的数组, 从dk_indices里面存储的是dk_entries的下标, 然后在3.6的dict实现的建议里面, 有:

For example, the dictionary:

.. code-block:: python

    d = {'timmy': 'red', 'barry': 'green', 'guido': 'blue'}

is currently stored as:

.. code-block:: python

    entries = [['--', '--', '--'],
               [-8522787127447073495, 'barry', 'green'],
               ['--', '--', '--'],
               ['--', '--', '--'],
               ['--', '--', '--'],
               [-9092791511155847987, 'timmy', 'red'],
               ['--', '--', '--'],
               [-6480567542315338377, 'guido', 'blue']]

Instead, the data should be organized as follows:

.. code-block:: python

    indices =  [None, 1, None, None, None, 0, None, 2]
    entries =  [[-9092791511155847987, 'timmy', 'red'],
                [-8522787127447073495, 'barry', 'green'],
                [-6480567542315338377, 'guido', 'blue']]

之前的dict和3.6的dict各种接口操作是一样的, 比如都是计算了hash之后, 拿到hash再经过mod运算, 得到hash表的下标, 比如某个key的hash=-8522787127447073495, 这个hash % 8 = 1, 就去entries这个hash表的下标1的数组

去比对hash值, 然后比对相等, 则返回, 如果不相等, 则二次探测. 区别的是3.6的dict中, hash表被单独提出来为indices, 然后原来的hash, key, value这个组合依然存储在entries这个数组内, 然后indices存储的是

entries的下标, 所以3.6的dict是先在计算hash表的下标, 比如hash=-8522787127447073495, 然后hash mod 8 = 1, 然后去查询indices数组下标为1元素, 里面是1, 表示应该去查询entries数组下标为1的元素, 然后去

查找entries数组下标为1的元素, 是一个hash, key, value对, 然后比对hash值, 相等则返回, 不相等, 则继续二次探测. 二次探测为: (next_j = ((5*j) + 1) mod (perturb >> PERTURB_SHIFT), 其中perturb=hash, PERTURB_SHIFT=5.


可以看到, 原来的dict是一个entries就是一个hash表, 然后下标对应存储的是hash值和key, value, 然后存储的空间就很浪费, 64位机器下是24 bit一个hash表的row, 所以之前存储

的话就要花费24 * 8 = 192 bit. 而3.6的dict则是hash数组是int数组, 元素为1 bit来, 表示entries数组的下标, 而 **entries表是一个插入的时候append only的数组**, 是紧凑的数组, 而花费的空间

为: 8(hash数组) + 24 * 3 = 80, 所以空间大幅度减少了.


排序的区别
-------------

之前的dict是"无序"的, 其实应该说是按hash值排序的, 在之前的dict中, keys的代码为:

https://hg.python.org/cpython/file/52f68c95e025/Objects/dictobject.c#l1180

.. code-block:: c

    static PyObject *
    dict_keys(register PyDictObject *mp) {
        ep = mp->ma_table;
        mask = mp->ma_mask;
        for (i = 0, j = 0; i <= mask; i++) {
            if (ep[i].me_value != NULL) {
                PyObject *key = ep[i].me_key;
                Py_INCREF(key);
                PyList_SET_ITEM(v, j, key);
                j++;
            }
        }
    }


可以看到, 遍历的时候的终止条件是i<=mask, 而mask则是hash表的长度-1, 也就是会遍历hash表, 所以得到的key就是hash排序的key


而3.6的keys函数为:

https://github.com/python/cpython/blob/v3.6.3/Objects/dictobject.c#L2179

.. code-block:: c

    static PyObject *
    dict_keys(PyDictObject *mp)
    {
        ep = DK_ENTRIES(mp->ma_keys);
        size = mp->ma_keys->dk_nentries;
        for (i = 0, j = 0; i < size; i++) {
            if (*value_ptr != NULL) {
                PyObject *key = ep[i].me_key;
                Py_INCREF(key);
                PyList_SET_ITEM(v, j, key);
                j++;
            }
            value_ptr = (PyObject **)(((char *)value_ptr) + offset);
        }
    }

可以看到, size是dk_nentries的大小, 也就是dk_entries的大小, 然后遍历的时候会从ep直接拿key对象的指针, 而ep就是dk_entries, 所以也就是直接遍历dk_entries

这个数组, 而这个数组是insert的时候append only的, 也就是保持了插入的顺序

hash table rbt(map)
---------------------

http://blog.csdn.net/ljlstart/article/details/51335687

https://www.zhihu.com/question/24506208

大概就是:

1. hash table内存比较大, map的话内存比较小

2. hash table是无序的, map的话是有序的

3. map比较稳定, 最差也就是logN, hash table好的时候可以说常数级, 但是这个常数级可能比logN大, 然后最坏的时候搜索要遍历整个hash table, 也就是O(N)
   
  也就是hash table搜索效率依赖于冲突, hash table冲突很大的话, 搜索就慢了, 可以打到O(N)



set
======

https://fanchao01.github.io/blog/2016/10/24/python-setobject/


一个hash table实现的, 然后遍历的时候就是遍历hash table, 所以看起来才是"无序"的


list
=======

https://www.laurentluce.com/posts/python-list-implementation/

列表也就是一个数组, 然后当添加或者删除一个元素的时候, 列表的长度会变化的, 下面是代码摘抄:

.. code-block:: c

    // https://github.com/python/cpython/blob/v3.6.3/Objects/listobject.c

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
        // 所以, 换句话说, 需要扩展内存大小的情况是, newsize大于已分配的内存, 需要缩减内存的情况是
        // newsize的大小小于已分配的一半
        if (allocated >= newsize && newsize >= (allocated >> 1)) {
            assert(self->ob_item != NULL || newsize == 0);
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
        // newsize >> 3是newsize往右移３位, 也就是newsize / 8, 毕竟往右移一位等于除以2
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

跟位置无关的操作, 比如append, pop的复杂度都是O(1), 其他跟位置有关的都是O(n), 比如insert, pop(index), remove(value)

