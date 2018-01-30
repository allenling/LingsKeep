

intern
============

参考1: http://guilload.com/python-string-interning/

参考2: http://www.laurentluce.com/posts/tag/python/

简单来说, intern会把字符串当成全局唯一一个, 然后可以减少内存分配.

python会在编译成字节码的时候把常量和常量计算的结果, 长度都要不大于20, 给intern掉, 计算操作比如字符串的+和*, 但是一旦涉及到变量计算, 就会重新分配内存了.

不带特殊符号(!, _等等)的字符串会被intern掉, 带特殊符号的不会.

使用了intern的字符串可用从LOAD_CONST操作码看出来的.

整数也可以intern的, 但是整数的intern应该是和小整数内存池有关.



字符串查找
===============

http://guilload.com/python-string-interning/


查找算法参考了BM算法和Horspool算法

