# 集群中获取客户端IP的问题
我们的服务有个需求：记录用户访问的IP，对比每次访问的IP是否一致，不一致则推出登录

目前使用的方式是：
通过请求头 X-Forwarded-For 获取客户端IP
或通过 ctx.request.ip 获取客户端IP

Koa 的 ctx.request.ip 的来源取决于几个因素，主要取决于你的应用是否运行在反向代理（例如 Nginx 或 Apache）之后，以及你的 Koa 应用是否设置了 app.proxy = true。

1. 没有反向代理，app.proxy = false (默认):

在这种情况下，ctx.request.ip 直接来自 Node.js 的 req.socket.remoteAddress 属性。 这个属性表示客户端的 IP 地址。

2. 有反向代理，app.proxy = false:

如果你的 Koa 应用运行在反向代理之后，并且 app.proxy 为 false，那么 ctx.request.ip 将会是反向代理服务器的 IP 地址，而不是实际客户端的 IP 地址。 这是因为请求到达你的 Koa 应用之前，已经经过了反向代理服务器。

3. 有反向代理，app.proxy = true:

当 app.proxy = true 时，Koa 会尝试从请求头中获取客户端的真实 IP 地址。 它会检查以下几个请求头，并按照优先级顺序依次尝试：

X-Forwarded-For：这是最常用的请求头，由反向代理服务器添加，包含客户端的 IP 地址。 如果这个头包含多个 IP 地址（例如，多个代理服务器），Koa 通常会取第一个 IP 地址作为客户端的 IP 地址。
X-Real-IP：一些反向代理服务器使用这个头来表示客户端的真实 IP 地址。
其他自定义的请求头：如果你的反向代理服务器使用了其他的自定义请求头来传递客户端 IP 地址，你可能需要在 Koa 中进行相应的配置。
如果以上请求头都不存在，Koa 会回退到 req.socket.remoteAddress，也就是反向代理服务器的 IP 地址。

req.socket.remoteAddress 的值来源于底层网络连接的远程地址。 更具体地说，它是 Node.js HTTP 服务器在接受客户端连接时，从操作系统获取的客户端 IP 地址。

这个 IP 地址是由操作系统在建立 TCP 连接时提供的。 当客户端向服务器发起连接请求时，操作系统会负责处理连接的建立，并记录下客户端的 IP 地址。 Node.js 的 HTTP 模块在接受连接后，会从操作系统获取这个 IP 地址，并将其存储在 req.socket.remoteAddress 属性中。

需要注意的是，req.socket.remoteAddress 提供的 IP 地址可能并非客户端的真实 IP 地址，尤其是在以下情况下：

使用反向代理: 如果你的服务器位于反向代理（如 Nginx 或 Apache）之后，req.socket.remoteAddress 将返回反向代理服务器的 IP 地址，而不是客户端的真实 IP 地址。
使用负载均衡器: 类似地，如果你的服务器位于负载均衡器之后，req.socket.remoteAddress 将返回负载均衡器的 IP 地址。
NAT 网络: 在使用 NAT 网络的情况下，req.socket.remoteAddress 可能返回 NAT 网关的 IP 地址，而不是客户端的真实 IP 地址。

目前我们的 koa 没有配置 proxy 字段


问题：
我们的 bff 服务启动了两个 pod，策略是两个 pod 分别在两个 node 上，这两个 pod 的 网段不同
同时在集群内配置了 ingress 来代理 bff 服务，也配置了 nodePort

那么在用 ip 访问时，两个 pod 获得的是 网关地址，而不是客户端的真实地址，网关地址不同，导致无法对比 ip 是否一致（总是不一致）
在用 域名 访问时，两个 pod 获得的是 反向代理的地址，这个地址是相同的，所以可以对比 ip 是否一致（总是一致）



