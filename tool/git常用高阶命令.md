本文章来源于：<https://github.com/Zeb-D/my-review> ，请star 强力支持，你的支持，就是我的动力。

[TOC]

------

### 背景

平常常用的git 命令无非是git add 、commit、push等；

但比如查看提交日志、pick、rebase这些估计有些经常使用；



### git log

简介打印日志

```
git -<N> --pretty=online //以一行的方式打印前面N行记录
```



### git rebase

一般用于精简提交记录，比如多个合并记录提交成一个，也可以把其它rebase过来 但不会产生Merge Into的记录；

1、rebase 其它分支到当前分支

```
git rebase <branch_name>
```

2、rebase 前面几条记录为一条

```
git rebase -i HEAD～<N> //<N>为前面几条记录

pick 2c29aad update md format
f abc9713 update md format
f d8745ee update md format

# Rebase 38956a9..d8745ee onto 38956a9 (3 commands)
#
# Commands:
# p, pick = use commit
# r, reword = use commit, but edit the commit message
# e, edit = use commit, but stop for amending
# s, squash = use commit, but meld into previous commit
# f, fixup = like "squash", but discard this commit's log message
# x, exec = run command (the rest of the line) using shell
# d, drop = remove commit
#
# These lines can be re-ordered; they are executed from top to bottom.
#
# If you remove a line here THAT COMMIT WILL BE LOST.
#
# However, if you remove everything, the rebase will be aborted.
#
# Note that empty commits are commented out

```

一般选择f （fixup），选择s （squash）会弹出继续编辑 选择记录；

```
[detached HEAD 873a136] update md format
 Date: Mon Feb 20 12:09:18 2023 +0800
 18 files changed, 92 insertions(+), 59 deletions(-)
 create mode 100644 image/network-https-connect-flow.png
Successfully rebased and updated refs/heads/master.

```

接下来就可以直接 push -f 到自己的分支了



3、rebase 当前记录到某个之前的commitID

```
git rebase -i <commitID>

如：
git rebase -i 873a1367
```

