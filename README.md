# Huangsir's blog

power by https://github.com/acyortjs/acyort

master存编译好的html文件，develop存acyort的配置。

大概的流程是：

issue修改后触发webhook -> ifttt -> circleci -> 编译文章 -> 覆盖master文件 -> push到master
