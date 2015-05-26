---
layout: post
title: "小水管支撑大流量-Graphite的IO优化实践"
description: ""
category: 
tags: []
---

首先推荐两片文章：   
[http://calvin1978.blogcn.com/articles/graphite.html](http://calvin1978.blogcn.com/articles/graphite.html)网上各种转载，我也不知道那篇才是原文     
[https://kevinmccarthy.org/blog/2013/07/18/10-things-i-learned-deploying-graphite/](https://kevinmccarthy.org/blog/2013/07/18/10-things-i-learned-deploying-graphite/)绝逼推荐，尤其是讲interval的那节    

做我们这行的，什么不多，就是请求特别多，什么不大，就是流量特别大-，-#

首先我们的Graphite只有一台小机器，用于投放的机器有12台。并且放Graphite的这台小机器还兼顾着放许多的三方监控，放sentry错误采集等等周边产品。你想想一个异性恋小正太站在12个彪悍凶残的兄贵中间，是一种什么样的体验-，-#

好了本来即便是如此，Graphite还是能hold住的，只是有时会比较卡。最近ifstat的时候发现，Graphite所在的服务器网络IO经常跑满，磁盘IOPS常年保持在500+。虽然没有什么问题，但总归是一种浪费，也是一个隐患。

好吧，其实我们有两台Graphite，一台放在AWSCN上，常年IOPS在500+，不过没关系，我们和成一台来描述O(∩_∩)O~

总之问题就是，网络带宽占用过高，磁盘IO过高，CPU过高。

Graphite有两种接受Metrics的方式，一种是用String接收，比如接受Statsd的传参。另一种是用Pakcle接收Metrics，使用Pakcle可以Batch发送一批Metrics(其实string也是可以的)，并且压缩效果比较好。

我们在AWSCN上申请的机器IOPS最高是300，500+的IOPS已经明显超出上限了。在carbon.conf中有一个选项叫`MAX_UPDATES_PER_SECOND`，调到100，重启，再看，IOPS立马掉下来了。目前没有发现副作用，副作用不明。

我一开始传Metrics时的做法，是在各台投放机上直接往Graphite机发送数据给到Statsd，Statsd汇总完转发给carbon-cache。那么既然网络带宽高，我们就得想办法减少数据的传输。

我去粗略翻阅了一下Statsd的文档，发现这东西就是用来聚合Metrics的，聚合之后，再将数据一并发送给carbon-cache，那假如我再各个投放机上装Statsd，是不是可以减少发送的频次从而降低网络IO。有了这个Idea后就去投放机上装了Statsd，并且把发送的地址全部改为了127.0.0.1。然后由每台投放机的Statsd将数据汇总后发到Graphite机上。这样一套下来，发现没！有！用！-，-#

这时重新去细细翻阅了一下Statsd，其实Statsd的主旨是讲不同的Metrics用相同的一种格式来表达。而并不是通过合并相同的Metrics来减少Metrics的数量，这样当然无法减少网络IO。

这时我又看到了carbon-aggregation。官方说这东西可以挂在carbon-cache之前，用来做一次预先的聚合，可以显著降低IO。

这里有几个比较关键的配置点：

```
[aggregation]
AGGREGATION_RULES = /etc/carbon/aggregation-rules.conf
DESTINATIONS = 192.168.1.20:2004
MAX_QUEUE_SIZE = 20000
```

AGGREGATION_RULES 配置聚合的方式，聚合的方式在官方github的example中有比较详细的注释，我这里就不再多重复一次了，在aggregation-rules.conf里，我直接增加了一行`stats.counters.logc.<metrics> (5) = sum
stats.counters.logc.<<metrics>>` 表示凡是`stats.counters.logc.`开头的所有Metrics，都在不改变Metrics名称的前提下，sum每5秒的数据成一条Metrics，然后发送到`DESTINATIONS`里。

DESTINATIONS是转发的地址，aggregation聚合后的数据会转发到这里，注意会有多个`DESTINATIONS`标签，我们使用的是[aggregation]下面的那个。

如果我们的Metrics过多，carbon-cache消耗不完，就会导致carbon-aggregation这边堆积大量的Metrics，一旦超过上限，carbon-aggregation就会直接丢弃掉，carbon-aggregation丢弃Metrics的时候，会写入一条日志。遇到这种情况时，我们需要设置MAX_QUEUE_SIZE。将MAX_QUEUE_SIZE适当调大一些，让carbon有足够的空间去完成他的工作。大家可以自行根据自己的情况调整这个参数。

仔细看aggregation的example的人会发现，如果在aggregation-rules改变了Metrics，可能会导致carbon-aggregaion发送double的数据。如`statsd.counters.logc.x.servers (60) = sum
statsd.counters.logc.<type>.servers`
这样一条Metrics，carbon-aggregation会发送`statsd.counter.logc.<type>.servers`的数据，也会发送聚合后的`statsd.counters.logc.x.servers`。

这是我们的流程已经改为：在每台投放机上部署Statsd和carbon-aggregation，Statsd采集本机的Metrics给到本机的carbon-aggregaion，经过聚合后在发送给Graphite机的carbon-cache。

到这里，再运行ifstat查看网络流量，已经变为了之前的1/10。

过了两天后，居然TMD丢数据。。。

解决方法是，在Graphite机器上也运行carbon-aggregation，配置与其他机器完全一致，只是DESTINATIONS为carbon-cache的地址。
然后投放机的carbon-aggregation的DESTINATIONS改为Graphite机得carbon-aggregation的Pakcle接收地址。
即将数据聚合完后，转发到Graphite机的carbon-aggregation上，再聚合一次，然后统一转发给carbon-cache，这样就可以解决Metrics丢失的问题。

Metrics丢失的原因没有确定，推测可能是有carbon-cache的BufferedPool引起的。不要问我是怎么知道的 T，T

carbon-aggregation的聚合时间不宜太长，否则Graphite图表上会出现大量丑陋的锯齿。我目前聚合的频率为5s一次，在Graphite上看不出任何锯齿。

对于多久聚合一次，其实并不需要太过纠结，以为一旦使用了carbon-aggregation，Metrics的发送数量就和请求并发量没有直接关联了，转而与统计的项目直接挂钩，因为不论有多大量的请求过来，每一个统计项最后都会被聚合为5秒一次的Metrics。

至此，在使用ifstat查看，Graphite机平均网络带宽占用已经不足100k了，波峰也在300k以内。而因为每台投放机都已经自己把最大量的Metrics聚合起来，所以Graphite机上的carbon-cache的cpu占用也是极低。聚合后的数据，对于Graphite的渲染似乎也是有些帮助的。

最后，虽然我的Metrics只有10w+/s，但是我相信在这样的结构下，即使是100w+的Metrics也是可以Hold住的。阿门~~
