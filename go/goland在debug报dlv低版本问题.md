本文章来源于：<https://github.com/Zeb-D/my-review> ，请star 强力支持，你的支持，就是我的动力。

[TOC]

------

### 一、现象

很久前没买GoLand正式版，加上最近go推了一些高级特性，就打算升级了下，因为我这边不是整体做了升级，

你可以理解只是简单升级了下GO_PATH相关的lib；

结果报了以下的错误：

```
Version of Delve is too old for this version of Go...
```



### 二、处理

网上查了问题，大致都是GoLand自带的dlv版本与go类库的版本太低导致，

需要进行对dlv升级；

```
`git clone https://github.com/go-delve/delve`
`cd delve`
`cd cmd/dlv`
`➜  dlv git:(master) ls
 cmds        dlv_test.go main.go     tools.go`
```

进行编译

```
`go build`
```

 编译后有个可执行文件

```
 `➜  dlv git:(master) ls
  cmds        dlv         dlv_test.go main.go     tools.go`
```

 在GoLand 进行配置

```

 `goland>help>Edit Custom Properties`
```

 增加你上面进行编译的path， 格式为

```
 `dlv.path=/Users/xxx/delve/cmd/dlv/dlv`
```

 对GoLand进行重启，对你的代码进行Debug，OK