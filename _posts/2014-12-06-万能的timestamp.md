---
layout: post
title: "万能的timestamp"
description: ""
category: 
tags: []
---

日常工作中经常需要做一些counter的工作，但是老老实实的用户点一下赞就加一个赞，或用户下载一个App就给这个App的下载数+1这种做法实在是太不符合老！板！的！风！格！了！  

假如，你的老板有这种需求，怎么办呢？这时时间戳出现了。

稳定，自增，全局，幂等，还有比这个更完美的解决方案么？

假如我是豌豆夹的码农(我不是豌豆夹的码农，我也不知道豌豆夹有没有这样搞，我就是举个栗子)，刚刚起步时，我们从GP上抓了100w的App下来。  
这时我们的产品就可以放出去了！吗？不行啊，下载数都是0啊，连qq都是0啊，这样拿出去怎么见人？不行哇！  

怎么办？要不直接上时间戳吧！下载数是时间戳，你这不是要火么？  
没关系哇，时间戳调慢一点，截取一段用来做前缀，中间一点稳定的hash值，后面再加一点随机数，最后再给他来个自增。回头一看，完美-，-#

服务器要部署在全球多个IDC节点上哇，MySQL要做Master-Master哇，不做也没关系哇，总之Id不能冲突哇。  
怎么办？UUID吧，到你公司倒闭100次人家照样好用哇。  
不行哇，我要自增啊。
自增你妹啊自增～～
不行哇，一定要自增哇，Dump数据，增量查询都方便哇。

自增就自增，来个`MongoId`吧，Timestamp+MachineId+ProcessId+SequenceId，做成16进制，大把够你用哇。[http://docs.mongodb.org/manual/reference/object-id](http://docs.mongodb.org/manual/reference/object-id)

什么？要数字？数你妹啊数字～～  
不行哇，强类型哇，臣妾做不到哇，你看看Twitter，Facebook都是数字呀~~  
哎哟我擦，还真是-，-#  
Twitter也用时间搓呀，果然时间戳才是解决问题的王道哇~  
[http://www.slideshare.net/davegardnerisme/unique-id-generation-in-distributed-systems](http://www.slideshare.net/davegardnerisme/unique-id-generation-in-distributed-systems)

```php
$current_sequence = 4095;
$machine = 1023;
$current_timestamp = intval(microtime(true)*1000);  // 1417848461815
$epoch = strtotime("2014-12-05")*1000;  // 1417737600000

$current_timestamp -= $epoch;  // 110861815

echo decbin($current_timestamp);
// 110100110111001110111110111

$current_timestamp <<= 22;
$machine <<= 12;

echo decbin($current_timestamp);
echo decbin($machine);
echo decbin($current_sequence);
echo decbin($current_timestamp|$machine);
// 1101001101110011101111101110000000000000000000000
// 0000000000000000000000000001111111111000000000000
// 0000000000000000000000000000000000000111111111111
// 1101001101110011101111101111111111111000000000000

$id = $current_timestamp|$machine|$current_sequence;

echo decbin($id);
// 1101001101110011101111101111111111111111111111111

echo $id;  // 464988158296063

```
