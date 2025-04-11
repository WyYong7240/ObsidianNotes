> 版本：0.61.2
> 以代理内网服务为例，以本地程序方式运行
> 本例以代理内网的Prometheus监控服务为例，将内网的Prometheus监控服务，能够在公网部署的Grafana中访问到并监控数据

# FRPS部署及配置
## 在具有公网IP的服务器上安装frps、frps.toml

~~~ bash
wget https://github.com/fatedier/frp/releases/download/v0.61.2/frp_0.61.2_linux_amd64.tar.gz

tar -zxf frp_0.61.2_linux_amd64.tar.gz -C /usr/local
cd /usr/local
mv frp_0.61.2_linux_amd64 frp
~~~

## 在服务器上配置frps为系统服务

~~~ bash
vim /usr/lib/systemd/system/frps.service
# 输入以下内容
# 服务名称，可自定义
Description = frp server for edgePrometheuse
After = network.target syslog.target
Wants = network.target

[Service]
Type = simple
# 启动frps的命令，需修改为您的frps的安装路径
ExecStart = /usr/local/frp/frps -c /usr/local/frp/frps.toml

[Install]
WantedBy = multi-user.target
~~~

## 配置frps

~~~ bash
vim /usr/local/frp/frps.toml

# 配置frps和frpc连接的端口，默认是7000
bindPort = 7000

~~~ 

## 启动frps

~~~ bash
systemctl enable frps
systemctl start frps
systemctl status frps
~~~

# FRPC的部署和配置

## 在需要代理服务的服务器上安装frpc、frpc.toml

~~~ bash
wget https://github.com/fatedier/frp/releases/download/v0.61.2/frp_0.61.2_linux_amd64.tar.gz

tar -zxf frp_0.61.2_linux_amd64.tar.gz -C /usr/local
cd /usr/local
mv frp_0.61.2_linux_amd64 frp
~~~

## 在服务器上配置frpc为系统服务

~~~ bash
vim /usr/lib/systemd/system/frps.service
# 输入以下内容
# 服务名称，可自定义
Description = frp server for edgePrometheuse
After = network.target syslog.target
Wants = network.target

[Service]
Type = simple
# 启动frps的命令，需修改为您的frps的安装路径
ExecStart = /usr/local/frp/frpc -c /usr/local/frp/frpc.toml

[Install]
WantedBy = multi-user.target
~~~

## 配置frps

~~~ bash
vim /usr/local/frp/frps.toml

# 配置内容如下
serverAddr = "39.98.76.224"  # frps服务器的IP
serverPort = 7000  # frps与frpc的连接端口

# 这是以域名方式访问，详见frp官方文档中的例子配置
# https://gofrp.org/zh-cn/docs/examples/vhost-http/
#[[proxies]]
#name = "prometheuse-metrics"
#type = "http"
#localPort = 9090
#customDomains = ["www.cloudK8sEdge1.com"]

[[proxies]]
name = "prometheuse-metrics" # 服务名称
type = "tcp"  # 使用的协议类型
localIP = "127.0.0.1" # 要代理的服务的IP
localPort = 9090 # 要代理的服务的端口
remotePort = 28080 # 远程frps访问该服务的端口号


#[[proxies]]
#name = "prometheuse-metrics"
#type = "http"
#localIP = "127.0.0.1"
#localPort = 22
#remotePort = 6000
~~~ 
最终，访问该服务的方式就是http://39.98.76.224:28080
记得将frps所在公网服务器的对应端口开放，比如7000和28080
## 启动frps

~~~ bash
systemctl enable frps
systemctl start frps
systemctl status frps
~~~

