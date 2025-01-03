# Pod 控制器
按照 Pod 的创建方式，可分为两种：
- 自主式 Pod：直接由用户创建，k8s 不负责管理，删除后就没有了，不会重建
- 控制器管理的 Pod：由控制器创建，k8s 负责管理，删除后会重建

Pod 控制器是管理 pod 的中间层，使用了 pod 控制器后，我们只要告诉控制器我们需要多少个什么样的 pod 就行了，他会根据配置自动管理 pod

k8s 中有很多类型的 pod 控制器，常见的有：
- ReplicationController：用于管理 pod 的副本数，已被废弃，被 ReplicaSet 替代
- ReplicaSet：保证指定数量的 pod 运行，支持 pod 数量变更和镜像版本变更
- Deployment：通过控制 ReplicaSet 来管理 pod，支持滚动更新和回滚
- HorizontalPodAutoscaler：可以根据集群负载自动控制 pod 数量，实现削峰填谷
- DaemonSet：在集群中的指定 node 上都运行一个副本，一般用于守护进程类的服务
- Job：用于管理一次性任务
- CronJob：用于管理定时任务
- StatefulSet：管理有状态应用，可以保证 pod 的持久化存储

## ReplicaSet
ReplicaSet 是 k8s 中用于管理 pod 副本数量的控制器，可以保证指定数量的 pod 运行，支持 pod 数量变更和镜像版本变更
他会监听 pod 状态，一旦 pod 发生故障，就会自动重启或重建

资源清单文件
```yaml
apiVersion: apps/v1  # 版本
kind: ReplicaSet  # 类型
metadata:
  name: nginx-replicaset  # 名称
  namespace: default  # 命名空间
  labels:  # 标签
    app: nginx
spec:
  replicas: 3  # 副本数
  selector:  # 选择器 用来指定该容器用来管理哪些pod
    matchLabels:  # 匹配标签
      app: nginx
    matchExpressions:  # 匹配表达式
      - key: app  # 键
        operator: In  # 操作符
        values: [nginx]  # 值
  template:  # 模板  当副本数量不足时，会使用 template 创建 pod
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.17.1
          ports:
            - containerPort: 80
```

```bash
# 创建 ReplicaSet
kubectl apply -f nginx-replicaset.yaml

# 查看 ReplicaSet
kubectl get replicaset

# 删除 ReplicaSet
kubectl delete replicaset nginx-replicaset
```

```bash
# 扩缩容
# 编辑 副本数量字段即可
# 编辑命令
kubectl edit replicaset nginx-replicaset
# 或者
kubectl scale replicaset nginx-replicaset --replicas=5
```

```bash
# 镜像升级
# 编辑 镜像版本字段即可
kubectl edit replicaset nginx-replicaset
# 或者
kubectl set image replicaset nginx-replicaset nginx=nginx:1.18.0
```


## Deployment
不直接管理 pod，而是通过 ReplicaSet 来管理 pod，支持滚动更新和回滚，比 ReplicaSet 更强大
- 支持 ReplicaSet 的所有功能
- 支持发布的停止、继续
- 支持版本的滚动升级和回滚

资源文件清单：
```yaml
apiVersion: apps/v1  # 版本
kind: Deployment  # 类型
metadata:
  name: nginx-deployment  # 名称
  namespace: default  # 命名空间
  labels:  # 标签
    app: nginx
spec:
  replicas: 3  # 副本数
  revisionHistoryLimit: 10  # 保留的版本数 默认为 10
  paused: false  # 是否暂停部署 默认 false
  progressDeadlineSeconds: 600  # 部署超时时间 默认 600s
  strategy:  # 部署策略
    type: RollingUpdate  # 滚动更新
    rollingUpdate:  # 滚动更新配置
      maxSurge: 25%  # 最大扩容数量 默认为 25% 可为数量，也可为百分比
      maxUnavailable: 25%  # 最大不可用数量 默认为 25%
  selector:  # 选择器
    matchLabels:  # 匹配标签
      app: nginx
    matchExpressions:  # 匹配表达式
      - key: app  # 键
        operator: In  # 操作符
        values: [nginx]  # 值
  template:  # 模板
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.17.1
          ports:
            - containerPort: 80
```

```bash
# 创建 Deployment
kubectl apply -f nginx-deployment.yaml

# 查看 Deployment
kubectl get deployment 

# 删除 Deployment
kubectl delete deployment nginx-deployment
```

```bash
# 扩缩容
# 编辑 副本数量字段即可
# 编辑命令
kubectl edit deployment nginx-deployment
# 或者
kubectl scale deployment nginx-deployment --replicas=5
```

### 镜像更新
Deployment 支持两种镜像更新策略 重建更新 和 滚动更新（默认）
```yaml
spec:
  strategy:
    type: RollingUpdate  # 滚动更新  杀死一部分就启动一部分，在更新过程中存在两个版本的 pod
    type: Recreate  # 重建更新  新的 pod 创建出来前，先删除所有旧的 pod
    rollingUpdate:
      maxSurge: 25%  # 最大扩容数量 默认为 25% 可为数量，也可为百分比
      maxUnavailable: 25%  # 最大不可用数量 默认为 25%
```

#### 重建更新
先设置策略为 Recreate
再创建一个 deployment
然后修改镜像版本
```bash
kubectl set image deployment nginx-deployment nginx=nginx:1.18.0
```
然后观察 pod 变化


### 版本回退
`kubectl rollout` 命令支持以下选项：
- status：查看部署状态
- history：查看部署历史
- undo：回退到上一个版本
- pause：暂停部署
- resume：继续部署
- restart：重启部署

```bash
# 查看部署状态
kubectl rollout status deployment nginx-deployment

# 查看部署历史
kubectl rollout history deployment nginx-deployment

# 版本回滚
kubectl rollout undo deployment nginx-deployment --to-revision=1
```

### 金丝雀发布
在一批 pod 更新后暂停更新，此时只有一部分 pod 是新版本，此时将一部分用户的请求转发到新版本的 pod，如果新版本没有问题，继续更新所有 pod，否则会退

```bash
# 更新 deployment 并配置暂停
kubectl set image deployment nginx-deployment nginx=nginx:1.18.0 && kubectl rollout pause deployment nginx-deployment

# 观察更新状态
kubectl rollout status deployment nginx-deployment

# 继续更新
kubectl rollout resume deployment nginx-deployment

# 回滚
kubectl rollout undo deployment nginx-deployment
```

## HorizontalPodAutoscaler
之前手动执行 kubectl scale 不符合 自动化的要求
HPA 通过监听 pod 的资源使用情况，实现自动调整 pod 数量
HPA 可以获取每个 pod 的利用率，然后和 HPA 中定义的指标进行比较，计算出需要伸缩的具体值，最后实现 pod 数量的调整

### 安装 metrics-server
metrics-server 是 k8s 的资源监控系统，用于收集和存储 pod 的资源使用情况
```bash
yum install git
git clone https://github.com/kubernetes-sigs/metrics-server.git
cd metrics-server/deploy/
vim metrics-server-deployment.yaml

# 添加一下配置
hostNetwork: true
image: registry.aliyuncs.com/google_containers/metrics-server:v0.6.4
args:
  - --kubelet-preferred-address-types=InternalIP,HostName,InternalDNS,ExternalDNS,ExternalIP
  - --kubelet-insecure-tls

# 安装
kubectl apply -f metrics-server-deployment.yaml

# 查看 metrics-server 状态
kubectl get pods -n kube-system

# 查看 资源
kubectl top node
kubectl top pod
```

### 准备 deployment 和 service
```bash
kubectl apply -f nginx-deployment.yaml
kubectl apply -f nginx-service.yaml

kubectl run nginx --image=nginx:1.17.1 --port=80
kubectl expose deployment nginx type=NodePort --port=80 --target-port=80
```

### 部署 HPA
```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: nginx-hpa
spec:
  minReplicas: 1  # 最小副本数
  maxReplicas: 10  # 最大副本数
  targetCPUUtilizationPercentage: 50  # 目标 CPU 使用率
  scaleTargetRef:  # 目标资源
    apiVersion: apps/v1
    kind: Deployment
    name: nginx
```

```bash
kubectl apply -f nginx-hpa.yaml

# 查看 HPA
kubectl get hpa
```


## DaemonSet
DaemonSet 用于在集群中的指定 node 上都运行一个副本，一般用于守护进程类的服务如日志收集、节点监控
- 当集群中增加一个 node 时，DaemonSet 会自动在新的 node 上创建一个 pod
- 当集群中删除一个 node 时，DaemonSet 会自动在新的 node 上删除一个 pod

资源清单文件：
```yaml
apiVersion: apps/v1  # 版本
kind: DaemonSet  # 类型
metadata:
  name: fluentd-elasticsearch  # 名称
  namespace: default  # 命名空间
  labels:  # 标签
    name: fluentd-elasticsearch
spec:
  revisionHistoryLimit: 10  # 保留的版本数 默认为 10
  updateStrategy:  # 更新策略
    type: RollingUpdate  # 滚动更新
    rollingUpdate:  # 滚动更新配置
      maxUnavailable: 1  # 最大不可用数量 默认为 1
  selector:  # 选择器
    matchLabels:  # 匹配标签
      name: fluentd-elasticsearch
    matchExpressions:  # 匹配表达式
      - key: name  # 键
        operator: In  # 操作符
        values: [fluentd-elasticsearch]  # 值
  template:  # 模板
    metadata:
      labels:
        name: fluentd-elasticsearch
    spec:
      containers:
        - name: fluentd-elasticsearch
          image: fluent/fluentd:latest
          ports:
            - containerPort: 24224
```




