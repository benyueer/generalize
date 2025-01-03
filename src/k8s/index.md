# k8s

## 组件
- master：集群的控制平面，负责集群的决策
  - APiServer：集群的入口，接收用户的命令，提供认证授权、API注册和发现等机制
  - Scheduler：负责集群资源调度，按照预定的调度策略将 Pod 调度到合适的Node上
  - ControllerManager：负责维护集群的状态，比如故障检测、自动扩展、滚动更新等
  - Etcd：分布式键值对数据库，用于存储集群的状态，各种资源的信息
- node：集群的数据平面，负责为容器提供运行环境 
  - Kubelet：负责维护容器的生命周期，通过 docker 控制包括容器的创建、启动、停止等
  - KubeProxy：负责为 Pod 提供网络支持，实现服务发现和负载均衡
  - Docker：负责容器的运行

## 概念
- Master：集群的控制平面，负责集群的决策
- Node：集群的数据平面，负责为容器提供运行环境
- Pod：K8s 中最小的可部署单元，一个 Pod 包含一个或多个容器，Pod 是 Kubernetes 中可以被管理和调度的基本单位
- Controller：控制器，通过它来管理 Pod 的运行
- Service：Service 是 Kubernetes 中用于管理网络连接和负载均衡的抽象概念，它定义了一组 Pod 的访问策略，通过 Service 可以实现 Pod 的负载均衡和网络连接的抽象
- Label：标签，用于标识和选择资源，可以附加到各种资源上，用于分类和选择
- NameSpace：命名空间，用于隔离资源，不同的 NameSpace 之间相互隔离，默认情况下，所有资源都属于 default NameSpace 