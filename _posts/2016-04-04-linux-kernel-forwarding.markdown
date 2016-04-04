---
layout: post
title:  "Linux内核转发技术"
date:   2016-04-03 17:01:00
comments: true
categories: Tech
---

什么? linux? 内核?! 也许你会说,"拜托,这种一看就让人头大的字眼, 我真的需要了解吗?"
有句流行语说得好,没有买卖,就没有杀害. 如果在日常中需要和流量打交道,那么为了不让
自己在面对来往的海量数据时手足无措, 最好还是了解一下如何管理,过滤和转发这些数据,
让信息能被传递到该去的地方. 哪怕自己不去亲自参与管理, 也可以通过简单的了解,
让自己在围观时多一份自信,少一点懵逼.

## 前言

在linux内核中,通常集成了带有封包过滤和防火墙功能的内核模块, 不同内核版本的模块名称不同,
在2.4.x版本及其以后的内核中, 其名称为`iptables`, 已取代了早期的`ipchains`和远古时期的`ipfwadm`.
在命令行中可以通过`lsmod | grep -i iptable`来查看当前加载的相关模块信息.

iptables作为内核模块, 由一些信息包过滤表组成,这些表包含内核用来控制信息包过滤处理的规则集.
与此同时, iptables也作为用户空间(userspace)的一个管理工具而存在,使得我们使插入,
修改和除去信息包过滤表中的规则变得非常容易,不需要每次修改规则后都重新编译内核.
本文主要讨论的也正是iptables在用户空间的功能.

## 基本概念

linux内核的转发机制主要通过查`表(tables)`来完成, 而iptables则用来设置,管理和检查linux内核中ip包过滤规则表.
table后面加了s说明可以定义多张表, 而每张表中又包含了若干`链路(chains)`, 链路表示一系列应用于匹配ip包的`规则(rules)`.
下面就对这"三座大山"分别加以解释. 为了使概念明了, 我们自底向上说明:

### 规则(rules)

规则又称为rule-specification, 其主要作用是匹配特定的ip封包, 并作出相应的动作, 其格式为:

```
rule-specification = [matches...] [target]
```
其中,

```
match = -m matchname [per-match-options]
target = -j targetname [per-target-options]
```
matchname表示匹配的格式的名称, targetname表示所执行的动作的名称. iptables定义了一系列内置的格式和动作,
如target为accept表示接受, masquerade表示执行类似路由器的动作(用于nat)等, 具体可以通过`man iptables-extensions`查看.

### 链路(chains)

所谓链路, 顾名思义就是表示ip数据包传输的路径, 一个封包的源和目的不同, 其走的路径即有可能不同,
就像路途中的朋友们, 在任何一个节点都有可能分道扬镳.这些节点有:

 - pre_routing : 外部数据刚刚进入时.
 - post_routing : 外部数据准备离开时.
 - input    : 数据包的目的地址为本地socket.
 - output   : 数据包由本地生成.
 - forward  : 数据包被本机转发.

事实上, 链路在内核中以钩子的形式存在, 在每个结点给用户预留了回调函数来处理封包(即用前面提到的规则).
ip封包从外部进入后,所经过的链路如下图所示:

![](http://images2015.cnblogs.com/blog/676200/201604/676200-20160402195959160-889058479.jpg)

网口接收到ip封包后, 首先经过mangle和nat表的pre_routing表的处理, 然后判断是否目的地址为本机的应用程序, 
若是则往左边的路径往下, 由接收程序处理完后再发出. 在未开启内核转发的情况下, 目的地址不为本机的ip包都会丢弃掉, 
若开启了转发则往右边路径将其从网口转发出去. 在图中每个链路点都能对ip包做相应的修改和过滤.

> **注意:** prerouting链只会匹配流的第一个包,也就是说,这个流的所有其他的包都不会被此链检查. 
> 因此prerouting链只能做网络地址转换,不能被用来做任何过滤.

### 表(tables)

当前最新版本的iptables中包含了五个独立的表, 工作时使用哪张表取决于内核选项以及当前应用了哪个模块. 各表说明如下:

- **filter**: filter表为(iptables命令)默认使用的表, 包含input,forward和output链路.
- **nat**: 当遇到一个创建了新链接的ip包时, 内核就会查找nat表, 其包含了prerouting和postrouting链路.
- **mangle**: mangle表用于专门的封包修改,如改变tos,ttl,mark等. 在内核2.4.17之前只包含prerouting和output链路,
在之后的版本中增加了input,forward和postrouting链路.
- **raw**: raw主要用来在连接跟踪中配置notrack行为, 其在netfilter的hook注册了更高优先级的回调,因此可以在ip_conntrack表
亦即其他ip表之前被调用. raw包含了prerouting和output链路.
- **security**: security表用于命令访问控制(mandatory access control, mac)网络的规则. mac由linux安全模块如selinux实现,
security表在filter表之后调用, 提供了input,output和forward链路.

## 具体应用

工具的产生终究要服务于生产, 光解释名词也不能形象地展现linux强大的内核转发机制,因此以几个小例子来说明iptables的
具体使用, 并依据上述介绍来写出有实际效用的脚本. iptables命令的一般格式如下:

    iptables [-t table] {-a|-c|-d} chain rule-specification

其中命令分为三部分,亦即上面说到的指定表,链路和规则

    -t table指定表的名字, 若不指定则默认为filter.
    -a chain表示在链路中增加规则, -c和-d分别表示检查和删除.
    剩余部分指定规则, 格式为`[matches...] [target]`

完整的命令可以通过iptables的manpages查看.

### 例1.作为防火墙

假设这么一种场景, 我们连接上了一个烦人的局域网, 为什么说它烦人呢? 因为局域网内有很多脚本小子,
来来回回扫描不说, 还在某些端口进行爆破. 因此我想简单生成一个防火墙, 除了网关不允许子网内任何其他的
ip对我进行连接, 甚至连ping都ping不到我.

需求明确, 那么如何实现呢? 其实很简单, 只需要以下命令(假设子网为192.168.1.0/24): 

```bash
#1. 清空现有规则
iptables -t filter -f
#2. 打开网关访问权限
iptables -t filter -a input -s 192.168.1.1 -j accept
#3. 指定不过滤ping的返回
iptables -t filter -a input -p icmp --icmp-type 0 -s 192.168.1.0/24 -j accept
#4. 关闭其他所有内网访问权限
iptables -t filter -a input -p all -s 192.168.1.0/24 -j drop
```
一般来说防火墙策略一般是从非信任->信任,先用策略(-p policy)关闭所有访问权限,再添加规则按需要逐条打开.
这里为了简单就在默认策略(accept)的基础上添加规则. 各个表的当前策略可以通过`iptables -t table -s`查看.
要注意的是所有规则是**按顺序检查**的, 一旦检查到符合的条件就会执行,而不往下继续检查,如果所有规则都不匹配,
则会执行默认的操作(默认策略).因此在逐条添加规则的时候最好是从小到大添加. 
在#3命令中,我们打开了icmp type为0的输入,即ping echo reply封包, 这样别人ping不到我的同时,我却能ping到别人,是不是很方便?

### 例2.作为路由器

在管制的网络中经常有这么一种情况, 即内网是绑定mac地址的, 客户端要接入路由器必须要网络管理员添加,
因为即使有wifi密码,连接上之后也无法获取ip,因而不能上网. 我的pc已经被添加到网络中,但是手机,平板之类
的设备不想一一要网管去添加, 那该怎么连上wifi呢? 解决办法有很多, 在windows下有各种xxx-wifi软件, linux的
networkmanager也有类似添加热点的解决方案. 这里讲的是iptables的解决办法.

现假定笔记本启动了热点at0,并已配置好dhcp服务. 我们的子网还是192.168.1.0/24, 热点的子网为10.0.0.0/24, 
共用同一块无线网卡wlan0. 此时子网的请求会到wlan0上, 但是目的地址不是我, 根据上图可以知道, 这时
ip封包应该往右边的路径转发出去, 不过需要出去前改变一下网络地址(即nat, 详见[p2p通信原理与实现][nat]).
设置nat转发的规则也很简单:

```bash
iptables -t nat -a postrouting -o wlan0 -j masquerade
```
这是在当我们既用wlan0上网,也用wlan0做路由器的时候配置的nat规则,但是这样性能会不太理想,
更普遍的情况是我们用一个网卡连接网络(假设为wlan0), 另一个网卡作为路由器(设为wlan1), 
这种情况下只需要将wlan1的流量转发到wlan0上:

```
iptables -t filter -a forward -i wlan1 -o wlan0 -j accept
iptables -t filter -a forward -i wlan0 -o wlan1 -m state --state established,related -j accept
iptables -t nat -a postrouting -o wlan0 -j masquerade
```
其中masquerade表示提供一种类似路由器的转发行为,即为出去的tcp/udp包改变源地址,为进来的包改变目的地址,
用-j snat可以实现同样功能, 只不过ip地址需要自己指定(这里为wlan0在内网中的地址). masquerade被专门设计
用于那些动态获取ip地址的连接,比如拨号上网,dhcp连接等.如果你有静态ip,使用snat target可以减少开销.

```
iptables -t nat -a postrouting -o wlan0 -p tcp -j snat --to-source [wlan0-ip]
# 这里不需要设置dnat, 因为snat会记住连接,把响应转发给对应的请求.不过为了例示还是写出来:
iptables -t nat -a prerouting -i wlan0 -d [wlan0-ip] -p tcp -j dnat --to [client-ip]
```
这里值得一提的是, iptables本质上只是过滤和处理数据, 所以准确说是**允许**将wlan1的流量转发到wlan0上,
事实上如果用默认策略, forward都是允许的, 不用额外设置.

### 例3.作为透明代理

不同的人对代理有不同的需求, 最常见的就是http代理, 一般提供了地址和端口号. 我们在浏览器中配置使用
代理并指定地址和端口后, 上网冲浪的请求会经过代理服务器接收,然后根据需要会从为我们去向目的网站请求内容,
或者从缓存中直接给我们返回内容. 假设我们现在已经配置好并运行了[squid][squid]代理服务器, 工作在3128端口.
和作为路由器类似, 不过除了改变ip还需要改变目的端口号:

```
iptables -t nat -a prerouting -i wlan1 -p tcp --dport 80 -j dnat --to [wlan0-ip]:3128
iptables -t nat -a prerouting -i wlan0 -p tcp --dport 80 -j redirect --to-port 3128
```

透明代理完整的iptables配置可以参考[set up squid in linux][linux-squid]. 

## 后记

对于linux内核转发的技术介绍感觉差不多了, 虽然没有完全表现出其强大的功能, 
但相信有需要的人可以根据基本规则来举一反三; 通过google查看别人的iptables"脚本",
也能获得很多灵感. 另外关于iptables的更多具体命令,要记得多使用简单的办法查阅(*man iptables*).

## 参考资料

- [iptables指南][ipts]
- [iptables/netfilter原理][littlenan]


博客地址:

- [pannzh.github.io](http://pannzh.github.io)
- [有价值炮灰-博客园](http://www.cnblogs.com/pannengzhi/)

*欢迎交流,文章转载请注明出处.*

[nat]:https://pannzh.github.io/tech/p2p/2015/10/31/p2p-over-middle-box.html
[ipts]:https://www.frozentux.net/iptables-tutorial/cn/iptables-tutorial-cn-1.1.19.html
[littlenan]:http://www.cnblogs.com/littlehann/p/3708222.html
[squid]:http://www.squid-cache.org/
[linux-squid]:http://www.cyberciti.biz/tips/linux-setup-transparent-proxy-squid-howto.html
