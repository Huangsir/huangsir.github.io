---
layout: post
title: "使用Logrotated清理日志文件"
description: ""
category: 
tags: []
---

题外话：Logrotated是依靠cron来执行的，他的脚本放在/etc/cron.daily下。其实许多定时执行的功能如apc cache清理过期数据等等，也是依赖cron执行的。

首先，要比较全面的了解Logrotated的使用，推荐阅读[鸟哥的私房菜Logrotated部分](http://linux.vbird.org/linux_basic/0570syslog.php#rotate_config)。

我这里讲一些我自己使用的情况。

```
/var/log/php5-fpm.log {
    rotate 12   #保留12份，也就是会有php5-fpm.log.1.gz ~ php5-fpm.log.12.gz
    maxage 12   #包括12天
    weekly  #每周执行一次
    missingok   #没有找到log文件也OK
    notifempty  #如果log文件是空的，就不做rotate
    compress    #gzip压缩，所以文件会是 php5-fpm.log.1.gz
    delaycompress   #和compress一起使用，转储的日志文件到下一次转储时才压缩
    minsize 10M     #小于1M的log文件不做rotate
    postrotate  #rotate后执行里面得命令
        invoke-rc.d php5-fpm reopen-logs > /dev/null
    endscript
}
```

rotate和maxage都是控制日志保留的，不过前者是以个数为单位，后者以天数为单位。

具体logrotated会在什么时候运行，这个需要看cron.daily会在什么时候运行。在/etc/crontab文件里可以看到运行时间。

一般自己写的程序，我习惯使用两种logrotated的方式。

如果log本身并没有特别作用，只是一些例行的log信息，那么可以直接删掉，一般会使用

```
weekly
missingok
rotate 0
su root audio
```

如果是需要保留一段时间以备查阅的，一般使用

```
daily
compress
missingok
rotate 14
su root audio
```

是否需要compress要看log文件的大小和具体的业务，保存几天也视情况而定。
不过一般情况下我会选择保存2周时间。

su root audio这个命令是用于解决权限问题的，一般不会加，
但是如果你发现你的log文件logrotated因为权限问题无法正确的归档，
就可以增加该命令。
[[参考]](https://linuxslut.net/logrotate-parent-directory-has-insecure-permissions/)

编辑好你的rotate脚本后，保存到/etc/logrotate.d下，最好对于每一个日志，
都编辑一个与之对应的logrotated脚本。当然也可以把多个脚本放在一个文件下。

保存好之后不需要重启什么服务，下次日志归档时就会自动生效了。

当然，如果你不知道你的脚本写的好不好的话，也可以手动运行logrotated来执行指定脚本。

```bash
sudo logrotate -vf <你的logrotated脚本路径>
```

使用-v会打印logrotated的执行过程，如果有报错可以直接看到。

在使用的过程中，我们有时会发现log文件无法归档，或者log文件归档后，没有再生成新的log文件。

比如说，我现在有一个log文件叫做celery.log，并不断有日志往这个文件里写入。  
在归档后，文件名变成了celery.log.1.gz。  
这时程序理应生成一个新的celery.log并往里面写数据，
但是却不见celery.log出现。  
这是因为程序一直持有老的celery.log的fd，在老的celery.log变成celery.log.1.gz后，并没有释放fd，所以就不会生成新的celery.log。

解决这个问题的方法要看具体案例，比如说php5-fpm就使用上文的postrotate语法段来解决。  
如果是我们自己写的python程序，使用logging的FileHandler时也会出现这个问题，可以改用WatchedFileHandler解决问题。  
当然使用RotatingFileHandler/TimedRotatingFileHandler来自己做rotate也是可以的。

