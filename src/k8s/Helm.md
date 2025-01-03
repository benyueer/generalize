# Helm
使用 yaml 文件部署应用 过于复杂，所以有了 Helm

Helm 是 Kubernetes 的包管理工具，用于 管理 Kubernetes 应用的 部署、升级、回滚、删除

Helm 的 核心概念：
- Chart：Helm 的 包，包含 应用的 所有 资源 和 配置
- Release：Helm 的 部署 实例，一个 Chart 可以 有 多个 Release
- Repository：Helm 的 仓库，用于 存储 Chart


## 安装
官网 下载 压缩包
解压后将 可执行文件放到 /usr/local/bin 目录下

## 配置 helm 仓库
```bash
helm repo add stable https://charts.helm.sh/stable
helm repo update

helm repo add aliyun https://kubernetes.oss-cn-hangzhou.aliyuncs.com/helm
helm repo update

helm repo list

helm repo remove aliyun
```

## 基本使用
3个命令
- helm install：安装 一个 新的 Release
- helm upgrade：升级 一个 已有的 Release
- helm rollback：回滚 一个 已有的 Release

### 使用 helm 部署一个应用
```bash
# 1 搜索应用名称
helm search repo nginx

# 2 根据搜索结果安装
helm install my-nginx nginx

# 3 查看 部署的 应用
helm list

# 4 查看 部署的 应用 详情
helm get manifest my-nginx
helm status my-nginx
```

## 简单自定义 Chart 
1 创建 Chart
```bash
helm create my-chart
# 会生成一个目录 里边包含 chart.yaml 和 values.yaml 文件 和 templates 目录 charts 目录等

```

- chart.yaml：Chart 的 元数据
- values.yaml：Chart 的 全局变量
- templates：Chart 的 模板 文件
  - 编写 yaml 文件 放到 templates 目录下，可自定义层级
- charts：Chart 的 依赖

2 在templates目录下 编写 yaml 文件
```yaml
# deplayment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-nginx
  template:
    metadata:
      labels:
        app: my-nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.25.2
          ports:
            - containerPort: 80


---
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: my-nginx
spec:
  selector:
    app: my-nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: NodePort
```

3 安装 my-chart
```bash
helm install my-chart ./my-chart
```
 
4 查看 部署的 应用
```bash
helm list

kubectl get pods
kubectl get svc
```

5 应用 升级
```bash
helm upgrade my-chart ./my-chart
```

6 删除 部署的 应用
```bash
helm uninstall my-chart
```


## Chart 模板
通过传递参数，动态生成模版，yaml 内容 动态传入参数生成

values.yaml 文件 里边 定义 全局变量

```yaml
# values.yaml
image: nginx:1.25.2
replicas: 1
```

templates 目录下 编写 yaml 文件
```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}
spec:
  replicas: {{ .Values.replicas }}
  selector:
    matchLabels:
      app: {{ .Release.Name }}
  template:
    metadata:
      labels:
        app: {{ .Release.Name }}
    spec:
      containers:
        - name: nginx
          image: {{ .Values.image }}
          ports:
            - containerPort: 80
```

```bash
# 1 查看 模板
helm template my-chart ./my-chart

# 2 安装 模板
helm install --dry-run --debug my-chart ./my-chart

# 3 升级 模板
helm upgrade --dry-run --debug my-chart ./my-chart

# --dry-run 只生成模板，不安装
# --debug 输出 调试信息

# 安装
helm install my-chart ./my-chart

# 升级
helm upgrade my-chart ./my-chart

# 查看 部署的 应用
kubectl get pods
```


## 常用命令
| 命令 | 说明 |示例 |
| --- | --- | --- |
| create | 创建一个 chart 并指定名字 |`helm create my-chart`|
| dependency | 管理 chart 依赖 |  |
| get | 下载一个 release |  |
| history | 获取 release 历史 |  |
| install | 安装一个 chart |  |
| list | 列出release |  |
| package  | 将 chart 目录打包到 chart 存档文档中 |  |
| pull | 从远程仓库下载 chart 并解压 |  |
| repo | chart 仓库操作 |  |
| rollback | 回滚版本 |  |
| search | 搜索 chart |  |
| show | 查看 chart 详情 |  |
| template | 渲染模板 |  |
| test | 测试 chart |  |
| uninstall | 删除 release |  |
| upgrade | 升级 release |  |
| verify | 验证 chart |  |
| status | 查看 release 状态 |  |
| version | 查看 helm 版本 |  |


## 模版语法
在chart中以 “下划线” 开头的文件，称为”子模版”
格式：`{{- define "模版名字" -}}` 模版内容 `{{- end -}}`
```yaml
{{- define "nginx.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" -}}
{{- end -}}

# 若 .Values.nameOverride 为空，则默认值为 .Chart.Name
```

引用模板，格式：`{{ include "模版名字" 作用域}}`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "nginx.fullname" . }}

```


内置对象
Chart 预定义对象可直接在各模板中使用
```yaml
Release：      代表Release对象，属性包含：Release.Name、Release.Namespace、Release.Revision等
Values：       表示 values.yaml 文件数据
Chart：        表示 Chart.yaml 数据
Files：        用于访问 chart 中非标准文件

Capabilities： 用于获取 k8s 集群的一些信息
   - Capabilities.KubeVersion.Major：K8s的主版本

Template：     表示当前被执行的模板
   - Name：表示模板名，如：mychart/templates/mytemplate.yaml
   - BasePath：表示路径，如：mychart/templates

```

变量
默认情况点`( . )`, 代表全局作用域，用于引用全局对象。
helm 全局作用域中有两个重要的全局对象：Values 和 Release
```yaml
# Values
# 这里引用了全局作用域下的Values对象中的key属性。 
{{ .Values.key }}
Values代表的就是values.yaml定义的参数，通过.Values可以引用任意参数。
例子：
{{ .Values.replicaCount }}

# 引用嵌套对象例子，跟引用json嵌套对象类似
{{ .Values.image.repository }}

# Release 
其代表一次应用发布，下面是Release对象包含的属性字段：
Release.Name       - release的名字，一般通过Chart.yaml定义，或者通过helm命令在安装应用的时候指定。
Release.Time       - release安装时间
Release.Namespace  - k8s名字空间
Release.Revision   - release版本号，是一个递增值，每次更新都会加一
Release.IsUpgrade  - true代表，当前release是一次更新.
Release.IsInstall  - true代表，当前release是一次安装
Release.Service:   - The service that is rendering the present template. On Helm, this is always Helm.

```

自定义模版变量
```yaml
# 变量名以$开始命名， 赋值运算符是 := (冒号+等号)
{{- $relname := .Release.Name -}}

引用自定义变量:
#不需要 . 引用
{{ $relname }}

```


include
```yaml
include 是一个函数，所以他的输出结果是可以传给其他函数的

# 例子1：
env:
  {{- include "xiaomage" . }}

# 结果：
          env:
- name: name
  value: xiaomage
- name: age
  value: secret
- name: favourite
  value: "Cloud Native DevSecOps"
- name: wechat
  value: majinghe11

# 例子2：
env:
  {{- include "xiaomage" . | indent 8}}

# 结果：
          env:
            - name: name
              value: xiaomage
            - name: age
              value: secret
            - name: favourite
              value: "Cloud Native DevSecOps"
            - name: wechat
              value: majinghe11

```


with
with 关键字可以控制变量的作用域,主要就是用来修改 . 作用域的，默认 . 代表全局作用域，with 语句可以修改 . 的含义
```yaml
# 例子：
# .Values.favorite 是一个 object 类型
{{- with .Values.favorite }}
drink: {{ .drink | default "tea" | quote }}   # 相当于.Values.favorite.drink
food:  {{ .food  | upper | quote }}
{{- end }}

```


Values 对象
values 对象的值有四个来源
- chart 包中的 values.yaml 文件
- 父 chart 包的 values.yaml 文件
- 使用 helm install 或者 helm upgrade 的 -f 或者 --values 参数传入的自定义的 yaml 文件
- 通过 --set 参数传入的值
```yaml
cat global.yaml 
course: k8s

cat mychart/templates/configmap.yaml 
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  course:  {{ .Values.course }}

helm install --name mychart --dry-run --debug -f global.yaml ./mychart/

```


管道
```yaml
k8s:  {{ quote .Values.course.k8s }} # 加双引号
k8s:  {{ .Values.course.k8s | upper | quote }} # 大写字符串加双引号
k8s:  {{ .Values.course.k8s | repeat 3 | quote }} # 加双引号和重复3次字符串

```


if/else 条件
```yaml
if/else 块是用于在模板中有条件地包含文本块的方法，条件块的基本结构

{{ if PIPELINE }}
   # Do something
{{ else if OTHER PIPELINE }}
   # Do something else
{{ else }}
   # Default case
{{ end }}

# 判断条件，如果值为以下几种情况，管道的结果为 false：
1. 一个布尔类型的假
2. 一个数字零
3. 一个空的字符串
4. 一个 nil(空或null)
5. 一个空的集合(map, slice, tuple, dict, array)
除了上面的这些情况外，其他所有的条件都为真。

# 例子
cat mychart/templates/configmap.yaml 
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: {{ .Values.hello | default "Hello World" | quote }}
  k8s:  {{ .Values.course.k8s | upper | quote | repeat 3 }}
  python:  {{ .Values.course.python | repeat 3 | quote }}
  {{ if eq .Values.course.python "django" }}web: true{{ end }}

helm install --name mychart --dry-run --debug ./mychart/

运行部分结果：
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mychart-configmap
data:
  myvalue: "Hello World"
  k8s:  "KLVCHEN""KLVCHEN""KLVCHEN"
  python:  "djangodjangodjango"
  web: true

# 空格控制
{{- if eq .Values.course.python "django" }}
web: true
{{- end }}

```


range 关键字
```yaml
range主要用于循环遍历数组类型。

语法1:
# 遍历map类型，用于遍历键值对象
# 变量key代表对象的属性名，val代表属性值
{{- range key,val := 键值对象 }}
{{ $key }}: {{ $val | quote }}
{{- end}}

语法2：
{{- range 数组 }}
{{ . | title | quote }} # . (点)，引用数组元素值。
{{- end }}

例子:
# values.yaml定义
# map类型
favorite:
  drink: coffee
  food: pizza
 
# 数组类型
pizzaToppings:
  - mushrooms
  - cheese
  - peppers
  - onions
 
# map类型遍历例子:
{{- range $key, $val := .Values.favorite }}
{{ $key }}: {{ $val | quote }}
{{- end}}
 
# 数组类型遍历例子:
{{- range .Values.pizzaToppings}}
{{ . | quote }}
{{- end}}

```