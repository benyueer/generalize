# frp
frp 是一个开源、简洁易用、高性能的内网穿透和反向代理软件，支持 tcp, udp, http, https等协议。frp 项目官网是 https://github.com/fatedier/frp


## frp工作原理
- 服务端运行，监听一个主端口，等待客户端的连接
- 客户端连接到服务端的主端口，同时告诉服务端要监听的端口和转发类型
- 服务端fork新的进程监听客户端指定的端口
- 外网用户连接到客户端指定的端口，服务端通过和客户端的连接将数据转发到客户端
- 客户端进程再将数据转发到本地服务，从而实现内网对外暴露服务的能力

## 配置
### 服务端配置
下载应用
解压后编辑配置文件

```ini
# frps.ini
[common]
# frp监听的端口，默认是7000，可以改成其他的
bind_port = 7000
# 授权码，请改成更复杂的
token = 52010  # 这个token之后在客户端会用到

# frp管理后台端口，请按自己需求更改
dashboard_port = 7500
# frp管理后台用户名和密码，请改成自己的
dashboard_user = admin
dashboard_pwd = admin
enable_prometheus = true

# frp日志配置
log_file = /var/log/frps.log
log_level = info
log_max_days = 3
```

启动 frp 服务
```sh
sudo mkdir -p /etc/frp
sudo cp frps.ini /etc/frp
sudo cp frps /usr/bin
sudo cp systemd/frps.service /usr/lib/systemd/system/
sudo systemctl enable frps
sudo systemctl start frps
```

防火墙开放端口
```sh
# 添加监听端口
sudo firewall-cmd --permanent --add-port=7000/tcp
# 添加管理后台端口
sudo firewall-cmd --permanent --add-port=7500/tcp
sudo firewall-cmd --reload
```

验证服务端是否启动成功
访问 http://server_ip:管理端口 查看服务状态

### 客户端配置
下载应用
解压后编辑配置文件

```ini
frpc.ini
# 客户端配置
[common]
server_addr = 服务器ip
server_port = 7000 # 与frps.ini的bind_port一致
token = 52010  # 与frps.ini的token一致

# 配置ssh服务
[ssh]
type = tcp
local_ip = 127.0.0.1
local_port = 22
remote_port = 6000  # 这个自定义，之后再ssh连接的时候要用

# 配置http服务，可用于小程序开发、远程调试等，如果没有可以不写下面的
[web]
type = http
local_ip = 127.0.0.1
local_port = 8080
subdomain = test.hijk.pw  # web域名
remote_port = 自定义的远程服务器端口，例如8080
```


开放防火墙


启动客户端
```sh
./frpc -c frpc.ini
```