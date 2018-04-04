参考
====

.. [1] http://www.wklken.me/posts/2015/08/29/python-source-memory-1.html, 有两章

.. [2] 深入python编程: 1.6内存分配
 
.. [3] http://deeplearning.net/software/theano/tutorial/python-memory-management.html
 
.. [4] http://www.evanjones.ca/memoryallocator/

.. [5] https://docs.python.org/3/c-api/memory.html


主要参考字参考 [1]_, 先看参考 [1]_, 再看下面

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


流程小结
===========

分配流程:

1. 从可用的pool(usedpools)列表中拿到指定idx的pool链表中的第一个, 如果存在pool, 则返回, 否则走2

2. 如果没有可用的pool, 分一个, 走3

3. 如何分? 如果有可用的arena, 从arena中分一个pool, 走4, 如果没有可用的arena, 走5

4. 从可用的arena中分一个pool, 如果有回收过的pool, 空的pool, 可用的pool就是它, 如果没有, 重新划一个4kb作为可用的pool, 无论哪一种, 最后都要把可用的pool加入usedpools数组中

5. 没有可用的arena, 新建一组(很多个)arena, 然后返回第一个作为可用的arena, 然后走4


回收流程差不多, 就是反着来

新分配arena
==============

arena是保存多个pool的地方, arena的大小是256KB, 一个pool是4kb, 那么一个arena就有64个pool.

arena包含了64个pool, 每个pool都是4kb, 至于每个pool的划分大小是多少, 根据具体情况来的, 而不是说每一个arena中一定是分别有一个分配8字节的pool, 一个分配16字节的pool

比如可能, 一个arena中64个pool都是固定分配n(比如32)字节的pool, 一个arena中有n个固定分配大小是4字节的pool, m个分配大小是64字节的pool.

需要在实际拿到一个可用的pool(4kb大小)的时候, 需要对这个pool进行初始化, 也就是决定这个pool是固定分配多大的内存

可用的arena, unused_arena_objects, 是areans这个全局arena数组的一部分.

cpython/Objects/obmalloc.c

.. code-block:: c

    static struct arena_object*
    new_arena(void)
    {
        struct arena_object* arenaobj;

        // 省略代码
    
        // 如果unused_arena_objects是NULL, 需要重新分配
        if (unused_arena_objects == NULL) {
            uint i;
            uint numarenas;
            size_t nbytes;
    
            /* Double the number of arena objects on each allocation.
             * Note that it's possible for `numarenas` to overflow.
             */
            // 每次分配都是上一次的两倍, 也就是多一倍的空间
            // 比如第一次是16, 第一次分配的大小是固定的, INITIAL_ARENA_OBJECTS=16
            // 然后第二次arena的长度就变为32
            numarenas = maxarenas ? maxarenas << 1 : INITIAL_ARENA_OBJECTS;
            if (numarenas <= maxarenas)
                return NULL;                /* overflow */
    #if SIZEOF_SIZE_T <= SIZEOF_INT
            if (numarenas > SIZE_MAX / sizeof(*arenas))
                return NULL;                /* overflow */
    #endif
    
            // 新的arenas的总长度
            // 比如第二次是numarenas = 32, nbytes = 32 * sizeof(*arenas)
            nbytes = numarenas * sizeof(*arenas);
            // 调用PyMem_RawRealloc去增加数组长度
            arenaobj = (struct arena_object *)PyMem_RawRealloc(arenas, nbytes);
            if (arenaobj == NULL)
                return NULL;
            // 这里为什么要重新赋值
            // 这是因为有可能realloc之后的内存地址会变为, 但是会把老数据给赋值到新地址上
            // 这里重新赋值是保险
            arenas = arenaobj;
    
            /* We might need to fix pointers that were copied.  However,
             * new_arena only gets called when all the pages in the
             * previous arenas are full.  Thus, there are *no* pointers
             * into the old array. Thus, we don't have to worry about
             * invalid pointers.  Just to be sure, some asserts:
             */
            assert(usable_arenas == NULL);
            assert(unused_arena_objects == NULL);
    
            /* Put the new arenas on the unused_arena_objects list. */
            // 然后数组中, 每一个新的arenas结构都初始化为0
            // address表示的是pool的地址, 地址为0
            // 表示这个arena中的pool没有分配具体的空间
            for (i = maxarenas; i < numarenas; ++i) {
                arenas[i].address = 0;              /* mark as unassociated */
                arenas[i].nextarena = i < numarenas - 1 ?
                                       &arenas[i+1] : NULL;
            }
    
            /* Update globals. */
            // 更新全局的变量
            unused_arena_objects = &arenas[maxarenas];
            maxarenas = numarenas;
        }
    
        // 下面是初始化一个可用的arena对象给调用层

        /* Take the next available arena object off the head of the list. */
        assert(unused_arena_objects != NULL);
        arenaobj = unused_arena_objects;

        unused_arena_objects = arenaobj->nextarena;

        // 这个assert表示如果address是0的话, 表示该arena对象没有初始化过
        assert(arenaobj->address == 0);

        // 为新的arena对象用来存储pool分配内存空间
        address = _PyObject_Arena.alloc(_PyObject_Arena.ctx, ARENA_SIZE);
        if (address == NULL) {
            /* The allocation failed: return NULL after putting the
             * arenaobj back.
             */
            arenaobj->nextarena = unused_arena_objects;
            unused_arena_objects = arenaobj;
            return NULL;
        }

        // 这里address就是指向分配了的256kb的内存块了
        arenaobj->address = (uintptr_t)address;
    
        ++narenas_currently_allocated;
        ++ntimes_arena_allocated;
        if (narenas_currently_allocated > narenas_highwater)
            narenas_highwater = narenas_currently_allocated;
        arenaobj->freepools = NULL;
        /* pool_address <- first pool-aligned address in the arena
           nfreepools <- number of whole pools that fit after alignment */
        
        // !!!!!!这里pool_address被初始化为address!!!!
        arenaobj->pool_address = (block*)arenaobj->address;

        //包括nfreepools的个数等等
        arenaobj->nfreepools = ARENA_SIZE / POOL_SIZE;
        assert(POOL_SIZE * arenaobj->nfreepools == ARENA_SIZE);
        excess = (uint)(arenaobj->address & POOL_SIZE_MASK);
        if (excess != 0) {
            --arenaobj->nfreepools;
            arenaobj->pool_address += POOL_SIZE - excess;
        }
        arenaobj->ntotalpools = arenaobj->nfreepools;
    
        return arenaobj;
    }

其中,

1. pool_address是下一个可用的pool的地址, 比如初始化下, pool_address是100, 划分了第一个pool, 然后pool_address就是下一个pool的地址, 也就是100 + 4kb

2. 新的arena的freepools被设置为NULL, 表示该arena没有回收过pool, 需要划分新的pool, 也就是需要划分下一个4kb空间
   freepools是一个单链表, 每当arena中的一个pool被回收, 那么会加入到freepools的头部: *freepools = pool -> old_free_pools -> old_free_pools_next ->*
   注意的是, freepools是说一个pool, 其中的block有被回收的, 并且所有的block都是可用的, 比如其中有n个block是回收了的, m个是未使用的, m可能等于0
   更多参考下面的usedpools一节

注意点:

1. unused_arena_objects指向的是全局areans数组的一部分, 当需要对arena数组扩容的之后, 有


.. code-block:: python

    '''
    
    
    arenas +-----+ arena指针0 ------> arena指针1 ----> ... ----> 指针16 ---> 指针17 ----> 指针18 ---> ... ---> 指针31
                                                                   +            +
                                                                   |            |
                                                                   |            |
    usable_arenas        +-------------->>>>-----------------------+            |
                                                                                |
    unused_arena_objects +-------------->>>>------------------------------------+    

    
    '''


2. arenas数组的realloc的时候, 为什么需要 *arenas = arenaobj;* 重新赋值? 这是因为realloc的时候内存地址可能会变化, 变化的时候会把
   老数据给复制到新的内存地址上, 重新赋值是保证arenas指针始终指向全局的arenas数组


3. arena中的address为什么一开始赋值为0? 因为arena对象只是保存了计数等信息, 真正的保存pool的内存块一开始是没有分配的.
   当需要新分配pool内存块的时候, 会把新分配出来的pool的内存块的地址信息保存到address属性上.


.. code-block:: python

    '''
    
    1. 一开始扩容完arenas数组之后, usable_arenas有
    
    arenas +---------+ usable_arenas +-----------+ address = 0
                     |          
                     + arena
                     |
                     + arena
    
    
    
    2. 然后调用address = _PyObject_Arena.alloc(_PyObject_Arena.ctx, ARENA_SIZE);分配一个ARENA_SIZE(256KB)大小的的内存块存储pool
    
    
    
    arenas +---------+ usable_arenas +-----------+ address -- 等于pool的地址, 也就是arena的地址 --->  +----------------------+
                     |                           |                                                    | 256KB的内存空间      |
                     + arena指针2                + pool_address ---------------------------------->   +----------------------+
                                                 |
                                                 + freepool = NULL
    '''

一旦address有值, 表示该arena已经分配了具体的pool的存储空间了

usedpools
============

参考 [1]_中有比较具体的解释, 先看参考 [1]_的说明, 然后下面是自己的理解

先来看看pool_header的结构

.. code-block:: c

    struct pool_header {
        union { block *_padding;
                uint count; } ref;          /* number of allocated blocks    */
        block *freeblock;                   /* pool's free list head         */
        struct pool_header *nextpool;       /* next pool of this size class  */
        struct pool_header *prevpool;       /* previous pool       ""        */
        uint arenaindex;                    /* index into arenas of base adr */
        uint szidx;                         /* block size class index        */
        uint nextoffset;                    /* bytes to virgin block         */
        uint maxnextoffset;                 /* largest valid nextoffset      */
    };

其中nextpool, prevpool, freeblock, ref都是４字节

再看看usedpools的定义:

.. code-block:: c

    typedef struct pool_header *poolp;
    
    // 具体定义先省略
    static poolp usedpools [] = {}

所以, usedpools是一个pool的头结构的的指针数组, 也就是usedpools是pool的数组, 但是存储的是头结构的地址, 但是头结构的地址也就是pool的地址了.

  *usedpools数组: 维护着所有处于used状态的pool, 当申请内存的时候, 会通过usedpools寻找到一块可用的(处于used状态的)pool, 从中分配一个block*
  
  --- 参考1

而在源码注释中有:

.. code-block:: c

    /*
    
    Major obscurity:  While the usedpools vector is declared to have poolp
    entries, it doesn't really.  It really contains two pointers per (conceptual)
    poolp entry, the nextpool and prevpool members of a pool_header.  The
    excruciating initialization code below fools C so that
    
        usedpool[i+i]
    
    "acts like" a genuine poolp, but only so long as you only reference its
    nextpool and prevpool members.  The "- 2*sizeof(block *)" gibberish is
    compensating for that a pool_header's nextpool and prevpool members
    immediately follow a pool_header's first two members:
    
        union { block *_padding;
                uint count; } ref;
        block *freeblock;
    
    */

这一段注释, 总结起来就是, usedpools中, 用两个槽位表示nextpool和prevpool这两个指针, 然后由于usedpools是定义为pool_header结构的指针,

所以为了用两个8字节去模拟nextpool和prevpool, 必须对地址进行补偿计算, 也就是进行减去 *2\*szieof(block *)* 的计算.

所以, 也就是简单点, 把usedpools中的两个槽位看成nextpool和prevpool指针就好了, 比如index=5, 那么就是usedpools的[10]和usedpools[11]

分别表示nextpool和prevpool(debug出来也确实如此)

.. code-block:: python

    '''

    usedpools +---------+ u0 --------> u1 ---- u2 -----> u3 -----> u4 -------> u5 ---> ....
                          |            |       |          |        |            |
                          +------------+       +----------+        +------------+
                          index=0              index=1的            index=1的next
                          next和prev           next和prev           和prev
                          usedpools[2*0]       usedpools[1+1]       usedpools[2+2]

                       
    '''

一开始, usedpools初始化的时候, 显然, next == prev == myself

关于一个pool的状态, 来自源码注释:

.. code-block:: c

    /*
    used == partially used, neither empty nor full
        At least one block in the pool is currently allocated, and at least one
        block in the pool is not currently allocated (note this implies a pool
        has room for at least two blocks).
        This is a pool's initial state, as a pool is created only when malloc
        needs space.
        The pool holds blocks of a fixed size, and is in the circular list headed
        at usedpools[i] (see above).  It's linked to the other used pools of the
        same size class via the pool_header's nextpool and prevpool members.
        If all but one block is currently allocated, a malloc can cause a
        transition to the full state.  If all but one block is not currently
        allocated, a free can cause a transition to the empty state.
    
    full == all the pool's blocks are currently allocated
        On transition to full, a pool is unlinked from its usedpools[] list.
        It's not linked to from anything then anymore, and its nextpool and
        prevpool members are meaningless until it transitions back to used.
        A free of a block in a full pool puts the pool back in the used state.
        Then it's linked in at the front of the appropriate usedpools[] list, so
        that the next allocation for its size class will reuse the freed block.
    
    empty == all the pool's blocks are currently available for allocation
        On transition to empty, a pool is unlinked from its usedpools[] list,
        and linked to the front of its arena_object's singly-linked freepools list,
        via its nextpool member.  The prevpool member has no meaning in this case.
        Empty pools have no inherent size class:  the next time a malloc finds
        an empty list in usedpools[], it takes the first pool off of freepools.
        If the size class needed happens to be the same as the size class the pool
        last had, some pool initialization can be skipped.
    */


1. used, 也就是一个pool正在使用(至少有一个block已经被分配), 但是不是full状态(至少有一个block可以被分配)
   这个状态的pool会被加入到usedpool中

2. full, 没有可以分配的block了, 这个状态的pool会被从usedpools中移除(On transition to full, a pool is unlinked from its usedpools[] list)，
   当该pool变为不满的时候, 也就是有一个block被释放的时候, pool又会变为used状态(A free of a block in a full pool puts the pool back in the used state),
   此时会被插入到usedpools中(Then it's linked in at the front of the appropriate usedpools[] list)
   所以下一次获取block的时候, 会优先从该pool中获取(由full变为used状态)(that the next allocation for its size class will reuse the freed block)

3. empty, 所有的block都是可分配状态(可能有n个是回收的block, 有m个是未使用的, m可能等于0), 会被从usedpools中移除(On transition to empty, a pool is unlinked from its usedpools[] list),
   然后加入到对应的arena中的freepools这个单链表中(and linked to the front of its arena_object's singly-linked freepools list)
   然后从arena中拿一个可用的pool的时候, 优先拿freepools(the next time a malloc finds an empty list in usedpools[], it takes the first pool off of freepools)
   然后需要根据idx去确定是否初始化, 比如被回收的pool的idx是4, 而我们需要的pool的idx是3, 则需要把被回收的pool的idx和分配大小设置一下


上面三个状态基本上就展示了分配的策略

从可用的pool拿到block
===========================

先查看参考 [1]_

cpython/Objects/obmalloc.c

.. code-block:: c

    static void *
    _PyObject_Alloc(int use_calloc, void *ctx, size_t nelem, size_t elsize)
    {
        
        // 如果是小对象, 小于512字节
        if ((nbytes - 1) < SMALL_REQUEST_THRESHOLD) {
            LOCK();
            /*
             * Most frequent paths first
             */
            // 拿到index
            size = (uint)(nbytes - 1) >> ALIGNMENT_SHIFT;
            // 拿到usedpools
            pool = usedpools[size + size];
            if (pool != pool->nextpool) {
                // pool != pool->nextpool
                // 说明该index下已经分配了一个pool
                // 从pool中拿block
    
                ++pool->ref.count;
                bp = pool->freeblock;
                assert(bp != NULL);
                // if的判断是, nextpool是否是NULL, 如果是NULL, 表示
                // 已经打到最大的数据块了, 走if下面的代码
                if ((pool->freeblock = *(block **)bp) != NULL) {
                    // 代码先省略
                }
                /*
                 * Reached the end of the free list, try to extend it.
                 */
                 // 如果nextpool打到最后了, 看看其中是否有其他可用的block
                 // 因为pool的中block可用不可用不一定是连续的
                if (pool->nextoffset <= pool->maxnextoffset) {
                   // if里面表示有可用的block
                   // 代码省略
                }
    
                //下面是说这个pool分配了一个block之后
                // 已经是最后一个可用的block了, 则把该pool从usedpools中移除
                /* Pool is full, unlink from used pools. */
                next = pool->nextpool;
                pool = pool->prevpool;
                next->prevpool = pool;
                pool->nextpool = next;
                UNLOCK();
                if (use_calloc)
                    memset(bp, 0, nbytes);
                return (void *)bp;
    
            }
            // 省略代码
    
        }
    }

上面的流程表示在usedpools中存在可用的pool, 此时pool != pool->nextpool, 简单来说就是:

1. 如果当前的pool还有可分配的block, 可分配存在两者情况: 没有达到最大地址以及有回收的block, 则返回

2. 如果pool是full了, 则把usedpools中的槽位置设置为next == prev == myself, 这样下一次进来的时候就需要分配额外的pool了
   也就是走if(pool != pool->nextpool)外面的代码块

拿可用arena中的pool
=======================

如果pool == pool->next, 说明没有处于used的pool, 则需要从arena中拿一个pool, 然后初始化这个pool, 把它加入到usedpools链表中

这里有两者情况, 如果arnea中有回收过的pool(freepool链表), 那么从freepool中拿, 如果freepool是NULL, 需要重新划分出4kb


.. code-block:: c

    static void *
    _PyObject_Alloc(int use_calloc, void *ctx, size_t nelem, size_t elsize)
    {
    
        if ((nbytes - 1) < SMALL_REQUEST_THRESHOLD) {
        
            if (pool != pool->nextpool) {
                // 省略代码
                // 看上一节的流程
    
            }
    
           // 如果没有可用的arena
           if (usable_arenas == NULL) {
                /* No arena has a free pool:  allocate a new arena. */
                // 下面这个宏是说如果定义了pool机制最大内存使用的话校验一下
    #ifdef WITH_MEMORY_LIMITS
                if (narenas_currently_allocated >= MAX_ARENAS) {
                    UNLOCK();
                    goto redirect;
                }
    #endif
                // 拿到一个新的arena
                usable_arenas = new_arena();
                if (usable_arenas == NULL) {
                    UNLOCK();
                    goto redirect;
                }
                usable_arenas->nextarena =
                    usable_arenas->prevarena = NULL;
            }
            assert(usable_arenas->address != 0);
    
            // 拿到arena的freepools
            pool = usable_arenas->freepools;
    
            // 下面的if说明从arena中拿到的pool可用
            if (pool != NULL) {
            
                // 这里是各种校验arena的代码


                // 然后走init_pool去初始化pool
                init_pool:
                    // 先省略代码
    
            }
    
        // 如果可用的arena中的的freepools为NULL
        // 那么需要从arena中划分下一个4kb空间作为新的pool
        // 先省略代码
    
    
        }
    
    
    }

所以, 如果没有处于used状态的pool, 那么

1. 如果没有可用的arena, 也就是usable_arenas这个全局变量, 调用 *new_arena* 那么去拓展arenas数组, 该函数参考之前的arena的分析

2. 拿到可用的arena之后, 拿到其中的freepool, 也就是先拿一个回收过的pool, 如果拿到了, 则进行初始化, 也就是init_pool代码块.
   初始化的时候注意一下, 比如freepool被释放的pool的index=3, 如果我们需要的pool的index是4, 则需要重新把pool初始化为固定分配4*8=32字节空间的pool
   所以这里就说明了, arena中的pool可以被重复使用, 并且回收的pool可以重新初始化成另外一个定额分配的pool.


3. 如果arena中的freepool不存在, 则需要额外划分4kb(pool的大小), 然后更新usable_arenas中的计数, 然后走init_pool初始化这个新的4kb空间


init_pool
==============

.. code-block:: c

    static void *
    _PyObject_Alloc(int use_calloc, void *ctx, size_t nelem, size_t elsize)
    {
    
        if ((nbytes - 1) < SMALL_REQUEST_THRESHOLD) {
        
            if (pool != pool->nextpool) {
                // 省略代码
            }
    
           if (usable_arenas == NULL) {
               // 拿新的arena
           }

           // 先拿回收过的pool
           pool = usable_arenas->freepools;
           // 如果存在回收过的pool
           if (pool != NULL) {

               // 然后freepools等于下一个pool的下一个pool
               // !!!!这里有点疑惑!!!!
               usable_arenas->freepools = pool->nextpool;
           
               // 这里是各种校验arena的代码
               // 然后走init_pool去初始化pool
               init_pool:
                   next = usedpools[size + size]; /* == prev */
                   // 加入到usedpool中
                   pool->nextpool = next;
                   pool->prevpool = next;
                   next->nextpool = pool;
                   next->prevpool = pool;
                   pool->ref.count = 1;
                   // 下面的if表示, pool的szidx和我们需要的有可能不一样
                   // 不一样的情况是, 这个是一个回收过的pool并且其szidx本来就不一样
                   // 或者这个pool的新分配的, 新分配的pool的szidx是DUMMY_SIZE_IDX
                   if (pool->szidx == size) {
                       /* Luckily, this pool last contained blocks
                        * of the same size class, so its header
                        * and free list are already initialized.
                        */
                       bp = pool->freeblock;
                       assert(bp != NULL);
                       pool->freeblock = *(block **)bp;
                       UNLOCK();
                       if (use_calloc)
                           memset(bp, 0, nbytes);
                       return (void *)bp;
                   }
                   /*
                    * Initialize the pool header, set up the free list to
                    * contain just the second block, and return the first
                    * block.
                    */
                   // 好的, 拿到的pool需要设置一下固定分配大小
                   // 以及szidx等等
                   pool->szidx = size;
                   size = INDEX2SIZE(size);
                   bp = (block *)pool + POOL_OVERHEAD;
                   pool->nextoffset = POOL_OVERHEAD + (size << 1);
                   pool->maxnextoffset = POOL_SIZE - size;
                   pool->freeblock = bp + size;
                   *(block **)(pool->freeblock) = NULL;
                   UNLOCK();
                   if (use_calloc)
                       memset(bp, 0, nbytes);
                   return (void *)bp;
    
           }
    
        // 如果可用的arena中的的freepools为NULL
        // 那么需要从arena中划分下一个4kb空间作为新的pool
        // 先省略代码
    
    
        }
    
    
    }

从arena中划分一个额外的4kb
===============================

进入到这个流程的话, 就是说usedpools中, 没有可用的pool了, 也就是pool == pool->nextpool, 并且

1. 或者要么需要新的arena, 需要分配新的arnea, 然后进入2

2. 或者usable_arneas中没有回收过的freeblock了, 需要划分新的4kb作为pool


.. code-block:: c

    static void *
    _PyObject_Alloc(int use_calloc, void *ctx, size_t nelem, size_t elsize)
    {
    
        if ((nbytes - 1) < SMALL_REQUEST_THRESHOLD) {
        
            // usedpools中有没有可用的pool
            if (pool != pool->nextpool) {
                // 省略代码
            }
    
           // 有没有usable_arena
           if (usable_arenas == NULL) {
               // 拿新的arena
           }

           // 先拿回收过的pool
           pool = usable_arenas->freepools;
           // 如果存在回收过的pool
           if (pool != NULL) {

               // 然后freepools等于下一个pool的下一个pool
               // !!!!这里有点疑惑!!!!
               usable_arenas->freepools = pool->nextpool;
           
               // 这里是各种校验arena的代码
               // 然后走init_pool去初始化pool
               init_pool:
                   // 代码省略
    
           }
    
            // arena中划分新的4kb空间

            assert(usable_arenas->nfreepools > 0);
            assert(usable_arenas->freepools == NULL);

            pool = (poolp)usable_arenas->pool_address;
            assert((block*)pool <= (block*)usable_arenas->address +
                                   ARENA_SIZE - POOL_SIZE);

            // 初始化新的pool的结构
            pool->arenaindex = (uint)(usable_arenas - arenas);
            assert(&arenas[pool->arenaindex] == usable_arenas);

            // 全新的pool的szidx是DUMMY_SIZE_IDX
            pool->szidx = DUMMY_SIZE_IDX;
            usable_arenas->pool_address += POOL_SIZE;

            // usable_arneas中的nfreepools减少一个
            --usable_arenas->nfreepools;

            if (usable_arenas->nfreepools == 0) {
                assert(usable_arenas->nextarena == NULL ||
                       usable_arenas->nextarena->prevarena ==
                       usable_arenas);
                /* Unlink the arena:  it is completely allocated. */
                usable_arenas = usable_arenas->nextarena;
                if (usable_arenas != NULL) {
                    usable_arenas->prevarena = NULL;
                    assert(usable_arenas->address != 0);
                }
            }

            // 走初始化过程
            goto init_pool;
    
        }
    
    
    }


回收内存
============

先看参考 [1]_

