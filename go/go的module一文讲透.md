本文章来源于：<https://github.com/Zeb-D/my-review> ，请star 强力支持，你的支持，就是我的动力。

[TOC]

------

### 序

无论是go还是rust、java等开发语言，都有调用其他包的场景，那么本文从 Go Module 的作者或维护者的视角，来聊聊在规划、发布和维护 Go Module 时需要考虑和注意什么事情，包括 go 项目仓库布局、Go Module 的发布、升级 module 主版本号、作废特定版本的 module。

注意：平时业务开发不用这么高阶使用，但凡有人涉及到开源项目的话，就会碰到各种问题，比如常见的大小版本号同时并行开发。

把握一个隐性规则，tag 永远递增（不会删除，更不会递减），就能避免很多问题。



### 引用module

#### 分支引用

```
go get -u your.domain/g1/demo@test
```

go module会自动将go mod 的依赖替换成

```
require your.domain/g1/demo@test v0.0.0-20241125145418-791598530d0e
```

部分说明v0.0.0表示不是用tag版本号，20241125145418 时间序列（后面部分是序列号），791598530d0e 是gi t commitId。



#### tag引用

```
go get -u your.domain/g1/demo@v0.0.1
```

这里是一个单module引用



### 私有的Go Module

在某 module 尚未发布到类似 GitHub 这样的网站前，如何 import 这个本地的 module？

如何拉取私有 module？



#### 导入本地 module

Go Module 从 Go 1.11 版本开始引入到 Go 中，现在它已经成为了 Go 语言的依赖管理与构建的标准，因此，我也一直建议你彻底抛弃 Gopath 构建模式，全面拥抱 Go Module 构建模式。

当我们的项目依赖已发布在 GitHub 等代码托管站点的公共 Go Module 时，Go 命令工具可以很好地完成依赖版本选择以及 Go Module 拉取的工作。

假设你有一个项目，这个项目中的 module a 依赖 module b，而 module b 是你另外一个项目中的 module，它本来是要发布到 github.com/user/b 上的。



但此时此刻，module b 还没有发布到公共托管站点上，它源码还在你的开发机器上。也就是说，go 命令无法在 github.com/user/b 上找到并拉取 module a 的依赖 module b，这时，如果你针对 module a 所在项目使用 go mod tidy 命令，就会收到类似下面这样的报错信息：

```
$go mod tidy
go: finding module for package github.com/user/b
github.com/user/a imports
    github.com/user/b: cannot find module providing package github.com/user/b: module github.com/user/b: reading https://goproxy.io/github.com/user/b/@v/list: 404 Not Found
    server response:
    not found: github.com/user/b@latest: terminal prompts disabled
    Confirm the import path was entered correctly.
    If this is a private repository, see https://golang.org/doc/faq#git_https for additional information.
```

这个时候，我们就可以借助 go.mod 的 replace 指示符，来解决这个问题。解决的步骤是这样的。首先，我们需要在 module a 的 go.mod 中的 require 块中，手工加上这一条（这也可以通过 go mod edit 命令实现）：

require github.com/user/b v1.0.0

注意了，这里的 v1.0.0 版本号是一个“假版本号”，目的是满足 go.mod 中 require 块的语法要求。

然后，我们再在 module a 的 go.mod 中使用 replace，将上面对 module b v1.0.0 的依赖，替换为本地路径上的 module b:

```
replace github.com/user/b v1.0.0 => module b的本地源码路径
```

这样修改之后，go 命令就会让 module a 依赖你本地正在开发、尚未发布到代码托管网站的 module b 的源码了。



#### Go workspace

Go 核心团队在 Go 社区的帮助下，2022 年 2 月发布的 Go 1.18 版本中加入了 Go 工作区（Go workspace，也译作 Go 工作空间）辅助构建机制。

基于这个机制，我们可以将多个本地路径放入同一个 workspace 中，这样，在这个 workspace 下各个 module 的构建将优先使用 workspace 下的 module 的源码。工作区配置数据会放在一个名为 go.work 的文件中，这个文件是开发者环境相关的，因此并不需要提交到源码服务器上，这就解决了上面“伪造 go.mod”方案带来的那些问题。



#### 拉取私有 module 的需求与参考方案

用 Go 命令拉取项目依赖的公共 Go Module，已不再是“痛点”，我们只需要在每个开发机上为环境变量 GOPROXY，配置一个高效好用的公共 GOPROXY 服务，就可以轻松拉取所有公共 Go Module 了：

![img](https://static001.geekbang.org/resource/image/eb/31/eb6dfca9868738a55162323ccf45ba31.png?wh=621x451)



但随着公司内 Go 使用者和 Go 项目的增多，“重造轮子”的问题就出现了。抽取公共代码放入一个独立的、可被复用的内部私有仓库成为了必然，这样我们就有了拉取私有 Go Module 的需求。

一些公司或组织的所有代码，都放在公共 vcs 托管服务商那里（比如 github.com），私有 Go Module 则直接放在对应的公共 vcs 服务的 private repository（私有仓库）中。如果你的公司也是这样，那么拉取托管在公共 vcs 私有仓库中的私有 Go Module，也很容易，见下图：

![img](https://static001.geekbang.org/resource/image/d6/yy/d6b950bddc40aaa48601e2a75159b8yy.png?wh=656x361)



也就是说，只要我们在每个开发机上，配置公共 GOPROXY 服务拉取公共 Go Module，同时再把私有仓库配置到 GOPRIVATE 环境变量，就可以了。这样，所有私有 module 的拉取，都会直连代码托管服务器，不会走 GOPROXY 代理服务，也不会去 GOSUMDB 服务器做 Go 包的 hash 值校验。

当然，这个方案有一个前提，那就是每个开发人员都需要具有访问公共 vcs 服务上的私有 Go Module 仓库的权限，凭证的形式不限，可以是 basic auth 的 user 和 password，也可以是 personal access token（类似 GitHub 那种），只要按照公共 vcs 的身份认证要求提供就可以了。

不过，更多的公司 / 组织，可能会将私有 Go Module 放在公司 / 组织内部的 vcs（代码版本控制）服务器上，就像下面图中所示：

![img](https://static001.geekbang.org/resource/image/19/54/191c16d8ee23f926ae1bcc323af67b54.png?wh=776x341)

那么这种情况，我们该如何让 Go 命令，自动拉取内部服务器上的私有 Go Module 呢？这里给出两个参考方案。



##### 通过直连组织公司内部的私有 Go Module 服务器拉取。

![img](https://static001.geekbang.org/resource/image/4b/9b/4b18a305f8982678844a7904186c8d9b.png?wh=1056x381)



在这个方案中，我们看到，公司内部会搭建一个内部 goproxy 服务（也就是上图中的 in-house goproxy）。

这样做有两个目的，一是为那些无法直接访问外网的开发机器，以及 ci 机器提供拉取外部 Go Module 的途径，二来，由于 in-house goproxy 的 cache 的存在，这样做还可以加速公共 Go Module 的拉取效率。

另外，对于私有 Go Module，开发机只需要将它配置到 GOPRIVATE 环境变量中就可以了，这样，Go 命令在拉取私有 Go Module 时，就不会再走 GOPROXY，而会采用直接访问 vcs（如上图中的 git.yourcompany.com）的方式拉取私有 Go Module。

这个方案十分适合内部有完备 IT 基础设施的公司。这类型的公司内部的 vcs 服务器都可以通过域名访问（比如 git.yourcompany.com/user/repo），因此，公司内部员工可以像访问公共 vcs 服务那样，访问内部 vcs 服务器上的私有 Go Module。



##### 将外部 Go Module 与私有 Go Module 都交给内部统一的 GOPROXY 服务去处理

![img](https://static001.geekbang.org/resource/image/81/d2/8109cbaca802a189fa60c1140016f2d2.png?wh=1021x481)

在这种方案中，开发者只需要把 GOPROXY 配置为 in-house goproxy，就可以统一拉取外部 Go Module 与私有 Go Module。

但由于 go 命令默认会对所有通过 goproxy 拉取的 Go Module，进行 sum 校验（默认到 sum.golang.org)，而我们的私有 Go Module 在公共 sum 验证 server 中又没有数据记录。因此，开发者需要将私有 Go Module 填到 GONOSUMDB 环境变量中，这样，go 命令就不会对其进行 sum 校验了。



不过这种方案有一处要注意：in-house goproxy 需要拥有对所有 private module 所在 repo 的访问权限，才能保证每个私有 Go Module 都拉取成功。



你可以对比一下上面这两个参考方案，看看你更倾向于哪一个，我推荐第二个方案。在第二个方案中，我们可以将所有复杂性都交给 in-house goproxy 这个节点，开发人员可以无差别地拉取公共 module 与私有 module，心智负担降到最低。



#### 统一 Goproxy 方案的实现思路与步骤

我们先为后续的方案实现准备一个示例环境，它的拓扑如下图：

![img](https://static001.geekbang.org/resource/image/89/35/89f7d377ab4688b635aa82c203aac935.png?wh=1018x482)



##### 选择一个 GOPROXY 实现

[Go module proxy 协议规范](https://pkg.go.dev/cmd/go@master#hdr-Module_proxy_protocol)发布后，Go 社区出现了很多成熟的 Goproxy 开源实现，比如有最初的 athens，还有国内的两个优秀的开源实现：goproxy.cn和goproxy.io 等。其中，goproxy.io 在官方站点给出了[企业内部部署的方法](https://goproxy.io/zh/docs/enterprise.html)，所以今天我们就基于 goproxy.io 来实现我们的方案。



我们在上图中的 in-house goproxy 节点上执行这几个步骤安装 goproxy：

```
$mkdir ~/.bin/goproxy
$cd ~/.bin/goproxy
$git clone https://github.com/goproxyio/goproxy.git
$cd goproxy
$make
```

编译后，我们会在当前的 bin 目录（~/.bin/goproxy/goproxy/bin）下看到名为 goproxy 的可执行文件。

```
$mkdir /root/.bin/goproxy/goproxy/bin/cache
```

再启动 goproxy：

```
$./goproxy -listen=0.0.0.0:8081 -cacheDir=/root/.bin/goproxy/goproxy/bin/cache -proxy https://goproxy.io
goproxy.io: ProxyHost https://goproxy.io
```

启动后，goproxy 会在 8081 端口上监听（即便不指定，goproxy 的默认端口也是 8081），指定的上游 goproxy 服务为 goproxy.io。

不过要注意下：goproxy 的这个启动参数并不是最终版本的，这里我仅仅想验证一下 goproxy 是否能按预期工作。我们现在就来实际验证一下。

首先，我们在开发机上配置 GOPROXY 环境变量指向 10.10.20.20:8081：

```
// .bashrc
export GOPROXY=http://10.10.20.20:8081
```

生效环境变量后，执行下面命令：

```
$go get github.com/pkg/errors
```

结果和我们预期的一致，开发机顺利下载了 github.com/pkg/errors 包。我们可以在 goproxy 侧，看到了相应的日志：

```
goproxy.io: ------ --- /github.com/pkg/@v/list [proxy]
goproxy.io: ------ --- /github.com/pkg/errors/@v/list [proxy]
goproxy.io: ------ --- /github.com/@v/list [proxy]
goproxy.io: 0.146s 404 /github.com/@v/list
goproxy.io: 0.156s 404 /github.com/pkg/@v/list
goproxy.io: 0.157s 200 /github.com/pkg/errors/@v/list
```



在 goproxy 的 cache 目录下，我们也看到了下载并缓存的 github.com/pkg/errors 包：

```
$cd /root/.bin/goproxy/goproxy/bin/cache
$tree
.
└── pkg
    └── mod
        └── cache
            └── download
                └── github.com
                    └── pkg
                        └── errors
                            └── @v
                                └── list

8 directories, 1 file
```

这就标志着我们的 goproxy 服务搭建成功，并可以正常运作了。



##### 自定义包导入路径并将其映射到内部的 vcs 仓库

一般公司可能没有为 vcs 服务器分配域名，我们也不能在 Go 私有包的导入路径中放入 ip 地址，因此我们需要给我们的私有 Go Module 自定义一个路径，比如：mycompany.com/go/module1。我们统一将私有 Go Module 放在 mycompany.com/go 下面的代码仓库中。

那么，接下来的问题就是，当 goproxy 去拉取 mycompany.com/go/module1 时，应该得到 mycompany.com/go/module1 对应的内部 vcs 上 module1 仓库的地址，这样，goproxy 才能从内部 vcs 代码服务器上下载 module1 对应的代码，具体的过程如下：

![img](https://static001.geekbang.org/resource/image/1f/20/1f6fd2d693f78c77e60f118cf081c020.jpg?wh=1980x1080)

那么我们如何实现为私有 module 自定义包导入路径，并将它映射到内部的 vcs 仓库呢？



其实方案不止一种，这里我使用了 Google 云开源的一个名为 [govanityurls 的工具](https://github.com/GoogleCloudPlatform/govanityurls)，来为私有 module 自定义包导入路径。然后，结合 govanityurls 和 nginx，我们就可以将私有 Go Module 的导入路径映射为其在 vcs 上的代码仓库的真实地址。具体原理你可以看一下这张图：

![img](https://static001.geekbang.org/resource/image/b0/bd/b0eb4b4d2785d8f1a3af78b76e2b5dbd.jpg?wh=1980x1080)

首先，goproxy 要想不把收到的拉取私有 Go Module（mycompany.com/go/module1）的请求转发给公共代理，需要在其启动参数上做一些手脚，比如下面这个就是修改后的 goproxy 启动命令：

```
$./goproxy -listen=0.0.0.0:8081 -cacheDir=/root/.bin/goproxy/goproxy/bin/cache -proxy https://goproxy.io -exclude "mycompany.com/go"
```

这样，凡是与 -exclude 后面的值匹配的 Go Module 拉取请求，goproxy 都不会转给 goproxy.io，而是直接请求 Go Module 的“源站”。

而上面这张图中要做的，就是将这个“源站”的地址，转换为企业内部 vcs 服务中的一个仓库地址。然后我们假设 mycompany.com 这个域名并不存在（很多小公司没有内部域名解析能力），从图中我们可以看到，我们会在 goproxy 所在节点的 /etc/hosts 中加上这样一条记录：

```
127.0.0.1 mycompany.com
```

这样做了后，goproxy 发出的到 mycompany.com 的请求实际上是发向了本机。而上面这图中显示，监听本机 80 端口的正是 nginx，nginx 关于 mycompany.com 这一主机的配置如下：

```
// /etc/nginx/conf.d/gomodule.conf

server {
        listen 80;
        server_name mycompany.com;

        location /go {
                proxy_pass http://127.0.0.1:8080;
                proxy_redirect off;
                proxy_set_header Host $host;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

                proxy_http_version 1.1;
                proxy_set_header Upgrade $http_upgrade;
                proxy_set_header Connection "upgrade";
        }
}
```

我们看到，对于路径为 mycompany.com/go/xxx 的请求，nginx 将请求转发给了 127.0.0.1:8080，而这个服务地址恰恰就是 govanityurls 工具监听的地址。

govanityurls 这个工具，是前 Go 核心开发团队成员 Jaana B.Dogan 开源的一个工具，这个工具可以帮助 Gopher 快速实现自定义 Go 包的 go get 导入路径。

govanityurls 本身，就好比一个“导航”服务器。当 go 命令向自定义包地址发起请求时，实际上是将请求发送给了 govanityurls 服务，之后，govanityurls 会将请求中的包所在仓库的真实地址（从 vanity.yaml 配置文件中读取）返回给 go 命令，后续 go 命令再从真实的仓库地址获取包数据。

> 注：govanityurls 的安装方法很简单，直接 go install/go get github.com/GoogleCloudPlatform/govanityurls 就可以了。在我们的示例中，vanity.yaml 的配置如下：

```
host: mycompany.com

paths:
  /go/module1:
      repo: ssh://admin@10.10.30.30/module1
      vcs: git
```

也就是说，当 govanityurls 收到 nginx 转发的请求后，会将请求与 vanity.yaml 中配置的 module 路径相匹配，如果匹配 ok，就会将该 module 的真实 repo 地址，通过 go 命令期望的应答格式返回。

在这里我们看到，module1 对应的真实 vcs 上的仓库地址为：ssh://admin@10.10.30.30/module1。所以，goproxy 会收到这个地址，并再次向这个真实地址发起请求，并最终将 module1 缓存到本地 cache 并返回给客户端。



##### 开发机 (客户端) 的设置

前面示例中，我们已经将开发机的 GOPROXY 环境变量，设置为 goproxy 的服务地址。但我们说过，凡是通过 GOPROXY 拉取的 Go Module，go 命令都会默认把它的 sum 值放到公共 GOSUM 服务器上去校验。

但我们实质上拉取的是私有 Go Module，GOSUM 服务器上并没有我们的 Go Module 的 sum 数据。这样就会导致 go build 命令报错，无法继续构建过程。

因此，开发机客户端还需要将 mycompany.com/go，作为一个值设置到 GONOSUMDB 环境变量中：

```
export GONOSUMDB=mycompany.com/go
```

这个环境变量配置一旦生效，就相当于告诉 go 命令，凡是与 mycompany.com/go 匹配的 Go Module，都不需要在做 sum 校验了。

到这里，我们就实现了拉取私有 Go Module 的方案。



##### 方案的“不足”

开发者还是需要额外配置 GONOSUMDB 变量

由于 Go 命令默认会对从 GOPROXY 拉取的 Go Module 进行 sum 校验，因此我们需要将私有 Go Module 配置到 GONOSUMDB 环境变量中，这就给开发者带来了一个小小的“负担”。对于这个问题，我的解决建议是：公司内部可以将私有 go 项目都放在一个特定域名下，这样就不需要为每个 go 私有项目单独增加 GONOSUMDB 配置了，只需要配置一次就可以了。



新增私有 Go Module，vanity.yaml 需要手工同步更新

这是这个方案最不灵活的地方了，由于目前 govanityurls 功能有限，针对每个私有 Go Module，我们可能都需要单独配置它对应的 vcs 仓库地址，以及获取方式（git、svn or hg）。

关于这一点，我的建议是：在一个 vcs 仓库中管理多个私有 Go Module。相比于最初 go 官方建议的一个 repo 只管理一个 module，新版本的 go 在一个 repo 下管理多个 Go Module方面，已经有了长足的进步，我们已经可以通过 repo 的 tag 来区别同一个 repo 下的不同 Go Module。

不过对于一个公司或组织来说，这点额外工作与得到的收益相比，应该也不算什么！



无法划分权限

goproxy 所在节点需要具备访问所有私有 Go Module 所在 vcs repo 的权限，但又无法对 go 开发者端做出有差别授权，这样，只要是 goproxy 能拉取到的私有 Go Module，go 开发者都能拉取到。

不过对于多数公司而言，内部所有源码原则上都是企业内部公开的，这个问题似乎也不大。如果觉得这是个问题，那么只能使用前面提到的第一个方案，也就是直连私有 Go Module 的源码服务器的方案了。



### 仓库布局

是单 module 还是多 module？

如果没有单一仓库（monorepo）的强约束，那么在默认情况下，你选择一个仓库管理一个 module 是不会错的，这是管理 Go Module 的最简单的方式，也是最常用的标准方式。这种方式下，module 维护者维护起来会很方便，module 的使用者在引用 module 下面的包时，也可以很容易地确定包的导入路径。



#### 单 module

举个简单的例子，我们在 github.com/bigwhite/srsm 这个仓库下管理着一个 Go Module（srsm 是 single repo single module 的缩写）。

> 对于私有仓库的话，替换下域名，然后再拉包的时候 改下gitconfig 就差不多的



通常情况下，module path 与仓库地址保持一致，都是 github.com/bigwhite/srsm，这点会体现在 go.mod 中：

```
// go.mod
module github.com/bigwhite/srsm

go 1.17
```



然后我们对仓库打 tag，这个 tag 也会成为 Go Module 的版本号，这样，对仓库的版本管理其实就是对 Go Module 的版本管理。

如果这个仓库下的布局是这样的：

```
./srsm
├── go.mod
├── go.sum
├── pkg1/
│   └── pkg1.go
└── pkg2/
    └── pkg2.go
```

那么这个 module 的使用者可以很轻松地确定 pkg1 和 pkg2 两个包的导入路径，一个是 github.com/bigwhite/srsm/pkg1，另一个则是 github.com/bigwhite/srsm/pkg2。



注意：如果涉及到版本升级，我还是建议用tag 的方式来管理，因为这种方式可以用于多种场景。

比如 module 演进到了 v2.x.x 版本，那么以 pkg1 包为例，它的包的导入路径就变成了 github.com/bigwhite/srsm/v2/pkg1。

如果组织层面要求采用单一仓库（monorepo）模式，也就是所有 Go Module 都必须放在一个 repo 下，那我们只能使用单 repo 下管理多个 Go Module 的方法了。



#### 多 module

记得 Go Module 的设计者 Russ Cox 曾说过：“在单 repo 多 module 的布局下，添加 module、删除 module，以及对 module 进行版本管理，都需要相当谨慎和深思熟虑，因此，管理一个单 module 的版本库，几乎总是比管理现有版本库中的多个 module 要容易和简单”。

我们也用一个例子来感受一下这句话的深意。这里是一个单 repo 多 module 的例子，我们假设 repo 地址是 github.com/bigwhite/srmm。

这个 repo 下的结构布局如下（srmm 是 single repo multiple modules 的缩写）：

```
./srmm
├── module1
│   ├── go.mod
│   └── pkg1
│       └── pkg1.go
└── module2
    ├── go.mod
    └── pkg2
        └── pkg2.go
```

注意：子module 和package区别在于go.mod 不一样。

srmm 仓库下面有两个 Go Module，分为位于子目录 module1 和 module2 的下面，这两个目录也是各自 module 的根目录（module root）。

这种情况下，module 的 path 也不能随意指定，必须包含子目录的名字。



我们以 module1 为例分析一下，它的 path 是 github.com/bigwhite/srmm/module1，只有这样，Go 命令才能根据用户导入包的路径，找到对应的仓库地址和在仓库中的相对位置。同理，module1 下的包名同样是以 module path 为前缀的，比如：github.com/bigwhite/srmm/module1/pkg1。

在单仓库多 module 模式下，各个 module 的版本是独立维护的。因此，我们在通过打 tag 方式发布某个 module 版本时，tag 的名字必须包含子目录名。

比如：如果我们要发布 module1 的 v1.0.0 版本，我们不能通过给仓库打 v1.0.0 这个 tag 号来发布 module1 的 v1.0.0 版本，正确的作法应该是打 module1/v1.0.0 这个 tag 号。



你现在可能觉得这样理解起来也没有多复杂，但当各个 module 的主版本号升级时，你就会感受到这种方式带来的繁琐了，这个我们稍后再细说。

注：推荐单module好，不同package不用做切割，实在没办法再新起一个。



### 发布 Go Module

当我们的 module 完成开发与测试，module 便可以发布了。发布的步骤也十分简单，就是为 repo 打上 tag 并推送到代码服务器上就好了。



如果采用单 repo 单 module 管理方式，那么我们给 repo 打的 tag 就是 module 的版本。

如果采用的是单 repo 多 module 的管理方式，那么我们就需要注意在 tag 中加上各个 module 的子目录名，这样才能起到发布某个 module 版本的作用，否则 module 的用户通过 go get xxx@latest 也无法看到新发布的 module 版本。

而且，这里还有一个需要你特别注意的地方，如果你在发布正式版之前先发布了 alpha 或 beta 版给大家公测使用，那么你一定要提醒你的 module 的使用者，让他们通过 go get 指定公测版本号来显式升级依赖，比如：

```
$go get github.com/bigwhite/srsm@v1.1.0-beta.1
```

这样，go get 工具才会将使用者项目依赖的 github.com/bigwhite/srsm 的版本更新为 v1.1.0-beta.1。

而我们通过 go get github.com/bigwhite/srsm@latest，是不会获取到像上面 v1.1.0-beta.1 这样的发布前的公测版本的





### 作废特定版本的 Go Module

多数情况下，Go Module 的维护者可以正确地发布 Go Module。

但人总是会犯错的，作为 Go Module 的作者或维护者，我们偶尔也会出现这样的低级错误：将一个处于 broken 状态的 module 发布了出去。

那一旦出现这样的情况，我们该怎么做呢？我们继续向下看。

我们先来看看如果发布了错误的 module 版本会对 module 的使用者带去什么影响。



#### 错误版本的危害

我们直接来看一个例子。假设 bitbucket.org/bigwhite/m1 是我维护的一个 Go Module，它目前已经演进到 v1.0.1 版本了，并且有两个使用者 c1 和 c2，你可以看下这个示意图，能更直观地了解 m1 当前的状态：

![img](https://static001.geekbang.org/resource/image/23/0a/23931514c6ea70debaeb9e5709cec20a.jpg?wh=1920x861)



某一天，我一不小心，就把一个处于 broken 状态的 module 版本，m1@v1.0.2 发布出去了！此时此刻，m1 的 v1.0.2 版本还只存在于它的源仓库站点上，也就是 bitbucket/bigwhite/m1 中，在任何一个 GoProxy 服务器上都还没有这个版本的缓存。



这个时候，依赖 m1 的两个项目 c1 和 c2 依赖的仍然是 m1@v1.0.1 版本。也就是说，如果没有显式升级 m1 的版本，c1 和 c2 的构建就不会受到处于 broken 状态的 module v1.0.2 版本的影响，这也是 Go Module 最小版本选择的优点。



而且，由于 m1@v1.0.2 还没有被 GoProxy 服务器缓存，在 GOPROXY 环境变量开启的情况下，go list 是查不到 m1 有可升级的版本的：

```
// 以c2为例：
$go list -m -u all
github.com/bigwhite/c2
bitbucket.org/bigwhite/m1 v1.0.1
```



但如若我们绕开 GOPROXY，那么 go list 就可以查找到 m1 的最新版本为 v1.0.2（我们通过设置 GONOPROXY 来让 go list 查询 m1 的源仓库而不是代理服务器上的缓存）：

```
$GONOPROXY="bitbucket.org/bigwhite/m1" go list -m -u all
github.com/bigwhite/c2
bitbucket.org/bigwhite/m1 v1.0.1 [v1.0.2]
```



之后，如果某个 m1 的消费者，比如 c2，通过 go get bitbucket.org/bigwhite/m1@v1.0.2 对 m1 的依赖版本进行了显式更新，那就会触发 GOPROXY 对 m1@v1.0.2 版本的缓存。

这样一通操作后，module proxy 以及 m1 的消费者的当前的状态就会变成这样：

![img](https://static001.geekbang.org/resource/image/85/31/85fyy48dd62570758dd21666a4646631.jpg?wh=1920x972)

由于 Goproxy 服务已经缓存了 m1 的 v1.0.2 版本，这之后，m1 的其他消费者，比如 c1 就能够在 GOPROXY 开启的情况下查询到 m1 存在新版本 v1.0.2，即便它是 broken 的：

```
// 以c1为例：
$go list -m -u all
github.com/bigwhite/c1
bitbucket.org/bigwhite/m1 v1.0.1 [v1.0.2]
```

但是，一旦 broken 的 m1 版本（v1.0.2）进入到 GoProxy 的缓存，那么它的“危害性”就会“大肆传播”开。

这时 module m1 的新消费者都将受到影响！比如这里我们引入一个新的消费者 c3，c3 的首次构建就会因 m1 的损坏而报错：

![img](https://static001.geekbang.org/resource/image/73/00/737eb3121d9fa70387742de128eba500.jpg?wh=1920x1252)

糟糕的情况已经出现了！那我们怎么作废掉 m1@v1.0.2 版本来修复这个问题呢？

如果在 GOPATH 时代，废掉一个之前发的包版本是分分钟的事情，因为那时包消费者依赖的都是 latest commit。包作者只要 fix 掉问题、提交并重新发布就可以了。

但是在 Go Module 时代，作废掉一个已经发布了的 Go Module 版本还真不是一件能轻易做好的事情。

这很大程度是源于大量 Go Module 代理服务器的存在，Go Module 代理服务器会将已发布的 broken 的 module 缓存起来。下面我们来看看可能的问题解决方法。



#### 修复 broken 版本并重新发布

要解决掉这个问题，Go Module 作者有一个很直接的解决方法，就是：修复 broken 的 module 版本并重新发布。

它的操作步骤也很简单：

m1 的作者只需要删除掉远程的 tag: v1.0.2；

在本地 fix 掉问题，然后重新 tag v1.0.2 并 push 发布到 bitbucket 上的仓库中就可以了。



但这样做真的能生效么？

理论上发布的bitbucket能立马生效，但对于已经 get 到 broken v1.0.2 的消费者来说，他们只需清除掉本地的 module cache（go clean -modcache），对于 m1 的新消费者，他们直接得到的就是重新发布后的 v1.0.2 版本。



但现实的情况时，现在大家都是通过 Goproxy 服务来获取 module 的。

所以，一旦一个 module 版本被发布，当某个消费者通过他配置的 goproxy 获取这个版本时，这个版本就会在被缓存在对应的代理服务器上。

后续 m1 的消费者通过这个 goproxy 服务器获取那个版本的 m1 时，请求不会再回到 m1 所在的源代码托管服务器。



这样，即便 m1 的源服务器上的 v1.0.2 版本得到了重新发布，散布在各个 goproxy 服务器上的 broken v1.0.2 也依旧存在，并且被“传播”到各个 m1 消费者的开发环境中，而重新发布后的 v1.0.2 版本却得不到“传播”的机会，我们还是用一张图来直观展示下这种“窘境”：

![img](https://static001.geekbang.org/resource/image/c0/2b/c07fba8655615046c49110fb91bb662b.jpg?wh=1920x1252)

因此，从消费者的角度看，m1 的 v1.0.2 版本依旧是那个 broken 的版本，这种解决措施无效！



那你可能会问，如果 m1 的作者删除了 bitbucket 上的 v1.0.2 这个发布版本，各大 goproxy 服务器上的 broken v1.0.2 版本是否也会被同步删除呢？

遗憾地告诉你：不会。

因为Goproxy 服务器当初的一个设计目标，就是尽可能地缓存更多 module。所以，即便某个 module 的源码仓库都被删除了，这个 module 的各个版本依旧会缓存在 goproxy 服务器上，这个 module 的消费者依然可以正常获取这个 module，并顺利构建。



因此，goproxy 服务器当前的实现都没有主动删掉某个 module 缓存的特性。当然了，这可能也不是绝对的，毕竟不同 goproxy 服务的实现有所不同。



#### 发布 module 的新 patch 版本

上面这种情况下，Go 社区更为常见的解决方式就是发布 module 的新 patch 版本

我们依然以上面的 m1 为例，现在我们废除掉 v1.0.2，在本地修正问题后，直接打 v1.0.3 标签，并发布 push 到远程代码服务器上。

这样 m1 的消费者以及 module proxy 的整体状态就变成这个样子了：

![img](https://static001.geekbang.org/resource/image/b7/35/b74c8b9182f3f195c11ff2e20964ed35.jpg?wh=1920x1045)

在这样的状态下，我们分别看看 m1 的消费者的情况：

对于依赖 m1@v1.0.1 版本的 c1，在未手工更新依赖版本的情况下，它仍然可以保持成功的构建；

对于 m1 的新消费者，比如 c4，它首次构建时使用的就是 m1 的最新 patch 版 v1.0.3，跨过了作废的 v1.0.2，并成功完成构建；

对于之前曾依赖 v1.0.2 版本的消费者 c2 来说，这个时候他们需要手工介入才能解决问题，也就是需要在 c2 环境中手工升级依赖版本到 v1.0.3，这样 c2 也会得到成功构建。



那这样，我们错误版本的问题就得到了缓解。

从 Go 1.16 版本开始，Go Module 作者还可以在 go.mod 中使用新增加的 [retract 指示符](https://go.dev/ref/mod#go-mod-file-retract)，标识出哪些版本是作废的且不推荐使用的。retract 的语法形式如下：

```
// go.mod
retract v1.0.0           // 作废v1.0.0版本
retract [v1.1.0, v1.2.0] // 作废v1.1.0和v1.2.0两个版本
```

我们还用 m1 为例，我们将 m1 的 go.mod 更新为如下内容：

```
//m1的go.mod
module bitbucket.org/bigwhite/m1

go 1.17

retract v1.0.2
```



然后将 m1 放入 v1.0.3 标签中并发布。现在 m1 的消费者 c2 要查看 m1 是否有最新版本时，可以查看到以下内容（c2 本地环境使用 go1.17 版本）：

```
$GONOPROXY=bitbucket.org/bigwhite/m1 go list -m -u all
... ...
bitbucket.org/bigwhite/m1 v1.0.2 (retracted) [v1.0.3]
```



从 go list 的输出结果中，我们看到了 v1.0.2 版本上有了 retracted 的提示，提示这个版本已经被 m1 的作者作废了，不应该再使用，应升级为 v1.0.3。

但 retracted 仅仅是一个提示作用，并不影响 go build 的结果，c2 环境（之前在 go.mod 中依赖 m1 的 v1.0.2）下的 go build 不会自动绕过 v1.0.2，除非显式更新到 v1.0.3。



不过，上面的这个 retract 指示符适合标记要作废的独立的 minor 和 patch 版本，如果要提示用某个 module 的某个大版本整个作废，我们用 Go 1.17 版本引入的 Deprecated 注释行更适合。下面是使用 Deprecated 注释行的例子：

```
// Deprecated: use bitbucket.org/bigwhite/m1/v2 instead.
module bitbucket.org/bigwhite/m1
```



如果我们在 module m1 的 go.mod 中使用了 Deprecated 注释，那么 m1 的消费者在 go get 获取 m1 版本时，或者是通过 go list 查看 m1 版本时，会收到相应的作废提示，以 go get 为例：

```
$go get bitbucket.org/bigwhite/m1@latest
go: downloading bitbucket.org/bigwhite/m1 v1.0.3
go: module bitbucket.org/bigwhite/m1 is deprecated: use bitbucket.org/bigwhite/m1/v2 instead.
... ...
```

不过 Deprecated 注释的影响也仅限于提示，它不会影响到消费者的项目构建与使用。



### 升级 module 的 major 版本号

随着 module 的演化，总有一天 module 会出现不兼容以前版本的 change，这就到了需要升级 module 的 major 版本号的时候了。



在前面的讲解中，我们学习了 Go Module 的语义导入版本机制，也就是 Go Module 规定：如果同一个包的新旧版本是兼容的，那么它们的包导入路径应该是相同的。反过来说，如果新旧两个包不兼容，那么应该采用不同的导入路径。



而且，我们知道，Go 团队采用了将“major 版本”作为导入路径的一部分的设计。这种设计支持在同一个项目中，导入同一个 repo 下的不同 major 版本的 module，比如：

```
import (
    "bitbucket.org/bigwhite/m1/pkg1"   // 导入major版本号为v0或v1的module下的pkg1
    pkg1v2 "bitbucket.org/bigwhite/m1/v2/pkg1" // 导入major版本号为v2的module下的pkg1
)
```



我们可以认为：在同一个 repo 下，不同 major 号的 module 就是完全不同的 module，甚至同一 repo 下，不同 major 号的 module 可以相互导入。



这样一来，对于 module 作者 / 维护者而言，升级 major 版本号，也就意味着高版本的代码要与低版本的代码彻底分开维护，通常 Go 社区会采用为新的 major 版本建立新的 major 分支的方式，来将不同 major 版本的代码分离开，这种方案被称为“major branch”的方案。



major branch 方案对于多数 gopher 来说，是一个过渡比较自然的方案，它通过建立 vN 分支并基于 vN 分支打 vN.x.x 的 tag 的方式，做 major 版本的发布。



那么，采用这种方案的 Go Module 作者升级 major 版本号时要怎么操作呢？



我们以将 bitbucket.org/bigwhite/m1 的 major 版本号升级到 v2 为例看看。首先，我们要建立 v2 代码分支并切换到 v2 分支上操作，然后修改 go.mod 文件中的 module path，增加 v2 后缀：

```
//go.mod
module bitbucket.org/bigwhite/m1/v2

go 1.17
```



这里要特别注意一点，如果 module 内部包间有相互导入，那么在升级 major 号的时候，这些包的 import 路径上也要增加 v2，否则，就会存在在高 major 号的 module 代码中，引用低 major 号的 module 代码的情况，这也是 module 作者最容易忽略的事情。



这样一通操作后，我们就将 repo 下的 module 分为了两个 module 了，一个是原先的 v0/v1 module，在 master/main 分支上；新建的 v2 分支承载了 major 号为 2 的 module 的代码。major 号升级的这个操作过程还是很容易出错的，你操作时一定要谨慎。



对于消费者而言，在它依赖的 module 进行 major 版本号升级后，他们只需要在这个依赖 module 的 import 路径的后面，增加 /vN 就可以了（这里是 /v2），当然代码中也要针对不兼容的部分进行修改，然后 go 工具就会自动下载相关 module。



早期 Go 团队还提供了利用子目录分割不同 major 版本的方案，我们也看看这种方式怎么样。



我们还是以 bitbucket.org/bigwhite/m1 为例，如果这个 module 已经演化到 v3 版本了，那么这个 module 所在仓库的目录结构应该是这样的：

```
# tree m1
m1
├── pkg1
│   └── pkg1.go
├── go.mod
├── v2
│   ├── pkg1 
│   │   └── pkg1.go
│   └── go.mod
└── v3
    ├── pkg1 
    │   └── pkg1.go
    └── go.mod
```

这里我们直接用 vN 作为子目录名字，在代码仓库中将不同版本 module 放置在不同的子目录中，这样，go 命令就会将仓库内的子目录名与 major 号匹配并找到对应版本的 module。



从描述上看，似乎这种通过子目录方式来实现 major 版本号升级，会更“简单”一些。但我总感觉这种方式有些“怪”，而且，其他主流语言也很少有用这种方式进行 major 版本号升级的。

另外一旦使用这种方式，我们似乎也很难利用 git 工具在不同 major 版本之间进行代码的 merge 了。目前 Go 文档中似乎也不再提这种方案了，我个人也建议你尽量使用 major 分支方案。



在实际操作中，也有一些 Go Module 的仓库，始终将 master 或 main 分支作为最高 major 版本的分支，然后建立低版本分支来维护低 major 版本的 module 代码，比如：etcd、go-redis等。



这种方式本质上和前面建立 major 分支的方式是一样的，并且这种方式更符合一个 Go Module 演化的趋势和作者的意图，也就是低版本的 Go Module 随着时间的推移将渐渐不再维护，而最新最高版本的 Go Module 是 module 作者最想让使用者使用的版本。



但在单 repo 多 module 管理方式下，升级 module 的 major 版本号有些复杂，我们需要分为两种情况来考虑。



#### 第一种情况：repo 下的所有 module 统一进行版本发布。

在这种情况下，我们只需要向上面所说的那样建立 vN 版本分支就可以了，在 vN 分支上对 repo 下所有 module 进行演进，统一打 tag 并发布。当然 tag 要采用带有 module 子目录名的那种方式，比如：module1/v2.0.0。



etcd 项目对旗下的 Go Module 的统一版本发布，就是用的这种方式。如果翻看一下 etcd 的项目，你会发现 etcd 只会建立少量的像 release-3.4、release-3.5 这样的 major 分支，基于这些分支，etcd 会统一发布 moduleName/v3.4.x 和 moduleName/v3.5.x 版本。



#### 第二个情况：repo 下的 module 各自独立进行版本发布。

在这种情况下，简单创建一个 major 号分支来维护 module 的方式，就会显得不够用了，我们很可能需要建立 major 分支矩阵。

假设我们的一个 repo 下管理了多个 module，从 m1 到 mN，那么 major 号需要升级时，我们就需要将 major 版本号与 module 做一个组合，形成下面的分支矩阵：

![img](https://static001.geekbang.org/resource/image/89/64/8932b28df291e7efbc537d4843cfc464.jpeg)

以 m1 为例，当 m1 的 major 版本号需要升级到 2 时，我们建立 v2_m1 major 分支专门用于维护和发布 m1 module 的 v2.x.x 版本。



