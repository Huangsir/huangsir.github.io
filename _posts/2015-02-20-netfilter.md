---
layout: post
title: "netfilter参数"
description: ""
category: 
tags: []
---

死人杨先生在春节放假前几天，跑去机房升级了两台服务器的内存，**顺便**把这两台服务器从ubuntu12.04升级到14.04。  
这两台服务器负责的一个重要的业务，是使用rest对外发送各种消息或postback数据。  
升级之后，当天晚上发现大量rest请求出现`tcp i/o timeout`这样的错误，死人杨先生重启服务后继续再跑。

第二天又发生了许多的`tcp i/o
timeout`，因为死人杨先生又去了机房，只能老夫来抢救服务。检查了cpu和内存，都没有发现异常，load也很低，网卡中断也正常，syslog和dmesg没有错误信息。事情紧急，不能挂tcpdump这种耗时的工作(大多数发生问题的时候都已经不容许挂tcpdump或者strace这种高耗时的工作了)。细细思考了一下，插内存对于机器的影响是很小的，相比于插内存，更有可能出现问题的是升级系统。

细细思考了一下后，打电话问杨先生升级系统之后，sysctl有没有重新设置，杨先生的回复是没！有！-，-#（当时心中一千万只草泥马奔腾而过）

重新跑了sysctl后问题依然没有解决，发现有几个设置在14.04后已经没有了。  

改好后的参数几个参数  

```c
//net.core.somaxconn = 300000
net.core.somaxconn = 65535
//net.ipv4.netfilter.ip_conntrack_max = 655350
net.netfilter.nf_conntrack_max = 655350
//net.ipv4.netfilter.ip_conntrack_tcp_timeout_established = 86400
net.netfilter.nf_conntrack_tcp_timeout_established = 86400
```

**net.core.somaxconn**  

`net.core.somaxconn`定义了系统中每一个端口最大的监听队列的长度，这是个全局的参数。在突发的高并发请求时，可能会导致连接超时。在ubuntu14.04，这个值可以设置的上限是65535。但在ubuntu12.04的时候这个值可以设置很高。

**net.netfilter.nf\_conntrack\_max**  

`net.netfilter.nf_conntrack_max`默认65536，同时这个值和你的内存大小有关，如果内存128M，这个值最大8192，1G以上内存这个值都是默认65536。这个值决定了你作为网关的工作能力上限，所有局域网内通过这台网关对外的连接都将占用一个连接。

**net.netfilter.nf\_conntrack\_tcp\_timeout\_established**  

`net.netfilter.nf_conntrack_tcp_timeout_established`表示已建立的tcp连接的超时时间。这个值过大将导致一些可能已经不用的连接常驻于内存中，占用大量链接资源，从而可能导致NAT
ip_conntrack: table full的问题。

修改了几个参数之后重新跑了一下`sysctl -p /etc/sysctl.conf`，之后死人杨先生又调整了dns等等其他参数，问题基本解决。

这个故事告诉我们，假前不要随便上线奇怪的东西，任何不畏惧墨菲定律的行为都将会得到报应ಥ_ಥ

**参考：**  
[http://my.oschina.net/hongsheng/blog/151136](http://my.oschina.net/hongsheng/blog/151136)  
[http://www.cnblogs.com/fczjuever/archive/2013/04/17/3026694.html](http://www.cnblogs.com/fczjuever/archive/2013/04/17/3026694.html)
[http://www.itwhy.org/linux/%E4%BC%98%E5%8C%96%E4%BD%A0%E7%9A%84-netfilteriptables-%E7%BD%91%E5%85%B3.html](http://www.itwhy.org/linux/%E4%BC%98%E5%8C%96%E4%BD%A0%E7%9A%84-netfilteriptables-%E7%BD%91%E5%85%B3.html)
