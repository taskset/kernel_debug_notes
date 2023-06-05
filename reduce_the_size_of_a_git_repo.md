# 问题描述

工作中有时候会基于linux-stable仓库的某个分支来进行定制修改，但是linuxe-stable太大了，光git目录就占用4G+：

```
4.8G    ./.git
```

一般我们都是内外网隔离的环境，内网服务器器上存了完整的大仓库，这样导致了本地开发也要下载这么大的数据，既浪费了磁盘空间，也降低了效率。


# 解决方法

## 方法1：裁剪分支

linux-stable仓库中的分支非常多，从20多年前的linux-2.6.y 到最近的linux-6.2.y 全在上面，而我们一般只用到了其中某一个分支，所以内网开发的仓库只需要下载某一个分支即可，比如5.10.y:

```
git clone -b  linux-5.10.y  --single-branch  https://mirrors.tuna.tsinghua.edu.cn/git/linux-stable.git


此时的磁盘磁盘会下降一半多：

1.9G    ./.git
```

## 方法2：裁剪代码的历史记录

对于最求效率的程序员来说，1.9G 还是嫌大，我们还可以通过删除某个时间点之前的代码历史记录来进一步缩减空间。

比如我们只关注5.10本身，想裁剪掉v5.10 这个tag之前的代码提交记录，就可以这样来操作：

```

a, 还是要下载linux-stable仓库的代码（或者直接从内网的服务器上pull）

git clone https://mirrors.tuna.tsinghua.edu.cn/git/linux-stable.git

b, 查询出v5.10 tag 对应的commit id：

$ git show v5.10

commit 2c85ebc57b3e1817b6ce1a6b703928e113a90442 (tag: v5.10)
Author: Linus Torvalds <torvalds@linux-foundation.org>
Date:   Sun Dec 13 14:41:30 2020 -0800

    Linux 5.10

c, 通过功能强大的git rebase命令，删除v5.10 之前的所有commit log： 

git checkout --orphan tmp 2c85ebc57b3e1817b6ce1a6b703928e113a90442^

git commit -m "truncate history before v5.10: 2c85ebc57b3e1817b6ce1a6b703928e113a90442"

git rebase --onto tmp 2c85ebc57b3e1817b6ce1a6b703928e113a90442^  linux-5.10.y

git branch -D tmp

操作步骤参考的是这个文档：https://passingcuriosity.com/2017/truncating-git-history/

d, 进一步构建精简的git仓库：


新建一个空白的git repo:

mkdir linux-stable-lite && cd linux-stable-lite
git init .

关联到上一步裁剪历史记录的git仓库上：
git remote add xxx /path/to/linut-stable

fetch代码，并且切换到我们使用的分支上
git fetch xxx
git checkout xxx/linux-5.10.y -b linux-5.10.y

可以看到磁盘空间已经y进一步减少为:

230M        ./.git/

最后把linux-stable-lite传到内网，即可作为精简的开发基线。

为方便，我也把精简后的linux-stable放到github上了, 可以直接用:

https://github.com/systemd-learning/linux-stable-lite

```

## 方法3：只保留干净代码

上面的方法通过裁剪无关的分支以及裁剪过期的代码修改记录来缩减git仓库空间，.git文件夹从4.8G 减少到了230M，效果还是不错。
但如果还想继续缩减git仓库空间，那么最后还可以直接下载干净代码, 但缺点也很明显，没有修改记录，无法做小版本升级&降级，只能下载某一个小版本作为基线：

```

假设我们关注的是5.10.65这个基线版本，先查询出它的commitid：

$ git show v5.10.65
commit c31c2cca229aa5280d108618bb264c713840a4c2 (tag: v5.10.65)
Author: Greg Kroah-Hartman <gregkh@linuxfoundation.org>
Date:   Wed Sep 15 09:50:49 2021 +0200

    Linux 5.10.65

然后直接下载对应的压缩包即可：
wget https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/snapshot/linux-c31c2cca229aa5280d108618bb264c713840a4c2.tar.gz

或者在本地repo中根据commit 制作 压缩包：
git archive  c31c2cca229aa5280d108618bb264c713840a4c2 --format=tar.gz --output=./linux-c31c2cca229aa5280d108618bb264c713840a4c2.tar.gz

这个虽然完全省略了.git 文件夹的消耗，但如上所说，丢失了修改记录，无法做源码朔源，也难以做小版本升降级，所以不到万不得已，还是不要这么用。

```


