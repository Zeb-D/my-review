本文章来源于：<https://github.com/Zeb-D/my-review> ，请star 强力支持，你的支持，就是我的动力。

[TOC]

------

### 背景

记一次rust本地升级到最新版本，

之前版本为1.67 （2023-07），最新版本1.81 （2024-09），

当前环境为macbookpro（OS：10.12.4 古董级电脑），很多墙软件都安装不了。

之前环境没配置过，在国内网速只有十几kb，一直卡在下载这个环节。



### 目标

升级方式有brew方式，也有rustup自身工具链方式，本次操作为rustup进行升级。

将rust环境配置切到国内环境。



### 步骤

```
vi ~/.bash_profile
//添加阿里源
export RUSTUP_UPDATE_ROOT="https://mirrors.aliyun.com/rustup/rustup"
export RUSTUP_DIST_SERVER="https://mirrors.aliyun.com/rustup"

source ~/.bash_profile
```



更新最新版本

```
rustup update
// 安装后看下当前版本
rustc -V
```



