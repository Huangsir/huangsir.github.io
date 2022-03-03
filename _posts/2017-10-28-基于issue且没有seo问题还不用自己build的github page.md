---
layout: post
title: "基于issue且没有seo问题还不用自己build的github page"
date: 2017-10-28 14:28:34 +8000
categories: []
---
基于Github issue的Github page要么就是直接用JS现场生成HTML，外加Token来提高访问上限。要么就是写完issue之后要找个地方把代码Markdown build一下生成HTML后再po回Github上。

前者没法做SEO（好像我也不是很需要）；后者要自己部署东西，像我这种半年都不更新一次博客的，每次更新前还要检查部署的服务器是否还在.....

两个都不完美，所以研究出了一种基于Github Issue的，没有SEO问题的，还不需要自己Build的解决方案[坏笑]。

<!-- more -->

Github 用Issue写博客有集中方法，其中比较有代表性的好像是： 
* JS动态编译系的 https://github.com/wuhaoworld/github-issues-blog
* 预编译系的 https://github.com/acyortjs/acyort 

我的博客用的是 https://github.com/acyortjs/acyort 。因此需要在写好博客之后运行一下`acyort build`编译好静态HTML后再提交。

这种做法很不友好，我并不能随时随地的写或记录一些东西，因为需要回到电脑上编译。

因此我就找到了Github的Webhook，配置了一个Server，在我更新Issue后会Notify我的Server，然后编译并po到线上。

但是我很长时间都不会写一片博客，经常到要写的时候就已经忘记了要干什么才能正确发布一片博客，或者经常写完之后还要去服务器调半个小时的代码，非常不优雅。

因此我就想到了用CI帮忙编译。于是我就找到了`Travis CI`。经过研究发现Travis CI没有基于Issue的触发器。所以想要触发Travis CI要么产生一个Push，要么手动触发Travis CI的Trigger，还是不够优雅。

或者用Github的Webhook触发一个中转的Server，然后Server再通过API调用Travis CI来编译博客，那就又回到了要部署代码的那个状态。

----------   这里是解决方案的分割线   ----------

中转的Server用于连接Github的Webhook和Travis CI的API，这样的需求直指IFTTT。

不知道的同学可以访问这里（需要梯子）：https://ifttt.com/

![image](https://user-images.githubusercontent.com/3870517/32136474-027ca5da-bbd4-11e7-8800-4672b50edc5e.png)

IFTTT 专门处理 `If this then that` 的需求，而我们需要的是 If `Github的Issue有更新` then `触发Travis CI`。

但是IFTTT里并没有Travis CI的连接器，所以我们只能IFTTT里的自定义Webhooks来生成对Travis CI的调用。而再经过仔细的阅读Travis CI的文档后，我发现Travis CI的授权Token必须要加再HTTPHeader里。然而IFTTT的Webhooks没法随意增加Header。

因此我被迫放弃里Travis CI去寻找API里不需要操作太多Header，可以全部在URL或Body里解决的CI框架。

很快我找到了这个（需要梯子）：https://circleci.com/ 

----------   这里才是真正的解决方案的分割线   ----------

#### Circle CI 设置

1. 首先要注册，这一句是废话。
2. 给CircleCI授权你的GitHub权限，这句也是废话。
3. 在PROJECTS里找到你的博客项目，然后点击Setup Projcct。
4. System选Linux，Platform选最新，Language选NodeJS，把下面的Sample .yml 拷出来保存，然后点Start building。
5. *我真的要写那么详细么？这都不会还是老老实实写CSDN好啦...*
6. 点击你的项目，然后点Settings，找到下面的Permission部分。
> 因为我们需要在CI里编译然后Push，所以不能使用deploy key，因为deploy key没有Push权限。我们可以选择下面两种方法的其中一种来解决：

> 方法一：
> 我们可以在Checkout SSH key里点击`Add user key`来增加一个用户级别的Key。
  
> 方法二：
> 回到GitHub -> 找到项目 -> Settings -> Deploy keys -> Add deploy key -> <自己生成一个Token然后填进去> -> **勾选`Allow write access`** -> 保存。
> 回调CiecleCI -> 点击SSH Permission -> Add SSH key -> 把刚才生成的Token填进去 -> 保存。
     
> 这时我们在CiecleCI里才能Push代码。需要注意的是方法二只能操作本项目，如果编译后的HTML需要Push到其他Project上，建议使用方法一。

7. 创建一个API Token给IFTTT使用，名字就叫做`IFTTT`好了。点击自己头像 -> User Settings -> Personal API Token -> Create New Token -> 填入Token的名字IFTTT -> 保存。

查看CircleCI的[文档](https://circleci.com/docs/api/)可以看到，通过API触发Build的方法为`POST: /project/:vcs-type/:username/:project/tree/:branch`。

我的项目地址是`github.com/Huangsir/huangsir.github.io`，所以调用CircleCI的方法应该是

```
POST https://circleci.com/api/v1.1/project/github/Huangsir/huangsir.github.io/tree/develop?circle-token={{Token}}
```

这里的{{Token}}就填入刚刚申请给IFTTT使用的那个Token。

#### IFTTT

1. 首先我们要注册一个IFTTT，步骤我就不写了...
2. 点击`New Applet` -> this -> Webhooks -> Receive ad web request
3. 注意这里需要你填写`Event Name`，填写的`Event Name`必须和填写在GitHub上的连接的Event一直，否则触发不到。这里我们可以填写`build_github_page`，然后点击`Create Trigger`
![image](https://user-images.githubusercontent.com/3870517/32136663-4e2457fa-bbd7-11e7-9be9-ed083f9bc63e.png)
4. 点击 that -> Webhooks -> Make a web request
5. 填写上文我们获取到的CircleCI的调用地址，连带Token一起写进去。
> 例上文中，我的项目在`Huangsir/huangsir.github.io`上，那么我填写的地址是
 `https://circleci.com/api/v1.1/project/github/Huangsir/huangsir.github.io/tree/develop?circle-token={{Token}}`
> 将我们一开始在CircleCI上申请的Token替换到{{Token}}上。
6. 保存并启用设置。


在IFTTT的[Webhooks](https://ifttt.com/maker_webhooks)里，点击Documentation查看使用文档，可以看到连接的组成格式为`https://maker.ifttt.com/trigger/{event}/with/key/XXXXXXXXX`。

其中{event}为我们要触发的Applets，需要和上文设置的Event Name一致。本例子中要填写的是build_github_page，所以整条连接应该是`https://maker.ifttt.com/trigger/build_github_page/with/key/XXXXXXXXX`

记下这个地址，这个地址将会填写到GitHub上。

#### GitHub

1. 进入你存放Issue的项目 -> 点击Settings -> Webhooks -> Add Webhooks。
2. 将上文中IFTTT的触发连接填入Payload URL里，Content-Type选application/json。
3. 勾选 Let me select individual events. 然后再展开的列表中选择Issues，其他都不选。
4. 勾选Active，然后保存。

至此，全部配置完成，当我修改了一个Issue时，会触发的流程如下：

![image](https://user-images.githubusercontent.com/3870517/32145342-e1ad4c1c-bc94-11e7-8894-7dad0723d01f.png)

#### .circleci/config.yml 配置

根据[官方文档](https://circleci.com/docs/2.0/sample-config/)来编写问题不大。

```yaml
version: 2
jobs:
  build:
    docker:
      # 这里使用Docker官方的Node源
      - image: node:latest
      
    # 项目的目录，可以随便写一个
    working_directory: ~/repo
    
    # 只触发某个branch的更新，这里我使用develop来编译博客，然后再合并到master来Push。
    branches:
      only:
        - develop

    # 执行步骤，基本就是语义化的写法，大家根据自己的需求改改应该就可以的了。
    steps:
      - checkout
      - run: npm install acyort -g
      - run: git config --global user.email "rj.huangsir@gmail.com"
      - run: git config --global user.name "Huangsir"
      - run: git checkout master
      - run: git reset --hard origin/master
      - run: git checkout develop
      - run: git clone https://github.com/acyortjs/theme-donob.git themes/donob
      - run: acyort build
      - run: git checkout master
      - run: cp -R public/* .
      - run: rm -R public
      - run: ls -alh
      - run: git status
      - run: git add .
      - run: git commit -am "build by circleci - `date`"
      - run: git push origin master
```

保存到 `.circleci/config.yml` 下，然后Push，搞定。

最后，如果发现用不了，请自行排查问题。

最后后，IFTTT需要定期检查确保Applet依然稳定运行，所以虽然我这么配置了，但是可能过半年后待我写下一篇博客时依然要重新调试一遍...