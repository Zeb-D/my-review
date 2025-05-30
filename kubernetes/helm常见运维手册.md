# Helm 的作用与常见操作指南

Helm 是 Kubernetes 的包管理工具，相当于 Linux 系统中的 yum/apt，它简化了 Kubernetes 应用的部署和管理。以下是 Helm 的核心作用和常见操作：

## 一、Helm 的核心作用

### 1. 应用打包

- 将复杂的 Kubernetes 资源（Deployment/Service/ConfigMap 等）打包成 **Chart**
- 支持版本控制（通过 **Release** 管理）

### 2. 模板化部署

- 使用 **Go 模板语言** 动态生成 YAML
- 通过 `values.yaml` 实现配置与部署分离

### 3. 依赖管理

- 通过 `Chart.yaml` 声明依赖的其他 Charts
- 自动解决依赖关系（类似 npm/pip）

### 4. 生命周期管理

- 支持 **安装/升级/回滚/删除** 全生命周期操作
- 保留每次 Release 的历史记录

## 二、Helm 核心概念

| 概念        | 说明                                                         |
| :---------- | :----------------------------------------------------------- |
| **Chart**   | Helm 应用包，包含运行应用所需的所有资源定义                  |
| **Release** | Chart 的运行实例，每次安装都会生成新 Release                 |
| **Repo**    | Chart 仓库，存储和共享 Charts 的位置                         |
| **Values**  | 可覆盖的配置参数，支持文件(`values.yaml`)和命令行(`--set`)两种传递方式 |

## 三、Helm 常见操作

### 1. 基础操作命令

#### 仓库管理

```
# 添加仓库
helm repo add bitnami https://charts.bitnami.com/bitnami

# 更新仓库
helm repo update

# 列出仓库
helm repo list
```

#### Chart 操作

```
# 搜索 Chart
helm search repo nginx

# 查看 Chart 信息
helm show chart bitnami/nginx

# 下载 Chart 到本地
helm pull bitnami/nginx --untar
```

### 2. 安装与卸载

```
# 安装 Chart（生成随机 release 名称）
helm install bitnami/nginx --generate-name

# 指定 release 名称安装
helm install my-nginx bitnami/nginx

# 使用自定义 values 安装
helm install -f custom-values.yaml my-nginx bitnami/nginx

# 卸载 release
helm uninstall my-nginx
```

### 3. 升级与回滚

```
# 检查升级可能的变化（dry-run）
helm upgrade --install my-nginx bitnami/nginx -n default --dry-run

# 执行升级
helm upgrade my-nginx bitnami/nginx --set replicaCount=3

# 查看历史版本
helm history my-nginx

# 回滚到特定版本
helm rollback my-nginx 2
```

### 4. 状态查看

```
# 列出所有 release
helm list -A  # -A 查看所有命名空间

# 查看 release 状态
helm status my-nginx

# 查看生成的实际 YAML
helm get manifest my-nginx

# 查看使用的 values
helm get values my-nginx
```

## 四、高级操作

### 1. 创建自定义 Chart

```
# 创建新 Chart
helm create my-chart

# 目录结构
my-chart/
├── charts/          # 子 Chart
├── Chart.yaml       # Chart 元数据
├── templates/       # 模板文件
│   ├── deployment.yaml
│   ├── _helpers.tpl # 辅助模板
│   └── service.yaml
└── values.yaml      # 默认配置
```

### 2. 模板调试

```
# 渲染模板（不实际部署）
helm template my-chart

# 检查模板语法
helm lint my-chart

# 调试特定值
helm install --debug --dry-run my-release ./my-chart
```

### 3. 依赖管理



```
# Chart.yaml 示例
dependencies:
  - name: mysql
    version: "9.0.0"
    repository: "https://charts.bitnami.com/bitnami"
    condition: mysql.enabled
```



```
# 更新依赖
helm dependency update my-chart

# 打包 Chart（包含依赖）
helm package my-chart
```

## 五、生产环境最佳实践

1. **使用 Values 管理环境差异**

   

   ```
   # 目录结构
   ├── values-dev.yaml
   ├── values-staging.yaml
   └── values-prod.yaml
   
   # 部署时指定环境
   helm install -f values-prod.yaml my-app ./my-chart
   ```

2. **版本控制策略**

   - 遵循语义化版本 (SemVer)
   - 每次修改都递增 Chart 版本

3. **安全建议**

   

   ```
   # 使用 --atomic 自动回滚失败的升级
   helm upgrade --atomic my-app ./my-chart
   
   # 限制权限（使用 ServiceAccount）
   helm install --create-namespace -n my-ns --serviceaccount my-sa my-app ./my-chart
   ```

4. **CI/CD 集成**

   

   ```
   # GitLab CI 示例
   deploy:
     stage: deploy
     image: alpine/helm
     script:
       - helm upgrade --install my-app ./my-chart -f values-$ENV.yaml
     only:
       - main
   ```

## 六、常见问题解决

1. **错误：release already exists**

   ```
   # 使用 --replace 替换现有 release
   helm install --replace my-app ./my-chart
   ```

2. **查看失败原因**

   ```
   kubectl get pods -l app.kubernetes.io/instance=my-app
   kubectl logs <pod-name>
   ```

3. **清理失败 release**

   ```
   helm list --failed
   helm uninstall <release-name>
   ```

Helm 极大简化了 Kubernetes 应用的管理复杂度，通过模板化和版本控制实现了真正的 "Infrastructure as Code"。掌握这些核心操作后，可以高效管理复杂的云原生应用部署。





# Helm 进行镜像服务升级与回滚操作指南

Helm 提供了完整的应用生命周期管理能力，可以方便地进行镜像版本升级和回滚操作。以下是具体实现方法：

## 一、镜像升级操作

### 1. 通过命令行直接升级

```
# 方法1：使用 --set 直接指定新镜像
helm upgrade <RELEASE_NAME> <CHART_NAME> \
  --set image.repository=<NEW_IMAGE_REPO> \
  --set image.tag=<NEW_TAG>

# 示例：将nginx升级到1.23.0版本
helm upgrade my-nginx bitnami/nginx \
  --set image.repository=bitnami/nginx \
  --set image.tag=1.23.0
```

### 2. 通过 values 文件升级

```
# 方法2：修改values.yaml后升级
# 编辑values.yaml中的image部分
image:
  repository: bitnami/nginx
  tag: 1.23.0
  pullPolicy: IfNotPresent

# 执行升级
helm upgrade -f values.yaml my-nginx bitnami/nginx
```

### 3. 查看升级效果

```
# 查看新镜像是否生效
kubectl get pods -l app.kubernetes.io/instance=my-nginx \
  -o jsonpath='{.items[*].spec.containers[*].image}'

# 查看升级历史（准备回滚时使用）
helm history my-nginx
```

## 二、镜像回滚操作

### 1. 查看历史版本

```
helm history my-nginx

# 输出示例
REVISION  UPDATED                   STATUS     CHART        APP VERSION DESCRIPTION
1         Mon Aug 1 10:00:00 2022   deployed   nginx-13.0.0 1.21.0     Install complete
2         Mon Aug 1 11:00:00 2022   deployed   nginx-13.0.0 1.22.0     Upgrade complete
3         Mon Aug 1 12:00:00 2022   failed     nginx-13.0.0 1.23.0     Upgrade failed
```

### 2. 回滚到指定版本

```
# 回滚到上一个版本
helm rollback my-nginx 2

# 指定等待时间（默认300秒）
helm rollback my-nginx 2 --wait --timeout 600s

# 强制回滚（即使资源不存在也继续）
helm rollback my-nginx 2 --force
```

### 3. 验证回滚结果

```
# 查看当前使用的镜像
kubectl get deploy my-nginx -o jsonpath='{.spec.template.spec.containers[0].image}'

# 查看回滚后的Pod状态
kubectl get pods -l app.kubernetes.io/instance=my-nginx
```

## 三、生产环境最佳实践

### 1. 升级前检查

```
# 干运行检查（不实际执行）
helm upgrade --dry-run --debug my-nginx bitnami/nginx --set image.tag=1.23.0

# 显示差异
helm diff upgrade my-nginx bitnami/nginx --set image.tag=1.23.0
```

### 2. 安全升级策略

```
# 原子性升级（失败自动回滚）
helm upgrade --atomic my-nginx bitnami/nginx --set image.tag=1.23.0

# 分批次升级（金丝雀发布）
helm upgrade --set "image.tag=1.23.0" \
  --set "updateStrategy.rollingUpdate.maxUnavailable=25%" \
  my-nginx bitnami/nginx
```

### 3. 版本控制建议

```
# 为每次升级添加描述信息
helm upgrade --description "Upgrade to nginx 1.23.0 for CVE-2023-XXXX" \
  my-nginx bitnami/nginx --set image.tag=1.23.0
```

## 四、常见问题解决方案

### 1. 升级后Pod未更新

```
# 检查Deployment是否更新
kubectl describe deployment my-nginx | grep Image

# 强制重新部署（当image未变化但需要重新部署时）
kubectl rollout restart deployment/my-nginx
```

### 2. 回滚失败处理

```
# 查看回滚状态
helm status my-nginx

# 手动删除卡住的资源
kubectl delete pod my-nginx-xxxx --force --grace-period=0

# 使用--force强制回滚
helm rollback my-nginx 2 --force
```

### 3. 多容器镜像管理

对于包含多个容器的应用：

```
# 升级指定容器的镜像
helm upgrade my-app ./my-chart \
  --set containers[0].image.repository=nginx \
  --set containers[0].image.tag=1.23.0 \
  --set containers[1].image.repository=redis \
  --set containers[1].image.tag=6.2
```

## 五、CI/CD 集成示例

```
# GitLab CI 示例
stages:
  - deploy

upgrade-image:
  stage: deploy
  image: alpine/helm
  script:
    - helm upgrade --install --atomic
      --set image.tag=${CI_COMMIT_SHA}
      --namespace ${NAMESPACE}
      my-app ./chart/
  only:
    - main
```

通过以上方法，您可以安全高效地使用 Helm 管理 Kubernetes 中的镜像升级和回滚操作。关键点是：

1. 升级前使用 `--dry-run` 验证
2. 使用 `--atomic` 确保升级原子性
3. 通过 `helm history` 和 `helm rollback` 实现快速回滚
4. 在生产环境中采用分批次升级策略
