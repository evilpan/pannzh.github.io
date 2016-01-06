---
layout: post
title:  "成为中间人的几种方式"
date:   2015-11-27 19:15:26
comments: true
categories: Tech MITM
---

前面几篇文章已经介绍了常见的中间人攻击方式，即窃听、修改和重定向（钓鱼）。当然这都是建立在我们完全掌握
了局域网的所有流量的前提下的。在[ARP欺骗与中间人攻击][arp-mitm]中就已经介绍了一种最简单的一种控制流量的
方法，即ARP欺骗。但是ARP的方式往往对网络的负载造成较大压力，从而对别人的网速造成影响不说，如果目标稍微
有点意识，也能很容易发现和防御ARP攻击。因此我们需要其他更多的选择来成为中间人，本文就来介绍一下各种不同的
中间人方式。

## ICMP重定向攻击

ICMP的全称为Internet Control Message Protocol，即网络控制报文协议，被封装在IP层数据中。它是TCP/IP协议族的一个
附属协议，用于在IP主机、路由器之间传递控制消息。控制消息是指网络通不通、主机是否可达、路由是否可用等网络
本身的消息。这些控制消息虽然并不传输用户数据，但是对于用户数据的传递起着重要的作用。

### ICMP原理

ICMP报文主要分为4个部分，分别是类型（type），代码（code），校验和（checksum）和对应类型的数据报（datagram）。
type为0-16，如我们常用的ping命令就基于ICMP协议，type为8（echo ping request）。而code字段为0-5，通常路由器无法转发
IP报文时会给源头发送方发送ICMP目的不可到达报文，code代表发送失败的原因，如0代表`net unreachable`,1代表`host unreachable`
等等。下图是一个简要的概述，协议详情可以查看**RFC792**。

![ICMP](http://images2015.cnblogs.com/blog/676200/201511/676200-20151121194922249-882105359.jpg)

### ICMP攻击方法

由于ICMP协议中有重定向的报文类型，那么我们就可以伪造一个ICMP信息然后发送给局域网中的客户端，并伪装自己是
一个更好的路由通路。从而导致目标所有的上网流量都会发送到我们指定的接口上，达到和ARP欺骗同样的效果。我们接收
到客户端的流量后，经过玩耍再将其转发给真正的网关就可以了。

以ettercap举例实现如下：

    #ettercap  -i wlan0 -Tq -M icmp:11:22:33:44:55/192.168.1.1 

    其中11:22:33:44:55为网关MAC地址，192.168.1.1为网关IP，运行后将会重定向所有经过指定网关的链接。当然也可以指定target。

值得一提的是，ICMP重定向攻击是半双工的，只有客户端被重定向了。也就是说我们只能得到目标到路由器的外出
流量，而无法得到路由器到目标的流量。原因是网关不接受同一内网的重定向请求信息。另外要注意在我们进行中间人活动的
时候，**不要修改payload的长度**，原因也是因为tcp数据无法双向更新。

## DHCP攻击

DHCP相信大家都不会陌生，其全称为Dynamic Host Configuration Protocol，动态主机配置协议。作为局域网的一个网络
协议，主要作用是给内网客户端自动分配IP地址。其中DHCP server的端口号为67，DHCP client端口号为68，采用UDP协议通讯。

### DHCP请求过程

客户端连接到一个局域网后，第一件事就是向网关（DHCP server）请求分配一个可用的IP地址，具体的交互过程如下：

![DHCP](http://images2015.cnblogs.com/blog/676200/201511/676200-20151121211503890-2024287785.jpg)

    1. DHCP Client以广播的方式发出DHCP Discover报文。
    2. 所有的DHCP Server都能够接收到DHCP Client发送的DHCP Discover报文，所有的DHCP Server都会给出响应，向DHCP Client发送一个DHCP Offer报文。
    3. DHCP Offer报文中“Your(Client) IP Address”字段就是DHCP Server能够提供给DHCP Client使用的IP地址，且DHCP Server会将自己的IP地址放在“option”字段中以便DHCP Client区分不同的DHCP Server。DHCP Server在发出此报文后会存在一个已分配IP地址的纪录。
    4. DHCP Client只能处理其中的一个DHCP Offer报文，一般的原则是DHCP Client处理最先收到的DHCP Offer报文。
    5. DHCP Client会发出一个广播的DHCP Request报文，在选项字段中会加入选中的DHCP Server的IP地址和需要的IP地址。
    6. DHCP Server收到DHCP Request报文后，判断选项字段中的IP地址是否与自己的地址相同。如果不相同，DHCP Server不做任何处理只清除相应IP地址分配记录；如果相同，DHCP Server就会向DHCP Client响应一个DHCP ACK报文，并在选项字段中增加IP地址的使用租期信息。
    7. DHCP Client接收到DHCP ACK报文后，检查DHCP Server分配的IP地址是否能够使用。如果可以使用，则DHCP Client成功获得IP地址并根据IP地址使用租期自动启动续延过程；如果DHCP Client发现分配的IP地址已经被使用，则DHCP Client向DHCPServer发出DHCP Decline报文，通知DHCP Server禁用这个IP地址，然后DHCP Client开始新的地址申请过程。
    8. DHCP Client在成功获取IP地址后，随时可以通过发送DHCP Release报文释放自己的IP地址，DHCP Server收到DHCP Release报文后，会回收相应的IP地址并重新分配。
    9. 在使用租期超过某个时刻处，DHCP Client会以单播或者广播形式向DHCP Server发送DHCPRequest报文来续租IP地址。

### DHCP攻击方法

由于一个局域网内可以有多个DHCP服务器，因此我们可以伪装成一个DHCP服务器，在目标发出DHCP请求的时候,抢先为其提供
一个合法的IP地址，这样攻击者就能操纵GW参数并且监控和修改外出的流量。注意DHCP方式的攻击也是半双工的，和ICMP一样
不能修改payload长度。

在本机运行dhcp-server很简单，但是要使得DHCP Offer和DHCP ACK要比网关要快才行，为此有两种策略：

#### 一. 加快自己的DHCP服务器响应速度

ettercap就是这么做的，其不检测IP是否已经分配就立刻响应Client的DHCP Request请求，因此我们要确保分配的ip池是完全
可用的：

    #ettercap -i wlan0 -Tq -M dhcp:192.168.1.150,160-200/255.255.255.0/192.168.1.1

其中192.168.1.150,160-200为ip池，255.255.255.0为子网掩码，192.168.1.1为DNS服务器，这个命令会响应DHCP Offer和Request，
如果想只响应DHCP Request，可以不指定ip池。

#### 二. 减慢原DHCP服务器的响应速度

由于DHCP服务器会响应所有的DHCP请求，我们可以伪造来自不同MAC地址的DHCP Discover或者Request报文使得原来的DHCP服务器耗尽
其ip池，甚至由于忙着响应大量的DHCP请求而陷入瘫痪：

    #yersinia dhcp -attack 1 -interface wlan0

其中-attack 1表示一直发送discover请求来对DHCP服务器进行拒绝服务（DOS）攻击。当然也可以用其他小工具如dhcpstarv来耗尽
网关的DHCP服务器IP池。这时再打开我们的DHCP server并使目标断开重新连接，这样目标自然就只能获得我们分配的IP了。

## 端口盗用攻击

端口盗用（Port Stealing）技术常用来在当交换网络不能进行ARP欺骗（例如固定ARP映射）的时候进行嗅探。在网关为交换机的
情况下，虽然无法进行传统的ARP欺骗（即MAC-IP欺骗），但交换机上同样维护着一个MAC缓存表，可以动态更新。

为此我们了解一下交换机的工作过程：首先交换机内部有一个Port-to-MAC的的缓存表，记录着每一个端口下存在着哪些MAC地址。
这个表一开始是空的，并从来往数据帧中学习和更新。比如PortA端口对应的HostA发送了一个数据，那么HostA的物理地址MACA就会
被记录下来，并且在交换机的Port-to-Mac缓存表中增加一条记录。随后所有发往MACA的数据都会从PortA输出。

与传统的ARP欺骗类似，只不过不同点是我们要污染的是交换机的Port-MAC缓存表。由于该表是动态更新的，我们可以伪造源MAC地址
不断向网关发送数据包，来更新交换机的缓存表，当数据足够多时，原来的PortA-MACA记录就会被冲掉。其结果是交换机会把发给MACA
的数据洪泛发送给所有端口，当然也包括攻击者拥有的端口，从而达到截取目标会话的目的。用ettercap实现如下：

    #ettercap -i wlan0 -Tq -M port /192.168.1.1/ /192.168.1.104/

该命令将截取192.168.1.1和192.168.1.104之间的所有流量，但也同时会接收到两者的所有流量。
端口盗用的方法只能截获流量而不能对其进行修改，并且只能工作于以太网交换机上。值得注意的是，该方法会对局域网的网络性能造成
比较明显的影响，需要谨慎使用。

## NDP攻击

NDP(Neighbor Discovery Protocol)是IPv6协议的一个重要组成，取代了IPv4的ARP，ICMP路由发现和重定向功能。在IPv6中使用NDP
来发现直接相连的邻居信息，包括邻接设备的设备名称、软/硬件版本、连接端口等，另外还可提供设备的ID、端口地址、硬件平台等信息。
支持NDP设备都维护NDP邻居信息表(neighbor cache)，邻居信息表中的每一表项都是可以老化的，一旦老化时间到，NDP自动删除相应的邻居表项。
同时，用户可以清除当前的NDP信息以重新收集邻接信息。

因此，我们通过给目标发送ND requests/replies报文，来污染目标的邻居信息表，从而达到伪装成网关的目的，例如：

    #ettercap -i wlan0 -Tq -M ndp:remote //fe80::260d:afff:fe6e:f378/ //2001:db8::2:1/

其中地址分别为网关和目标的IPv6地址。当然目前对IPv6的推广还不普及，但随着IPv4地址耗尽，最终普及IPv6也只是一个时间过程罢了。针对
IPv6的攻击也有待发掘。

## 邪恶的孪生AP攻击(Evil Twin AP)

上面介绍的几种中间人攻击方法都有一个共同的前提，即要求我们和目标在同一个局域网之中。然而有时候我们无法入侵内网，比如WiFi用了
WPA/WPA2 PSK + 强口令的方式加密，且有意识地关闭了WPA等一些有漏洞的功能。这时想要破解进入其局域网显然不太可能，但是仍然可以对
内网中的客户端进行中间人攻击。

### 攻击原理

这就要说到我们网络设备自动连接WiFi的过程，要决定是否连接一个AP，首先会看其是否曾经连接过。AP的特征往往通过ESSID，BSSID和加密
方式来进行判断，因此我们只要能在本机建立一个相同ESSID，相同MAC地址以及相同加密方式的AP，并且信号强度大于原AP，那么客户端就很有
可能优先连接上我们的假AP。这时只要在本地给目标非配号DHCP地址和DNS服务器，再进行流量转发，就可以神不知鬼不觉地对其进行中间人攻击了。

### 实现过程

Evil Twin AP的实现过程和传统的“钓鱼热点”大同小异，不同在于钓鱼热点往往是不加密的，以期不明真相的路人进行连接，数据广撒网的攻击。而
孪生AP则更有针对性，比如怀疑邻居浏览虐童网站或者进行恐怖活动，就需要对其进行监听采证，然后向警察叔叔举报。当然，要注意自己的安全。
具体实现过程我分为三步：

**建立Fake AP**

这里使用aircrack-ng工具集来进行操作，关于aircrack-ng的基本用法可以参考我之前写的[WIFI密码破解笔记][wifi-crack]，首先选择一个网络接口，
将其设置为混杂模式：

    airmon-ng start wlan0

假设混杂模式（monitor）的无线接口为mon0，我们可以用其建立一个AP：

    airbase-ng -P -a $ap_mac -e $ap_name -c $channel mon0

其中-a指定AP的BSSID（即MAC地址），-e指定AP的名称（ESSID）， -c指定信道。全部指定和待攻击的AP一样则会建立一个Evil Twin AP，否则建立的是一个
开放的AP，设置好后运行ifconfig应该会看到一个at0的虚拟网络接口，即我们的AP。

**打开本地DHCP服务器**

这时候如果目标连接上了我们的AP，当然是上不了网的，甚至连动态IP也获取不了，因此要先设置好我们的DHCP Server。根据DHCP Server工具不同，配置文件
的路径也会有点不一样，一般是`/ect/dhcp/dhcpd.conf`，修改IP池范围，子网掩码，DNS服务器地址，如果有需要也可以修改lease时间。改好后运行：

    ifconfig at0 up 192.168.2.1 netmask 255.255.255.0 

重新设置at0的地址和掩码，其中192.168.2.1是我们的假AP地址，然后添加一条路由记录：

    route add -net 192.168.2.0 netmask 255.255.255.0 gw 192.168.2.1

启动（或重启）DHCP Server：

    service isc-dhcp-server restart
    
这样目标连接上我们的WIFI后就可以分配到一个192.168.2.0/24中的IP地址了（IP池内）。

**开启转发**

转发用到iptables，我在[ARP欺骗与中间人攻击][arp-mitm]中已经介绍过了，这里就不再赘述，具体命令如下：

    iptables -t nat -A POSTROUTING -o wlan0 -s 192.168.2.0/24 -j MASQUERADE

目的是把192.168.2.0/24子网内的流量转发到能上网的接口上，这里是wlan0。值得一说的是，wlan0开启了混杂模式又要转发，对网络性能可能有点影响，最好是额外
有个网卡，或者是有线（eth0）。然后用aireplay-ng对目标进行短暂的掉线重连，就有可能连上我们的AP了。当然这只是最基本的功能，为实现最好效果，我们可以
加大假AP的发射功率，要提高网络性能也可可以优化iptables转发规则，限于篇幅就不再细说。

## 后记

通过介绍了几种常见协议，也介绍了几种相应的中间人攻击方法。虽然现在协议对安全的关注越来越多，但不论如何总要在速度和安全性做出折衷。
新的协议部署和推广都需要一段很长的时间，即便有了安全的协议也仍然会有各种无法避免的漏洞。更何况如果局域网内主机的通讯都像防贼一样
重重加密，那也有点得不偿失。其实中间人攻击的手段还有很多，要懂得发动自己的想象力。比如通过钓鱼或者路由器漏洞的方式获得路由器密码，
这样通过修改路由配置，就成为真正的“中间人了”，而这也是我最喜欢的一种方式，呵呵。

[arp-mitm]: http://pannzh.github.io/tech/mitm/2015/11/01/arp-mitm.html
[wifi-crack]: http://pannzh.github.io/tech/mitm/2015/10/31/wifi-crack.html
