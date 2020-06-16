[toc]

------



### kubeadm 一键部署

在理解了容器技术之后，为什么不用容器部署 Kubernetes 呢？

这样，我只要给每个 Kubernetes 组件做一个容器镜像，然后在每台宿主机上用 docker run 指令启动这些组件容器，部署不就完成了吗？

事实上，在 Kubernetes 早期的部署脚本里，确实有一个脚本就是用 Docker 部署 Kubernetes 项目的，这个脚本相比于 SaltStack 等的部署方式，也的确简单了不少。

**但是，这样做会带来一个很麻烦的问题，即：如何容器化 kubelet。**

kubelet 是 Kubernetes 项目用来操作 Docker 等容器运行时的核心组件。可是，除了跟容器运行时打交道外，kubelet 在配置容器网络、管理容器数据卷时，都需要直接操作宿主机。

而如果现在 kubelet 本身就运行在一个容器里，那么直接操作宿主机就会变得很麻烦。对于网络配置来说还好，kubelet 容器可以通过不开启 Network Namespace（即 Docker 的 host network 模式）的方式，直接共享宿主机的网络栈。可是，要让 kubelet 隔着容器的 Mount Namespace 和文件系统，操作宿主机的文件系统，就有点儿困难了。

比如，如果用户想要使用 NFS 做容器的持久化数据卷，那么 kubelet 就需要在容器进行绑定挂载前，在宿主机的指定目录上，先挂载 NFS 的远程目录。

可是，这时候问题来了。由于现在 kubelet 是运行在容器里的，这就意味着它要做的这个“mount -F nfs”命令，被隔离在了一个单独的 Mount Namespace 中。即，kubelet 做的挂载操作，不能被“传播”到宿主机上。

对于这个问题，有人说，可以使用 setns() 系统调用，在宿主机的 Mount Namespace 中执行这些挂载操作；也有人说，应该让 Docker 支持一个–mnt=host 的参数。

但是，到目前为止，在容器里运行 kubelet，依然没有很好的解决办法，我也不推荐你用容器去部署 Kubernetes 项目。正因为如此，kubeadm 选择了一种妥协方案：

> 把 kubelet 直接运行在宿主机上，然后使用容器部署其他的 Kubernetes 组件。

所以，你使用 kubeadm 的第一步，是在机器上手动安装 kubeadm、kubelet 和 kubectl 这三个二进制文件。当然，kubeadm 的作者已经为各个发行版的 Linux 准备好了安装包，所以你只需要执行：

```

$ apt-get install kubeadm
```



接下来，你就可以使用“kubeadm init”部署 Master 节点了。



#### kubeadm init 的工作流程

当你执行 kubeadm init 指令后，kubeadm 首先要做的，是一系列的检查工作，以确定这台机器可以用来部署 Kubernetes。这一步检查，我们称为“Preflight Checks”，它可以为你省掉很多后续的麻烦。

其实，Preflight Checks 包括了很多方面，比如：

- Linux 内核的版本必须是否是 3.10 以上？
- Linux Cgroups 模块是否可用？机器的 hostname 是否标准？
- 在 Kubernetes 项目里，机器的名字以及一切存储在 Etcd 中的 API 对象，都必须使用标准的 DNS 命名（RFC 1123）。用户安装的 kubeadm 和 kubelet 的版本是否匹配？
- 机器上是不是已经安装了 Kubernetes 的二进制文件？
- Kubernetes 的工作端口 10250/10251/10252 端口是不是已经被占用？
- ip、mount 等 Linux 指令是否存在？
- Docker 是否已经安装？
- ……

在通过了 Preflight Checks 之后，kubeadm 要为你做的，是生成 Kubernetes 对外提供服务所需的各种证书和对应的目录。

Kubernetes 对外提供服务时，除非专门开启“不安全模式”，否则都要通过 HTTPS 才能访问 kube-apiserver。这就需要为 Kubernetes 集群配置好证书文件。

kubeadm 为 Kubernetes 项目生成的证书文件都放在 Master 节点的 /etc/kubernetes/pki 目录下。在这个目录下，最主要的证书文件是 ca.crt 和对应的私钥 ca.key。

此外，用户使用 kubectl 获取容器日志等 streaming 操作时，需要通过 kube-apiserver 向 kubelet 发起请求，这个连接也必须是安全的。kubeadm 为这一步生成的是 apiserver-kubelet-client.crt 文件，对应的私钥是 apiserver-kubelet-client.key。

除此之外，Kubernetes 集群中还有 Aggregate APIServer 等特性，也需要用到专门的证书，这里我就不再一一列举了。需要指出的是，你可以选择不让 kubeadm 为你生成这些证书，而是拷贝现有的证书到如下证书的目录里：

```

/etc/kubernetes/pki/ca.{crt,key}
```

这时，kubeadm 就会跳过证书生成的步骤，把它完全交给用户处理。

证书生成后，kubeadm 接下来会为其他组件生成访问 kube-apiserver 所需的配置文件。这些文件的路径是：/etc/kubernetes/xxx.conf：

```

ls /etc/kubernetes/
admin.conf  controller-manager.conf  kubelet.conf  scheduler.conf
```

这些文件里面记录的是，当前这个 Master 节点的服务器地址、监听端口、证书目录等信息。这样，对应的客户端（比如 scheduler，kubelet 等），可以直接加载相应的文件，使用里面的信息与 kube-apiserver 建立安全连接。



接下来，kubeadm 会为 Master 组件生成 Pod 配置文件。我已经在上一篇文章中和你介绍过 Kubernetes 有三个 Master 组件 kube-apiserver、kube-controller-manager、kube-scheduler，而它们都会被使用 Pod 的方式部署起来。

你可能会有些疑问：这时，Kubernetes 集群尚不存在，难道 kubeadm 会直接执行 docker run 来启动这些容器吗？

在 Kubernetes 中，有一种特殊的容器启动方法叫做“Static Pod”。它允许你把要部署的 Pod 的 YAML 文件放在一个指定的目录里。这样，当这台机器上的 kubelet 启动时，它会自动检查这个目录，加载所有的 Pod YAML 文件，然后在这台机器上启动它们。

从这一点也可以看出，kubelet 在 Kubernetes 项目中的地位非常高，在设计上它就是一个完全独立的组件，而其他 Master 组件，则更像是辅助性的系统容器。

在 kubeadm 中，Master 组件的 YAML 文件会被生成在 /etc/kubernetes/manifests 路径下。比如，kube-apiserver.yaml：

```

apiVersion: v1
kind: Pod
metadata:
  annotations:
    scheduler.alpha.kubernetes.io/critical-pod: ""
  creationTimestamp: null
  labels:
    component: kube-apiserver
    tier: control-plane
  name: kube-apiserver
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-apiserver
    - --authorization-mode=Node,RBAC
    - --runtime-config=api/all=true
    - --advertise-address=10.168.0.2
    ...
    - --tls-cert-file=/etc/kubernetes/pki/apiserver.crt
    - --tls-private-key-file=/etc/kubernetes/pki/apiserver.key
    image: k8s.gcr.io/kube-apiserver-amd64:v1.11.1
    imagePullPolicy: IfNotPresent
    livenessProbe:
      ...
    name: kube-apiserver
    resources:
      requests:
        cpu: 250m
    volumeMounts:
    - mountPath: /usr/share/ca-certificates
      name: usr-share-ca-certificates
      readOnly: true
    ...
  hostNetwork: true
  priorityClassName: system-cluster-critical
  volumes:
  - hostPath:
      path: /etc/ca-certificates
      type: DirectoryOrCreate
    name: etc-ca-certificates
  ...
```



关于一个 Pod 的 YAML 文件怎么写、里面的字段如何解读，我会在后续专门的文章中为你详细分析。在这里，你只需要关注这样几个信息：

- 这个 Pod 里只定义了一个容器，它使用的镜像是：k8s.gcr.io/kube-apiserver-amd64:v1.11.1 。这个镜像是 Kubernetes 官方维护的一个组件镜像。
- 这个容器的启动命令（commands）是 kube-apiserver --authorization-mode=Node,RBAC …，这样一句非常长的命令。其实，它就是容器里 kube-apiserver 这个二进制文件再加上指定的配置参数而已。
- 如果你要修改一个已有集群的 kube-apiserver 的配置，需要修改这个 YAML 文件。
- 这些组件的参数也可以在部署时指定

在这一步完成后，kubeadm 还会再生成一个 Etcd 的 Pod YAML 文件，用来通过同样的 Static Pod 的方式启动 Etcd。所以，最后 Master 组件的 Pod YAML 文件如下所示：

```

$ ls /etc/kubernetes/manifests/
etcd.yaml  kube-apiserver.yaml  kube-controller-manager.yaml  kube-scheduler.yaml
```

而一旦这些 YAML 文件出现在被 kubelet 监视的 /etc/kubernetes/manifests 目录下，kubelet 就会自动创建这些 YAML 文件中定义的 Pod，即 Master 组件的容器。

Master 容器启动后，kubeadm 会通过检查 localhost:6443/healthz 这个 Master 组件的健康检查 URL，等待 Master 组件完全运行起来。

**然后，kubeadm 就会为集群生成一个 bootstrap token。**在后面，只要持有这个 token，任何一个安装了 kubelet 和 kubadm 的节点，都可以通过 kubeadm join 加入到这个集群当中。

这个 token 的值和使用方法，会在 kubeadm init 结束后被打印出来。

在 token 生成之后，kubeadm 会将 ca.crt 等 Master 节点的重要信息，通过 ConfigMap 的方式保存在 Etcd 当中，供后续部署 Node 节点使用。这个 ConfigMap 的名字是 cluster-info。

kubeadm init 的最后一步，就是安装默认插件。Kubernetes 默认 kube-proxy 和 DNS 这两个插件是必须安装的。它们分别用来提供整个集群的服务发现和 DNS 功能。其实，这两个插件也只是两个容器镜像而已，所以 kubeadm 只要用 Kubernetes 客户端创建两个 Pod 就可以了。



#### kubeadm join 的工作流程

kubeadm init 生成 bootstrap token 之后，你就可以在任意一台安装了 kubelet 和 kubeadm 的机器上执行 kubeadm join 了。可是，为什么执行 kubeadm join 需要这样一个 token 呢？

因为，任何一台机器想要成为 Kubernetes 集群中的一个节点，就必须在集群的 kube-apiserver 上注册。可是，要想跟 apiserver 打交道，这台机器就必须要获取到相应的证书文件（CA 文件）。可是，为了能够一键安装，我们就不能让用户去 Master 节点上手动拷贝这些文件。

所以，kubeadm 至少需要发起一次“不安全模式”的访问到 kube-apiserver，从而拿到保存在 ConfigMap 中的 cluster-info（它保存了 APIServer 的授权信息）。而 bootstrap token，扮演的就是这个过程中的安全验证的角色。

只要有了 cluster-info 里的 kube-apiserver 的地址、端口、证书，kubelet 就可以以“安全模式”连接到 apiserver 上，这样一个新的节点就部署完成了。接下来，你只要在其他节点上重复这个指令就可以了。



#### 配置 kubeadm 的部署参数

kubeadm 部署 Kubernetes 集群最关键的两个步骤，kubeadm init 和 kubeadm join。相信你一定会有这样的疑问：kubeadm 确实简单易用，可是我又该如何定制我的集群组件参数呢？比如，我要指定 kube-apiserver 的启动参数，该怎么办？

在这里，我强烈推荐你在使用 kubeadm init 部署 Master 节点时，使用下面这条指令：

```

$ kubeadm init --config kubeadm.yaml
```

这时，你就可以给 kubeadm 提供一个 YAML 文件（比如，kubeadm.yaml），它的内容如下所示（我仅列举了主要部分）：

```

apiVersion: kubeadm.k8s.io/v1alpha2
kind: MasterConfiguration
kubernetesVersion: v1.11.0
api:
  advertiseAddress: 192.168.0.102
  bindPort: 6443
  ...
etcd:
  local:
    dataDir: /var/lib/etcd
    image: ""
imageRepository: k8s.gcr.io
kubeProxy:
  config:
    bindAddress: 0.0.0.0
    ...
kubeletConfiguration:
  baseConfig:
    address: 0.0.0.0
    ...
networking:
  dnsDomain: cluster.local
  podSubnet: ""
  serviceSubnet: 10.96.0.0/12
nodeRegistration:
  criSocket: /var/run/dockershim.sock
  ...
```



通过制定这样一个部署参数配置文件，你就可以很方便地在这个文件里填写各种自定义的部署参数了。比如，我现在要指定 kube-apiserver 的参数，那么我只要在这个文件里加上这样一段信息：

```

...
apiServerExtraArgs:
  advertise-address: 192.168.0.103
  anonymous-auth: false
  enable-admission-plugins: AlwaysPullImages,DefaultStorageClass
  audit-log-path: /home/johndoe/audit.log
```

然后，kubeadm 就会使用上面这些信息替换 /etc/kubernetes/manifests/kube-apiserver.yaml 里的 command 字段里的参数了。

而这个 YAML 文件提供的可配置项远不止这些。比如，你还可以修改 kubelet 和 kube-proxy 的配置，修改 Kubernetes 使用的基础镜像的 URL（默认的k8s.gcr.io/xxx镜像 URL 在国内访问是有困难的），指定自己的证书文件，指定特殊的容器运行时等等。这些配置项，就留给你在后续实践中探索了。





### 生产环境部署

#### 准备工作

最直接的办法，自然是到公有云上申请几个虚拟机。当然，如果条件允许的话，拿几台本地的物理服务器来组集群是最好不过了。这些机器只要满足如下几个条件即可：

- 满足安装 Docker 项目所需的要求，比如 64 位的 Linux 操作系统、3.10 及以上的内核版本；
- x86 或者 ARM 架构均可；
- 机器之间网络互通，这是将来容器之间网络互通的前提；
- 有外网访问权限，因为需要拉取镜像；
- 能够访问到gcr.io、quay.io这两个 docker registry，因为有小部分镜像需要在这里拉取；
- 单机可用资源建议 2 核 CPU、8 GB 内存或以上，再小的话问题也不大，但是能调度的 Pod 数量就比较有限了；30 GB 或以上的可用磁盘空间，这主要是留给 Docker 镜像和日志文件用的。

> 备注：在开始部署前，我推荐你先花几分钟时间，回忆一下 Kubernetes 的架构。

实践的目标：

- 在所有节点上安装 Docker 和 kubeadm；
- 部署 Kubernetes Master；
- 部署容器网络插件；
- 部署 Kubernetes Worker；
- 部署 Dashboard 可视化插件；
- 部署容器存储插件。

#### 安装 kubeadm 和 Docker

直接在 root 用户下进行操作。

```

$ curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
$ cat <<EOF > /etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF
$ apt-get update
$ apt-get install -y docker.io kubeadm
```

> 提示：如果 apt.kubernetes.io 因为网络问题访问不到，可以换成中科大的 Ubuntu 镜像源 deb http://mirrors.ustc.edu.cn/kubernetes/apt kubernetes-xenial main。

在上述安装 kubeadm 的过程中，kubeadm 和 kubelet、kubectl、kubernetes-cni 这几个二进制文件都会被自动安装好。另外，这里我直接使用 Ubuntu 的 docker.io 的安装源，原因是 Docker 公司每次发布的最新的 Docker CE（社区版）产品往往还没有经过 Kubernetes 项目的验证，可能会有兼容性方面的问题。



#### 部署 Kubernetes 的 Master 节点

kubeadm 可以一键部署 Master 节点。不过，既然要部署一个“完整”的 Kubernetes 集群，那我们不妨模拟生产环境：通过配置文件来开启一些实验性功能。

编写了一个给 kubeadm 用的 YAML 文件（名叫：kubeadm.yaml）：

```

apiVersion: kubeadm.k8s.io/v1alpha1
kind: MasterConfiguration
controllerManagerExtraArgs:
  horizontal-pod-autoscaler-use-rest-clients: "true"
  horizontal-pod-autoscaler-sync-period: "10s"
  node-monitor-grace-period: "10s"
apiServerExtraArgs:
  runtime-config: "api/all=true"
kubernetesVersion: "stable-1.11"
```

这个配置中，我给 kube-controller-manager 设置了：

```

horizontal-pod-autoscaler-use-rest-clients: "true"
```

这意味着，将来部署的 kube-controller-manager 能够使用自定义资源（Custom Metrics）进行自动水平扩展。

其中，“stable-1.11”就是 kubeadm 帮我们部署的 Kubernetes 版本号，即：Kubernetes release 1.11 最新的稳定版，在我的环境下，它是 v1.11.1。你也可以直接指定这个版本，比如：kubernetesVersion: “v1.11.1”。

```

$ kubeadm init --config kubeadm.yaml
```

就可以完成 Kubernetes Master 的部署了，这个过程只需要几分钟。部署完成后，kubeadm 会生成一行指令：

```

kubeadm join 10.168.0.2:6443 --token 00bwbx.uvnaa2ewjflwu1ry --discovery-token-ca-cert-hash sha256:00eb62a2a6020f94132e3fe1ab721349bbcd3e9b94da9654cfe15f2985ebd711
```

这个 kubeadm join 命令，就是用来给这个 Master 节点添加更多工作节点（Worker）的命令。我们在后面部署 Worker 节点的时候马上会用到它，所以找一个地方把这条命令记录下来。

此外，kubeadm 还会提示我们第一次使用 Kubernetes 集群所需要的配置命令：

```

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

而需要这些配置命令的原因是：Kubernetes 集群默认需要加密方式访问。所以，这几条命令，就是将刚刚部署生成的 Kubernetes 集群的安全配置文件，保存到当前用户的.kube 目录下，kubectl 默认会使用这个目录下的授权信息访问 Kubernetes 集群。



如果不这么做的话，我们每次都需要通过 export KUBECONFIG 环境变量告诉 kubectl 这个安全配置文件的位置。

现在，我们就可以使用 kubectl get 命令来查看当前唯一一个节点的状态了：

```

$ kubectl get nodes

NAME      STATUS     ROLES     AGE       VERSION
master    NotReady   master    1d        v1.11.1
```

可以看到，这个 get 指令输出的结果里，Master 节点的状态是 NotReady，这是为什么呢？在调试 Kubernetes 集群时，最重要的手段就是用 kubectl describe 来查看这个节点（Node）对象的详细信息、状态和事件（Event），我们来试一下：

```

$ kubectl describe node master

...
Conditions:
...

Ready   False ... KubeletNotReady  runtime network not ready: NetworkReady=false reason:NetworkPluginNotReady message:docker: network plugin is not ready: cni config uninitialized
```

通过 kubectl describe 指令的输出，我们可以看到 NodeNotReady 的原因在于，我们尚未部署任何网络插件。另外，我们还可以通过 kubectl 检查这个节点上各个系统 Pod 的状态，其中，kube-system 是 Kubernetes 项目预留的系统 Pod 的工作空间（Namepsace，注意它并不是 Linux Namespace，它只是 Kubernetes 划分不同工作空间的单位）：

```

$ kubectl get pods -n kube-system

NAME               READY   STATUS   RESTARTS  AGE
coredns-78fcdf6894-j9s52     0/1    Pending  0     1h
coredns-78fcdf6894-jm4wf     0/1    Pending  0     1h
etcd-master           1/1    Running  0     2s
kube-apiserver-master      1/1    Running  0     1s
kube-controller-manager-master  0/1    Pending  0     1s
kube-proxy-xbd47         1/1    NodeLost  0     1h
kube-scheduler-master      1/1    Running  0     1s
```

可以看到，CoreDNS、kube-controller-manager 等依赖于网络的 Pod 都处于 Pending 状态，即调度失败。这当然是符合预期的：因为这个 Master 节点的网络尚未就绪。



#### 部署网络插件

在 Kubernetes 项目“一切皆容器”的设计理念指导下，部署网络插件非常简单，只需要执行一句 kubectl apply 指令，以 Weave 为例：

```

$ kubectl apply -f https://git.io/weave-kube-1.6
```

部署完成后，我们可以通过 kubectl get 重新检查 Pod 的状态：

```

$ kubectl get pods -n kube-system

NAME                             READY     STATUS    RESTARTS   AGE
coredns-78fcdf6894-j9s52         1/1       Running   0          1d
coredns-78fcdf6894-jm4wf         1/1       Running   0          1d
etcd-master                      1/1       Running   0          9s
kube-apiserver-master            1/1       Running   0          9s
kube-controller-manager-master   1/1       Running   0          9s
kube-proxy-xbd47                 1/1       Running   0          1d
kube-scheduler-master            1/1       Running   0          9s
weave-net-cmk27                  2/2       Running   0          19s
```

可以看到，所有的系统 Pod 都成功启动了，而刚刚部署的 Weave 网络插件则在 kube-system 下面新建了一个名叫 weave-net-cmk27 的 Pod，一般来说，这些 Pod 就是容器网络插件在每个节点上的控制组件。

Kubernetes 支持容器网络插件，使用的是一个名叫 CNI 的通用接口，它也是当前容器网络的事实标准，市面上的所有容器网络开源项目都可以通过 CNI 接入 Kubernetes，比如 Flannel、Calico、Canal、Romana 等等，它们的部署方式也都是类似的“一键部署”。

至此，Kubernetes 的 Master 节点就部署完成了。如果你只需要一个单节点的 Kubernetes，现在你就可以使用了。不过，在默认情况下，Kubernetes 的 Master 节点是不能运行用户 Pod 的，所以还需要额外做一个小操作。在本篇的最后部分，我会介绍到它。



#### 部署 Kubernetes 的 Worker 节点

Kubernetes 的 Worker 节点跟 Master 节点几乎是相同的，它们运行着的都是一个 kubelet 组件。唯一的区别在于，在 kubeadm init 的过程中，kubelet 启动后，Master 节点上还会自动运行 kube-apiserver、kube-scheduler、kube-controller-manger 这三个系统 Pod。

所以，相比之下，部署 Worker 节点反而是最简单的，只需要两步即可完成。

第一步，在所有 Worker 节点上执行“安装 kubeadm 和 Docker”一节的所有步骤。

第二步，执行部署 Master 节点时生成的 kubeadm join 指令：

```

$ kubeadm join 10.168.0.2:6443 --token 00bwbx.uvnaa2ewjflwu1ry --discovery-token-ca-cert-hash sha256:00eb62a2a6020f94132e3fe1ab721349bbcd3e9b94da9654cfe15f2985ebd711
```



#### 通过 Taint/Toleration 调整 Master 执行 Pod 的策略

默认情况下 Master 节点是不允许运行用户 Pod 的。而 Kubernetes 做到这一点，依靠的是 Kubernetes 的 Taint/Toleration 机制。

它的原理非常简单：一旦某个节点被加上了一个 Taint，即被“打上了污点”，那么所有 Pod 就都不能在这个节点上运行，因为 Kubernetes 的 Pod 都有“洁癖”。



除非，有个别的 Pod 声明自己能“容忍”这个“污点”，即声明了 Toleration，它才可以在这个节点上运行

其中，为节点打上“污点”（Taint）的命令是：

```

$ kubectl taint nodes node1 foo=bar:NoSchedule
```

这时，该 node1 节点上就会增加一个键值对格式的 Taint，即：foo=bar:NoSchedule。其中值里面的 NoSchedule，意味着这个 Taint 只会在调度新 Pod 时产生作用，而不会影响已经在 node1 上运行的 Pod，哪怕它们没有 Toleration。

那么 Pod 又如何声明 Toleration 呢？我们只要在 Pod 的.yaml 文件中的 spec 部分，加入 tolerations 字段即可：

```

apiVersion: v1
kind: Pod
...
spec:
  tolerations:
  - key: "foo"
    operator: "Equal"
    value: "bar"
    effect: "NoSchedule"
```

这个 Toleration 的含义是，这个 Pod 能“容忍”所有键值对为 foo=bar 的 Taint（ operator: “Equal”，“等于”操作）。

现在回到我们已经搭建的集群上来。这时，如果你通过 kubectl describe 检查一下 Master 节点的 Taint 字段，就会有所发现了：

```

$ kubectl describe node master

Name:               master
Roles:              master
Taints:             node-role.kubernetes.io/master:NoSchedule
```

可以看到，Master 节点默认被加上了node-role.kubernetes.io/master:NoSchedule这样一个“污点”，其中“键”是node-role.kubernetes.io/master，而没有提供“值”。

此时，你就需要像下面这样用“Exists”操作符（operator: “Exists”，“存在”即可）来说明，该 Pod 能够容忍所有以 foo 为键的 Taint，才能让这个 Pod 运行在该 Master 节点上：

```

apiVersion: v1
kind: Pod
...
spec:
  tolerations:
  - key: "foo"
    operator: "Exists"
    effect: "NoSchedule"
```

当然，如果你就是想要一个单节点的 Kubernetes，删除这个 Taint 才是正确的选择：

```

$ kubectl taint nodes --all node-role.kubernetes.io/master-
```

如上所示，我们在“node-role.kubernetes.io/master”这个键后面加上了一个短横线“-”，这个格式就意味着移除所有以“node-role.kubernetes.io/master”为键的 Taint。

到了这一步，一个基本完整的 Kubernetes 集群就部署完毕了。是不是很简单呢？

有了 kubeadm 这样的原生管理工具，Kubernetes 的部署已经被大大简化。更重要的是，像证书、授权、各个组件的配置等部署中最麻烦的操作，kubeadm 都已经帮你完成了。

接下来，我们再在这个 Kubernetes 集群上安装一些其他的辅助插件，比如 Dashboard 和存储插件。



#### 部署 Dashboard 可视化插件

在 Kubernetes 社区中，有一个很受欢迎的 Dashboard 项目，它可以给用户提供一个可视化的 Web 界面来查看当前集群的各种信息。毫不意外，它的部署也相当简单：

```

$ kubectl apply -f 
$ $ kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-rc6/aio/deploy/recommended.yaml
```

部署完成之后，我们就可以查看 Dashboard 对应的 Pod 的状态了：

```

$ kubectl get pods -n kube-system

kubernetes-dashboard-6948bdb78-f67xk   1/1       Running   0          1m
```

需要注意的是，由于 Dashboard 是一个 Web Server，很多人经常会在自己的公有云上无意地暴露 Dashboard 的端口，从而造成安全隐患。所以，1.7 版本之后的 Dashboard 项目部署完成后，默认只能通过 Proxy 的方式在本地访问。具体的操作，你可以查看 Dashboard 项目的官方文档。

而如果你想从集群外访问这个 Dashboard 的话，就需要用到 Ingress，我会在后面的文章中专门介绍这部分内容。



#### 部署容器存储插件

接下来，让我们完成这个 Kubernetes 集群的最后一块拼图：容器持久化存储。

在前面介绍容器原理时已经提到过，很多时候我们需要用数据卷（Volume）把外面宿主机上的目录或者文件挂载进容器的 Mount Namespace 中，从而达到容器和宿主机共享这些目录或者文件的目的。容器里的应用，也就可以在这些数据卷中新建和写入文件。



可是，如果你在某一台机器上启动的一个容器，显然无法看到其他机器上的容器在它们的数据卷里写入的文件。这是容器最典型的特征之一：无状态。

而容器的持久化存储，就是用来保存容器存储状态的重要手段：存储插件会在容器里挂载一个基于网络或者其他机制的远程数据卷，使得在容器里创建的文件，实际上是保存在远程存储服务器上，或者以分布式的方式保存在多个节点上，而与当前宿主机没有任何绑定关系。这样，无论你在其他哪个宿主机上启动新的容器，都可以请求挂载指定的持久化存储卷，从而访问到数据卷里保存的内容。这就是“持久化”的含义。

由于 Kubernetes 本身的松耦合设计，绝大多数存储项目，比如 Ceph、GlusterFS、NFS 等，都可以为 Kubernetes 提供持久化存储能力。在这次的部署实战中，我会选择部署一个很重要的 Kubernetes 存储插件项目：Rook。

Rook 项目是一个基于 Ceph 的 Kubernetes 存储插件（它后期也在加入对更多存储实现的支持）。不过，不同于对 Ceph 的简单封装，Rook 在自己的实现中加入了水平扩展、迁移、灾难备份、监控等大量的企业级功能，使得这个项目变成了一个完整的、生产级别可用的容器存储插件。

得益于容器化技术，用几条指令，Rook 就可以把复杂的 Ceph 存储后端部署起来：

```

$ kubectl apply -f https://raw.githubusercontent.com/rook/rook/master/cluster/examples/kubernetes/ceph/common.yaml

$ kubectl apply -f https://raw.githubusercontent.com/rook/rook/master/cluster/examples/kubernetes/ceph/operator.yaml

$ kubectl apply -f https://raw.githubusercontent.com/rook/rook/master/cluster/examples/kubernetes/ceph/cluster.yaml
```

在部署完成后，你就可以看到 Rook 项目会将自己的 Pod 放置在由它自己管理的两个 Namespace 当中：

```

$ kubectl get pods -n rook-ceph-system
NAME                                  READY     STATUS    RESTARTS   AGE
rook-ceph-agent-7cv62                 1/1       Running   0          15s
rook-ceph-operator-78d498c68c-7fj72   1/1       Running   0          44s
rook-discover-2ctcv                   1/1       Running   0          15s

$ kubectl get pods -n rook-ceph
NAME                   READY     STATUS    RESTARTS   AGE
rook-ceph-mon0-kxnzh   1/1       Running   0          13s
rook-ceph-mon1-7dn2t   1/1       Running   0          2s
```



这样，一个基于 Rook 持久化存储集群就以容器的方式运行起来了，而接下来在 Kubernetes 项目上创建的所有 Pod 就能够通过 Persistent Volume（PV）和 Persistent Volume Claim（PVC）的方式，在容器里挂载由 Ceph 提供的数据卷了。

而 Rook 项目，则会负责这些数据卷的生命周期管理、灾难备份等运维工作。关于这些容器持久化存储的知识，我会在后续章节中专门讲解。



这时候，你可能会有个疑问：为什么我要选择 Rook 项目呢？其实，是因为这个项目很有前途。如果你去研究一下 Rook 项目的实现，就会发现它巧妙地依赖了 Kubernetes 提供的编排能力，合理的使用了很多诸如 Operator、CRD 等重要的扩展特性（这些特性我都会在后面的文章中逐一讲解到）。这使得 Rook 项目，成为了目前社区中基于 Kubernetes API 构建的最完善也最成熟的容器存储插件。我相信，这样的发展路线，很快就会得到整个社区的推崇。

> 备注：其实，在很多时候，大家说的所谓“云原生”，就是“Kubernetes 原生”的意思。而像 Rook、Istio 这样的项目，正是贯彻这个思路的典范。在我们后面讲解了声明式 API 之后，相信你对这些项目的设计思想会有更深刻的体会。





集群的部署过程并不像传说中那么繁琐，这主要得益于：

- kubeadm 项目大大简化了部署 Kubernetes 的准备工作，尤其是配置文件、证书、二进制文件的准备和制作，以及集群版本管理等操作，都被 kubeadm 接管了。
- Kubernetes 本身“一切皆容器”的设计思想，加上良好的可扩展机制，使得插件的部署非常简便。



```
kubeadm最新版本已经为1.12，很多人遇到提示版本不对，重新安装低版本就好了
apt remove kubelet kubectl kubeadm
apt install kubelet=1.11.3-00
apt install kubectl=1.11.3-00
apt install kubeadm=1.11.3-00
```

kubeadm 1.14 配置文件

```
apiVersion: kubeadm.k8s.io/v1beta1
kind: ClusterConfiguration
controllerManager:
    extraArgs:
        horizontal-pod-autoscaler-use-rest-clients: "true"
        horizontal-pod-autoscaler-sync-period: "10s"
        node-monitor-grace-period: "10s"
apiServer:
    extraArgs:
        runtime-config: "api/all=true"
kubernetesVersion: "stable-1.14"
```

相关文档：https://godoc.org/k8s.io/kubernetes/cmd/kubeadm/app/apis/kubeadm/v1beta1

备注这个针对版本号为1.14的



最新版本的kubeadm.yaml

```

cat <<EOF > kubeadm.yaml
apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
controllerManager:
    extraArgs:
        horizontal-pod-autoscaler-use-rest-clients: "true"
        horizontal-pod-autoscaler-sync-period: "10s"
        node-monitor-grace-period: "10s"
apiServer:
    extraArgs:
        runtime-config: "api/all=true"
imageRepository: registry.aliyuncs.com/google_containers
kubernetesVersion: "v1.17.0"
EOF

同时，部署网络插件的命令：
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"

"kubeadm config print init-defaults"这个命令可以告诉我们kubeadm.yaml版本信息。
```





### 实践

#### 普通操作

有了容器镜像之后，你需要按照 Kubernetes 项目的规范和要求，将你的镜像组织为它能够“认识”的方式，然后提交上去。

Kubernetes 的必备技能：编写配置文件。

> 这些配置文件可以是 YAML 或者 JSON 格式的。为方便阅读与理解，在后面的讲解中，我会统一使用 YAML 文件来指代它们。

Kubernetes 跟 Docker 等很多项目最大的不同，就在于它不推荐你使用命令行的方式直接运行容器（虽然 Kubernetes 项目也支持这种方式，比如：kubectl run），而是希望你用 YAML 文件的方式，即：把容器的定义、参数、配置，统统记录在一个 YAML 文件中，然后用这样一句指令把它运行起来：

```

$ kubectl create -f 我的配置文件
```

这么做最直接的好处是，你会有一个文件能记录下 Kubernetes 到底“run”了什么。比如下面这个例子：

```

apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```

像这样的一个 YAML 文件，对应到 Kubernetes 中，就是一个 API Object（API 对象）。当你为这个对象的各个字段填好值并提交给 Kubernetes 之后，Kubernetes 就会负责创建出这些对象所定义的容器或者其他类型的 API 资源。

可以看到，这个 YAML 文件中的 Kind 字段，指定了这个 API 对象的类型（Type），是一个 Deployment。所谓 Deployment，是一个定义多副本应用（即多个副本 Pod）的对象，我在前面的文章中（也是第 9 篇文章《从容器到容器云：谈谈 Kubernetes 的本质》）曾经简单提到过它的用法。此外，Deployment 还负责在 Pod 定义发生变化时，对每个副本进行滚动更新（Rolling Update）。

在上面这个 YAML 文件中，我给它定义的 Pod 副本个数 (spec.replicas) 是：2。而这些 Pod 具体的又长什么样子呢？

为此，我定义了一个 Pod 模版（spec.template），这个模版描述了我想要创建的 Pod 的细节。在上面的例子里，这个 Pod 里只有一个容器，这个容器的镜像（spec.containers.image）是 nginx:1.7.9，这个容器监听端口（containerPort）是 80。

> Pod 就是 Kubernetes 世界里的“应用”；而一个应用，可以由多个容器组成。

需要注意的是，像这样使用一种 API 对象（Deployment）管理另一种 API 对象（Pod）的方法，在 Kubernetes 中，叫作“控制器”模式（controller pattern）。在我们的例子中，Deployment 扮演的正是 Pod 的控制器的角色。

这样的每一个 API 对象都有一个叫作 Metadata 的字段，这个字段就是 API 对象的“标识”，即元数据，它也是我们从 Kubernetes 里找到这个对象的主要依据。这其中最主要使用到的字段是 Labels。

顾名思义，Labels 就是一组 key-value 格式的标签。而像 Deployment 这样的控制器对象，就可以通过这个 Labels 字段从 Kubernetes 中过滤出它所关心的被控制对象。



比如，在上面这个 YAML 文件中，Deployment 会把所有正在运行的、携带“app: nginx”标签的 Pod 识别为被管理的对象，并确保这些 Pod 的总数严格等于两个。



而这个过滤规则的定义，是在 Deployment 的“spec.selector.matchLabels”字段。我们一般称之为：Label Selector。

另外，在 Metadata 中，还有一个与 Labels 格式、层级完全相同的字段叫 Annotations，它专门用来携带 key-value 格式的内部信息。所谓内部信息，指的是对这些信息感兴趣的，是 Kubernetes 组件本身，而不是用户。所以大多数 Annotations，都是在 Kubernetes 运行过程中，被自动加在这个 API 对象上。

一个 Kubernetes 的 API 对象的定义，大多可以分为 Metadata 和 Spec 两个部分。前者存放的是这个对象的元数据，对所有 API 对象来说，这一部分的字段和格式基本上是一样的；而后者存放的，则是属于这个对象独有的定义，用来描述它所要表达的功能。

在了解了上述 Kubernetes 配置文件的基本知识之后，我们现在就可以把这个 YAML 文件“运行”起来。正如前所述，你可以使用 kubectl create 指令完成这个操作：

```
$ kubectl create -f nginx-deployment.yaml
```

然后，通过 kubectl get 命令检查这个 YAML 运行起来的状态是不是与我们预期的一致：

```

$ kubectl get pods -l app=nginx
NAME                                READY     STATUS    RESTARTS   AGE
nginx-deployment-67594d6bf6-9gdvr   1/1       Running   0          10m
nginx-deployment-67594d6bf6-v6j7w   1/1       Running   0          10m
```

kubectl get 指令的作用，就是从 Kubernetes 里面获取（GET）指定的 API 对象。可以看到，在这里我还加上了一个 -l 参数，即获取所有匹配 app: nginx 标签的 Pod。需要注意的是，**在命令行中，所有 key-value 格式的参数，都使用“=”而非“:”表示。**



从这条指令返回的结果中，我们可以看到现在有两个 Pod 处于 Running 状态，也就意味着我们这个 Deployment 所管理的 Pod 都处于预期的状态。此外， 你还可以使用 kubectl describe 命令，查看一个 API 对象的细节，比如：

```

$ kubectl describe pod nginx-deployment-67594d6bf6-9gdvr
Name:               nginx-deployment-67594d6bf6-9gdvr
Namespace:          default
Priority:           0
PriorityClassName:  <none>
Node:               node-1/10.168.0.3
Start Time:         Thu, 16 Aug 2018 08:48:42 +0000
Labels:             app=nginx
                    pod-template-hash=2315082692
Annotations:        <none>
Status:             Running
IP:                 10.32.0.23
Controlled By:      ReplicaSet/nginx-deployment-67594d6bf6
...
Events:

  Type     Reason                  Age                From               Message

  ----     ------                  ----               ----               -------
  
  Normal   Scheduled               1m                 default-scheduler  Successfully assigned default/nginx-deployment-67594d6bf6-9gdvr to node-1
  Normal   Pulling                 25s                kubelet, node-1    pulling image "nginx:1.7.9"
  Normal   Pulled                  17s                kubelet, node-1    Successfully pulled image "nginx:1.7.9"
  Normal   Created                 17s                kubelet, node-1    Created container
  Normal   Started                 17s                kubelet, node-1    Started container
```

在 kubectl describe 命令返回的结果中，你可以清楚地看到这个 Pod 的详细信息，比如它的 IP 地址等等。其中，有一个部分值得你特别关注，它就是 Events（事件）。在 Kubernetes 执行的过程中，对 API 对象的所有重要操作，都会被记录在这个对象的 Events 里，并且显示在 kubectl describe 指令返回的结果中。

比如，对于这个 Pod，我们可以看到它被创建之后，被调度器调度（Successfully assigned）到了 node-1，拉取了指定的镜像（pulling image），然后启动了 Pod 里定义的容器（Started container）。所以，这个部分正是我们将来进行 Debug 的重要依据。如果有异常发生，你一定要第一时间查看这些 Events，往往可以看到非常详细的错误信息。

接下来，如果我们要对这个 Nginx 服务进行升级，把它的镜像版本从 1.7.9 升级为 1.8，要怎么做呢？

很简单，我们只要修改这个 YAML 文件即可。

```

...    
    spec:
      containers:
      - name: nginx
        image: nginx:1.8 #这里被从1.7.9修改为1.8
        ports:
      - containerPort: 80
```



可是，这个修改目前只发生在本地，如何让这个更新在 Kubernetes 里也生效呢？我们可以使用 kubectl replace 指令来完成这个更新：

```

 $ kubectl replace -f nginx-deployment.yaml
```

我推荐你使用 kubectl apply 命令，来统一进行 Kubernetes 对象的创建和更新操作，具体做法如下所示：

```

$ kubectl apply -f nginx-deployment.yaml

# 修改nginx-deployment.yaml的内容

$ kubectl apply -f nginx-deployment.yaml
```

这样的操作方法，是 Kubernetes“声明式 API”所推荐的使用方法。也就是说，作为用户，你不必关心当前的操作是创建，还是更新，你执行的命令始终是 kubectl apply，而 Kubernetes 则会根据 YAML 文件的内容变化，自动进行具体的处理。

而这个流程的好处是，它有助于帮助开发和运维人员，围绕着可以版本化管理的 YAML 文件，而不是“行踪不定”的命令行进行协作，从而大大降低开发人员和运维人员之间的沟通成本。

**当应用本身发生变化时，开发人员和运维人员可以依靠容器镜像来进行同步；当应用部署参数发生变化时，这些 YAML 文件就是他们相互沟通和信任的媒介。**

以上，就是 Kubernetes 发布应用的最基本操作了。



#### Volume操作

在 Kubernetes 中，Volume 是属于 Pod 对象的一部分。所以，我们就需要修改这个 YAML 文件里的 template.spec 字段，如下所示：

```

apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.8
        ports:
        - containerPort: 80
        volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: nginx-vol
      volumes:
      - name: nginx-vol
        emptyDir: {}
```

可以看到，我们在 Deployment 的 Pod 模板部分添加了一个 volumes 字段，定义了这个 Pod 声明的所有 Volume。它的名字叫作 nginx-vol，类型是 emptyDir。那什么是 emptyDir 类型呢？

它其实就等同于我们之前讲过的 Docker 的隐式 Volume 参数，即：不显式声明宿主机目录的 Volume。所以，Kubernetes 也会在宿主机上创建一个临时目录，这个目录将来就会被绑定挂载到容器所声明的 Volume 目录上。

> 难看到，Kubernetes 的 emptyDir 类型，只是把 Kubernetes 创建的临时目录作为 Volume 的宿主机目录，交给了 Docker。这么做的原因，是 Kubernetes 不想依赖 Docker 自己创建的那个 _data 目录。

而 Pod 中的容器，使用的是 volumeMounts 字段来声明自己要挂载哪个 Volume，并通过 mountPath 字段来定义容器内的 Volume 目录，比如：/usr/share/nginx/html。当然，Kubernetes 也提供了显式的 Volume 定义，它叫作 hostPath。比如下面的这个 YAML 文件：

```

 ...   
    volumes:
      - name: nginx-vol
        hostPath: 
          path:  " /var/data"
```

这样，容器 Volume 挂载的宿主机目录，就变成了 /var/data。在上述修改完成后，我们还是使用 kubectl apply 指令，更新这个 Deployment:

```

$ kubectl apply -f nginx-deployment.yaml
```

接下来，你可以通过 kubectl get 指令，查看两个 Pod 被逐一更新的过程：

```

$ kubectl get pods
NAME                                READY     STATUS              RESTARTS   AGE
nginx-deployment-5c678cfb6d-v5dlh   0/1       ContainerCreating   0          4s
nginx-deployment-67594d6bf6-9gdvr   1/1       Running             0          10m
nginx-deployment-67594d6bf6-v6j7w   1/1       Running             0          10m
$ kubectl get pods
NAME                                READY     STATUS    RESTARTS   AGE
nginx-deployment-5c678cfb6d-lg9lw   1/1       Running   0          8s
nginx-deployment-5c678cfb6d-v5dlh   1/1       Running   0          19s
```

从返回结果中，我们可以看到，新旧两个 Pod，被交替创建、删除，最后剩下的就是新版本的 Pod。这个滚动更新的过程，我也会在后续进行详细的讲解。然后，你可以使用 kubectl describe 查看一下最新的 Pod，就会发现 Volume 的信息已经出现在了 Container 描述部分：

```

...
Containers:
  nginx:
    Container ID:   docker://07b4f89248791c2aa47787e3da3cc94b48576cd173018356a6ec8db2b6041343
    Image:          nginx:1.8
    ...
    Environment:    <none>
    Mounts:
      /usr/share/nginx/html from nginx-vol (rw)
...
Volumes:
  nginx-vol:
    Type:    EmptyDir (a temporary directory that shares a pod's lifetime)
```

> 作为一个完整的容器化平台项目，Kubernetes 为我们提供的 Volume 类型远远不止这些

最后，你还可以使用 kubectl exec 指令，进入到这个 Pod 当中（即容器的 Namespace 中）查看这个 Volume 目录：

```

$ kubectl exec -it nginx-deployment-5c678cfb6d-lg9lw -- /bin/bash
# ls /usr/share/nginx/html
```

此外，你想要从 Kubernetes 集群中删除这个 Nginx Deployment 的话，直接执行：

```

$ kubectl delete -f nginx-deployment.yaml
```





像这样的 Kubernetes API 对象，往往由 Metadata 和 Spec 两部分组成，其中 Metadata 里的 Labels 字段是 Kubernetes 过滤对象的主要手段。在这些字段里面，容器想要使用的数据卷，也就是 Volume，正是 Pod 的 Spec 字段的一部分。而 Pod 里的每个容器，则需要显式的声明自己要挂载哪个 Volume。