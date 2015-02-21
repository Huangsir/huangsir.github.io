---
layout: post
title: "ubuntu上安装和配置graphite+statsd"
description: ""
category: 
tags: []
---

常见的开源监控工具有nagios, ganglia, graphite等等，各有各的特点。
第一次听说graphite是在nsq的文档上，他声称官方支持使用graphite+statsd来统计metric。
于是乎便捣鼓了一下。

ubuntu12.04和ubuntu14.04上的安装方法略有不同，总体来说我更喜欢ubuntu14.04的使用方法。

***

**ubuntu12.04**

首先，安装各种依赖。graphite会安装到/opt/graphite，不伦有没有使用virtualenv，不过我们依然建议使用virtualenv。
graphite-web的各种依赖可以看[这里](https://github.com/graphite-project/graphite-web/blob/master/requirements.txt)。  

这里我没有贴出所需要得依赖，因为我自己也忘记了。不想看依赖的可以像我一样，装到哪里发现不行的时候再去补充依赖。

需要注意的是django和django-tagging两个依赖，django版本不兼容也已经不是新鲜事了，如果发现django报错，可能是你的版本使用的不对。  
我这里使用的是

```python
Django (1.6.1)
django-tagging (0.3.1)
```

系统库也需要补充一些`sudo apt-get install libcairo2-dev`  

```python
pip install whisper
pip install carbon
pip install graphite-web
```

这里介绍一下，whisper是一个文件数据库，用来存储采集到的metrics。你采集到的metrics会扔到carbon里，然后carbon会存储到whisper上。
graphite-web从whisper中拿出各种metrics来画图，画图用的库我们刚刚用apt-get安装了。  
metrics是什么？比如说`stats.cpu.host-01.cpu-00`就是一个metrics，也是一个key，他的value是50，就表示host-01这台机器的cpu-00现在是50%的状态。  
使用点号分割。但是我们不需要在意这些细节。

安装完后，你会发现/opt/graphite下多了很多东西，把/opt/graphite/conf下面的*.example的.example去掉cp到对应的目录即可。默认graphite会使用2003，2004端口。

启动carbon

```bash
/opt/graphite/bin/carbon-cache.py start
```

制造一些metrics发送

```bash
vim /etc/hosts
127.0.0.1   graphite
python /opt/graphite/examples/example-client.py
```

数据会存放在

```bash
/opt/graphite/storage/whisper
```

现在去看看，是不是有一些东西了。

同步数据库

```bash
python /opt/graphite/webapp/graphite/manage.py syncdb
```

启动graphite-web

```bash
python /opt/graphite/webapp/graphite/manage.py runserver 0.0.0.0:12222
```

打开浏览器 http://127.0.0.1:12222，看看是不是有了一些东西。

***

**ubuntu14.04**

14.04的官方库里就有了graphite，可以直接安装。老夫装完后升级到14.04又重新用新的方法装一次-，-#

```bash
sudo apt-get install graphite-web graphite-carbon
```

apt-get安装后，配置文件在/etc/graphite/, /etc/carbon/下。  
数据库放在/var/lib/graphite/下。  
graphite的uwsgi文件在/usr/share/graphite-web下。  
graphite的py文件在/usr/lib/python2.7/dist-packages/graphite  
graphite的可执行文件为graphite-manage，可以直接运行  
配置的example在/usr/share/doc下  

***

**配置Graphite**

14.04的配置在/etc/graphite/local_settings.py里，ubuntu12.04在/opt里有对应的配置文件。**往后的配置如无特别说明都以ubuntu14.04为主，因为我已经没有12.04的环境，找不回配置文件所在了。**

```python
SECRET_KEY = 'yw7HIHFkW459HuswEUAZOn12NgQGx+z6\n'
```

可以使用python来生成

```python
import os
os.urandom(24).encode('base64')
```

```python
#修改时区
TIME_ZONE = 'Asia/Shanghai'
#打开RemoteUser身份认证（不知道干嘛用的）
USE_REMOTE_USER_AUTHENTICATION = True
```

配置好后同步数据库:

```bash
#ubuntu14.04
sudo graphite-manage syncdb

#ubuntu12.04
python /opt/graphite/webapp/graphite/manage.py syncdb
```

**配置Carbon**

配置carbon开机启动（这项配置ubuntu12.04里没有）

```bash
sudo vim /etc/default/graphite-carbon

CARBON_CACHE_ENABLED=true
```

开启carbon的log rotation

```bash
sudo vim /etc/carbon/carbon.conf

ENABLE_LOGROTATION=true
```

配置storage schemas，在最上面添加一个新的schema

```
sudo vim /etc/carbon/storage-schemas.conf

[statsd]
pattern = ^stats\.
retentions = 15s:1d,1m:7d,15m:5y
```

上面这段statsd的意思是：

* 匹配以`stats.`为开头的所有metrics
* 每隔15s创建一个数据点，保存1d，请求最近1d的数据时使用这套数据
* 每隔1m创建一个数据点，保存7d，请求7d内得数据时使用这套数据
* 每隔15m创建一个数据点，保存5y，请求5y内得数据时使用这套数据

修改数据聚合的方法aggregation method

```
sudo vim /etc/carbon/storage-aggregation.conf

#可以改为以下(网上抄的)
[min]
pattern = \.min$
xFilesFactor = 0.1
aggregationMethod = min

[max]
pattern = \.max$
xFilesFactor = 0.1
aggregationMethod = max

[count]
pattern = \.count$
xFilesFactor = 0
aggregationMethod = sum

[lower]
pattern = \.lower(_\d+)?$
xFilesFactor = 0.1
aggregationMethod = min

[upper]
pattern = \.upper(_\d+)?$
xFilesFactor = 0.1
aggregationMethod = max

[sum]
pattern = \.sum$
xFilesFactor = 0
aggregationMethod = sum

[gauges]
pattern = ^.*\.gauges\..*
xFilesFactor = 0
aggregationMethod = last

[default_average]
pattern = .*
xFilesFactor = 0.5
aggregationMethod = average
```

其中的xFileFactor的值代表carbon做聚合的最小百分比值，根据你的具体情况修改。

启动carbon-cache

```bash
sudo service carbon-cache start
```

**Nginx配置**

```
sudo vim /etc/nginx/site-enabled/graphite

upstream graphite {
    server unix:///run/uwsgi/app/graphite/socket;
    keepalive 600;
}
server {
    listen 80;
    server_name XXXX;

    access_log off;
    error_log off;

    location / {
        uwsgi_pass graphite;
        include uwsgi_params;
    }
}
```

**Uwsgi配置**

```
sudo vim /etc/uwsgi/apps-available/graphite.ini

[uwsgi]
processes = 8
uid = _graphite
gid = _graphite
master = True
chdir = /usr/share/graphite-web
wsgi-file = graphite.wsgi
chmod-socket = 666
enable-threads = true
```

启动服务

```bash
sudo service carbon-cache restart
sudo service nginx restart
sudo service uwsgi restart
```

打开浏览器 http://XXXX:80

**安装statsd**

一般使用statsd来收集各种metrics。然后传给carbon，carbon再保存到whisper里。

安装所需要的依赖

```bash
sudo apt-get install git nodejs devscripts debhelper
```

下载并编译statsd

```bash
mkdir statsd_build
cd statsd_build
git clone https://github.com/etsy/statsd.git
cd statsd
dpkg-buildpackage
cd ..
sudo dpkg -i *.deb
```

关闭statsd的legacy namespacing

```
sudo vim /etc/statsd/localConfig.js

{
  graphitePort: 2003
, graphiteHost: "localhost"
, port: 8125
, graphite: {
    legacyNamespace: false
  }
}
```

上文所配置的carbon配置，已经是按照statsd的格式来配置了，所以这里不再重复。

重启服务

```bash
sudo service carbon-cache restart
sudo service statsd start
```

访问graphite，可以看到statsd自身的一些记录。statsd有许多的client，大家可以自己上github上搜索自己语言使用的client。

需要删掉数据的时候，可以直接到whisper下删除对应的文件，当然要先停止掉carbon和statsd的服务。

到这里已经可以正常的看到图像了，如果不行，可以自己再去google。

**配置Dashboard**

Graphite的原装web实在是看不下去，当然人家主打的也不是这个界面，所以有许多扩展界面出现。可以参考[这里](http://dashboarddude.com/blog/2013/01/23/dashboards-for-graphite/)

豆瓣也出了一个[graphite-index](https://github.com/douban/graph-index)，主要是配置方便。

用过kibana3的同学可以考虑使用[grafana](https://github.com/torkelo/grafana)，该项目是一个基于kibana3的改版而来的Dashboard，配置方法和kibana3很相似。

这里我选择的是golang版的grafana [gofana](https://github.com/jwilder/gofana)，为什么要使用这个呢，因为他配置起来很方便，可以直接使用docker来安装。而且使用起来也和kibana类似。

安装得方法在github上写得十分详细，我这里就不再重复。

这里要注意几点，kibana系列的Dashboard是自己用js绘图的，对于数据量很大的图表，绘制起来会很慢，原装的就不会有这个问题。我现在平均1s有1k+的metrics采集，查看7d的数据已经卡成狗，是有必要修改一下storage_schema的。

如果使用的是grafana，使用起来会有跨域问题（kibana是由js直接发起请求的）。解决这个问题的方法我查了很多，比如在nginx的返回值里手动添加CORS等。  
我这里选择了另外一种方式，直接修改graphite-web的配置，增加CORS。

安装django-cors-headers

```bash
pip install django-cors-headers
```

修改graphite-web的app_settings.py

```
sudo vim /usr/lib/python2.7/dist-packages/graphite/app_settings.py

MIDDLEWARE_CLASSES = (
  'corsheaders.middleware.CorsMiddleware',  # 添加这一行
  '...',
)
```

编辑/etc/graphite/local_settings.py

```
sudo vim /etc/graphite/local_settings.py

#添加一行
CORS_ORIGIN_ALLOW_ALL = True
```

**参考+推荐**

[http://note.axiaoxin.com/contents/install-and-use-graphite-on-ubuntu14.04.html](http://note.axiaoxin.com/contents/install-and-use-graphite-on-ubuntu14.04.html)
[http://segmentfault.com/blog/duoduo3_69/1190000000744706](http://segmentfault.com/blog/duoduo3_69/1190000000744706)
[https://kevinmccarthy.org/blog/2013/07/18/10-things-i-learned-deploying-graphite/](https://kevinmccarthy.org/blog/2013/07/18/10-things-i-learned-deploying-graphite/)
