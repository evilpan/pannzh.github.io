---
layout: post
title:  "P2P通信标准协议(四)之SIP"
date:   2016-01-05 22:11:00
comments: true
categories: Tech P2P
---

在前面[几篇文章][p2p]中我们介绍了建立p2p通信的一般协议(簇),以及一种完整的NAT传输解决方案ICE,
但是对于多用户的通信情况,还有一些通用协议来实现标准化的管理,如之前讲过的SDP和SIP等,[SIP(Session Initiation Protocol)][rfc-sip],
是属于应用层的控制协议,主要用于在一个或多个参与者之间创建,修改和中止会话(sessions).会话的类型包括IP电话,
多媒体流分发和多媒体会议等.

## SIP简介

SIP邀请(invitations)用于创建携带会话描述(如SDP信息)的会话,允许参与者使用一系列兼容的媒体类型.
SIP使用一种叫代理服务器的元素来帮助对用户当前位置进行转发,对用户进行验证和授权,并为用户提供相应的功能.
SIP同时也提供了注册函数以允许用户上传他们的当前地址供代理服务器使用.SIP协议运行在多个不同的传输协议之上.

SIP支持5个方面来建立和中止多媒体会话:

- 用户地址(User location): 决定了用来通讯的终端系统.
- 用户状态(User availability): 决定了被呼叫端的是否愿意加入通讯.
- 用户性能(User capabilities): 决定了多媒体类型和媒体使用的参数.
- 会话建立(Session setup): "响铃",在呼叫端和被呼叫端建立起会话.
- 会话管理(Session management): 包括传输和中止会话,修改会话参数以及调用服务.

SIP不是一个垂直集成的通讯系统,而是作为一个组件与其他协议共同运作,如RTP等实时传输协议等.另外SIP不提供服务,
只提供可以用来实现各种服务的原语.比如,SIP可以定位用户并且传输一个不透明的对象到其当前地址.如果这个原语用来
传输SDP,终端就能得知会话的一些参数;如果同样的原语用来传输一张照片,那也可以实现一种"显示来电者头像"的服务.
由此可见,一种原语通常用来实现多种不同的服务.

## SIP工作过程

下图描述了SIP的基本功能:定位一个终端,产生通讯请求,建立会话以及结束会话.

                     atlanta.com  . . . biloxi.com
                 .      proxy              proxy     .
               .                                       .
       Alice's  . . . . . . . . . . . . . . . . . . . .  Bob's
      softphone                                        SIP Phone
         |                |                |                |
         |    INVITE F1   |                |                |
         |--------------->|    INVITE F2   |                |
         |  100 Trying F3 |--------------->|    INVITE F4   |
         |<---------------|  100 Trying F5 |--------------->|
         |                |<-------------- | 180 Ringing F6 |
         |                | 180 Ringing F7 |<---------------|
         | 180 Ringing F8 |<---------------|     200 OK F9  |
         |<---------------|    200 OK F10  |<---------------|
         |    200 OK F11  |<---------------|                |
         |<---------------|                |                |
         |                       ACK F12                    |
         |------------------------------------------------->|
         |                   Media Session                  |
         |<================================================>|
         |                       BYE F13                    |
         |<-------------------------------------------------|
         |                     200 OK F14                   |
         |------------------------------------------------->|
         |                                                  |

                         图 1: SIP会话建立

图中描述了两个用户Alice和Bob交换SIP信息的过程.(信息表示为Fn.) 首先,Alice在其PC上使用了SIP终端(假设是软件电话),
并且通过互联网打给Bob. 其中我们看到有两个代理服务器atlanta和biloxi,用来帮助双方进行会话建立.这个典型的排列经常
被称为`SIP之梯(SIP trapezoid)`.

Alice呼叫Bob时,使用的是Bob的SIP身份信息,一种特定类型URI称为SIP URI,形式和E-mail地址类似,包含了用户名和主机名.
在本例中,Bob的地址为`sip:bob@biloxi.com`,biloxi是Bob的SIP服务提供商;同样,Bob联系Alice时也通过其SIP地址`sip:alice@atlanta.com`
来进行通信. SIP同样提供了安全的链接SIPS,和HTTPS类似,主要通过TLS进行内容加密, 加密的地址格式为`sips:alice@atlanta.com`

SIP基于一种类HTTP的请求/响应传输模型.每次传输包含一个调用了特定方法或函数的请求,以及至少一个响应.在本例中,
传输开始时Alice发送了一个INVITE请求到Bob的SIP URI. INVITE请求包含一系列头部(header)字段.头部字段被称为属性,
提供了关于报文的额外信息. 产生INVITE请求的终端包含了一个独特的通话标识符,目的地址,Alice的地址以及Alice
希望与Bob建立的会话的类型信息. 一个INVITE请求的例子如下,其中Alice的SDP信息没有显示出来:

      INVITE sip:bob@biloxi.com SIP/2.0
      Via: SIP/2.0/UDP pc33.atlanta.com;branch=z9hG4bK776asdhds
      Max-Forwards: 70
      To: Bob <sip:bob@biloxi.com>
      From: Alice <sip:alice@atlanta.com>;tag=1928301774
      Call-ID: a84b4c76e66710@pc33.atlanta.com
      CSeq: 314159 INVITE
      Contact: <sip:alice@pc33.atlanta.com>
      Content-Type: application/sdp
      Content-Length: 142

      (Alice的SDP信息,略)

                         图 2: Alice发送的请求报文

其中第一行包含了方法的名字(INVITE).后面的一些行则是一系列头部区段,各个头部字段的含义在下一节会说到.

由于Alice不知道Bob的准确地址,因此报文会先发送到Alice的SIP服务提供商,atlanta.com. 这个地址是可以在Alice
的终端(软件电话)上面进行配置的,当然也可以通过DHCP之类的协议来发现. SIP服务器接收SIP请求并按照其目的
进行转发. 在本例中, 代理服务器接收INVITE请求后,给Alice返回100(Trying)响应,表示请求正在进行转发.
SIP响应用一个三位数来表示状态,包含了和INVITE请求中同样的To, From, Call-ID, CSeq 和 branch(via内)参数,
从而允许Alice的终端将其与请求相联系. 代理服务器atlanta.com通过DNS等方法得到Bob的服务提供商地址.
并且在转发的报文中的via字段加上自己的地址信息. biloxi.com代理服务器接收到INVITE请求,并且返回100响应给
atlanta.com. 代理服务器查找其数据库(通常称为定位服务),其中包含了Bob的当前IP地址. 同时代理在转发请求前
也在头部的via字段加上自己的地址.

Bob的终端(SIP电话)接收到INVITE请求后,会提示Bob这是来自Alice的来电.同时Bob的终端返回180响应,
表示正在呼叫,响应一直转发回到Alice的终端,从而使Alice也能知道对方电话正在响.每个代理都通过头部的Via
字段来决定响应的发送方向,并且从via顶部去掉自己的地址信息. 因此虽然发送请求的时候用到DNS和定位服务,
但是发送响应的时候则不需要.

在本例中,Bob决定接起电话. 此时Bob的SIP电话发送200响应表示呼叫被应答.200响应包含了信息体(SDP)
表明Bob希望建立的会话类型.因此,这形成了两次SDP信息交换过程:Alice发送给Bob,然后Bob发送给Alice.
这个两次交换提供了基本的协商能力,并且基于简单的[offer/answer模型][rfc-offer-answer]. 如果Bob
不想接电话或者正在与别人通话,就会返回一个错误响应,从而不建立多媒体会话. Bob发送的200响应结构大体如下:

      SIP/2.0 200 OK
      Via: SIP/2.0/UDP server10.biloxi.com
         ;branch=z9hG4bKnashds8;received=192.0.2.3
      Via: SIP/2.0/UDP bigbox3.site3.atlanta.com
         ;branch=z9hG4bK77ef4c2312983.1;received=192.0.2.2
      Via: SIP/2.0/UDP pc33.atlanta.com
         ;branch=z9hG4bK776asdhds ;received=192.0.2.1
      To: Bob <sip:bob@biloxi.com>;tag=a6c85cf
      From: Alice <sip:alice@atlanta.com>;tag=1928301774
      Call-ID: a84b4c76e66710@pc33.atlanta.com
      CSeq: 314159 INVITE
      Contact: <sip:bob@192.0.2.4>
      Content-Type: application/sdp
      Content-Length: 131
      
      (Bob的SDP信息,略)

                         图 3: Bob发送的响应报文

Bob的SIP电话增加了一个tag参数到报文头部,这个tag会被两个端点合并到对话里,并且会在(本次通话)所有以后的请求和响应中包含.
Contact头包含了一个Bob能直接连接的URI,Content-Type 和 Content-Length表示消息体(没贴出来)的格式信息.
在本例中,代理服务器也可以拓展自己的功能,比如当接收到Bob返回的486(Busy Here)响应,则可以向Bob的语音信箱等
发送INVITE请求;一个代理服务器可以同时向多个地址发送请求,这种并行查找的特性通常称之为分叉(forking).

在200(OK)响应返回到Alice的软件电话上之后,电话停止响铃,并通知Alice对方已经接听,同时发送一个ACK报文到Bob
的终端,表示响应已经收到. 在本例中,ACK直接发送给Bob,而不通过两个代理服务器,是因为两个端点都知道了对方的地址,
因此不需要再通过代理去查找.

收到ACK之后,Alice和Bob就可以互相通信了. 通信完成之后,假设Bob先挂断电话,并产生一个BYE报文,直接发送给Alice,
Alice收到后确认请求,并返回200(OK)响应,从而结束此次会话.注意这里没有发送ACK,因为ACK只有在确认INVITE请求的响应时才发送.

注册(Registration)是另一个SIP常用的操作. 用户通过注册使得代理服务器能知道其当前的地址信息. 例如Bob
可以在初始化时,向biloxi.com发送注册请求(`REGISTER`),后者也称为注册商(registrar).
注册商会将Bob的SIP URI和其当前地址相关联起来(通常称为绑定),并把这个映射信息存储到服务器端的数据库中,
亦即上文说到的定位服务. 通常注册服务器和对应域名的代理服务器都是同一地址的,因此要知道SIP服务器的类型的
区别体现在逻辑上而不是物理上.

## SIP协议结构

SIP是一个分层的协议,这意味着其行为由一系列同级但独立的段(stage)描述. SIP的最底层为语法和编码,其中编码由
BNF语法(Backus-Naur Form grammar)指定; SIP第二层为为运输层(transport layer),定义了客户端和服务端如何发送和接收
请求和响应;第三层为事务层(transaction layer),事务层是SIP的基础组件,一次事务包括发送的请求和对应的响应,
事务层处理应用层的重传,请求/响应匹配和超时等;事务层之上称为事务用户(TU, transaction user),每个SIP
实体(除了无状态的代理),都是一个TU.

所有的SIP元素,包括用户客户端(UAC),服务器(UAS),无状态(stateless)或者全状态(stateful)的代理,
以及注册商,都包含一个区分彼此的内核(core). 其中除了无状态的代理,其他元素的内核都是事务用户.
UAC和UAS的内核行为依赖于方法,对于所有方法有一些通用规则,这里不细说. 对于UAC而言,这些规则支配
着请求报文的构造.

## SIP报文格式

SIP是基于文本(text-based)的协议,并且使用UTF-8字符集.一条SIP报文要么是从客户端到服务端的请求,
要么是服务端到客户端的响应;两种类型的报文都包含一个起始行,一个或者多个头部区域,一个表示头部结束的空行,
以及(可选的)正文部分(message body),每个部分以CRLF隔开:

         generic-message  =  start-line
                             *message-header
                             CRLF
                             [ message-body ]
         start-line       =  Request-Line / Status-Line

除了字符集的区别,大多数SIP的报文和头部语法都与HTTP/1.1相同,虽然如此,但SIP不是HTTP协议的拓展.

### SIP Request

SIP请求的报文首行都包含一个`请求行(Request-Line)`,请求行又包括方法名,请求URI以及协议版本,
并以SP(空格)分割除了在行尾,请求行不允许出现任何回车(CR)和换行(LF),元素中也不能出现行间的空字符(LWS, linear whitespace).

      Request-Line  =  Method SP Request-URI SP SIP-Version CRLF

其中:

- Method: 表示方法,RFC3261定义了六个方法,分别是:
    - REGISTER: 用来注册联系人信息.
    - INVITE, ACK, CANCEL, BYE: 这四个方法用于会话的建立.
    - OPTIONS: 用来发现服务器的性能(capabilities).
- Request-URI: 即SIP或者SIPS URI,用来表示请求要送往的服务或用户信息,其中**不能包括**控制字符,
    也不能包含在"<>"之中.
- SIP-Version: SIP的版本号,与RFC3261对应的是"SIP/2.0".和HTTP/1.1的处理类似,但不同点为SIP处理
    版本号是以字符串的格式,虽然这在实践中并没什么太大关系.

### SIP Response

SIP响应与请求不同,其起始行为`状态行(Status-Line)`,状态行包括协议版本,状态码以及对应的状态文字说明,
和请求行类似,个元素以空格分隔,中间也不能出现换行和回车.

    Status-Line  =  SIP-Version SP Status-Code SP Reason-Phrase CRLF

状态码(Status-Code)由3位数字组成,表示请求的结果. 状态码的第一位表示响应的种类:

- 1xx(表示100-199,下同): 临时响应(Provisional),表示请求已经被收到,但还在处理之中.
- 2xx: 成功(Success), 请求被成功接收,理解以及被接受.
- 3xx: 重定向(Redirection), 可能需要重新选择发送地址以完成请求.
- 4xx: 客户端错误(Client Error), 请求包含错误的语法,或者不能被服务器完成.
- 5xx: 服务端错误(Server Error), 服务器处理一个合法的请求失败.
- 6xx: 失败(Global Failure), 请求在任何服务器都无法被处理.

### Header Fields

SIP报文的头部和HTTP的头类似, 也有同样的性质,如在多个头部区域指定同一个属性的值时可以合并成一个头部,
并使field-value以逗号分隔等,头部的格式如下:

      field-name: field-value

冒号两边可以加任意空格,但是一般不建议这样做,而是使filed-name和冒号间不留空格,并使冒号和field-value只见留一个空格.
以图2中Alice的Request请求报文为例,大致介绍其中一些常见的Header field-name:

- Via: 包含了发送方想接受此次请求对应响应的地址,这里是pc33.atlanta.com, 
        并且还包含了识别此次传输事务的分支参数(branch parameter).
- Max-Forwards: 用来限制请求的最大跳数(max hops),在每个hop之后递减少.
- To: 包含了此次请求的目的用户的显示姓名Bob(display name)以及SIP/SIPS URI(sip:bob@biloxi.com)
- From: 包含了此次请求的发送方的显示姓名Alice和URI, 除此之外还有一个tag参数,包含了随即的字符串,
        将用于添加在URI中,主要用于验证和区分.
- Call-ID: 包含了此次通话中全局不同的标识,由随机字符串和发送端的主机名或IP地址组合而成.To,From和Call-ID
           字段完全定义了一个Alice和Bob的端到端的SIP关系,并表示为当前对话(dialog).
- CSeq: 或者写为Command Sequence,包含了一个整数(CSeq号)和方法名,CSeq号在本次对话中随着每次新的请求而递增.
- Contact: 包含代表直接连接Alice的SIP/SIPS URI. 和Via段不同的是,Via告诉其他单位要往那发送响应,
            而Contact告诉其他单位要往哪发(以后的)请求.
- Content-Length: 消息体的长度.
- Content-Type: 消息体(message body)的格式, 如SDP信息则为"application/sdp",关于SDP可以参考前一篇博客[P2P通信标准协议(三)之ICE][ice].

## 后记

本文简单介绍了SIP协议的结构和报文格式, 其中有很多细节都没有深入, 因此篇幅只有[原文/RFC3261][rfc-sip]的十分之一,
如果要根据协议来设计实际的应用,还是需要仔细看一遍协议的原文. 至此, P2P通信系列的介绍也就告一段落了.
P2P的去中心化,一直是个很令人振奋的话题,无论是在信息技术上,还是在金融,政治上,都有无限潜力.
最近的一系列文章主要是P2P入门以及实现简单的VOIP应用, 下一阶段应该会研究下内容分发协议(如bittorrent),
不过以我的拖延症来看,那肯定是很久之后的事了.

[p2p]:http://pannzh.github.io/categories.html#p2p-ref
[stun]:http://pannzh.github.io/tech/p2p/2015/12/12/p2p-standard-protocol-stun.html
[turn]:http://pannzh.github.io/tech/p2p/2015/12/15/p2p-standard-protocol-turn.html
[ice]:http://pannzh.github.io/tech/p2p/2015/12/20/p2p-standard-protocol-ice.html
[classic-stun]:http://www.rfc-editor.org/info/rfc3489
[rfc-stun]:http://www.rfc-editor.org/info/rfc5389
[rfc-turn]:http://www.rfc-editor.org/info/rfc5766
[rfc-ice]:http://www.rfc-editor.org/info/rfc5245
[rfc-sip]:http://www.rfc-editor.org/info/rfc3261
[rfc-offer-answer]:http://www.rfc-editor.org/info/rfc3264
[rfc-sdp]:http://www.rfc-editor.org/info/rfc4566
[TurnServer]:https://github.com/pannzh/TurnServer
[PJSIP]:http://www.pjsip.org
[mitm1]:http://pannzh.github.io/tech/2015/11/01/mitm-detail-1.html
