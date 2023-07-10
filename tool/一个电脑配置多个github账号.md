本文章来源于：<https://github.com/Zeb-D/my-review> ，请star 强力支持，你的支持，就是我的动力。

[TOC]

------

### 背景

平时有学习后总结的习惯，资料都放到了github上，此时就会出现公司工作账号、开源账号、个人账号多重切换，

公司会分配一个电脑、自己可能有N个电脑，对此的上下文切换是比较严重，资料存在跨多端切换，这是非常不太好。

故本文主要看看怎么在一个机子上配置多个账号；





### 思路

ssh 方式链接到 Github，需要唯一的公钥，如果想同一台电脑绑定两个Github 帐号，需要两个条件:

1. 能够生成两对 私钥/公钥
2. push 时，可以区分两个账户，推送到相应的仓库

解决方案:

1. 生成 私钥/公钥 时，密钥文件命名避免重复
2. 设置不同 Host 对应同一 HostName 但密钥不同
3. 取消 git 全局`用户名/邮箱`设置，为每个仓库独立设置 用户名/邮箱



### 操作方法

1. 查看已有 `密钥`

- Mac 下输入命令 `ls ~/.ssh/`，看到 `id_rsa` 与 `id_rsa_pub` 则说明已经有一对密钥，若没有也可以执行。

1. 生成新的公钥，并命名为 `id_rsa_2` (保证与之前密钥文件名称不同即可)

- `ssh-keygen -t rsa -f ~/.ssh/id_rsa_2 -C "Zeb-D@qq.com"`

在 `.ssh` 文件夹下新建 `config` 文件并编辑，另不同 Host 实际映射到同一 `HostName`，但密钥文件不同。Host 前缀可自定义，例子中`Zeb-D`

```ruby
# default                                                                       
Host github.com
HostName github.com
User user.name
IdentityFile ~/.ssh/id_rsa
# two                                                                           
Host Zeb-D.github.com
HostName github.com
User user.name2
IdentityFile ~/.ssh/id_rsa_2
```

将生成的 `id_rsa.pub`，`id_rsa_2.pub`内容copy 到对应的 repo，见[Your personal account] ->[SSH keys].

- 参考教程: [使用SSH密钥连接Github【图文教程】](http://www.xuanfengge.com/using-ssh-key-link-github-photo-tour.html)

1. 测试 ssh 链接

```ruby
ssh -T git@github.com
➜  .ssh ssh -T git@Zeb-D.github.com
The authenticity of host 'github.com (20.205.243.166)' can't be established.
ED25519 key fingerprint is SHA256:+DiY3wvvV6TuJJhbpZisF/zLDA0zPMSvHdkr4UvCOqU.
This key is not known by any other names
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'github.com' (ED25519) to the list of known hosts.
Enter passphrase for key '/Users/Zeb-D/.ssh/id_rsa_2':
Hi Zeb-D! You've successfully authenticated, but GitHub does not provide shell access.
# 出现上边这句，表示链接成功
```

取消全局 用户名/邮箱设置，并进入项目文件夹单独设置

```php
全局配置
git config -global user.name "aaa-bb"
git config --global user.email "aaa-bb@cc.com"

单独项目配置
git config user.name "Zeb-D"
git config user.email "1406721322@qq.com"
```

Clone 项目，使用哪个账号，就加上默认的前缀(config文件)：

```
➜  my git clone git@Zeb-D.github.com:Zeb-D/my-review.git
Cloning into 'my-review'...
Enter passphrase for key '/Users/Zeb-D/.ssh/id_rsa_2':
remote: Enumerating objects: 1302, done.
```

提交：

```
➜  my-review git:(master) git push
Enter passphrase for key '/Users/Zeb-D/.ssh/id_rsa_2': 
Enumerating objects: 6, done.
Counting objects: 100% (6/6), done.
Delta compression using up to 12 threads
Compressing objects: 100% (4/4), done.
Writing objects: 100% (4/4), 2.44 KiB | 2.44 MiB/s, done.
Total 4 (delta 1), reused 0 (delta 0), pack-reused 0
remote: Resolving deltas: 100% (1/1), completed with 1 local object.
To Zeb-D.github.com:Zeb-D/my-review.git
   94d0c9a..b7c1982  master -> master
➜  my-review git:(master) 
```



### 参考资料

- [如何同一台电脑配置多个git或github账号](http://notes.seirhsiao.com/2016/01/24/2014-09-30-github-multiple-account-and-multiple-repository/)
- [在一台电脑上使用两个Github账号](http://blog.lessfun.com/blog/2014/06/11/two-github-account-in-one-client/)
- [一台电脑绑定两个github帐号教程](https://www.jianshu.com/p/3fc93c16ad2d)
