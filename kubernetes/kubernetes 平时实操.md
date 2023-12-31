在容器这块，个人本地平时用到比较多的是docker+k8s（插件），之前用的是minikube 一直没调好，容易卡卡的现象；

```
在docker的设置页面
看到Docker Engine
看到Kubernetes
直接启用
```

```
然后brew install kubectl 客户端
然后进行尝试 kubectl cluster-info
kubeclt get nodes
```

```
如果还没成功，则看下dokcerDestop 的关于docker、Kubernetes 是不是处于runing状态
```



高版本创建serviceaccount不会默认生成secret，导致使用token去链接api-server不了

```
vim role-admin.yaml
```

```
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: admin-crb
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
subjects:
- kind: ServiceAccount
  name: admin-sa
  namespace: kube-system
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-sa
  namespace: kube-system
  labels:
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
---
apiVersion: v1
kind: Secret
metadata:
  name: admin-secret
  namespace: kube-system
  annotations:
    kubernetes.io/service-account.name: "admin-sa"
type: kubernetes.io/service-account-token

```

```
kubectl apply role-admin.yaml
```

```
如果有问题的话，先反向删除：kubectl delete role-admin.yaml
再编辑，再apply下
```

