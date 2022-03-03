---
layout: post
title: "git flow release finish的时候自动生成Changelog"
date: 2016-02-23 19:22:19 +8000
categories: [Git, 技术, Linux]
---
<!-- datetime: 2016-02-23 19:22:19 -->

大部分人应该都没有写Changelog的习惯吧？我也没有...

<!-- more -->

git有提供hook的方法，可以实现帮助我们实现许多自定义的功能。在git flow大热之后，许多的企业项目，都采用git flow的流程来作为项目开发的标准流程。

有一天老夫在网上逛荡的时候，注意到一些服务每次在发布版本的时候，都有提供详细的Changelog，然而老夫自己的项目里基本上是不会有这种东西的，而且我感觉也没有人有兴趣每天在上面更新项目的Changelog。作为一个极（ma）客（nong），我们应该是用技术的手法来减少工作。

于是乎，我就想到了在git flow发布release的时候，自动生成一份Changelog。

在网上翻了翻，其实有不少网友已经有提供了现成的脚本，老夫随意拿了一份来，改了改~~

git flow在安装的时候，本身已经对git的hook进行了一定的扩展，feature，release，hotfix都有start和finish的hook。我就选择使用release的finish的hook来实现自动生成Changelog的功能。

我希望能够达到的效果是：当我发布一个版本时（git flow release finish），程序能够自动将上一个版本，到这一次发版之间，所有的commit，全部输出到Changelog.md这个文件上，然后将这个文件的修改commit到版本库里。

我们新建一个项目，就叫做test_hook好了，然后在项目里使用git init初始化git目录，然后使用git flow init初始化git flow配置。

这时进入项目里.git/hooks，可以看到里面有一些hook的sample，如果没有找到这个目录，可以自己make一个。

然后我们在hooks目录下添加一个文件名为pre-flow-release-finish的文件。

文件脚本如下：

```
#!/bin/bash
#
# Runs before git flow release finish
#
# Positional arguments:
# $1    The version (including the version prefix)
# $2    The origin remote
# $3    The full branch name (including the release prefix)
#
# The following variables are available as they are exported by git-flow:
#
# MASTER_BRANCH - The branch defined as Master
# DEVELOP_BRANCH - The branch defined as Develop
#
VERSION=$1
ORIGIN=$2
BRANCH=$3

if [ `echo "$VERSION" | grep -P "v\d+.+"` != $VERSION ]; then
    exit 0
fi

REPOADDR=`git config --get remote.origin.url | sed -s 's/.git$//g'`
if [ ${REPOADDR:0:3} == "git" ]; then
    REPOADDR=`echo $REPOADDR | sed -s 's/:/\//g;s/git\@/https:\/\//g'`
fi
LINKADDR=$REPOADDR/commit/

rm Changelog.md 2>/dev/null
while read TAG; do
    if [ ! $CURRENT ]; then
        REV="--all"
        CURRENT=1
        echo "     " >> Changelog.md
        echo "*$VERSION (CURRENT)*" >> Changelog.md
        echo "---" >> Changelog.md
    else
        REV=$TAG
        echo "     " >> Changelog.md
        echo *$NEXT* >> Changelog.md
        echo "---" >> Changelog.md
    fi

    echo "     " >> Changelog.md
    git log --no-merges --date=short  --pretty=format:"- %ad (%an) %s -> [view commit](${LINKADDR}%h)" $TAG..$NEXT | grep -v "Edit Changelog.md" >> Changelog.md
    echo "     " >> Changelog.md

    NEXT=$TAG
done < <(git for-each-ref --sort='*authordate' --format='%(tag)' refs/tags | grep 'v\d+.+' -P | tac)

COMMITID=`git rev-list $NEXT | head -n 1`
LINKADDR=$REPOADDR/commit/$COMMITID
echo "     " >> Changelog.md
echo *$NEXT* >> Changelog.md
echo "---" >> Changelog.md
echo "     " >> Changelog.md
git log --no-merges --date=short  --pretty=format:"- %ad (%an) %s -> [view commit](${LINKADDR}%h)" $NEXT >> Changelog.md
echo "     " >> Changelog.md

git add Changelog.md
git commit -am "Edit Changelog.md"

exit 0
```

上面的逻辑表示他会读取这次release的tag，如果tag匹配v\d+.+，如v1.1这样的格式，那么就可以继续进项后续的流程。也就是说，我们release的tag，必须是满足类似v1.1或v2.12.4这样的格式，才能够正确被脚本读取，其余格式的tag都会被直接忽略，当做一个普通的commit来处理。

因为我们的Changelog里，每个commit id要能够对应到具体的页面上。而.git/config有可能记录的是以git为scheme的URI，所以这里需要一个简陋的转换，把git://的地址转为使用https://的地址。

然后就是把旧的Changelog.md删掉，同时生成新的Changelog.md。

最后就是把这个新的Changelog.md的修改commit上去，之后git flow release finish的流程就完成了。这时的版本库里就携带有最新的Changelog信息。

更多对git hook的了解可以参阅[官方文档](https://git-scm.com/book/zh/v1/%E8%87%AA%E5%AE%9A%E4%B9%89-Git-Git%E6%8C%82%E9%92%A9)
