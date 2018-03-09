参考
====

1. http://www.wklken.me/posts/2015/08/29/python-source-memory-1.html

2. 深入python编程: 1.6内存分配

3. http://deeplearning.net/software/theano/tutorial/python-memory-management.html

4. http://www.evanjones.ca/memoryallocator/

5. https://docs.python.org/3/c-api/memory.html

python中的pool/arena的内存策略很想linux内核的slab机制

C语言中malloc/calloc
=======================

malloc从堆中分配出来的是未初始化的内存, 需要用memset来初始化获得的内存块

calloc从堆中分配出来的是已初始化的内存, 分配出来的内存块全部被初始化为0

参考: http://www.cnblogs.com/52php/p/5794342.html


整体结构
=========

cpython/Objects/obmalloc.c

.. code-block:: c

    /* An object allocator for Python.
        Object-specific allocators
        _____   ______   ______       ________
       [ int ] [ dict ] [ list ] ... [ string ]       Python core         |
    +3 | <----- Object-specific memory -----> | <-- Non-object memory --> |
        _______________________________       |                           |
       [   Python's object allocator   ]      |                           |
    +2 | ####### Object memory ####### | <------ Internal buffers ------> |
        ______________________________________________________________    |
       [          Python's raw memory allocator (PyMem_ API)          ]   |
    +1 | <----- Python memory (under PyMem manager's control) ------> |   |
        __________________________________________________________________
       [    Underlying general-purpose allocator (ex: C library malloc)   ]
     0 | <------ Virtual memory allocated for the python process -------> |
    
       =========================================================================
        _______________________________________________________________________
       [                OS-specific Virtual Memory Manager (VMM)               ]
    -1 | <--- Kernel dynamic storage allocation & management (page-based) ---> |
        __________________________________   __________________________________
       [                                  ] [                                  ]
    -2 | <-- Physical memory: ROM/RAM --> | | <-- Secondary storage (swap) --> |
    
    */

1. +3层是每一个对象自己的缓存, 也就是对象的free_list了, 每一种类型对象自己管

2. +2层是python对小对象的缓存池, 小对象是大小小于512 byte的, 每次分配回收都会优先经过这个缓冲池中

3. +1层是python自己封装的, 和os无关的分配接口.
   
   提供统一的raw memory管理接口, 封装的原因: 不同操作系统 C 行为不一定一致, 保证可移植性, 相同语义相同行为(摘抄自参考1)


+3层: free_list
=================

+3层主要是每一类对象会在对象的demalloc中, 针对小对象进行缓存.

用tuple对象做个例子:

cpython/Objects/tupleobject.c

.. code-block:: c

    /* Speed optimization to avoid frequent malloc/free of small tuples */
    #ifndef PyTuple_MAXSAVESIZE
    #define PyTuple_MAXSAVESIZE     20  /* Largest tuple to save on free list */
    #endif
    #ifndef PyTuple_MAXFREELIST
    #define PyTuple_MAXFREELIST  2000  /* Maximum number of tuples of each size to save */
    #endif

    static PyTupleObject *free_list[PyTuple_MAXSAVESIZE];

python中内存管理的一个机制, 对于小对象是放在缓存中而不是还给os, 这样避免小对象经常的分配回收内存.

free_list就是小对象的空闲数组, 然后PyTuple_MAXSAVESIZE是表示小于某个大小的tuple才会放入到free_list中, 而PyTuple_MAXFREELIST是free_list的长度

如果free_list是满的, 那么直接从+1层分配一个.

long, unicode, dict/list/tuple/set等等数据结构都有自己的free_list, 或者自己的缓存池.


+2层内存池
===============

+2层是python中主要的小对象内存池, 这里的小对象大小是指小于512 byte的对象.

每一个小对象的分配和回收都会经过这个内存池, 然后决定是否是真正的从os拿到内存或者把内存返回os.

3.6之前, 对象的分配默认并不是走pool机制, 3.6之后, 默认就是走pool机制.

*Changed in version 3.6: The default allocator is now pymalloc instead of system malloc().*

--- 参考5: memory-interface

cpython/PC/pyconfig.h

.. code-block:: c

    /* Use Python's own small-block memory-allocator. */
    #define WITH_PYMALLOC 1


关于pymalloc:

*Python has a pymalloc allocator optimized for small objects (smaller or equal to 512 bytes) with a short lifetime.

It uses memory mappings called “arenas” with a fixed size of 256 KB. It falls back to PyMem_RawMalloc() and PyMem_RawRealloc() for allocations larger than 512 bytes.*

-- 参考5: The pymalloc allocator



分配策略(pool)
================


.. code-block:: c

    /*
     * Allocation strategy abstract:
     *
     * For small requests, the allocator sub-allocates <Big> blocks of memory.
     * Requests greater than SMALL_REQUEST_THRESHOLD bytes are routed to the
     * system's allocator.
     *
     * Small requests are grouped in size classes spaced 8 bytes apart, due
     * to the required valid alignment of the returned address. Requests of
     * a particular size are serviced from memory pools of 4K (one VMM page).
     * Pools are fragmented on demand and contain free lists of blocks of one
     * particular size class. In other words, there is a fixed-size allocator
     * for each size class. Free pools are shared by the different allocators
     * thus minimizing the space reserved for a particular size class.
     *
     * This allocation strategy is a variant of what is known as "simple
     * segregated storage based on array of free lists". The main drawback of
     * simple segregated storage is that we might end up with lot of reserved
     * memory for the different free lists, which degenerate in time. To avoid
     * this, we partition each free list in pools and we share dynamically the
     * reserved space between all free lists. This technique is quite efficient
     * for memory intensive programs which allocate mainly small-sized blocks.
     *
     * For small requests we have the following table:
     *
     * Request in bytes     Size of allocated block      Size class idx
     * ----------------------------------------------------------------
     *        1-8                     8                       0
     *        9-16                   16                       1
     *       17-24                   24                       2
     *       25-32                   32                       3
     *       33-40                   40                       4
     *       41-48                   48                       5
     *       49-56                   56                       6
     *       57-64                   64                       7
     *       65-72                   72                       8
     *        ...                   ...                     ...
     *      497-504                 504                      62
     *      505-512                 512                      63
     *
     *      0, SMALL_REQUEST_THRESHOLD + 1 and up: routed to the underlying
     *      allocator.
     */

1. 需要分配大小大于小对象大小(512 byte)的对象将会直接去调用os的malloc去分配.

2. 分配的单位是block, 一个block是8 byte, 是为了内存对齐.

3. pool是一组连续内存(4k), 可以看成是数组了. pool的大小是4K, 每一个pool分配的空间是固定的, 根据pool的size class idx, 每一个pool划分的单位大小不一样.
   
   比如idx=3的pool划出的单位空间是32字节, 那么一个28字节的对象为了内存对齐, 则需要划出32字节, 也就是由idx为3的pool划分.

对齐设置
------------

block的大小

.. code-block:: c

    #define ALIGNMENT               8               /* must be 2^N */
    #define ALIGNMENT_SHIFT         3

对象对应的poll
-----------------

计算对象应该从哪个pool操作.

.. code-block:: c

    #define INDEX2SIZE(I) (((uint)(I) + 1) << ALIGNMENT_SHIFT)

比如I=28, 那么(unint)(I) = 28 / 8 = 3, 所以最终: (3+1)<<3 = 32


小对象大小
------------

设置为512字节

.. code-block:: c

    #define SMALL_REQUEST_THRESHOLD 512
    #define NB_SMALL_SIZE_CLASSES   (SMALL_REQUEST_THRESHOLD / ALIGNMENT)

pool大小
------------

4K

.. code-block:: c

    #define SYSTEM_PAGE_SIZE        (4 * 1024)
    #define SYSTEM_PAGE_SIZE_MASK   (SYSTEM_PAGE_SIZE - 1)

    /*
     * Size of the pools used for small blocks. Should be a power of 2,
     * between 1K and SYSTEM_PAGE_SIZE, that is: 1k, 2k, 4k.
     */
    #define POOL_SIZE               SYSTEM_PAGE_SIZE        /* must be 2^N */
    #define POOL_SIZE_MASK          SYSTEM_PAGE_SIZE_MASK


整个内存池大小
-----------------

整个小对象内存池的带下默认设置为64M

.. code-block:: c

    #ifndef SMALL_MEMORY_LIMIT
    #define SMALL_MEMORY_LIMIT      (64 * 1024 * 1024)      /* 64 MB -- more? */


arena
=============

每一批编号0-63的pool组成一个arena, 每次分配的时候从可用的arena中拿到可用的pool去分配.

也就是每一个pool有4k, 64个pool有4 * 64 = 256, 所以每一个arena就是256KB了.

当然, arena的大小和pool的大小定义并没有关系, 上面推论是说明arena和pool的关系.

.. code-block:: c

    #define ARENA_SIZE              (256 << 10)     /* 256KB */
    
    #ifdef WITH_MEMORY_LIMITS
    #define MAX_ARENAS              (SMALL_MEMORY_LIMIT / ARENA_SIZE)
    #endif

初始化多少个arena?

.. code-block:: c

    /* How many arena_objects do we initially allocate?
     * 16 = can allocate 16 arenas = 16 * ARENA_SIZE = 4MB before growing the
     * `arenas` vector.
     */
    #define INITIAL_ARENA_OBJECTS 16

分配流程
==========

分配的时候都是通过pool来操作, arena只是pool的管理.

DRY原则, 看参考1, 很详细


回收内存
============

1. arena中所有pool都是闲置的(empty), 将arena内存释放, 返回给操作系统

2. 如果arena中之前所有的pool都是占用的(used), 现在释放了一个pool(empty), 需要将 arena加入到usable_arenas, 会加入链表表头

3. 如果arena中empty的pool个数n, 则从useable_arenas开始寻找可以插入的位置. 将arena插入. (useable_arenas是一个有序链表, 按empty pool的个数, 保证empty pool数量越多, 被使用的几率越小, 最终被整体释放的机会越大)

4. 其他情况, 不对arena 进行处理

上面摘抄自参考1, 详细请看参考1.

