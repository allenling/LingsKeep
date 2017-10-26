**ElasticSearch v5.4.0**

一些公司每天使用 Elasticsearch 索引检索 PB 级数据， 但我们中的大多数都起步于规模稍逊的项目。即使我们立志成为下一个 Facebook，我们的银行卡余额却也跟不上梦想的脚步。 我们需要为今日所需而构建，但也要允许我们可以灵活而又快速地进行水平扩展。

https://www.elastic.co/guide/cn/elasticsearch/guide/current/scale.html

shard/repli/node
==================


shard: 分片
-------------

分片即一个 Lucene 索引 ，一个 Elasticsearch 索引即一系列分片的集合。 你的应用程序与索引进行交互，Elasticsearch 帮助你将请求路由至相应的分片。


分片是一个功能完整的搜索引擎，它拥有使用一个节点上的所有资源的能力。 我们这个拥有6个分片（3个主分片和3个副本分片）的索引可以最大扩容到6个节点，每个节点上存在一个分片，并且每个分片拥有所在节点的全部资源。

**主分片的数目在索引创建时就已经确定了下来。** 实际上，这个数目定义了这个索引能够 存储 的最大数据量

*但是如果我们想要扩容超过6个节点怎么办呢？* 

主分片的数量不能调整，但是副分片可以呀，比如3主3副的结构把副本大小从１增到２(每个主分片2个副分片)之后，就有９个分片了(之前是6个), 3主6副.
这意味着我们可以将集群扩容到9个节点，每个节点上一个分片。相比原来3个节点时，集群搜索性能可以提升 3 倍。

当然，如果只是在相同节点数目的集群上增加更多的副本分片并不能提高性能,
**因为每个分片从节点上获得的资源会变少。** 你需要增加更多的硬件资源来提升吞吐量。

但是更多的副本分片数提高了数据冗余量：按照上面的节点配置，我们可以在失去2个节点的情况下不丢失任何数据。

https://www.elastic.co/guide/cn/elasticsearch/guide/current/_scale_horizontally.html


为什么主分片不能变更?
-----------------------

文档应该路由到哪个分片的计算公式是: shard = hash(routing) % number_of_primary_shards, 所以一旦主分片数量变了, 路由就就不成立了.

比如主分片由３变为４，那么有可能某个文档被路由到3这个主分片, 但是原来的分片序号最多是2(0, 1, 2), 找不到3这个分片.

(这么看来主分片数量变了的话需要重新均衡文档到新的分片数量下, 不过es没做这么做, 为什么?) 

https://www.elastic.co/guide/cn/elasticsearch/guide/current/routing-value.html

主/副节点交互
---------------

所有的写操作都必须在主分片上操作好然后把数据复制的副本分片上. 读的话主分片和副分片都能接收. 所以一般用负载均衡去轮询所有的node.

https://www.elastic.co/guide/cn/elasticsearch/guide/current/distrib-write.html

但是，当一个操作(写)在主分片上完成，但是异步复制到副本的还没有完成的时候，此时搜出会出现副本返回未找到(当轮询到副本分片的时候), 主分片返回数据(轮询到主分片的时候), 怎么破?

https://www.elastic.co/guide/cn/elasticsearch/guide/current/distrib-read.html


mget/bulk
-----------

mget 和 bulk API 的 模式类似于单文档模式。区别在于协调节点知道每个文档存在于哪个分片中。 它将整个多文档请求分解成 每个分片 的多文档请求，并且将这些请求并行转发到每个参与节点。
协调节点把所有的批量操作结果聚合起来然后返回.

批量操作是一行一个json的形式, 具体文档.

plus: SLA?

https://www.elastic.co/guide/cn/elasticsearch/guide/current/distrib-multi-doc.html

所以, 协调节点是怎么工作的?

当一个搜索请求被发送到某个节点时，这个节点就变成了协调节点。 这个节点的任务是广播查询请求到所有相关分片并将它们的响应整合成全局排序后的结果集合，这个结果集合会返回给客户端。
然后协调节点会轮询所有分片(主分片和副分片)

这里有两个轮询, 轮询所有节点，处理请求的节点成为协调节点, 协调节点轮询分片(怎么轮询分片?(分布式查询一节有)). 所以协调节点是处理请求的任意节点，所以轮询就轮询所有节点就好了

https://www.elastic.co/guide/cn/elasticsearch/guide/current/_query_phase.html


分布式查询
===============

一个 CRUD 操作只对单个文档进行处理，文档的唯一性由 _index, _type, 和 routing values （通常默认是该文档的 _id ）的组合来确定。 这表示我们确切的知道集群中哪个分片含有此文档。

搜索需要一种更加复杂的执行模型因为我们不知道查询会命中哪些文档: 这些文档有可能在集群的任何分片上。 一个搜索请求必须询问我们关注的索引（index or indices）的所有分片的某个副本来确定它们是否含有任何匹配的文档。

但是找到所有的匹配文档仅仅完成事情的一半。 在 search 接口返回一个 ``page`` 结果之前，多分片中的结果必须组合成单个排序列表。 为此，搜索被执行成一个两阶段过程，我们称之为 query then fetch 

也就是先搜索，然后聚合然后排序. 恩恩, 内存中快排, timsort什么的~~~


发起查询
----------

在初始 查询阶段 时， 查询会 **广播** 到索引中每一个分片拷贝（主分片或者副本分片）。 每个分片在本地执行搜索并构建一个匹配文档的 _优先队列_。

分布式查询的过程(看文档), 大概是将每个分片根据排序返回的前N个结果再排序为前N个(也就是top n结果，是不是想到了堆). 注意，这里返回的结果只是id和排序得分，并不是完整文档,

完整文档是在下面的取回查询数据的时候获取的.

**这里注意的是每个分片的意思, 其实是每个主分片或者主分片的副本**, 本质上是所有主分片的查询结果, 因为每个分片的副分片数据都是一样的，只有查询主分片才有意义，但是

查询主分片的时候，查询其副分片其实是一样的意思，所以应该是查询每一个主分片或者主分片的副分片, 比如主分片1, 2, 3, 其副分片: 11, 22, 33, 查询的时候查询

1, 22, 33和1, 2, 3都是可以的(**可以通过文档的图来理解，文档中图片中第二步就是查询了p1和r0,也就是主分片1本身p1和主分片0的副本分片r0**)

轮询分片的意思也可以理解为轮询的是主分片和其副分片~~~

https://www.elastic.co/guide/cn/elasticsearch/guide/current/distributed-search.html


取回查询
------------

这里就是query and fetch的fetch阶段, 将上一个阶段的m * top N(m是分片数)全排序(注意，这里不是维护top N了，因为有分页的存在，所有top N不行)之后，

根据分页得到的id去取需要的文档. 这样可以避免回去很多不必须的文档.

**但是我有个疑问，所有分片的top N经过排序之后，一定是所有数据的top N么**, 恩，答案是必然的，没证明.

不负责的证明: 将设a1, a2, a3和b1, b2, b3, b4分别是两个序列的前3个和前4个值, 然后返回给协调节点的是a1, a2, a3和b1, b2, b3, 是否存在两个序列的前3个

值中包含了b4. 证明, 两个序列的前3个要么全在第二个序列里面, 也就是b1, b2, b3 > (a3, a2, a3), 要么再第一个序列里面a1, a2, a3 > (b1, b2, b3), 要么就是两个序列混排

之后的前3个, 前两种情况中，b4不可能出现，第三种混排的情况也不可能出现有b4, 因为若存在b4是前3个的话，这就跟b4是第二个序列的第四个向违背，所以不可能.

深分页(大分页, 比如第1000页):

实际上， “深分页” 很少符合人的行为。当2到3页过去以后，人会停止翻页，并且改变搜索标准。会不知疲倦地一页一页的获取网页直到你的服务崩溃的罪魁祸首一般是机器人或者web spider。

https://www.elastic.co/guide/cn/elasticsearch/guide/current/_fetch_phase.html




搜索结果
===========

善用analyze api

如果不对某一特殊的索引或者类型做限制，就会搜索集群中的所有文档。Elasticsearch 转发搜索请求到每一个主分片或者副本分片，汇集查询出的前10个结果，并且返回给我们。

比如搜索a, 会搜索所有的文档, 包括用户和文章, 可以指定搜索索引文章或者用户. 比如很多公司的搜索的时候都会给你选择你是要搜索用户还是文章什么的.

https://www.elastic.co/guide/cn/elasticsearch/guide/current/multi-index-multi-type.html


除非指定属性，否则es会搜索_all这个属性，这个属性的内容是各个属性拼接起来的字符串

https://www.elastic.co/guide/cn/elasticsearch/guide/current/search-lite.html


精确值/全文
------------





乐观锁并发控制
==================

主分片异步复制数据到副本分片是异步的, 并且是复制整个文档而不是复制更新操作(类比mysql, 复制表而不是复制binlog), 所以是乱序的.

https://www.elastic.co/guide/cn/elasticsearch/guide/current/_partial_updates_to_a_document.html

更新有冲突的时候, 类似mvcc

https://www.elastic.co/guide/cn/elasticsearch/guide/current/optimistic-concurrency-control.html


如何加速搜索
==============

最新的索引放在内存, 控制刷盘(fsync)时间(es默认是1s)



倒排索引
=========

倒排索引? http://blog.csdn.net/hguisu/article/details/7962350

文本 -> 分析器分词 -> 提取分析处理词汇(词根去重, 一律小写等等) -> 词汇和文档的映射表, 包括频率位置等相关度，用于计算相关性 -> 入库

        后面这几步都可以并行                                                                  

查询的时候也需要对查询词汇进行相同的分词和分析果然, 不然怎么查询.


*并行*

  并行任务的特点就是独立的了, 比如reduce, 有n行文本，每一个有m个数字, 求所有数字之和, 可以各个worker计算一行存起来, 然后一起把结果加一下，并行了!
  
  比如提出词汇, 每个worker处理一行, 用相同的处理规则生成字典, 存起来，然后merge(merge是不是想起了归并排序!!!)一下.

NLP? 分词


setting/mapping
=================

mapping是字段和类型的映射(就是为字段设置类型), 这个很影响查询. mapping可以反过来理解，不是字段 -> 类型的映射, 而是 字段Є域.

es中有类型，并且类型有对应的搜索方法, 某个字段被归类与某个类型(域), 搜索的时候就按那个域的搜索方式去搜索.

可以自定义映射.


复杂对象和数据建模
----------------------




相关性和排序
=================

相关性, Elasticsearch的相似度算法 被定义为检索词频率/反向文档频率， TF/IDF. 输出相关度分析查询的时候加上/_explain

例如检索词honymoon

检索词(TF): 检索词 `honeymoon` 在这个文档的 `tweet` 字段中的出现次数。次数越多相关度越高

反向文档频率(IDF): 检索词 `honeymoon` 在索引上所有文档的 `tweet` 字段中出现的次数。次数越高相关度越低

字段长度准则: 在这个文档中， `tweet` 字段内容的长度 -- 内容越长，值越小。

https://www.elastic.co/guide/cn/elasticsearch/guide/current/relevance-intro.html


docValues?
-----------

因此，搜索和聚合是相互紧密缠绕的。搜索使用倒排索引查找文档，聚合操作收集和聚合 doc values 里的数据。

还是看例子吧

https://www.elastic.co/guide/cn/elasticsearch/guide/current/docvalues.html

https://www.elastic.co/guide/cn/elasticsearch/guide/current/_deep_dive_on_doc_values.html


游标查询
==========

cursor, 游标查询会取某个时间点的快照数据。 查询初始化之后索引上的任何变化会被它忽略。 它通过保存旧的数据文件来实现这个特性，结果就像保留初始化时的索引 视图 一样。

深度分页的代价根源是结果集全局排序，如果去掉全局排序的特性的话查询结果的成本就会很低。 游标查询用字段 _doc 来排序。 这个指令让 Elasticsearch 仅仅从还有结果的分片返回下一批结果。

大概意思是初始化的游标的时候，会生产一个数据快照, 然后所有的查询都是基于这个快照. 然后查询的时候是通过游标移动来实现的，比如分页的时候先查询前10个, 然后

记住游标的位置，接着查询下一10个的时候，游标直接往下走就好了. 这个跟之前的分页区别就是，之前的分页是要全局排序的，而这里的cursor是在快照的基础上，移动游标来查询的.

然后游标有过期时间的，过期之后就重新生成游标，然后又生成了数据快照.

话说，这样有点像缓存~~~

https://www.elastic.co/guide/cn/elasticsearch/guide/current/scroll.html


类比于mysql的cursor?


水平扩容
=========

Elasticsearch 的默认设置会伴你走过很长的一段路，但为了发挥它最大的效用，你需要考虑数据是如何流经你的系统的。 我们将讨论两种常见的数据流：时序数据（时间驱动相关性，例如日志或社交网络数据流），以及基于用户的数据（拥有很大的文档集但可以按用户或客户细分）。

注意的是 **时序数据**, **基于用户的数据**

https://www.elastic.co/guide/cn/elasticsearch/guide/current/scale.html


更新索引
==========


重建索引，一般使用索引别名

做好准备：在你的应用中使用别名而不是索引名。然后你就可以在任何时候重建索引。别名的开销很小，应该广泛使用。

https://www.elastic.co/guide/cn/elasticsearch/guide/current/index-aliases.html

Elasticsearch 通过另一种方式来支持分片分裂。你总是可以把你的数据 **重新索引至一个拥有适当分片个数的新索引（参阅 重新索引你的数据）** 。 和移动分片比起来这依然是一个更加密集的操作，依然需要足够的剩余空间来完成，但至少你可以控制新索引的分片个数了。
https://www.elastic.co/guide/cn/elasticsearch/guide/current/reindex.html




嵌套对象查询
==============


内嵌的json结构一般会被es打扁平, 比如

"followers": [
    { "age": 35, "name": "Mary White"},
    { "age": 26, "name": "Alex Jones"},
    { "age": 19, "name": "Lisa Smith"}
]

会被打平成
{
    "followers.age":    [19, 26, 35],
    "followers.name":   [alex, jones, lisa, smith, mary, white]
}

{age: 35} 和 {name: Mary White} 之间的相关性已经丢失了，因为每个多值域只是一包无序的值，而不是有序数组。这足以让我们问，“有一个26岁的追随者？”

https://www.elastic.co/guide/cn/elasticsearch/guide/current/complex-core-fields.html#inner-objects

要查询相关性的话，可以把字段类型设置为nested, 这样查询就会有相关性，原理就是:

*嵌套对象 就是来解决这个问题的。将 comments 字段类型设置为 nested 而不是 object 后,每一个嵌套对象都会被索引为一个 隐藏的独立文档*

例子:

{
  "title": "Nest eggs",
  "body":  "Making your money work...",
  "tags":  [ "cash", "shares" ],
  "comments": [ 
    {
      "name":    "John Smith",
      "comment": "Great article",
      "age":     28,
      "stars":   4,
      "date":    "2014-09-01"
    },
    {
      "name":    "Alice White",
      "comment": "More like this please",
      "age":     31,
      "stars":   5,
      "date":    "2014-10-22"
    }
  ]
}

如果不是nested类型的话，es会这么处理:

{
  "title":            [ eggs, nest ],
  "body":             [ making, money, work, your ],
  "tags":             [ cash, shares ],
  "comments.name":    [ alice, john, smith, white ],
  "comments.comment": [ article, great, like, more, please, this ],
  "comments.age":     [ 28, 31 ],
  "comments.stars":   [ 4, 5 ],
  "comments.date":    [ 2014-09-01, 2014-10-22 ]
}

如果是nested的话, 每个comments下的元素都是一个隐藏的文档

//下面两个都是根文档下的隐藏文档

{ 
  "comments.name":    [ john, smith ],
  "comments.comment": [ article, great ],
  "comments.age":     [ 28 ],
  "comments.stars":   [ 4 ],
  "comments.date":    [ 2014-09-01 ]
}
{ 
  "comments.name":    [ alice, white ],
  "comments.comment": [ like, more, please, this ],
  "comments.age":     [ 31 ],
  "comments.stars":   [ 5 ],
  "comments.date":    [ 2014-10-22 ]
}

//下面是根文档

{ 
  "title":            [ eggs, nest ],
  "body":             [ making, money, work, your ],
  "tags":             [ cash, shares ]
}


然后嵌套文档一般是和根文档绑定一起的，不能单独操作

*在独立索引每一个嵌套对象后,对象中每个字段的相关性得以保留。我们查询时,也仅仅返回那些真正符合条件的文档。

不仅如此,由于嵌套文档直接存储在文档内部,查询时嵌套文档和根文档联合成本很低,速度和单独存储几乎一样。

嵌套文档是隐藏存储的,我们不能直接获取。如果要增删改一个嵌套对象,我们必须把整个文档重新索引才可以。值得注意的是,查询的时候返回的是整个文档,而不是嵌套文档本身。*


https://www.elastic.co/guide/cn/elasticsearch/guide/current/nested-objects.html


嵌套对象排序
-----------------

nested_path, nested_filter

https://www.elastic.co/guide/cn/elasticsearch/guide/current/nested-sorting.html


嵌套对象聚合
---------------

https://www.elastic.co/guide/cn/elasticsearch/guide/current/nested-aggregation.html#nested-aggregation


嵌套对象的优势和劣势
----------------------------------

嵌套对象 在只有一个主要实体时非常有用，这个主要实体包含有限个紧密关联但又不是很重要的实体，例如我们的 blogpost 对象包含评论对象。 在基于评论的内容查找博客文章时， nested 查询有很大的用处，并且可以提供更快的查询效率。

嵌套模型的缺点如下：

当对嵌套文档做增加、修改或者删除时，整个文档都要重新被索引。嵌套文档越多，这带来的成本就越大。
查询结果返回的是整个文档，而不仅仅是匹配的嵌套文档。尽管目前有计划支持只返回根文档中最佳匹配的嵌套文档，但目前还不支持。
有时你需要在主文档和其关联实体之间做一个完整的隔离设计。这个隔离是由 父子关联 提供的。

https://www.elastic.co/guide/cn/elasticsearch/guide/current/nested-aggregation.html#nested-aggregation



关联查询
===========

Elasticsearch ，和大多数 NoSQL 数据库类似，是扁平化的。索引是独立文档的集合体。 文档是否匹配搜索请求取决于它是否包含所有的所需信息。

Elasticsearch 中单个文档的数据变更是 ACIDic 的， 而涉及多个文档的事务则不是。当一个事务部分失败时，无法回滚索引数据到前一个状态。

https://www.elastic.co/guide/cn/elasticsearch/guide/current/relations.html

有user和content这两个索引, es中典型的存储如下

user:

{'name': 'john', 'gender': 'M', 'id': 1}

content:

{'title': 't', 'user_id': 1}


比如要查询用户名为john用户的的文档，可以

应用层自己查
--------------

https://www.elastic.co/guide/cn/elasticsearch/guide/current/application-joins.html

先查询所有name带有john的用户的id, 然后通过id去查询content里面的user_id

冗余数据
----------

https://www.elastic.co/guide/cn/elasticsearch/guide/current/denormalization.html

content的存储带上user的一些基本信息

{'title': 't', 'user': {'id': 1, 'name': 'john'}

*但是这样的话，如果用户修改了自己的名字，除了要更新user索引，还要更新content索引, 但是代价不大

当然，数据非规范化也有弊端。 第一个缺点是索引会更大因为每个博客文章文档的 _source 将会更大，并且这里有很多的索引字段。这通常不是一个大问题。数据写到磁盘将会被高度压缩，而且磁盘已经很廉价了。Elasticsearch 可以愉快地应付这些额外的数据。

更重要的问题是，如果用户改变了他的名字，他所有的博客文章也需要更新了。幸运的是，用户不经常更改名称。即使他们做了， 用户也不可能写超过几千篇博客文章，所以更新博客文章通过 scroll 和 bulk APIs 大概耗费不到一秒。

https://www.elastic.co/guide/cn/elasticsearch/guide/current/denormalization-concurrency.html*


聚合
-----

桶的概念

https://www.elastic.co/guide/cn/elasticsearch/guide/current/top-hits.html


并发
-----

各种锁

https://www.elastic.co/guide/cn/elasticsearch/guide/current/concurrency-solutions.html


父子文档
==========

https://www.elastic.co/guide/cn/elasticsearch/guide/current/parent-child.html


