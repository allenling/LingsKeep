################
马尔科夫蒙特卡洛
################

马尔科夫蒙特卡洛(Markov chain Monte Carlo，MCMC)方法, 下面简称MCMC

IOTA的tangle中, 选择两个事务的算法利用了MCMC(加权随机游走算法), 让选择两个事务不遵循"随机性", 因为随机性并不好.

官方的实现貌似是所谓的加权随机, 参考了MCMC算法.

.. [1] https://www.jiqizhixin.com/articles/2017-12-24-6

.. [2] https://zhuanlan.zhihu.com/p/25610149

.. [3] https://zhuanlan.zhihu.com/p/26453269

.. [4] http://www.cnblogs.com/pinard/p/6632399.html

.. [5] http://huangweiran.club/2018/01/29/Introduction%20to%20Markov%20chain/

.. [6] https://www.zhihu.com/question/20254139

.. [7] http://www.ruanyifeng.com/blog/2015/07/monte-carlo-method.html

.. [8] https://www.cnblogs.com/daniel-D/p/3388724.html

马尔科夫链
==============

MCMC的第一个MC则是马尔科夫链模型

参考1是通俗地解释MCMC, 也是翻译文章, 有点拗口

参考2是简单介绍(其实也不简单)

参考3才是真的比较形象

参考4和5都有解释和公式


一句话理解: 状态转移方程, 会最终收敛到一组状态.


蒙特卡洛算法
===============

参考 [6]_, [7]_

抽样然后从概率得出近似最优解, 比如求积分, 比如一个简单的例子: 投针求圆周率的值


MCMC
=======

参考 [8]_


