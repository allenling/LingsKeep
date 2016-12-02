http1.1
========
明文报文, 也就是txt格式的, ascii码格式


0. 建立连接
1. URI/URL/URN, fragment, 绝对/相对URL
2. 方法
3. 报文格式
---------------

start-line(CRLF)
headers(CRLF)
CRLF(空行)
body

3.1 请求报文格式
~~~~~~~~~~~~~~~~~~

METHOD PATH HTTP_VERSION
HEADERS
BODY


如
GET /path/ http/1.1
Connection: keep-alive
Content-type: application/json
{'data': 1}

3.2 返回报文格式
~~~~~~~~~~~~~~~~~~~

HTTP_VERSION STATUS STATUS_NAME
BODY

如
http/1.1 200 OK
{'data': 1}

4. 请求头信息含义
---------------------

4.1 Cconnection: Keep-Alive/Close
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

http1.0协议头里可以设置Connection:Keep-Alive。在header里设置Keep-Alive可以在一定时间内复用连接，具体复用时间的长短可以由服务器控制，一般在15s左右。到http1.1之后Connection的默认值就是Keep-Alive，如果要关闭连接复用需要显式的设置Connection:Close。一段时间内的连接复用对PC端浏览器的体验帮助很大，因为大部分的请求在集中在一小段时间以内。但对移动app来说，成效不大，app端的请求比较分散且时间跨度相对较大。所以移动端app一般会从应用层寻求其它解决方案，长连接方案或者伪长连接方案：

4.2 Transfer Encoding: chunked
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

若response的头中有Transfer Encoding: chunked, 则连接不关闭, 服务器继续使用该连接发送数据直到chuncked为0.

4.3 Accept*
~~~~~~~~~~~~~~~~

Accept开头的头部含义是内容协商, 包括客户端接受说明形式的内容格式(如Accept: application/json), 什么样的编码(Accept-encoding: utf-t), 上面样的字符集(Accept-charset: gbk)

在rest_framework中, request.accepted_media_type是由你定义的renderers和wsgi中的HTTP_ACCEPT来决定的.  简单来说就是, 如果Accept命中我们设置的renderer, 则返回. 所以一般Accept: \*/\*是application/json. 如renderer和accept不能命中,
则报500, 不能满足客户端accept错误.

一般我们设置renders是
renderers = [rest_framework.renderers.JSONRenderer, rest_framework.renderers.BrowsableAPIRenderer]
media_type_set = 客户端传入的头中的Accept

判断流程在rest_framework.negotiation.DefaultContentNegotiation.select_renderer

.. code-block:: python

   class DefaultContentNegotiation(BaseContentNegotiation):
       def select_renderer(self, request, renderers, format_suffix=None):
           for media_type_set in order_by_precedence(accepts):
               for renderer in renderers:
                   for media_type in media_type_set:
                       if media_type_matches(renderer.media_type, media_type):
                           # Return the most specific media type as accepted.
                           media_type_wrapper = _MediaType(media_type)
                           if (
                               _MediaType(renderer.media_type).precedence >
                               media_type_wrapper.precedence
                           ):
                               # Eg client requests '*/*'
                               # Accepted media type is 'application/json'
                               full_media_type = ';'.join(
                                   (renderer.media_type,) +
                                   tuple('{0}={1}'.format(
                                       key, value.decode(HTTP_HEADER_ENCODING))
                                       for key, value in media_type_wrapper.params.items()))
                               return renderer, full_media_type
                           else:
                               # Eg client requests 'application/json; indent=8'
                               # Accepted media type is 'application/json; indent=8'
                               return renderer, media_type

4.4 X-Forwarded-For X-Real-IP和RemoteAddress
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

https://imququ.com/post/x-forwarded-for-header-in-http.html

上面是一下这种情况

+-------------+      +-----------------------------+
| client----> | -->  | (80)nginx2(9009) -----> app |
+-------+-----+      +-----------------------------+

在下面情况

+------------------------------------------+
| client---->  (80)nginx2(9009) -----> app |
+-------+----------------------------------+

在nginx+gunicorn+django的情况下, 三者都是127.0.0.1


还有一种情况, 比如说, 本地一个nginx, 监听8006端口, 然后反向代理的proxy_pass是一个公开的服务, 比如叫data.serve.com, 该公开服务前面也有一个nginx
           
                                 data.serve.com

+-------------------------+      +-----------------------+
| client  --> (8006)nginx1|----> | (80)nginx2 -----> app |
+-------------------------+      +-----------------------+


当我访问127.0.0.1:8006/path, nginx1会将请求发送到data.serve.com域名下, 此时请求的header中, 有{'Host': '127.0.0.1', 'RemoteAddress': '127.0.0.1', 'X-Forwarded-For': '127.0.0.1', 'X-Real-IP': '127.0.0.1'}

发送到data.serve.com的时候, 头会变为{'Host': '127.0.0.1', 'RemoteAddress': '192.168.88.214', 'X-Forwarded-For': '127.0.0.1', 'X-Real-IP': '127.0.0.1'}, RemoteAddress变为我本机的地址, 其他都是127.0.0.1.

由于nginx配置是按虚拟主机名称来匹配, 所以若匹配不到对应的虚拟主机名, 这里是data.serve.com, 则会交给默认的主机, 也就是default_serve(该配置有可能在/ect/nginx/conf/nginx_common.conf中), 一般在
default_serve中配置返回444或者403.(http://nginx.org/en/docs/http/request_processing.html)

所以会找不到虚拟主机, 所以发请求的时候, 需要加上Host这个header为Host: data.serve.com. 这样就能找到对应的虚拟主机了.

4.5 跨域请求
~~~~~~~~~~~~~~~~

Origin和Host一般一起使用. 跨域的时候将自己请求的Host和自己的Origin发送给服务器, 如Host: www.test.com, Origin: www.my.com. 若Host返回Allow-Control-Allow-Origin: www.my.com, 则说明
允许自己域名发起请求, 若不允许, 则返回502

https://en.wikipedia.org/wiki/Cross-origin_resource_sharing


5. 响应头
-------------

5.1 X-Frame-Options
~~~~~~~~~~~~~~~~~~~~~~~~~~

X-Frame-Options，是为了减少点击劫持（Clickjacking）而引入的一个响应头。Chrome4+、Firefox3.6.9+、IE8+均支持，详细的浏览器支持情况看这里。使用方式如下：
这个响应头支持三种配置：
DENY：不允许被任何页面嵌入；
SAMEORIGIN：不允许被本域以外的页面嵌入；
ALLOW-FROM uri：不允许被指定的域名以外的页面嵌入（Chrome现阶段不支持）；
如果某页面被不被允许的页面以<iframe>或<frame>的形式嵌入，IE会显示类似于“此内容无法在框架中显示”的提示信息，Chrome和Firefox都会在控制台打印信息。由于嵌入的页面不会加载，这就减少了点击劫持的发生。

5.2 X-XSS-Protection
~~~~~~~~~~~~~~~~~~~~~~~

XSS, 跨站脚本攻击.也就是钓鱼, 发送一个链接, 你点击进入的是我自己的网站, 里面的页面是银行的登陆账号, 你点击登陆, 我就发送登陆信息给银行, 若成功, 我就知道你正确的登陆信息. 或者我在社交媒体上发布一个文章, onclick的时候, 
发送当前的cookie到我的网站, 所以一旦有人点击了我的文章, 就会发送他的cookie给我, 我也就拿到了用户的sessionid, 我就可以冒充用户发起请求了, 所以我们要加上CSRF, 校验请求的令牌.

解决是转义用户输入.

顾名思义，这个响应头是用来防范XSS的。最早我是在介绍IE8的文章里看到这个，现在主流浏览器都支持，并且默认都开启了XSS保护，用这个header可以关闭它。它有几种配置：
0：禁用XSS保护；
1：启用XSS保护；
1; mode=block：启用XSS保护，并在检查到XSS攻击时，停止渲染页面（例如IE8中，检查到攻击时，整个页面会被一个#替换）；
浏览器提供的XSS保护机制并不完美，但是开启后仍然可以提升攻击难度，总之没有特别的理由，不要关闭它。


6. 缓存

概述: http://imweb.io/topic/5795dcb6fb312541492eda8c

agent                                            server
                    GET /test.css HTTP/1.1
       --------------------------------------->
        HTTP/1.1 200 OK Last-Modified-Since: xxx
        Etag: yyyy
       <----------------------------------------

                GET /test.css HTTP/1.1
            If-Modified-Since: xxxx Etag: yyy
       ---------------------------------------->

            HTTP/1.1 304 Not Modified
       <----------------------------------------

cache-control: no-cache可以等于cache-control: max-age=10; must-revalidate;

关于no-cache和max-age: http://stackoverflow.com/questions/1046966/whats-the-difference-between-cache-control-max-age-0-and-no-cache.

简单来说就是max-age是agent应该(SHOULD)发起请求重新验证资源是否修改, 而no-cache是必须(MUST)发起请求验证资源是否有修改

chrome中对于资源是否会发起请求重新获取资源: http://stackoverflow.com/questions/11245767/is-chrome-ignoring-control-cache-max-age

其实都不一定, 在不开F12的情况下, 在开启wireshark来查看chrome是否发送请求获取资源(像img), 都没有一定的规律, 比如, 第一次请求之后, 服务器返回304, 我下一次刷新(不管是点击url回车, 或者reload), 都不一定
发起请求, 也就是有可能发起请求, 也有可能是chrome伪造的response, 如200 (from cache), 304 (from cache), 这些response的的Date头的时间都是上一次response时间.

比如, 开着F12, ctrl+shift+r也有可能不会发送请求, 而是200 (from cache)

7. 各种中间角色intermediary, 包括gateway,agent,proxy,还有inbound和outbound的含义, upstream/downstream

http2.0
========

一般http2是在TLS上建立的, 也就是https的形式.

http2主要是用来解决线头阻塞问题

2.1. http/1.1 pipelining and header-of-line blocking(线头阻塞)
-------------------------------------------------------------------

HTTP Pipelining其实是把多个HTTP请求放到一个TCP连接中一一发送，而在发送过程中不需要等待服务器对前一个请求的响应；只不过，客户端还是要按照发送请求的顺序来接收响应。

c                               s

|  --(包含多个请求1,2,3,4,5)->  |
|                               |
|                               |
|  <----------------------------|
|                               |

请求2，3，4，5不用等请求1的response返回之后才发出，而是几乎在同一时间把request发向了服务器。2，3，4，5及所有后续共用该连接的请求节约了等待的时间，极大的降低了整体延迟。

head of line blocking并没有完全得到解决，server的response还是要求依次返回，遵循FIFO(first in first out)原则。也就是说如果请求1的response没有回来，2，3，4，5的response也不会被送回来。head of line blocking并没有完全得到解决，server的response还是要求依次返回，遵循FIFO(first in first out)原则。也就是说如果请求1的response没有回来，2，3，4，5的response也不会被送回来。

但就像在超市收银台或者银行柜台排队时一样，你并不知道前面的顾客是干脆利索的还是会跟收银员/柜员磨蹭到世界末日（译者注：不管怎么说，服务器（即收银员/柜员）是要按照顺序处理请求的，如果前一个请求非常耗时（顾客磨蹭），那么后续请求都会受到影响），这就是所谓的线头阻塞（Head of line blocking）。


2.2. http/2 NPN和ALPN
--------------------------

切换到http2.0

1. 客户端发送带有Upgrade: h2c的头的request给服务器
2. 服务器若不支持, 直接返回200就行, 若支持, 返回101
3. 客户端收到101, 发送http2.0请求给服务器

协议协商的方式有NPN, ALPN

ALPN和NPN的主要区别在于：谁来决定该次会话所使用的协议。在ALPN的描述中，是让客户端先发送一个协议优先级列表给服务器，由服务器最终选择一个合适的。而NPN则正好相反，客户端有着最终的决定权。

NPN:

Client                                      Server

ClientHello (NPN extension)  -------->
                                            ServerHello (NPN extension &#038; list of protocols)
                                            Certificate*
                                            ServerKeyExchange*
                                            CertificateRequest*
                             <--------      ServerHelloDone
Certificate*
ClientKeyExchange
CertificateVerify*
[ChangeCipherSpec]
EncryptedExtensions
Finished                     -------->
                                            [ChangeCipherSpec]
                             <--------      Finished
Application Data             <------->      Application Data

1.客户端通过 ClinetHello 发送一个空的 NPN 扩展字段
2. 服务端通过 NPN 扩展返回支持的协议列表
3. 客户端在 ChangeCipherSpec 之后 Finished 之前发送 EncryptedExtensions 选择某一个协议

ALPN:

Client                                                          Server

ClientHello (ALPN extension &#038; list of protocols)  -------->
                                                                 ServerHello (ALPN extension &#038; selected protocols)
                                                                 Certificate*
                                                                 ServerKeyExchange*
                                                                 CertificateRequest*
                                                  <--------      ServerHelloDone
Certificate*
ClientKeyExchange
CertificateVerify*
[ChangeCipherSpec]
Finished                                          -------->
                                                                 [ChangeCipherSpec]
                                                  <--------      Finished
Application Data                                  <------->      Application Data



1. 客户端通过 ALPN 扩展将自己支持的应用层协议发送给服务端
2. 服务端选择其中某个协议，并将结果通过 ALPN 扩展发送给客户端
3. SSL 协商完成后进行正常通讯

2.3. http/2的服务器推送, 也称为缓存推送
----------------------------------------

也就是发送请求A的response的时候, 会加上请求B的response内容, 即使请求B还没有请求, 这样客户端就可以缓存请求B的返回, 然后请求B发生的时候, 使用缓存在本地的请求B的response

2.4. http/2帧结构
---------------------

http/2中, 最重要的就是头压缩已经多路复用(多路复用有优先级, 流量控制)

众所周知 ，在 HTTP/1.1 协议中 「浏览器客户端在同一时间，针对同一域名下的请求有一定数量限制。超过限制数目的请求会被阻塞」。

多路复用在AMQP协议中也有应用, 也就是一个连接内可以定义多个channel, 这些channel都通过一个连接来首发数据, 在http/2中, channel就是帧上的流标志为, 标识该帧属于哪个流. 接收方需要

解包发过来的帧, 根据channel/流来封装多个发送给同一channel/流的帧成一个完整的数据包, http/2使用优先级来标识流的优先级, 流量控制来控制具体的流的数据流量.

流标志位就是下面的那个Stream identitier, 流的最大数量为2**32-1, 若达到最大流数字之后要建立新的流, 就必须开启新连接.

1. 客户端时必须关闭连接, 创建一个新连接

2. 服务端发送一个goway的frame通知客户端需要建立新连接.

+-----------------------------------------------+
|                Length (24)                    |
+---------------+---------------+---------------+
|  Type (8)     |  Flags (8)    |
+-+-------------+---------------+-------------------------------+
|R|                Stream Identifier (31)                       |
+=+=============================================================+
|                  Frame Payload (0...)                       ...
+---------------------------------------------------------------+

帧长度Length：无符号的自然数，24个比特表示，仅表示帧负载所占用字节数，不包括帧头所占用的9个字节. 
   默认大小区间为为0~16384(2^14)，一旦超过默认最大值2^14(16384)，发送方将不再允许发送，除非接收到接收方定义的SETTINGS_MAX_FRAME_SIZE（一般此值区间为2^14 ~ 2^24）值的通知。
    
帧类型Type：8个比特表示，定义了帧负载的具体格式和帧的语义，HTTP/2规范定义了10个帧类型，这里不包括实验类型帧和扩展类型帧

帧保留比特为R：在HTTP/2语境下为保留的比特位，固定值为0X0

帧的标志位Flags：8个比特表示，服务于具体帧类型，默认值为0x0。

流标识符Stream Identifier：无符号的31比特表示无符号自然数。0x0值表示为帧仅作用于连接，不隶属于单独的流。 奇数是客户端发起的, 偶数是服务端发起的.

2.4.1 帧类型(TYPE)
~~~~~~~~~~~~~~~~~

1. REST
++++++++++++

很多app客户端都有取消图片下载的功能场景，对于http1.x来说，是通过设置tcp segment里的reset flag来通知对端关闭连接的。这种方式会直接断开连接，下次再发请求就必须重新建立连接。http2.0引入RST_STREAM类型的frame，可以在不断开连接的前提下取消某个request的stream，表现更好。


2.5. why http/2
------------------

1. 开发http2的其中一个主要原因就是修复HTTP pipelining。如果在你的应用场景里本来就不需要pipelining，那么确实很有可能http2对你没有太大帮助。虽然这并不是唯一的提升，但显然这是非常重要的一个。

2. 多路复用

3. 小规模的REST API和采用HTTP 1.x的简单程序可能并不会从迁移到http2中获得多大的收益。但至少，迁移至http2对绝大部分用户来讲几乎是没有坏处的。

4. http/2只适用大网站? 完全不是这样。因为缺乏内容分发网络(content distributed network, cdn)，小网站的网络延迟往往较高，而多路复用的能力可以极大的改善在高网络延迟下的体验。大型网站往往已经将内容分发到各处，所以速度其实已经非常快了。+

2.6. http/2 头部压缩
----------------------

简单来讲就是

1. 服务端和客户端都维护一个http头的静态编码, 比如get方法对应数字2, 这些静态类型就直接使用数字来表示

2. 维护一个动态字典, 来表示动态内容, 比如cookie: xxxxx, 浏览器将cookie: xxxx对应为字符/数字是多少, 将这个消息告知服务器, 服务器存下来, 这样双方就有了约束. 发送请求的时候, 这部分内容就使用字符/数字来表示

3. 支持基于静态哈夫曼码表的哈夫曼编码（Huffman Coding）； 

现在大家都知道tcp有slow start的特性，三次握手之后开始发送tcp segment，第一次能发送的没有被ack的segment数量是由initial tcp window大小决定的。这个initial tcp window根据平台的实现会有差异，但一般是2个segment或者是4k的大小（一个segment大概是1500个字节），也就是说当你发送的包大小超过这个值的时候，要等前面的包被ack之后才能发送后续的包，显然这种情况下延迟更高。intial window也并不是越大越好，太大会导致网络节点的阻塞，丢包率就会增加，具体细节可以参考IETF这篇文章。http的header现在膨胀到有可能会超过这个intial window的值了，所以更显得压缩header的重要性。

2.7. http/2 流量控制
-------------------------

全部由接收端决定, 客户端(如浏览器)也可能是接收端(在push_promise的情况下)

只有DATA帧受流量控制影响.

每个流都有一个初始窗口大小, 然后每发送一个帧, 就用window的大小减去帧大小, 直到window等于0.

若不想要流量控制, 可以这么做

    1. 两端（收发）保有一个流量控制窗口（window）初始值。
    2. 发送端每发送一个DATA帧，就把window递减，递减量为这个帧的大小，要是window小于帧大小，那么这个帧就必须被拆分。如果window等于0，就不能发送任何帧
    3. 接收端可以发送 WINDOW_UPDATE帧给发送端，发送端以帧内指定的Window Size Increment作为增量，加到window上


2.8 请求优先级
---------------------



tcp/ip
========

osi七层网络模型

+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          应用层          |       HTTP/FTP/SMTP        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          表示层          |       XDR/ASN.1            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          会话层          |       ASAP/SSH/X.22/RPC    |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          传输层          |       TCP/UDP              |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          网络层          |       IP/ICMP              |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          数据链路层      |       以太网/令牌环/PPP    |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          物理层          |       光纤/无线电          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

1. 三次握手, 四次挥手
-----------------------

1.1 那为什么非要三次呢？
~~~~~~~~~~~~~~~~~~~~~~~~~

（1）第一次握手：Client将标志位SYN置为1，随机产生一个值seq=J，并将该数据包发送给Server，Client进入SYN_SENT状态，等待Server确认。

（2）第二次握手：Server收到数据包后由标志位SYN=1知道Client请求建立连接，Server将标志位SYN和ACK都置为1，ack=J+1，随机产生一个值seq=K，并将该数据包发送给Client以确认连接请求，Server进入SYN_RCVD状态。

（3）第三次握手：Client收到确认后，检查ack是否为J+1，ACK是否为1，如果正确则将标志位ACK置为1，ack=K+1，并将该数据包发送给Server，Server检查ack是否为K+1，ACK是否为1，如果正确则连接建立成功，Client和Server进入ESTABLISHED状态，完成三次握手，随后Client与Server之间可以开始传输数据了。


怎么觉得两次就可以完成了。那TCP为什么非要进行三次连接呢？在谢希仁的《计算机网络》中是这样说的：

为了防止已失效的连接请求报文段突然又传送到了服务端，因而产生错误。

在书中同时举了一个例子，如下：

“已失效的连接请求报文段”的产生在这样一种情况下：client发出的第一个连接请求报文段并没有丢失，而是在某个网络结点长时间的滞留了，以致延误到连接释放以后的某个时间才到达server。本来这是一个早已失效的报文段。但server收到此失效的连接请求报文段后，就误认为是client再次发出的一个新的连接请求。于是就向client发出确认报文段，同意建立连接。假设不采用“三次握手”，那么只要server发出确认，新的连接就建立了。由于现在client并没有发出建立连接的请求，因此不会理睬server的确认，也不会向server发送数据。但server却以为新的运输连接已经建立，并一直等待client发来数据。这样，server的很多资源就白白浪费掉了。采用“三次握手”的办法可以防止上述现象发生。例如刚才那种情况，client不会向server的确认发出确认。server由于收不到确认，就知道client并没有要求建立连接。

1.2 那四次分手又是为何呢？
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

（1）第一次挥手：Client发送一个FIN，用来关闭Client到Server的数据传送，Client进入FIN_WAIT_1状态。
（2）第二次挥手：Server收到FIN后，发送一个ACK给Client，确认序号为收到序号+1（与SYN相同，一个FIN占用一个序号），Server进入CLOSE_WAIT状态。
（3）第三次挥手：Server发送一个FIN，用来关闭Server到Client的数据传送，Server进入LAST_ACK状态。
（4）第四次挥手：Client收到FIN后，Client进入TIME_WAIT状态，接着发送一个ACK给Server，确认序号为收到序号+1，Server进入CLOSED状态，完成四次挥手。

简单来说需要双方都确认没有数据发送给对方,之后才关闭连接. 一方要关闭连接,意味着自己没有数据发送给对方,但是还是可以接收对方发送过来的数据.

TCP协议是一种面向连接的、可靠的、基于字节流的运输层通信协议。TCP是全双工模式，这就意味着，当主机1发出FIN报文段时，只是表示主机1已经没有数据要发送了，主机1告诉主机2，它的数据已经全部发送完毕了；但是，这个时候主机1还是可以接受来自主机2的数据；当主机2返回ACK报文段时，表示它已经知道主机1没有数据发送了，但是主机2还是可以发送数据到主机1的；当主机2也发送了FIN报文段时，这个时候就表示主机2也没有数据要发送了，就会告诉主机1，我也没有数据要发送了，之后彼此就会愉快的中断这次TCP连接。如果要正确的理解四次分手的原理，就需要了解四次分手过程中的状态变化。

FIN_WAIT_1: 这个状态要好好解释一下，其实FIN_WAIT_1和FIN_WAIT_2状态的真正含义都是表示等待对方的FIN报文。而这两种状态的区别是：FIN_WAIT_1状态实际上是当SOCKET在ESTABLISHED状态时，它想主动关闭连接，向对方发送了FIN报文，此时该SOCKET即进入到FIN_WAIT_1状态。而当对方回应ACK报文后，则进入到FIN_WAIT_2状态，当然在实际的正常情况下，无论对方何种情况下，都应该马上回应ACK报文，所以FIN_WAIT_1状态一般是比较难见到的，而FIN_WAIT_2状态还有时常常可以用netstat看到。（主动方）

FIN_WAIT_2：上面已经详细解释了这种状态，实际上FIN_WAIT_2状态下的SOCKET，表示半连接，也即有一方要求close连接，但另外还告诉对方，我暂时还有点数据需要传送给你(ACK信息)，稍后再关闭连接。（主动方）

CLOSE_WAIT：这种状态的含义其实是表示在等待关闭。怎么理解呢？当对方close一个SOCKET后发送FIN报文给自己，你系统毫无疑问地会回应一个ACK报文给对方，此时则进入到CLOSE_WAIT状态。接下来呢，实际上你真正需要考虑的事情是察看你是否还有数据发送给对方，如果没有的话，那么你也就可以 close这个SOCKET，发送FIN报文给对方，也即关闭连接。所以你在CLOSE_WAIT状态下，需要完成的事情是等待你去关闭连接。（被动方）

LAST_ACK: 这个状态还是比较容易好理解的，它是被动关闭一方在发送FIN报文后，最后等待对方的ACK报文。当收到ACK报文后，也即可以进入到CLOSED可用状态了。（被动方）

TIME_WAIT: 表示收到了对方的FIN报文，并发送出了ACK报文，就等2MSL后即可回到CLOSED可用状态了。如果FINWAIT1状态下，收到了对方同时带FIN标志和ACK标志的报文时，可以直接进入到TIME_WAIT状态，而无须经过FIN_WAIT_2状态。（主动方）

CLOSED: 表示连接中断。

为什么等待2MSL再关闭
~~~~~~~~~~~~~~~~~~~~~~~~~~~

为什么TIME_WAIT状态需要经过2MSL(最大报文段生存时间)才能返回到CLOSE状态？

原因有二： 
一、保证TCP协议的全双工连接能够可靠关闭 
二、保证这次连接的重复数据段从网络中消失

先说第一点，如果Client直接CLOSED了，那么由于IP协议的不可靠性或者是其它网络原因，导致Server没有收到Client最后回复的ACK。那么Server就会在超时之后继续发送FIN，此时由于Client已经CLOSED了，就找不到与重发的FIN对应的连接，最后Server就会收到RST而不是ACK，Server就会以为是连接错误把问题报告给高层。这样的情况虽然不会造成数据丢失，但是却导致TCP协议不符合可靠连接的要求。所以，Client不是直接进入CLOSED，而是要保持TIME_WAIT，当再次收到FIN的时候，能够保证对方收到ACK，最后正确的关闭连接。

再说第二点，如果Client直接CLOSED，然后又再向Server发起一个新连接，我们不能保证这个新连接与刚关闭的连接的端口号是不同的。也就是说有可能新连接和老连接的端口号是相同的。一般来说不会发生什么问题，但是还是有特殊情况出现：假设新连接和已经关闭的老连接端口号是一样的，如果前一次连接的某些数据仍然滞留在网络中，这些延迟数据在建立新连接之后才到达Server，由于新连接和老连接的端口号是一样的，又因为TCP协议判断不同连接的依据是socket pair，于是，TCP协议就认为那个延迟的数据是属于新连接的，这样就和真正的新连接的数据包发生混淆了。所以TCP连接还要在TIME_WAIT状态等待2倍MSL，这样可以保证本次连接的所有数据都从网络中消失。

2. 报文格式
--------------

TCP和UDP都是32位/4字节为单位

2.1 UDP报文格式
~~~~~~~~~~~~~~~~

+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          Source Port          |       Destination Port        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          数据包长度           |      校验值                   |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                             数据                              |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+



2.2 TCP
~~~~~~~~~~

+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          Source Port          |       Destination Port        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                        Sequence Number                        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Acknowledgment Number                      |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Data |           |U|A|P|R|S|F|                               |
| Offset| Reserved  |R|C|S|S|Y|I|            Window             |
|       |           |G|K|H|T|N|N|                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|           Checksum            |         Urgent Pointer        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Options                    |    Padding    |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                             data                              |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

2.2.1 Data Offset
~~~~~~~~~~~~~~~~

数据偏移位置, 单位是32位/4字节.

因为Options的变长的, 但是会通过padding(填充)来使得头部是32位的整数倍. 而数据是Options和padding之后开始, 所以Data Offset就是表示data是从哪里开始.

2.2.2 Checksum
~~~~~~~~~~~~~~

校验和

校验和包括TCP的头(包括options)和数据

2.2.3 Options + Padding
~~~~~~~~~~~~~~~~~~~~~~~~

TCP头部大小最大是60个字节, 包括Options之前的20个固定字节. 所以这里的Options+Padding最多就是40个字节.


3. UDP面向无连接

UDP的面向无连接是发送前不会发送syn建立连接, 只需要目标端口号, 然后就只管发送, 不保证数据到达, 不重发, 只管发送而已.

4. ip选择路由

5. nagle算法和tcp_nodelay


