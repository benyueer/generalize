# Service
pod 的 IP 地址是会变的，所以需要一个稳定的访问地址，这就是 Service 的作用
service 会对提供同一个服务的 pod 进行聚合，进而提供一个统一的访问入口

service 只是一个概念，起作用的实际上是 `kube-proxy` 组件，每个 node  上都运行着一个 `kube-proxy`服务进程，当创建 service 的时候会通过 api-server 向 etcd 中写入 service 的信息，`kube-proxy` 会监听 etcd 中 service 的变化，然后根据 service 的信息创建 iptable 规则，将 service 的访问地址映射到后端的 pod 上

kube-proxy 支持三种工作模式：
- userspace：通过 iptables 规则将 service 的访问地址映射到后端的 pod 上
  - kube-proxy 会为每一个 service 创建一个监听端口，发向 ClusterIp 的请求会通过 iptables 规则转发到监听端口上，然后通过监听端口转发到后端的 pod 上
  - 该模式下，kube-proxy 充当一个 四层均衡器的角色
  - 由于运行在 userspace 中，会增加内核和用户空间之间的数据拷贝，性能较差
- iptables
  - kube-proxy 会为 service 后端的每一个 pod 创建一个 iptables 规则，将发向 ClusterIp 的请求转发到后端的 pod 上
  - 该模式下，kube-proxy 不充当均衡器的角色，只负责创建 iptables 规则
  - 性能较好，但LB灵活性较差，当后端 pod 不可用时也无法重试
- ipvs
  - kube-proxy 监听 pod 的变化来配置相应的 ipvs 规则
  - ipvs 相对 iptables 性能更好，支持更多的负载均衡算法


配置 ipvs
此模式必须开启 ipvs 内核模块，否侧降级为 iptables 模式
```bash
# 开启 ipvs

kubectl edit cm kube-proxy -n kube-system # mode 改为 ipvs

kubectl delete pod -l k8s-app=kube-proxy -n kube-system  # 删除 kube-proxy 的 pod 重新生成
ipvsadm -Ln

```

## service 类型
资源清单文件：
```yaml
apiVersion: v1
kind: Service # 类型
metadata:
  name: nginx-service # 名称
  namespace: default # 命名空间
spec:
  selector: # 选择器 代理那些 pod
    app: nginx # 标签
  ports: # 端口
    - protocol: TCP # 协议
      port: 80 # 服务端口
      targetPort: 80 # 后端 pod 端口
      nodePort: 30080 # 节点端口
  sessionAffinity: ClientIP # 会话亲和性 
  clusterIP: 10.96.0.1 # service 的虚拟 ip
  # ----
  type: NodePort # 类型
  nodePort: 30080 # 节点端口
  loadBalancerIP: 192.168.1.100 # 负载均衡 ip
  externalIPs: # 外部 ip
    - 192.168.1.100 # 外部 ip
  externalName: # 外部名称
    - nginx.example.com # 外部名称
```

type 有四个选项：
- ClusterIP：默认类型，为 service 提供一个虚拟 ip，只能在集群内部访问
- NodePort：为 service 提供一个虚拟 ip，并映射到 node 的端口上，可以通过 node 的端口访问 service
- LoadBalancer：为 service 提供一个虚拟 ip，并映射到外部负载均衡器上，可以通过负载均衡器访问 service
- ExternalName：为 service 提供一个虚拟 ip，并映射到外部名称上，可以通过外部名称访问 service，即引入外部服务到集群内


## service 使用
先使用 Deployment 创建3个 pod
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.25.1
          ports:
            - containerPort: 80
```

```bash
# 创建 deployment
kubectl apply -f nginx-deployment.yaml

# 查看 pod
kubectl get pod -l app=nginx

# 修改 3个 pod  的 nginx 页面
kubectl exec -it nginx-deployment-6d4b494d4d-64444 -- /bin/bash
echo "hello" > /usr/share/nginx/html/index.html

# 测试访问
curl http://10.96.0.1
```

### ClusterIP service
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  type: ClusterIP
  clusterIP: 10.96.0.1
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

```bash
# 创建 service
kubectl apply -f nginx-service.yaml

# 查看 service
kubectl get svc nginx-service
kubectl describe svc nginx-service

# 查看 ipvs
ipvsadm -Ln
# 测试访问
curl http://10.96.0.1
```

> Endpoint
> Endpoint 是 service 的实际后端 pod 的地址，可以通过 `kubectl get endpoints` 查看
> 存储在 etcd 中，当 pod 发生变化时，会自动更新
> 一个 service 由一组 pod 组成，这些 pod 通过 endpoints 列表暴露给外部
> service 和 pod 联系是通过 endpoint 实现的


### Headless Service
不想使用 默认的 负载均衡
Headless Service 不分配 ClusterIP，如果想要访问 service 需要 service 的域名查询

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  type: ClusterIP
  clusterIP: None  # 不分配 ClusterIP
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```
```bash
# 创建 service
kubectl apply -f nginx-service.yaml

# 查看 service
kubectl get svc nginx-service
kubectl describe svc nginx-service

# 测试访问
curl nginx-service.default.svc.cluster.local
```

### NodePort service
将 service 的端口映射到 node 的端口上，可以通过 node 的端口访问 service
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  type: NodePort
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
      nodePort: 30080  # 映射到 node 的端口
```

```bash
# 创建 service
kubectl apply -f nginx-service.yaml

# 查看 service
kubectl get svc nginx-service
kubectl describe svc nginx-service

# 测试访问
curl http://192.168.1.100:30080
```

### LoadBalancer service
为 service 提供一个虚拟 ip，并映射到外部负载均衡器上，可以通过负载均衡器访问 service
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  type: LoadBalancer
```

# Ingress
service 对外暴露服务 一般是通过 NodePort 和 LoadBalancer 来实现，这两种方式都有缺陷：
- NodePort 会占用 node 的端口
- LB 需要外部负载均衡器，成本较高

Ingress 只需要一个 NodePort 或 LoadBalancer 就可以实现对外暴露多个 service
Ingress 相当于 7层负载均衡器，反向代理，工作原理类似于 nginx
在 ingress 里建立多个映射规则，ingress controller 通过监听这些规则并转化为 nginx 配置，然后对外提供服务
- ingress k8s 对 反向代理 的抽象，定义映射规则
- ingress controller 具体实现反向代理和负载均衡的程序，对 ingress 规则进行实现，具体实现方式有很多，如 nginx contour 等


## ingress 使用
环境准备
安装 ingress
```bash
mkdir ingress-controller
cd ingress-controller

# 获取 ingress-nginx
wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.0/deploy/mandatory.yaml
wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.0/deploy/static/provider/baremetal/service-nodeport.yaml

# 修改 mandatory.yaml 中的仓库

# 创建 ingress-nginx
kubectl apply -f mandatory.yaml
kubectl apply -f service-nodeport.yaml

# 查看 ingress-nginx
kubectl get pod -n ingress-nginx

# 查看 ingress-nginx 的 service
kubectl get svc -n ingress-nginx
```

准备 service 和 pod
```yaml
apiVersion: v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: dev
spec:
  replicas: 3
  selector:
    app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.25.1
          ports:
            - containerPort: 80

----

apiVersion: v1
kind: Deployment
metadata:
  name: tomcat-deployment
  namespace: dev
spec:
  replicas: 3
  selector:
    app: tomcat
  template:
    metadata:
      labels:
        app: tomcat
    spec:
      containers:
        - name: tomcat
          image: tomcat:9.0.82-jdk17-openjdk-slim
          ports:
            - containerPort: 8080
----


apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: dev
spec:
  selector:
    app: nginx
  type: ClusterIP
  clusterIP: None
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80

---

apiVersion: v1
kind: Service
metadata:
  name: tomcat-service
  namespace: dev
spec:
  selector:
    app: tomcat
  type: ClusterIP
  clusterIP: None
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 8080
```

```bash
# 创建 service 和 pod
kubectl apply -f nginx-service.yaml
kubectl apply -f nginx-deployment.yaml
kubectl apply -f tomcat-service.yaml
kubectl apply -f tomcat-deployment.yaml
```

### http 代理
```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ingress-http
  namespace: dev
spec:
  rules:
    - host: nginx.com
      http:
        paths:
          - path: /
            backend:
              serviceName: nginx-service
              servicePort: 80
    - host: tomcat.com
      http:
        paths:
          - path: /
            backend:
              serviceName: tomcat-service
              servicePort: 8080
```

```bash
# 创建 ingress
kubectl apply -f ingress-http.yaml

# 查看 ingress
kubectl get ingress -n dev
kubectl describe ingress ingress-http -n dev
```

### https 代理
```bash
# 创建证书
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/CN=nginx.com"

# 创建 secret
kubectl create secret tls tls-secret --cert=tls.crt --key=tls.key -n dev
```

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ingress-https
  namespace: dev
spec:
  tls:
    - hosts:
        - nginx.com
        - tomcat.com
      secretName: tls-secret
  rules:
    - host: nginx.com
      http:
        paths:
          - path: /
            backend:
              serviceName: nginx-service
              servicePort: 80
    - host: tomcat.com
      http:
        paths:
          - path: /
            backend:
              serviceName: tomcat-service
              servicePort: 8080
```

```bash
# 创建 ingress
kubectl apply -f ingress-https.yaml

# 查看 ingress
kubectl get ingress -n dev
kubectl describe ingress ingress-https -n dev
```