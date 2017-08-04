django, app, 依赖
=======================


基本以及最重要的一点是，django是一个单体应用!

不像微服务，app的边界是网络!


业务app之间不合适写install_requires
--------------------------------------

开源的app都是提供基础或者插件服务，跟业务无关，所以不会出现互相依赖的情况，这跟业务app就很有区别.

业务app不像其他基础设施app一样，只有单向依赖，业务app互相调用是非常正常的，比如a和b经常要用对方提供的方法来调用，这个时候在
install_requires中写上依赖就是互相引用了，先不管pip install怎么帮你解决互相依赖的情况，本身写互相依赖就很奇怪。


所以，我觉得业务app上就不写install_requires了，install_requires上只写上，只保留版本信息，只在项目的requiremenet.txt上写上各个业务app的版本。

因为单体应用的业务app就应该跟着项目走，业务app的功能只是体提供业务实现，这个时候业务app就不能单独看待，而是说他是项目的一部分而已，你要装，就
把整个项目装一下，不然单独装某个业务app其实是没有意义的。又因为项目上的requiremenets.txt已经写上了完整的依赖，包括基础服务, 这样app上的install_requires意义就不大了


业务app之间的互相调用
-------------------------

既然业务app之间没有了install_requires来依赖，又因为本身是一个单体应用，那么其实app之间可以直接调用的，比如a中调用b.utils.func，这样，但是这样app

代码之间就很又可能出现模块上的互相引用，当然，只要两个app之间相同模块互相调用就好, 比如不要出现这种情况: a.utils.afunc上调用了b.utils.bfunc1, 而b.utils.bfunc2上样调用了a.utils.afunc1就可以

但是写着写着不小心就又这种情况.

django中提供了信号机制来一定程度上解决互相调用的情况，但是信号机制是一种又因果关系的调用，比如说某个app发出了某个信号，然后调用监听这个信号的handler，

这里本身就有局限性，比如，a这个app需要b中某个model的信息，这个就用不到信号机制了.

我觉得既然业务app之间已经没有了install_requires这种东西，那么互相调用就走一个之间层吧，一个实例，它提供call方法，参数是(app.func, args, kwargs)这样的形式,

这样可以避免模块互相调用的情况.

然后，app提供的服务中既可以返回object，因为是单体的嘛，就算你不返回object我一样可以直接手动导, 也可以为返回序列化好的数据.


业务app的复用
---------------


按照上面的做法拆分业务app之后，复用怎么办呢?

比如现在project1用的是my-user这个app，然后project2也想用my-user这个app，但是my-user这个app中view有一些view, utils等模块是跟project1中的业务有关的，project2不能直接用，因为

比如project1中有另外一个业务app叫content，然后my-user中utils有get_user_content这个函数，这个函数会去调用content这个app的函数获取某个用户下content的信息，但是project2并

不需要content这个app，所以直接用my-user这个app肯定是不行的。

我们可以把my-user这个app中最基础的部分剥离出来，作为一个新的app，叫base-user的app, 这个base-user的app包含了最基础的部分，比如其中models中user等model，utils中获取user

的基本信息，等等，这样project2自己做一个user的app，叫project2-user，然后my-user和project2-user都依赖与base-user这个app，就可以了。


其他
------

serializer上的extra是个优秀的设计, 但是需要做减法, 全部取消外键的情况下，extra只允许callable的handler.



