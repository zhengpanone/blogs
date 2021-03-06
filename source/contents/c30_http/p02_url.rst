========================
30.2 URL基础知识
========================

本文转自segmentfault ,部分删减。原文链接:https://segmentfault.com/a/1190000017122748

1. 从浏览器接收url到开启网络请求线程
#. 开启网络线程到发出一个完整的http请求
#. 从服务器接收到请求到对应后台接收到请求
#. 后台和前台的http交互
#. http的缓存问题
#. 浏览器接收到http数据包后的解析流程
#. 跨域、web安全、hybrid等等

从浏览器接收url到开启网络请求线程
==============================================

多进程的浏览器
----------------------------

浏览器是多进程的，有一个主控进程，以及每一个tab页面都会新开一个进程（某些情况下多个tab会合并进程）。
进程可能包括主控进程，插件进程，GPU，tab页（浏览器内核）等等。

    1. Browser进程：浏览器的主进程（负责协调、主控），只有一个
    #. 第三方插件进程：每种类型的插件对应一个进程，仅当使用该插件时才创建
    #. GPU进程：最多一个，用于3D绘制
    #. 浏览器渲染进程（内核）：默认每个Tab页面一个进程，互不影响，控制页面渲染，脚本执行，事件处理等（有时候会优化，如多个空白tab会合并成一个进程）

多线程的浏览器内核

每一个tab页面可以看作是浏览器内核进程，然后这个进程是多线程的，它有几大类子线程：

GUI渲染线程

JS引擎线程

事件触发线程

定时器触发线程

http异步网络请求线程



解析URL

输入URL后，会进行解析（URL的本质就是统一资源定位符）

URL一般包括几大部分：

protocol，协议头，譬如有http，ftp，https等

host，主机域名或IP地址

port，端口号

path，目录路径

query，即查询参数

fragment，即 #后的hash值，一般用来定位到某个位置



网络请求都是单独的线程

每次网络请求时都需要开辟单独的线程进行，譬如如果URL解析到http协议，就会新建一个网络线程去处理资源下载。

因此浏览器会根据解析出得协议，开辟一个网络线程，前往请求资源。



启网络线程到发出一个完整的http请求



DNS查询得到IP

如果输入的是域名，需要进行dns解析成IP，大致流程：

如果浏览器有缓存，直接使用浏览器缓存，否则使用本机缓存，再没有的话就是用host
如果本地没有，就向dns域名服务器查询（当然，中间可能还会经过路由，也有缓存等），查询到对应的IP
注意，域名查询时有可能是经过了CDN调度器的（如果有cdn存储功能的话）。

而且，需要知道dns解析是很耗时的，因此如果解析域名过多，会让首屏加载变得过慢，可以考虑 dns-prefetch优化。
这一块可以深入展开，具体请去网上搜索，这里就不占篇幅了（网上可以看到很详细的解答）。

tcp/ip请求

http的本质就是 tcp/ip请求。
需要了解三次握手规则建立连接以及断开连接时的四次挥手。
tcp将http长报文划分为短报文，通过三次握手与服务端建立连接，进行可靠传输。

三次握手：

1.客户端给服务器发确实是当前服务器
2.服务器给客户端回应，我是你要访问的当前服务器
3.客户端回应，我是客户端

四次挥手：

1.发起者：关闭主动传输信息的通道，只能接收信息
2.接受者：收到通道关闭的信息
3.接受者：也关闭主动传输信息的通道
4.发起者：接收到数据，关闭通道，双方无法通信


tcp/ip的并发限制



浏览器对同一域名下并发的tcp连接是有限制的（2-10个不等）。
而且在http1.0中往往一个资源下载就需要对应一个tcp/ip请求。
所以针对这个瓶颈，又出现了很多的资源优化方案。（感兴趣的朋友请自行搜索，资料很多）


get和post的区别

这个东西网上的资料也很多，这儿就大概描述一下在tcp/ip层面的区别，在http层面的区别请读者自行搜索：
get和post本质都是tcp/ip。
get会产生一个tcp数据包，post两个。

具体就是：

get请求时，浏览器会把 headers和 data一起发送出去，服务器响应200（返回数据）。



post请求时，浏览器先发送 headers，服务器响应 100continue，浏览器再发送 data，服务器响应200（返回数据）。

然后有读者可能以前了解过OSI的七层：物理层、 数据链路层、 网络层、 传输层、 会话层、 表示层、 应用层

这儿就不班门弄虎了，列一下内容，需要深入理解的读者请自行搜索，计算机网络相关的资料。

1.应用层(dns,http) DNS解析成IP并发送http请求
2.传输层(tcp,udp) 建立tcp连接（三次握手）
3.网络层(IP,ARP) IP寻址
4.数据链路层(PPP) 封装成帧
5.物理层(利用物理介质传输比特流) 物理传输（然后传输的时候通过双绞线，电磁波等各种介质）
6.表示层：主要处理两个通信系统中交换信息的表示方式，包括数据格式交换，数据加密与解密，数据压缩与终端类型转换等
7.会话层：它具体管理不同用户和进程之间的对话，如控制登陆和注销过程



从服务器接收到请求到对应后台接收到请求



负载均衡

对于大型的项目，由于并发访问量很大，所以往往一台服务器是吃不消的，所以一般会有若干台服务器组成一个集群，然后配合反向代理实现负载均衡。（据说现在node在微服务的项目方面越来越猛，大并发也不在话下，正在研究node，希望后面能写一个心得）

简单的说：用户发起的请求都指向调度服务器（反向代理服务器，譬如安装了nginx控制负载均衡），然后调度服务器根据实际的调度算法，分配不同的请求给对应集群中的服务器执行，然后调度器等待实际服务器的HTTP响应，并将它反馈给用户

后台的处理

一般后台都是部署到容器中的，所以一般为：

1.先是容器接受到请求（如tomcat容器）
2.然后对应容器中的后台程序接收到请求（如java程序）
3.然后就是后台会有自己的统一处理，处理完后响应响应结果

概括下：
1.一般有的后端是有统一的验证的，如安全拦截，跨域验证
2.如果这一步不符合规则，就直接返回了相应的http报文（如拒绝请求等）
3.然后当验证通过后，才会进入实际的后台代码，此时是程序接收到请求，然后执行（譬如查询数据库，大量计算等等）
4.等程序执行完毕后，就会返回一个http响应包（一般这一步也会经过多层封装）
5.然后就是将这个包从后端发送到前端，完成交互



后台和前台的http交互



前后端交互时，http报文作为信息的载体。

http报文结构

报文一般包括了： 通用头部， 请求/响应头部， 请求/响应体。学过计算机网络的读者应超级熟悉。

通用头部

这也是开发人员见过的最多的信息，包括如下：

Request Url: 请求的web服务器地址
Request Method: 请求方式（Get、POST、OPTIONS、PUT、HEAD、DELETE、CONNECT、TRACE）
Status Code: 请求的返回状态码，如200代表成功
Remote Address: 请求的远程服务器地址（会转为IP）
譬如，在跨域拒绝时，可能是method为 options，状态码为 404/405等（当然，实际上可能的组合有很多）。
其中，Method的话一般分为两批次：



HTTP1.0定义了三种请求方法： GET, POST 和 HEAD方法。
HTTP1.1新增了五种请求方法：OPTIONS, PUT, DELETE, TRACE 和 CONNECT 方法。



相信知道RESTFUL的读者应该很熟悉，现在在前端后端开发使用频繁的也就是get,post,put,delete，也是我们熟知的四大操作"增删改查"。



状态码



这是进行请求和回应的关键信息，官方有最全的状态码信息，这儿就列几个常见的：



200——表明该请求被成功地完成，所请求的资源发送回客户端

304——自从上次请求后，请求的网页未修改过，请客户端使用本地缓存

400——客户端请求有错（譬如可以是安全模块拦截）

401——请求未经授权

403——禁止访问（譬如可以是未登录时禁止）

404——资源未找到

500——服务器内部错误

503——服务不可用


对于状态码



数字1开头的表示：请求已经接收，继续处理
数字2开头的表示：请求成功，已经被服务器成功处理
数字3开头的表示：需要客户端采取进一步的操作才能完成请求
数字4开头的表示：客户端看起来可能发生了错误，妨碍了服务器的处理
数字5开头的：表示服务器在处理请求的过程中有错误或者异常状态发生，也有可能是服务器意识到以当前的软硬件资源无法完成对请求的处理



请求/响应的头部



Accept: 接收类型，表示浏览器支持的MIME类型（对标服务端返回的Content-Type）

Accept-Encoding：浏览器支持的压缩类型,如gzip等,超出类型不能接收

Content-Type：客户端发送出去实体内容的类型

Cache-Control: 指定请求和响应遵循的缓存机制，如no-cache

If-Modified-Since：对应服务端的Last-Modified，用来匹配看文件是否变动，只能精确到1s之内，http1.0中

Expires：缓存控制，在这个时间内不会请求，直接使用缓存，http1.0，而且是服务端时间

Max-age：代表资源在本地缓存多少秒，有效时间内不会请求，而是使用缓存，http1.1中

If-None-Match：对应服务端的ETag，用来匹配文件内容是否改变（非常精确），http1.1中

Cookie：有cookie并且同域访问时会自动带上

Connection：当浏览器与服务器通信时对于长连接如何进行处理,如keep-alive

Host：请求的服务器URL

Origin：最初的请求是从哪里发起的（只会精确到端口）,Origin比Referer更尊重隐私

Referer：该页面的来源URL(适用于所有类型的请求，会精确到详细页面地址，csrf拦截常用到这个字段)

User-Agent：用户客户端的一些必要信息，如UA头部等


常用的响应头部



Access-Control-Allow-Headers: 服务器端允许的请求Headers

Access-Control-Allow-Methods: 服务器端允许的请求方法

Access-Control-Allow-Origin: 服务器端允许的请求Origin头部（譬如为*）

Content-Type：服务端返回的实体内容的类型

Date：数据从服务器发送的时间

Cache-Control：告诉浏览器或其他客户，什么环境可以安全的缓存文档

Last-Modified：请求资源的最后修改时间

Expires：应该在什么时候认为文档已经过期,从而不再缓存它

Max-age：客户端的本地资源应该缓存多少秒，开启了Cache-Control后有效

ETag：请求变量的实体标签的当前值

Set-Cookie：设置和页面关联的cookie，服务器通过这个头部把cookie传给客户端

Keep-Alive：如果客户端有keep-alive，服务端也会有响应（如timeout=38）

Server：服务器的一些相关信息

请求头部和响应头部是有对应关系的：例如
1.请求头部的 Accept要和响应头部的 Content-Type匹配，否则会报错。
2.跨域请求时，请求头部的 Origin要匹配响应头部的 Access-Control-Allow-Origin，否则会报跨域错误。
3.在使用缓存时，请求头部的 If-Modified-Since、 If-None-Match分别和响应头部的 Last-Modified、 ETag对应。

更多的对应关系请读者自行搜索。

请求/响应实体

做http请求时，除了头部，还有消息实体，一般来说，请求实体中会将一些需要的参数都放入进入（用于post请求）。譬如实体中可以放参数的序列化形式（ a=1&b=2这种），或者直接放表单对象（ FormData对象，上传时可以夹杂参数以及文件），等等。

而一般响应实体中，就是放服务端需要传给客户端的内容。一般现在的接口请求时，实体中就是对于的信息的json格式。


cookie以及优化

cookie是浏览器的一种本地存储方式，一般用来帮助客户端和服务端通信的，常用来进行身份校验，结合服务端的session使用。

常用的场景如下：

用户登陆后，服务端会生成一个session，session中有对于用户的信息（如用户名、密码等），然后会有一个sessionid（相当于是服务端的这个session对应的key），然后服务端在登录页面中写入cookie，值就是:jsessionid=xxx，然后浏览器本地就有这个cookie了，以后访问同域名下的页面时，自动带上cookie，自动检验，在有效时间内无需二次登陆。

一般来说，cookie是不允许存放敏感信息的（千万不要明文存储用户名、密码），因为非常不安全，如果一定要强行存储，首先，一定要在cookie中设置 httponly（这样就无法通过js操作了）。

另外，由于在同域名的资源请求时，浏览器会默认带上本地的cookie，针对这种情况，在某些场景下是需要优化的。

例如以下场景：

客户端在域名A下有cookie（这个可以是登陆时由服务端写入的）

然后在域名A下有一个页面，页面中有很多依赖的静态资源（都是域名A的，譬如有20个静态资源）

此时就有一个问题，页面加载，请求这些静态资源时，浏览器会默认带上cookie

也就是说，这20个静态资源的http请求，每一个都得带上cookie，而实际上静态资源并不需要cookie验证

此时就造成了较为严重的浪费，而且也降低了访问速度（因为内容更多了）



当然了，针对这种场景，是有优化方案的（多域名拆分）。具体做法就是：

将静态资源分组，分别放到不同的子域名下
而子域名请求时，是不会带上父级域名的cookie的，所以就避免了浪费
说到了多域名拆分，这里再提一个问题，那就是：

在移动端，如果请求的域名数过多，会降低请求速度（因为域名整套解析流程是很耗费时间的，而且移动端一般带宽都比不上pc）
此时就需要用到一种优化方案： dns-prefetch（让浏览器空闲时提前解析dns域名，不过也请合理使用，勿滥用）

gzip压缩

首先，明确 gzip是一种压缩格式，需要浏览器支持才有效（不过一般现在浏览器都支持），而且gzip压缩效率很好（高达70%左右）。然后gzip一般是由 apache、 tomcat等web服务器开启。

当然服务器除了gzip外，也还会有其它压缩格式（如deflate，没有gzip高效，且不流行），所以一般只需要在服务器上开启了gzip压缩，然后之后的请求就都是基于gzip压缩格式的，非常方便。

长连接与短连接

首先看 tcp/ip层面的定义：

长连接：一个tcp/ip连接上可以连续发送多个数据包，在tcp连接保持期间，如果没有数据包发送，需要双方发检测包以维持此连接，一般需要自己做在线维持（类似于心跳包）
短连接：通信双方有数据交互时，就建立一个tcp连接，数据发送完成后，则断开此tcp连接
然后在http层面：

http1.0中，默认使用的是短连接，也就是说，浏览器没进行一次http操作，就建立一次连接，任务结束就中断连接，譬如每一个静态资源请求时都是一个单独的连接
http1.1起，默认使用长连接，使用长连接会有这一行 Connection:keep-alive，在长连接的情况下，当一个网页打开完成后，客户端和服务端之间用于传输http的tcp连接不会关闭，如果客户端再次访问这个服务器的页面，会继续使用这一条已经建立的连接
注意： keep-alive不会永远保持，它有一个持续时间，一般在服务器中配置（如apache），另外长连接需要客户端和服务器都支持时才有效。



http2.0



http2.0不是https，它相当于是http的下一代规范（譬如https的请求可以是http2.0规范的）。然后简述下http2.0与http1.1的显著不同点：

http1.1中，每请求一个资源，都是需要开启一个tcp/ip连接的，所以对应的结果是，每一个资源对应一个tcp/ip请求，由于tcp/ip本身有并发数限制，所以当资源一多，速度就显著慢下来
http2.0中，一个tcp/ip请求可以请求多个资源，也就是说，只要一次tcp/ip请求，就可以请求若干个资源，分割成更小的帧请求，速度明显提升。
所以，如果http2.0全面应用，很多http1.1中的优化方案就无需用到了（譬如打包成精灵图，静态资源多域名拆分等）。
然后简述下http2.0的一些特性：

多路复用（即一个tcp/ip连接可以请求多个资源）
首部压缩（http头部压缩，减少体积）
二进制分帧（在应用层跟传送层之间增加了一个二进制分帧层，改进传输性能，实现低延迟和高吞吐量）
服务器端推送（服务端可以对客户端的一个请求发出多个响应，可以主动通知客户端）
请求优先级（如果流被赋予了优先级，它就会基于这个优先级来处理，由服务器决定需要多少资源来处理该请求。）
https

https就是安全版本的http，譬如一些支付等操作基本都是基于https的，因为http请求的安全系数太低了。

简单来看，https与http的区别就是： 在请求前，会建立ssl链接，确保接下来的通信都是加密的，无法被轻易截取分析

一般来说，如果要将网站升级成https，需要后端支持（后端需要申请证书等），然后https的开销也比http要大（因为需要额外建立安全链接以及加密等），所以一般来说http2.0配合https的体验更佳（因为http2.0更快了）

一般来说，主要关注的就是SSL/TLS的握手流程：

1.浏览器请求建立SSL链接，并向服务端发送一个随机数–Client random和客户端支持的加密方法，比如RSA加密，此时是明文传输。



2.服务端从中选出一组加密算法与Hash算法，回复一个随机数–Server random，并将自己的身份信息以证书的形式发回给浏览器 （证书里包含了网站地址，非对称加密的公钥，以及证书颁发机构等信息）



3.浏览器收到服务端的证书后

    验证证书的合法性（颁发机构是否合法，证书中包含的网址是否和正在访问的一样），如果证书信任，则浏览器会显示一个小锁头，否则会有提示

    用户接收证书后（不管信不信任），浏览会生产新的随机数–Premaster secret，然后证书中的公钥以及指定的加密方法加密 Premastersecret，发送给服务器。

    利用Client random、Server random和Premaster secret通过一定的算法生成HTTP链接数据传输的对称加密key- session key

    使用约定好的HASH算法计算握手消息，并使用生成的 session key对消息进行加密，最后将之前生成的所有信息发送给服务端。



4.服务端收到浏览器的回复

    利用已知的加解密方式与自己的私钥进行解密，获取 Premastersecret

    和浏览器相同规则生成 session key

    使用 session key解密浏览器发来的握手消息，并验证Hash是否与浏览器发来的一致

    使用 session key加密一段握手消息，发送给浏览器

5.浏览器解密并计算握手消息的HASH，如果与服务端发来的HASH一致，此时握手过程结束，



之后所有的https通信数据将由之前浏览器生成的 session key并利用对称加密算法进行加密。



http的缓存

前后端的http交互中，使用缓存能很大程度上的提升效率，而且基本上对性能有要求的前端项目都是必用缓存的。

强缓存与弱缓存
缓存可以简单的划分成两种类型： 强缓存（ 200fromcache）与 协商缓存（ 304）
区别如下：

强缓存（ 200fromcache）时，浏览器如果判断本地缓存未过期，就直接使用，无需发起http请求
协商缓存（ 304）时，浏览器会向服务端发起http请求，然后服务端告诉浏览器文件未改变，让浏览器使用本地缓存
对于协商缓存，使用 Ctrl+F5强制刷新可以使得缓存无效。但是对于强缓存，在未过期时，必须更新资源路径才能发起新的请求（更改了路径相当于是另一个资源了，这也是前端工程化中常用到的技巧）。

缓存头部简述
上述提到了强缓存和协商缓存，那它们是怎么区分的呢？答案是通过不同的http头部控制。
缓存中常用的几个头部：

If-None-Match/E-tag
If-Modified-Since/Last-Modified
Cache-Control/Max-Age
Prama/Expires
属于强缓存控制的：

(http1.1) Cache-Control/Max-Age
(http1.0) Pragma/Expires
注意： Max-Age不是一个头部，它是 Cache-Control头部的值。

属于协商缓存控制的：

(http1.1) If-None-Match/E-tag
(http1.0) If-Modified-Since/Last-Modified
可以看到，上述有提到 http1.1和 http1.0，这些不同的头部是属于不同http时期的。

头部的区别

首先明确，http的发展是从http1.0到http1.1，而在http1.1中，出了一些新内容，弥补了http1.0的不足。

http1.0中的缓存控制：

Pragma：严格来说，它不属于专门的缓存控制头部，但是它设置 no-cache时可以让本地强缓存失效（属于编译控制，来实现特定的指令，主要是因为兼容http1.0，所以以前又被大量应用）
Expires：服务端配置的，属于强缓存，用来控制在规定的时间之前，浏览器不会发出请求，而是直接使用本地缓存，注意，Expires一般对应服务器端时间，如 Expires：Fri,30Oct199814:19:41
If-Modified-Since/Last-Modified：这两个是成对出现的，属于协商缓存的内容，其中浏览器的头部是 If-Modified-Since，而服务端的是 Last-Modified，它的作用是，在发起请求时，如果 If-Modified-Since和 Last-Modified匹配，那么代表服务器资源并未改变，因此服务端不会返回资源实体，而是只返回头部，通知浏览器可以使用本地缓存。 Last-Modified，顾名思义，指的是文件最后的修改时间，而且只能精确到 1s以内
Max-Age相比Expires？

Expires使用的是服务器端的时间，但是有时候会有这样一种情况-客户端时间和服务端不同步。那这样，可能就会出问题了，造成了浏览器本地的缓存无用或者一直无法过期，所以一般http1.1后不推荐使用 Expires。而 Max-Age使用的是客户端本地时间的计算，因此不会有这个问题，因此推荐使用 Max-Age。
注意，如果同时启用了 Cache-Control与 Expires， Cache-Control优先级高。

E-tag相比Last-Modified？

Last-Modified：
    表明服务端的文件最后何时改变的
    它有一个缺陷就是只能精确到1s，
    然后还有一个问题就是有的服务端的文件会周期性的改变，导致缓存失效
E-tag：
    是一种指纹机制，代表文件相关指纹
    只有文件变才会变，也只要文件变就会变，
    也没有精确时间的限制，只要文件一遍，立马E-tag就不一样了
    如果同时带有 E-tag和 Last-Modified，服务端会优先检查 E-tag。


浏览器接收到http数据包后的解析流程



渲染流程大致如下：

1.解析HTML，构建DOM树
2.解析CSS，生成CSS规则树
3.合并DOM树和CSS规则，生成render树
4.布局render树（Layout/reflow），负责各元素尺寸、位置的计算
5.绘制render树（paint），绘制页面像素信息
6.浏览器会将各层的信息发送给GPU，GPU会将各层合成（composite），显示在屏幕上
找了个图





HTML解析，构建DOM

整个渲染步骤中，HTML解析是第一步。简单的理解，这一步的流程是这样的：浏览器解析HTML，构建DOM树。

Bytes → characters → tokens → nodes → DOM
假设有下面这样一个代码

.. code:: html

 <html>  
    <head>    
        <meta name="viewport" content="width=device-width,initial-scale=1">
        <link href="style.css" rel="stylesheet">
        <title>Critical Path</title>
    </head>
    <body>    
        <p>Hello<span>web performance</span> students!</p>
        <div><img src="awesome-photo.jpg"></div>  
    </body>
 </html>


浏览器的处理如下：





列举其中的一些重点过程：

Conversion转换：浏览器将获得的HTML内容（Bytes）基于他的编码转换为单个字符
Tokenizing分词：浏览器按照HTML规范标准将这些字符转换为不同的标记token。每个token都有自己独特的含义以及规则集
Lexing词法分析：分词的结果是得到一堆的token，此时把他们转换为对象，这些对象分别定义他们的属性和规则
DOM构建：因为HTML标记定义的就是不同标签之间的关系，这个关系就像是一个树形结构一样。例如：body对象的父节点就是HTML对象，然后段略p对象的父节点就是body对象
最后的DOM树如下：







跨域、web安全



跨域



为什么会跨域：

在浏览器同源策略限制下，向不同源（不同协议、不同域名或者不同端口）发送XHR请求，浏览器认为该请求不受信任，禁止请求，具体表现为请求后不正常响应
举个栗子









