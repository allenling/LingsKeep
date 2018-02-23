Dict
====================

python3.6的实现

1. 字典是散列表实现, 二次探测法进行hash探测
   
2. 使用key数组保证了key是插入顺序, key/value一般一起存在key数组.

3. 字典只会在添加k/v的时候扩张key数组长度, 但是删除操作不会缩减key数组, resize只针对key数组, hash表不会resize, 只有hash mask会变化.

   为什么不缩减数组长度呢? 注释说缩减要考虑到keys数组, 不好操作. 

4. 插入的时候, 会优先插入到dummy槽位, 和set一样, 但是不管插入的是empty还是dummy, usable总是减少1, 而set插入dummy的时候, fill不变.

5. resize的条件是可用空间(usable)为0, 也就是数组长度的2/3已经被使用, 那么扩容

6. resize每次都是最小长度(8)开始, 一直乘以2, 最后保证new_size要大于当前已用大小的两倍, new_size > used * 2.

7. resize之后, 新usable只会包含used的元素, 不包含dummy的元素, 所以新的usable可能会比老usable大.

8. 区分有split和combined两种模式.

9. dict根据key的不一样有不同的搜索策略

10. py2和3中, 调用dict函数和调用{}创建字典的区别和性能.



resize流程
===============

1. 关于dummy的槽位, dict也会优先占据dummy槽位, 这个和set一样.

2. 但是不管插入的是dummy还是empty槽位, 总是会减少usable的个数, 而set的话, 插入dummy槽位不会减少fill的个数.

2. 因为每次都是insert的时候, 总是赋值数组的dk_nentries这个下标位置, 所以这个位置只会增加, 不会减少, 也就是delete的时候不变, insert的时候加1

3. GROWTH_RATE是每次需要扩容的时候, 扩容的最小大小, 值是: used * 2 + size / 2 

.. code-block:: python

    '''
    x = {1: 1, 2: 2, 3: 3, 4: 4, 5: 5}, 此时usable = 0, used = 5, array_size = 8, dk_nentries=5
    
    hash表: xh = [-1, 0, 1, 2, 3, 4, -1, -1]

    数组为: xa = [1, 2, 3, 4, 5, NULL, NULL, NULL]

    xh中, -1表示空槽位, -2表示dummy, 存储的是xa的下标
    
    1. 然后x[6] = 6, 此时寻找第一个hash槽位, 6 & 7 = 6, 所以找到xh[6], 看到是empty, 返回. 由于usable = 0, 所以扩容, 最小大小是2 * 5 + 8 / 2 = 14.
       
    要求 new_array_size >= 14, 所以new_array_size = 16, used = 5, usable = 16 * 2/3 - used = 5, dk_nentries = 5:

    xh = [-1, 0, 1, 2, 3, 4, -1, -1, ...], 长度16

    xa = [1, 2, 3, 4, 5, NULL, ...], 长度16

    然后插入最后xa最后一位, xh[6]等于下标5:

    xh = [-1, 0, 1, 2, 3, 4, 5, -1, -1, ...], 长度16

    xa = [1, 2, 3, 4, 5, 6, NULL, NULL, ...], 长度16

    之后used += 1 = 6, usable -= 1 = 4, dk_nentries += 1 =6
    
    2. 然后一直赋值, 有x={1: 1, 2: 2, 3: 3, 4: 4, 5: 5, 6: 6, 7: 7, 8: 8, 9: 9, 10: 10}, x不会扩容, usable = 0, used = 10, dk_nentries=10

    xh = [-1, 0, 1, 2, 3, 4, 5, 6, 7, 8, 9, -1, ...], 长度16

    xa = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10, NULL, ...], 长度16
    
    3. 然后del x[1], 此时used = 9(used-=1), usable = 0(不变), dk_nentries=10, 不变

    xh = [-1, -2, 1, 2, 3, 4, 5, 6, 7, 8, 9, -1, ...], 长度16

    xa = [NULL, 2, 3, 4, 5, 6, 7, 8, 9, 10, NULL, ...], 长度16
    
    4. 继续del x[2], used = 8, usable = 0, dk_entries = 10, 不变

    xh = [-1, -2, -2, 2, 3, 4, 5, 6, 7, 8, 9, -1, ...], 长度16

    xa = [NULL, NULL, 3, 4, 5, 6, 7, 8, 9, 10, NULL, ...], 长度16
    
    5. 然后x[1] = 1

    5.1. 先第一次搜索, 第一个槽位是1, 发现是dummy, 然后保存起来到free_slot=1, 继续, 直到找到一个empty的槽位, 二次探测最后得到1->6->15, 15是空槽位.

    5.2. 插入的时候, 需要插入槽位1, 因为虽然15是空槽位, 但是先发现了一个dummy槽位置为free_slot=1, 所以要插入槽位1
    
    5.3 插入的时候发现usable是0, 需要扩容, 最小大小是28, 所以new_size = 32. 
    
    5.4. 新的xa数组需要把老key复制到新的key数组, 过程是顺序遍历xa, 只复制不为NULL的key, 然后放到新数组中, dk_entries初始为0, 遍历的时候dk_entries+=1

    new_xh = [-1, -1, -1, 0, 1, 2, 3, 4, 5, 6, 7, -1, -1, -1, ...], 长度是32

    new_xa = [3, 4, 5, 6, 7, 8, 9, 10, NULL, NULL, ...], 长度是32


    5.5 重新从new_xh和new_xa中找到1的可用位置, 明显, 第一个槽位1就是可用的, 所以把1插入到1的位置:

    new_xh = [-1, 8, -1, 0, 1, 2, 3, 4, 5, 6, 7, -1, ...], 长度是32

    new_xa = [3, 4, 5, 6, 7, 8, 9, 10, 1, NULL, ...], 长度是32
    
    '''


数据结构
=============

使用开放地址法的变长的哈希表, 比起b树结构, 查找更好一点, 并且实现更简单:

*Python’s dictionaries are implemented as resizable hash tables. 
Compared to B-trees, this gives better performance for lookup (the most common operation by far) under most circumstances, and the implementation is simpler.*

和树结构实现的map比较, 大概就是:

http://blog.csdn.net/ljlstart/article/details/51335687

https://www.zhihu.com/question/24506208

1. hash表内存比较大, map的话内存比较小

2. hash表是"无序的"(一般值hash表是hash值模运算顺序), map的话是有序的

3. map比较稳定, 最差也就是logN, hash table好的时候可以说常数级, 但是这个常数级可能比logN大, 然后最坏的时候搜索要遍历整个hash table, 也就是O(N)

   也就是hash table搜索效率依赖于冲突, hash table冲突很大的话, 搜索就慢了, 可以达到O(N)


dict in 3.6
==============

3.6已经重新实现了一个结构更紧凑的dict, 参考了 `pypy的实现 <https://docs.python.org/3/whatsnew/3.6.html#new-dict-implementation>`_,

由Raymond Hettinger在 `python-dev <https://mail.python.org/pipermail/python-dev/2012-December/123028.html>`_ 提出实现方式

比起3.5, 节省了20%-25%的内存, 并且现在keys返回是有序的，和key插入的顺序一样, 而3.6之前是"无序"的, 其实是hash表顺序.


如何压缩
--------------------

例如:

.. code-block:: python

    d = {'timmy': 'red', 'barry': 'green', 'guido': 'blue'}

老字典的存储形式为:

.. code-block:: python

    entries = [['--', '--', '--'],
               [-8522787127447073495, 'barry', 'green'],
               ['--', '--', '--'],
               ['--', '--', '--'],
               ['--', '--', '--'],
               [-9092791511155847987, 'timmy', 'red'],
               ['--', '--', '--'],
               [-6480567542315338377, 'guido', 'blue']]

新字典的存储形式为:

.. code-block:: python

    indices =  [None, 1, None, None, None, 0, None, 2]
    entries =  [[-9092791511155847987, 'timmy', 'red'],
                [-8522787127447073495, 'barry', 'green'],
                [-6480567542315338377, 'guido', 'blue']]

最大的区别在于: 

1. indices作为新hash表, 只存储entries的下标, 这样indices每一个元素的大小就减少到1字节.

2. entries是一个数组, 是append only的, 这样保证了插入顺序, keys方法只需要遍历entries数组就好了.

3. 查找的时候, 先根据hash值和hash表大小(indices数组)求出indices的下标, 然后同下标去找到entries对应的key/value.

3.6之前dict中, 一个entries就是一个hash表, 然后下标对应存储的是hash值和key, value, 然后存储的空间就很浪费, 64位机器下是hash每一个元素都占24 byte, 所以之前存储

的3个元素的话, 就要花费24 * 8 = 192 byte. 

而3.6的dict则是hash数组是int数组, 元素为4来, 表示entries数组的下标, 而 **entries表是一个插入的时候append only的数组**, 是紧凑的数组, 而花费的空间

为: 8(indices数组) + 24 * 3 = 80, 所以空间大幅度减少了. indices做了些优化, 不是用只用整型来代表entries下标的.

关于indices数组看起来是一个整数数组, 但是其实不是, 具体实现的是使用了共用体结构, 我也没看懂.

排序的区别
-------------

2.7中:

.. code-block:: python

    In [15]: x={'a': 1, 1: 'a'}
    
    In [16]: x
    Out[16]: {1: 'a', 'a': 1}
    
    In [17]: x.keys()
    Out[17]: ['a', 1]

看起来是有序的, 但是其实看看hash值:

.. code-block:: python

    In [18]: hash('a')
    Out[18]: 12416037344
    
    In [19]: hash('a') % 8
    Out[19]: 0
    
    In [20]: hash(1)
    Out[20]: 1
    
    In [21]: hash(1) % 2
    Out[21]: 1

字符串a的hash值在hash表(这里长度是8)的情况下, 模运算出来是0, 而1是1, 所以a会在1之前, 再看看1和2:

.. code-block:: python

    In [23]: x={2: 'b', 1: 'a'}
    
    In [24]: x.keys()
    Out[24]: [1, 2]
    
    In [25]: hash(1) % 8
    Out[25]: 1
    
    In [26]: hash(2) % 8
    Out[26]: 2

1的hash值模运算的结果小于2的结果, 所以keys出来就是1在2之前.

keys的区别在下面的keys函数部分

hash/二次探测
================

hash函数是去调用对象的hash方法, 如果没有定义那就报错咯. 

hash表寻址的时候是一个模运算, hash_value % sizeof(hash_table)

整数的hash值
--------------

整数的hash值就是其本身, 并不是一个random的算法, 整数的hash值就是本身数值, 这样对于一个大小为2**i的hash表, 2**1内的整数都没有冲突, 这样很方便.

可以理解为求整数在hash表的初始位置, 就是其整数的最低i位的值. 例如x=6, hash表大小为4, 那么进行hash表寻址的时候, x地址就是6%4, 也就是:

.. code-block:: 
   
    6  110
    4  100
    -------
    2  010
   
所以也就是最低i比特位的值. 所以这样求hash值在hash表的第一个位置的时候就很快.

*taking the low-order i bits as the initial table index is extremely fast*

如果hash表大小大于hash值, 那么可以用&符号, 比如6 & 32 = 6, 由于python中一定保证hash表大小(mask值)大于

冲突
------

最低i位作为hash值也不好, key的值的最低i比特都一样, 那么所有的key都会被放到同一个位置, 那么冲突就很大!, 比如

key的列表是[i<<2 for i in range(10)], 这个列表的最低两位都是0(明显都是4的倍数), 那么mod 4的时候都是0了.

但是基于这样一个事实: 绝大部分情况下, key的第一个位置就是可用的槽位了(当hash表的大小可用槽位大于2/3的时候).

所以我们继续保持获取首个槽位很快, 然后接下来就是解决碰撞问题了.

下一个槽位
-------------

第一个槽位冲突之后, 下一个槽位如果是直接+1或者-1的实现的话, 也不好, 比如有key为[1, 11, 2, 3, 4], hash表长10, 如果下一个槽位就是

冲突槽位+1的话, 11和1冲突之后, 11将会占据2的槽位, 那么2就必须进行1次额外探测, 3要进行2次额外探测, 4要进行3次额外探测, 失去了第一个位置就是

可用槽位的特点, 每一个元素都需要多次探测才能得到合适的空槽位.

python使用探测下一个槽位的公式是: j + 1 = ((5*j) + 1) mod 2**i, 2**i是hash表大小, hash大小是可以变化的.

然后加一个偏移量来帮助寻址, 整个二次探测的公式为:

.. code-block:: 

    PERTURB_SHIFT = 5
    perturb >>= PERTURB_SHIFT;
    j = (5*j) + 1 + perturb;
    next_j = j % (2**i)

一般perturb赋值为hash值, 并且PERTURB_SHIFT的值为5是一个权衡的结果:

*Selecting a good value for PERTURB_SHIFT is a balancing act.*

dummy状态
------------

一旦删除hash表的元素, 实际上并不会正在删除, 而是把它设置为dummy状态.

这样的好处是, 能够是得探测正常进行, 散列表探测流程是直到探测完所有的槽位或者探测到一个可用槽的时候才会停止.

比如, 假设散列表[1, 2], 1和2有同样的hash值, 2和1冲突之, 2经过二次探测到可用槽位在1的后面:

1. 1这个槽位被删除了, 如果我们直接删除, 设置为空的话, [None, 2]

2. 那么我查找2的时候, 我们首先会遇到None这个槽位, 那么根据二次散列的流程

   这个槽位是空, 则返回, 表示查找不到, 这样就不正确了.

3. 如果我们把这个槽位设置为dummy, 可以使得探测能继续进行下去, 继续探测到2, 探测成功


split/combined
================

dict区分有split和combined两种模式. `pep412 <https://www.python.org/dev/peps/pep-0412/#split-table-dictionaries>`_ 有介绍.

split
--------

pep412的motivation中提到, 之前__dict__是会把类中的属性的名字, 作为key复制到每一个key到对应的实例中的__dict__的,

这样内存有点浪费:

*The current dictionary implementation uses more memory than is necessary when used as a container 
for object attributes as the keys are replicated for each instance rather than being shared across many instances of the same class.*


而在多个实例之间共享key, 也就是共享类定义的属性, 这样会提高内存利用:

*By separating the keys (and hashes) from the values it is possible to share the keys between multiple dictionaries and improve memory use.*


split字典是在获取obj.__dict__属性的时候, 会生成并返回一个split模式的dict.

*When dictionaries are created to fill the __dict__ slot of an object, they are created in split form.*

.. code-block:: python

   class A:
       def __init__(self):
           self.a, self.b, self.c = 1, 2, 3
   a = A()
   d = a.__dict__

此时d就是一个split模式的字典.

split模式的字典下, 添加字符串的key会反射到object中:

.. code-block:: python

   d['new_key'] = 100
   a.new_key == 100

非字符串的key不会反射到object中:

.. code-block:: python

   d[10] = 'new_value'
   # 下面会报语法错误
   a.10


combined
-----------

除了访问obj.__dict__之外, 都是combined模式的字典, pep412:

*Explicit dictionaries (dict() or {}), module dictionaries and most other dictionaries are created as combined-table dictionaries.
A combined-table dictionary never becomes a split-table dictionary.
Combined tables are laid out in much the same way as the tables in the old dictionary, resulting in very similar performance.*


模式互转
-----------

一旦split模式的dict有删除操作, 那么就变成combined, combined的字典会转成split的, 只有访问__dict__属性的时候才会构造split模式.


.. code-block:: python

    class A:
        def __init__(self):
            self.a, self.b, self.c = 1, 2, 3
    a = A()
    m = a.__dict__

此时m就是一个split字典, 然后对其删除操作:


.. code-block:: python

   del m['a']

那么m就变成了一个combined字典

每次新建一个A对象实例, 访问实例__dict__属性都会新建一个split字典, 比如下面的n:

.. code-block:: python

   b = A()
   n = b.__dict__



区别
-----------

两者差别不大, 但是在resize的过程中, C代码有点区别, 简而言之, C代码显示: split模式的字典的size会随着key的增加减少而变大变小, 但是combined模式的字典却不会.

但是缩减的意义不大, 因为split经过删除之后是一个combined的dict, 那就不会收缩了, 所以split其实也不会收缩.

具体看resize部分.

look函数
===========

https://stackoverflow.com/questions/11162201/is-it-always-faster-to-use-string-as-key-in-a-dict

http://lewk.org/blog/python-dictionary-optimizations

look函数的作用是进行二次探测去查找空的槽位~~~

dict中的key是可以是任意对象的, 前提是能hash.

然后对于不同的key类型, dict会调用不同的搜索函数. 最快的是key是字符串的时候, 比key是int的时候还要快.

关键点在于非字符串 **对象** 比较的时候是调用PyObject_RichCompareBool这个函数， 是比较python对象的, 比较对象代价总是比较大的, 要比较类型什么的.

lookdict_unicode
-------------------

当key是字符串类型的时候, 因为字符串比对是不会出现异常的, 所以这个函数就不会去处理很多异常了, 所以快一点.

如果判断key不是字符串, 那么dict的look函数会变成lookdict.

cpython/Objects/dictobject.c

.. code-block:: c

    static Py_ssize_t
    lookdict_unicode(PyDictObject *mp, PyObject *key,
                     Py_hash_t hash, PyObject ***value_addr, Py_ssize_t *hashpos)
    {
        size_t i;
        size_t mask = DK_MASK(mp->ma_keys);
        Py_ssize_t ix, freeslot;
        PyDictKeyEntry *ep, *ep0 = DK_ENTRIES(mp->ma_keys);
    
        assert(mp->ma_values == NULL);
        // 这个if就是去确认key是不是字符串
        if (!PyUnicode_CheckExact(key)) {
            // 如果不是, look函数则设置为lookdict
            mp->ma_keys->dk_lookup = lookdict;
            return lookdict(mp, key, hash, value_addr, hashpos);
        }
    }


lookdict
----------

一般性的比较, 比较对象之

如果lookdict_unicode出现出错, 那么回进入到这个lookdict函数中.

并且这个函数就回不到lookdict_unicode或者lookdict_unicode_nodummy了.

lookdict_unicode_nodummy
----------------------------

这个函数的注释中有这一句话:

*Faster version of lookdict_unicode when it is known that no <dummy> keys will be present.*

这个dummy想了好久, 然后联想到dict中hash表的槽位状态有个叫dummy的, 表示这个槽位被删除过状态.

所以恍然大悟: 如果一个dict一开始的key都是字符串, 并且没有被删过, 那么会调用这个nodummy函数, 只要删除过dict的key, 那么都会回到上面两个函数中!!!

所以:

1. 新建字典, lookup函数就是nodummy

2. 一旦删除了dict的元素(del dict[key], dict.pop等等), 那么lookup函数就会变成上面两个函数之一

.. code-block:: c

    static Py_ssize_t
    lookdict_unicode_nodummy(PyDictObject *mp, PyObject *key,
                             Py_hash_t hash, PyObject ***value_addr,
                             Py_ssize_t *hashpos)
    {
        // 这里的判断和lookdict_unicode的一样
        // 判断key是否是字符串
        if (!PyUnicode_CheckExact(key)) {
            // 不是的话, 把字典的look函数设置为lookdict
            mp->ma_keys->dk_lookup = lookdict;
            return lookdict(mp, key, hash, value_addr, hashpos);
        }
    }

删除的时候替换掉look函数

cpython/Objects/dictobject.c

.. code-block:: c

    static int
    delitem_common(PyDictObject *mp, Py_ssize_t hashpos, Py_ssize_t ix,
                   PyObject **value_addr)
    {
        // 调用一个宏来设置look函数
        // 宏定义在下面
        ENSURE_ALLOWS_DELETIONS(mp);
        old_key = ep->me_key;
        ep->me_key = NULL;
        Py_DECREF(old_key);
        Py_DECREF(old_value);
    
        assert(_PyDict_CheckConsistency(mp));
        return 0;
    }

    // 替换look函数
    #define ENSURE_ALLOWS_DELETIONS(d) \
        if ((d)->ma_keys->dk_lookup == lookdict_unicode_nodummy) { \
            (d)->ma_keys->dk_lookup = lookdict_unicode; \
        }


look函数顺序
---------------

.. code-block:: python

   x = {1: 'a'}

上面的过程是先初始化空字典(下面的1), 然后调用insertdict来赋值1/'a'(下面的2).

1. new_dict, 初始化look函数值是lookdict_unicode_nodummy.

2. insertdict(d, k, v)调用lookdict_unicode_nodummy去搜索是否已经存key, 那么在

   lookdict_unicode_nodummy中会判断key是否是unicode, 如果不是, 那么look函数变为lookdict, **并且以后都是用lookdict了!!!**

3. 接2, 如果lookdict_unicode_nodummy检查key是字符串, 那么就继续.

4. 新建dict之后, 任何赋值的操作, dict[key] = v, 都会调用2中的insertdict, 不是字符串就会变成lookdict了.

5. 新建dict之后, 删除dict中的key, del dict[key]等, 如果删除的key不是字符串, 会调用lookdict_unicode_nodummy
   


keys函数
================

py2中直接遍历hash表:

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

.. code-block:: c

    static PyObject *
    dict_keys(PyDictObject *mp)
    {
        ep = DK_ENTRIES(mp->ma_keys);
        size = mp->ma_keys->dk_nentries;
        for (i = 0, j = 0; i < size; i++) {
            // 值为空表示被删除了
            if (*value_ptr != NULL) {
                PyObject *key = ep[i].me_key;
                Py_INCREF(key);
                PyList_SET_ITEM(v, j, key);
                j++;
            }
            value_ptr = (PyObject **)(((char *)value_ptr) + offset);
        }
    }

可以看到, size是数据数组的大小, 不是hash表的长度, 然后遍历的时候会从ep直接拿key对象的指针, 而ep就是dk_entries, 也就是数据数组, 所以也就是直接遍历数据数组

而不是hash表. 数据数组是insert的时候append only的, 也就是保持了插入的顺序

ipython打印
--------------

但是ipython中你回车出来看到的依然是ascii顺序的:

.. code-block:: python

    In [44]: x={'b': 1, 'a': 2}
    
    In [45]: x
    Out[45]: {'a': 2, 'b': 1}
    
    In [46]: list(x.keys())
    Out[46]: ['b', 'a']

ipython的问题: https://github.com/ipython/ipython/issues?utf8=%E2%9C%93&q=dict


dict内置函数
===============

参考: https://doughellmann.com/blog/2012/11/12/the-performance-impact-of-using-dict-instead-of-in-cpython-2-7-2/


py2中, 调用dict和直接用{}来创建字典相比, dict会更慢一点.


但是在我测试下来:

.. code-block::

    python2 -m timeit -n 1000000 -r 5 -v 'dict()'
    raw times: 0.0955 0.095 0.0958 0.0945 0.0954
    1000000 loops, best of 5: 0.0945 usec per loop

    python2 -m timeit -n 1000000 -r 5 -v '{}'
    raw times: 0.0415 0.0296 0.0293 0.0293 0.0295
    1000000 loops, best of 5: 0.0293 usec per loop

    python3.6 -m timeit -n 1000000 -r 5 -v 'dict()'
    raw times: 0.144 0.138 0.138 0.14 0.153
    1000000 loops, best of 5: 0.138 usec per loop

    python3.6 -m timeit -n 1000000 -r 5 -v '{}'
    raw times: 0.0348 0.0352 0.0368 0.0348 0.0352
    1000000 loops, best of 5: 0.0348 usec per loop

py2中, dict确实比{}慢一点, 但是py3中, dict却比{}快了挺多的.

但是调用dict的过程, py3和py2是一样的:

1. 先调用dict_new(tp_new)生成一个key初始长度的空字典

2. 调用dict_init(tp_init)去merge从dict中传入的参数字典

3. merge的过程是在PyDict_Merge中调用dict_merge处理的.


结果上的不一致也没太明白



----



PyDictObject
================

这个结构就表示了一个字典, ma_keys和ma_values则是分别存放key和value的地方, 但是

对于小字典的话, 会把key和value都存在ma_keys中, 大字典就把key放在ma_keys中, ma_values放的是value

.. code-block:: c

   typedef struct _dictkeysobject PyDictKeysObject;

    typedef struct {
        PyObject_HEAD
    
        // 字典中的元素实际个数
        Py_ssize_t ma_used;
    
        // 字典的版本, 一旦字典被改变, 版本也会改变
        // pep509
        uint64_t ma_version_tag;
    
        //  一个dictkeys对象
        PyDictKeysObject *ma_keys;
    
        // split模式下的value数组
        PyObject **ma_values;
    } PyDictObject;

PyDictKeysObject最终对应于_dictkeysobject, 下面是主要的字段:

cpython/Objects/dict-common.h

.. code-block:: c

    struct _dictkeysobject {

        // hash表的大小, 大小必须是2的n次方
        Py_ssize_t dk_size;
    
        // 搜索的函数, 解释在下面搜索函数部分
        dict_lookup_func dk_lookup;
    
        // 可用槽位置, dk_size * 2/3
        Py_ssize_t dk_usable;
    
        Py_ssize_t dk_nentries;
    
        // 下面是一个8字节的公用体
        // 用来作为indices数组
        union {
            int8_t as_1[8];
            int16_t as_2[4];
            int32_t as_4[2];
    #if SIZEOF_VOID_P > 4
            int64_t as_8[1];
    #endif
        } dk_indices;
    
    };
 
几个长度
----------

1. ma_used是dict的实际长度, 也就是元素的个数, 每次insert/del都是加减.

2. dk_nentries是entries数组的当前插入的下标, 插入完成之后自增1. 当插入的时候会一直增加, 删除的时候不变.

3. dk_usable = size_hash * 2/ 3 - maused.
   
4. 之所以dk_nentries是只增不减, 是因为这个值是保证key是插入顺序.


关于dict的空间变化, 在下面的resize部分.


新建字典
===========

使用{}新建字典的字节码是BUILD_MAP, 流程是先创建一个空字典, 然后一个个setitem

.. code-block:: c

    TARGET(BUILD_MAP) {
        Py_ssize_t i;
        // 初始化空字典
        PyObject *map = _PyDict_NewPresized((Py_ssize_t)oparg);
        if (map == NULL)
            goto error;
        // for循环setitem
        for (i = oparg; i > 0; i--) {
            int err;
            PyObject *key = PEEK(2*i);
            PyObject *value = PEEK(2*i - 1);
            err = PyDict_SetItem(map, key, value);
            if (err != 0) {
                Py_DECREF(map);
                goto error;
            }
        }

_PyDict_NewPresized
=====================

.. code-block:: c

    PyObject *
    _PyDict_NewPresized(Py_ssize_t minused)
    {
        const Py_ssize_t max_presize = 128 * 1024;
        Py_ssize_t newsize;
        PyDictKeysObject *new_keys;
    
        // 计算dict的大小
        // 只能预分配最大长度
        if (minused > USABLE_FRACTION(max_presize)) {
            newsize = max_presize;
        }
        else {
            // 没有超过最大预分配长度, 则计算size
            // 要满足newsize=2**n, 并且newsize*2/3 > minsize
            Py_ssize_t minsize = ESTIMATE_SIZE(minused);
            newsize = PyDict_MINSIZE;
            while (newsize < minsize) {
                newsize <<= 1;
            }
        }
        assert(IS_POWER_OF_2(newsize));
    
        // 新建keys对象
        new_keys = new_keys_object(newsize);
        if (new_keys == NULL)
            return NULL;
        // 新建dict
        return new_dict(new_keys, NULL);
    }


_PyDict_NewPresized会根据size, 新建一个keys对象, 然后调用new_dict去新建一个dict, 

new_keys_object
==================

这个函数负责初始化keys和values

.. code-block:: c

    static PyDictKeysObject *new_keys_object(Py_ssize_t size)
    {
        PyDictKeysObject *dk;
        Py_ssize_t es, usable;
    
        // 下面是校验长度和可用个数
        assert(size >= PyDict_MINSIZE);
        assert(IS_POWER_OF_2(size));
    
        usable = USABLE_FRACTION(size);
        // 省略了代码
    
        // 小字典从free_list拿出来
        if (size == PyDict_MINSIZE && numfreekeys > 0) {
            dk = keys_free_list[--numfreekeys];
        }
        else {
            // 大字典嘛, 分配一个
            dk = PyObject_MALLOC(sizeof(PyDictKeysObject)
                                 - Py_MEMBER_SIZE(PyDictKeysObject, dk_indices)
                                 + es * size
                                 + sizeof(PyDictKeyEntry) * usable);
            if (dk == NULL) {
                PyErr_NoMemory();
                return NULL;
            }
        }
        // 下面就是初始化了
        DK_DEBUG_INCREF dk->dk_refcnt = 1;
        dk->dk_size = size;
        dk->dk_usable = usable;
        dk->dk_lookup = lookdict_unicode_nodummy;
        dk->dk_nentries = 0;
        // 初始化hash表, 意思懂了, 但是如何映射的嘛, 没看懂
        memset(&dk->dk_indices.as_1[0], 0xff, es * size);
        PyDictKeyEntry *tmp = DK_ENTRIES(dk);
        // 这里初始化PyDictKeyEntry的数组， 意思看懂了, 但是细节没看懂
        // DK_ENTRIES这个宏有点难看懂
        memset(DK_ENTRIES(dk), 0, sizeof(PyDictKeyEntry) * usable);
        return dk;
    }

这个函数只是根据长度, 分配一个空的PyDictObject, 然后初始化look函数.

1. 长度有两个, 一个hash表的长度, 也就是要保持2的n次方, 值是保存在dk_size里面, 但是其对应的hash表, 也就是indices变量,

   只是一个8字节的固定长度的共用体而已, 如何映射, 没看懂(c语言比较渣), 但是按照上面的设计思路, 把indices当做一个

   数组就好了.
   
2. 一个是PyDictKeyEntry数组的长度, 会预分配, 使用的是指针移动的方式去赋值, 长度为hash表长度的2/3.

3. look函数默认初始化为lookdict_unicode_nodummy.


new_dict
==========

这里只是把keys对象赋值到dict对象中而已

.. code-block:: c

    static PyObject *
    new_dict(PyDictKeysObject *keys, PyObject **values)
    {
        PyDictObject *mp;
        assert(keys != NULL);
        if (numfree) {
            // 从free_list中拿一个
            mp = free_list[--numfree];
            assert (mp != NULL);
            assert (Py_TYPE(mp) == &PyDict_Type);
            _Py_NewReference((PyObject *)mp);
        }
        else {
            // 否则从内存中新分配一个dict
            mp = PyObject_GC_New(PyDictObject, &PyDict_Type);
            if (mp == NULL) {
                DK_DECREF(keys);
                free_values(values);
                return NULL;
            }
        }
        // 赋值传入的keys对象
        mp->ma_keys = keys;
        mp->ma_values = values;
        mp->ma_used = 0;
        mp->ma_version_tag = DICT_NEXT_VERSION();
        assert(_PyDict_CheckConsistency(mp));
        return (PyObject *)mp;
    }

lookdict
============

这个函数是一般性的搜索函数, 注意的是, **该只会返回一个空槽位的标志, 但是hashops可能是dummy槽位的**


.. code-block:: c

    static Py_ssize_t
    lookdict(PyDictObject *mp, PyObject *key,
             Py_hash_t hash, PyObject ***value_addr, Py_ssize_t *hashpos)
    {
        size_t i, mask;
        Py_ssize_t ix, freeslot;
        int cmp;
        PyDictKeysObject *dk;
        PyDictKeyEntry *ep0, *ep;
        PyObject *startkey;
    
    top:
        // 拿到初始数据
        dk = mp->ma_keys;
        mask = DK_MASK(dk);
        ep0 = DK_ENTRIES(dk);

        // 拿到第一个hash表槽位
        i = (size_t)hash & mask;
    
        // 拿到槽位中对应的下标
        ix = dk_get_index(dk, i);

        // 下标是空的, 那么返回
        if (ix == DKIX_EMPTY) {
            if (hashpos != NULL)
                *hashpos = i;
            *value_addr = NULL;
            return DKIX_EMPTY;
        }

        // 如果是dummy的, 记住它, 然后继续下面!!!
        if (ix == DKIX_DUMMY) {
            freeslot = i;
        }
        else {

            // 如果不是空也不是dummy, 说明hash一样
            ep = &ep0[ix];
            assert(ep->me_key != NULL);

            // 如果key一样, 说明是一个元素, 返回
            if (ep->me_key == key) {
                *value_addr = &ep->me_value;
                if (hashpos != NULL)
                    *hashpos = i;
                return ix;
            }
            // 如果hash值一样, 则继续比较
            if (ep->me_hash == hash) {
                startkey = ep->me_key;
                Py_INCREF(startkey);

                // 调用对象比较函数
                cmp = PyObject_RichCompareBool(startkey, key, Py_EQ);
                Py_DECREF(startkey);

                //下面比较起来小于0, 说明同一个hash不同的对象, 错误
                if (cmp < 0) {
                    *value_addr = NULL;
                    return DKIX_ERROR;
                }
                // 比较值大于0, 说明是likely的, 返回
                if (dk == mp->ma_keys && ep->me_key == startkey) {
                    if (cmp > 0) {
                        *value_addr = &ep->me_value;
                        if (hashpos != NULL)
                            *hashpos = i;
                        return ix;
                    }
                }
                else {
                    /* The dict was mutated, restart */
                    goto top;
                }
            }
            // freeslot表示这个槽位被占据了
            freeslot = -1;
        }
    
        // 上面找到的是dummy
        // 下面继续开放地址法
        for (size_t perturb = hash;;) {
            perturb >>= PERTURB_SHIFT;
            i = ((i << 2) + i + perturb + 1) & mask;
            ix = dk_get_index(dk, i);

            // 找到空槽位, 返回
            if (ix == DKIX_EMPTY) {
                // 这里hashops声明为py_size_t, 当然不是NULL
                if (hashpos != NULL) {
                    // 这里, hashpos会优先获取free_slot的位置, 也就是
                    // 如果之前发现了dummy的, 就优先拿dummy位置
                    // 如果之前找到的是一个被占据的槽位, 那么这里就返回当前的空槽位置
                    *hashpos = (freeslot == -1) ? (Py_ssize_t)i : freeslot;
                }
                *value_addr = NULL;
                return ix;
            }

            // 依然是dummy的, 继续
            // 这里如果freeslot是-1的话, 说明之前的位置不是dummy
            // 因为我们要记住第一个dummy位置, 所以这里判断下
            // 如果freeslot不是-1, 那么之前肯定找到了一个dummy槽位
            // 如果不是, 那么现在找到的dummy就是第一个
            if (ix == DKIX_DUMMY) {
                if (freeslot == -1)
                    freeslot = i;
                continue;
            }

            // 下面继续之前的比较
            ep = &ep0[ix];
            assert(ep->me_key != NULL);
            if (ep->me_key == key) {
                if (hashpos != NULL) {
                    *hashpos = i;
                }
                *value_addr = &ep->me_value;
                return ix;
            }
            if (ep->me_hash == hash) {
                startkey = ep->me_key;
                Py_INCREF(startkey);
                cmp = PyObject_RichCompareBool(startkey, key, Py_EQ);
                Py_DECREF(startkey);
                if (cmp < 0) {
                    *value_addr = NULL;
                    return DKIX_ERROR;
                }
                if (dk == mp->ma_keys && ep->me_key == startkey) {
                    if (cmp > 0) {
                        if (hashpos != NULL) {
                            *hashpos = i;
                        }
                        *value_addr = &ep->me_value;
                        return ix;
                    }
                }
                else {
                    /* The dict was mutated, restart */
                    goto top;
                }
            }
        }
        assert(0);          /* NOT REACHED */
        return 0;
    }

1. 搜索函数一定会返回empty或者error

2. 但是返回的hashops会优先返回找到的第一个free_slot, 也就是dummy的槽位, 而freeslot=-1, 说明之前找到的槽位是被占据的

3. 代码中槽的位置是保存在hashops中, 并且声明的时候是Py_ssize_t, 所以初始值是0, 不是NULL, 所以代码中很多if hashops == NULL这个判断感觉有点~~~怪怪    的

 
insertdict
=============

PyDict_SetItem将会调用insertdict去将key/value插入到keys对象中.

每次向dict插入key/value的时候, 比如调用dict[key] = value, 调用该函数:


.. code-block:: c

    static int
    insertdict(PyDictObject *mp, PyObject *key, Py_hash_t hash, PyObject *value)
    {
        PyDictKeyEntry *ep, *ep0;
        Py_ssize_t hashpos, ix;
    
        Py_INCREF(key);
        Py_INCREF(value);

        // 这里判断split模式的dict插入一个非字符串的key
        if (mp->ma_values != NULL && !PyUnicode_CheckExact(key)) {
            if (insertion_resize(mp) < 0)
                goto Fail;
        }

        // 调用look函数
        // ix一定会返回empty或者error
        // 但是hashpos有可能是dummy的位置
        ix = mp->ma_keys->dk_lookup(mp, key, hash, &value_addr, &hashpos);
        if (ix == DKIX_ERROR)
            goto Fail;
    
        // 这里的条件是如果是split模式的dict, 并且
        // mp-ma_used和mp->ma_keys不相等的时候需要resize
        // 两者不相等的时候是有可能大字典变为小字典, 导致不相等的
        if (_PyDict_HasSplitTable(mp) &&
           ((ix >= 0 && *value_addr == NULL && mp->ma_used != ix) ||
            (ix == DKIX_EMPTY && mp->ma_used != mp->ma_keys->dk_nentries))) {
           if (insertion_resize(mp) < 0)
               goto Fail;
           find_empty_slot(mp, key, hash, &value_addr, &hashpos);
           ix = DKIX_EMPTY;
        }
    
        // 这个if是如果搜索出来的槽位可用, 那么插入
        if (ix == DKIX_EMPTY) {
            // 但是没有可用个数了
            if (mp->ma_keys->dk_usable <= 0) {
                // 所以需要resize
                if (insertion_resize(mp) < 0)
                    goto Fail;

                // 新hash的新下标, hashpos会变
                find_empty_slot(mp, key, hash, &value_addr, &hashpos);
            }
            // ep0是PyDictKeyEntry的数组
            ep0 = DK_ENTRIES(mp->ma_keys);

            // 插入key数组
            ep = &ep0[mp->ma_keys->dk_nentries];

            // 设置hash对应位置为key数组下标
            dk_set_index(mp->ma_keys, hashpos, mp->ma_keys->dk_nentries);

            // 更新value对应的PyDictKeyEntry对象
            ep->me_key = key;
            ep->me_hash = hash;
            // 如果是split模式, value应该赋值到values数组
            if (mp->ma_values) {
                assert (mp->ma_values[mp->ma_keys->dk_nentries] == NULL);
                mp->ma_values[mp->ma_keys->dk_nentries] = value;
            }
            else {
                ep->me_value = value;
            }
            // 各种更新PyDictObject对象
            mp->ma_used++;
            mp->ma_version_tag = DICT_NEXT_VERSION();

            // 插入的时候, 不管是empty还是dummy的槽位置
            // 这里dk_usable都会减1
            mp->ma_keys->dk_usable--;
            mp->ma_keys->dk_nentries++;
            assert(mp->ma_keys->dk_usable >= 0);
            assert(_PyDict_CheckConsistency(mp));
            return 0;
        }

        // 下面有些代码, 没太看懂, 省略
    
    }

1. 插入的时候, 不管是empty还是dummy的槽位置, dk_usable都会减1

append only
-------------------

插入key的时候, dk_entries总是下一个连续下标, 所以:

.. code-block:: c

    ep = &ep0[mp->ma_keys->dk_nentries]
    // 下面是ep的赋值
  
就是key数组的append操作, 并且每次插入都是append only.


resize之后重新获得hash
------------------------

第一次拿到hashpos:

.. code-block:: c

    ix = mp->ma_keys->dk_lookup(mp, key, hash, &value_addr, &hashpos);

然后发现resize, 那么需要重新求hashpos

.. code-block:: c

    if (mp->ma_keys->dk_usable <= 0) {
        // 所以需要resize
        if (insertion_resize(mp) < 0)
            goto Fail;
    
        // 新hash的新下标, hashpos会变
        find_empty_slot(mp, key, hash, &value_addr, &hashpos);
    }



resize过程
=============

1. resize只会在插入的时候发生

2. 一旦dict的元素个数大于hash表的2/3的时候, 需要重新分配一个更大的key数组.

3. resize的要求是新长度至少大于GROWTH_RATE: 2*used + size / 2

4. resize只会返回的PyDictKeysObject一定是一个combined的.

.. code-block:: c 

    static int
    insertion_resize(PyDictObject *mp)
    {
        // 插入的时候设置最小长度
        return dictresize(mp, GROWTH_RATE(mp));
    }


    static int
    dictresize(PyDictObject *mp, Py_ssize_t minsize)
    {
        // 这里设置新长度, 新长度的是2**n次方, 并且大于最小长度
        /* Find the smallest table size > minused. */
        for (newsize = PyDict_MINSIZE;
             newsize < minsize && newsize > 0;
             newsize <<= 1)
            ;

        // 这里赋值一份老的keys和values
        oldkeys = mp->ma_keys;
        oldvalues = mp->ma_values;
        /* Allocate a new table. */
        // 分配新的keys和values
        mp->ma_keys = new_keys_object(newsize);
        if (mp->ma_keys == NULL) {
            mp->ma_keys = oldkeys;
            return -1;
        }
        // New table must be large enough.
        // 再次校验下长度, 并且顺手设置下look函数
        assert(mp->ma_keys->dk_usable >= mp->ma_used);
        if (oldkeys->dk_lookup == lookdict)
            mp->ma_keys->dk_lookup = lookdict;
        // resize的是, 返回的永远是一个combined模式的dict
        // 所以这里的mp->ma_values设置为NULL
        mp->ma_values = NULL;
        ep0 = DK_ENTRIES(oldkeys);
        // 这里如果老values有值, 表示是一个split表
        // 但是为了方便, 把老的values也设置到老的keys里面
        // 这样接下来复制两个数组的循环就只需要考虑老keys就好了
        if (oldvalues != NULL) {
            for (i = 0; i < oldkeys->dk_nentries; i++) {
                if (oldvalues[i] != NULL) {
                    Py_INCREF(ep0[i].me_key);
                    ep0[i].me_value = oldvalues[i];
                }
            }
        }
        /* Main loop */
        // 把老keys的数据复制到新的keys中
        // 遍历keys数组
        for (i = 0; i < oldkeys->dk_nentries; i++) {
            PyDictKeyEntry *ep = &ep0[i];
            // 只复制非NULL数据
            if (ep->me_value != NULL) {
                insertdict_clean(mp, ep->me_key, ep->me_hash, ep->me_value);
            }
        }
        // 更新已用的个数, 比如之前是8, 用了5个之后触发resize
        // 新表长度是16, 那么可用个数是10, 然后我们复制5个数据进去之后
        // 明显可用个数就要减5了
        mp->ma_keys->dk_usable -= mp->ma_used;
        if (oldvalues != NULL) {
            /* NULL out me_value slot in oldkeys, in case it was shared */
            for (i = 0; i < oldkeys->dk_nentries; i++)
                ep0[i].me_value = NULL;
            DK_DECREF(oldkeys);
            if (oldvalues != empty_values) {
                free_values(oldvalues);
            }
        }
        else {
            assert(oldkeys->dk_lookup != lookdict_split);
            assert(oldkeys->dk_refcnt == 1);
            DK_DEBUG_DECREF PyObject_FREE(oldkeys);
        }
        return 0;
    }

insertdict_clean
===================

这个函数是resize的时候, 设置hash表下标和更新dk_nentries的.

.. code-block:: c

    static void
    insertdict_clean(PyDictObject *mp, PyObject *key, Py_hash_t hash,
                     PyObject *value)
    {
        size_t i;
        PyDictKeysObject *k = mp->ma_keys;
        size_t mask = (size_t)DK_SIZE(k)-1;
        PyDictKeyEntry *ep0 = DK_ENTRIES(mp->ma_keys);
        PyDictKeyEntry *ep;
    
        assert(k->dk_lookup != NULL);
        assert(value != NULL);
        assert(key != NULL);
        assert(PyUnicode_CheckExact(key) || k->dk_lookup == lookdict);

        // i是hash表的下标, 这里&mask就是模运算了
        i = hash & mask;
        // 看看需不需要探测
        for (size_t perturb = hash; dk_get_index(k, i) != DKIX_EMPTY;) {
            perturb >>= PERTURB_SHIFT;
            i = mask & ((i << 2) + i + perturb + 1);
        }

        // append到keys数组
        ep = &ep0[k->dk_nentries];
        assert(ep->me_value == NULL);
        dk_set_index(k, i, k->dk_nentries);

        // 插入下标自增1
        k->dk_nentries++;
        ep->me_key = key;
        ep->me_hash = hash;
        ep->me_value = value;
    }


所以, insert的时候, 对于keys数组和hash表:

1. ma_used自增1

2. dk_usable减少1, dk_nentries自增1

pop
============

.. code-block:: c

    /* Internal version of dict.pop(). */
    PyObject *
    _PyDict_Pop_KnownHash(PyObject *dict, PyObject *key, Py_hash_t hash, PyObject *deflt)
    {
        mp = (PyDictObject *)dict;
    
        // 这里如果dict是空的, 那么报错
        if (mp->ma_used == 0) {
            if (deflt) {
                Py_INCREF(deflt);
                return deflt;
            }
            _PyErr_SetKeyError(key);
            return NULL;
        }
        // 查询要删除的key的hash下标, hash表下标会赋值到hashpos
        ix = (mp->ma_keys->dk_lookup)(mp, key, hash, &value_addr, &hashpos);
        if (ix == DKIX_ERROR)
            return NULL;
        // key不存在, 报错
        if (ix == DKIX_EMPTY || *value_addr == NULL) {
            if (deflt) {
                Py_INCREF(deflt);
                return deflt;
            }
            _PyErr_SetKeyError(key);
            return NULL;
        }
    
        // Split table doesn't allow deletion.  Combine it.
        // split模式的字典不能删除, resize成一个新的combined的字典
        // 这里会触发resize, 返回一个combined字典
        if (_PyDict_HasSplitTable(mp)) {
            if (dictresize(mp, DK_SIZE(mp->ma_keys))) {
                return NULL;
            }
            ix = (mp->ma_keys->dk_lookup)(mp, key, hash, &value_addr, &hashpos);
            assert(ix >= 0);
        }
        // 下面都是一些置空和减少操作

        old_value = *value_addr;
        assert(old_value != NULL);
        *value_addr = NULL;

        // ma_used自减1
        mp->ma_used--;
        mp->ma_version_tag = DICT_NEXT_VERSION();

        // 设置hash表对应槽位为dummy
        dk_set_index(mp->ma_keys, hashpos, DKIX_DUMMY);

        // 拿到ep对象
        ep = &DK_ENTRIES(mp->ma_keys)[ix];
        // 这个是设置look函数的
        ENSURE_ALLOWS_DELETIONS(mp);

        old_key = ep->me_key;
        // 也就是key数组的value设NULL, 表示删除
        ep->me_key = NULL;
        Py_DECREF(old_key);

        assert(_PyDict_CheckConsistency(mp));
        return old_value;
    
    }

删除操作
===========

调用del删除key的时候

.. code-block:: c

    int
    _PyDict_DelItem_KnownHash(PyObject *op, PyObject *key, Py_hash_t hash)
    {
        mp = (PyDictObject *)op;
        // 查询
        ix = (mp->ma_keys->dk_lookup)(mp, key, hash, &value_addr, &hashpos);
        if (ix == DKIX_ERROR)
            return -1;
        // key不存在, 报错
        if (ix == DKIX_EMPTY || *value_addr == NULL) {
            _PyErr_SetKeyError(key);
            return -1;
        }
        assert(dk_get_index(mp->ma_keys, hashpos) == ix);
    
        // Split table doesn't allow deletion.  Combine it.
        // 跟pop一样, split变成combined
        if (_PyDict_HasSplitTable(mp)) {
            if (dictresize(mp, DK_SIZE(mp->ma_keys))) {
                return -1;
            }
            ix = (mp->ma_keys->dk_lookup)(mp, key, hash, &value_addr, &hashpos);
            assert(ix >= 0);
        }
        // 里面都是一些置空和减少的操作
        return delitem_common(mp, hashpos, ix, value_addr);
    }
   
delitem_common
===============

这里是del操作的时候的置空操作

.. code-block:: c

    static int
    delitem_common(PyDictObject *mp, Py_ssize_t hashpos, Py_ssize_t ix,
                   PyObject **value_addr)
    {
        PyObject *old_key, *old_value;
        PyDictKeyEntry *ep;
    
        old_value = *value_addr;
        assert(old_value != NULL);
        *value_addr = NULL;

        // ma_used自减1
        mp->ma_used--;
        mp->ma_version_tag = DICT_NEXT_VERSION();
        ep = &DK_ENTRIES(mp->ma_keys)[ix];

        // 设置dummy
        dk_set_index(mp->ma_keys, hashpos, DKIX_DUMMY);
        ENSURE_ALLOWS_DELETIONS(mp);
        old_key = ep->me_key;

        // 设置key数组槽位为NULL表示删除
        ep->me_key = NULL;
        Py_DECREF(old_key);
        Py_DECREF(old_value);
    
        assert(_PyDict_CheckConsistency(mp));
        return 0;
    }


删除操作
=============

pop和del操作对key数组有同样的操作:


1. ma_used自减1, 然后dk_nentries不变, dk_usable不变

2. 设置hash表槽位为dummy

3. 设置keys数组对应槽位为NULL, 表示删除


