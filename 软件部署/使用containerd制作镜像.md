---
tags:
  - containerd
  - buildkit
  - 镜像制作
---
# 工具下载
- containerd的安装就不介绍了，可以参考文章[[搭建Kubernetes集群ECS]]中的处于Ubuntu环境下的containerd部署
- buildkit
	buildkit是Docker公司开源的下一代镜像构建工具，支持OCI标准的镜像构建，项目地址：https://github.com/moby/buildkit
## BuildKit安装
1. buildkit包下载与安装
	``` bash
	wget https://github.com/moby/buildkit/releases/download/v0.10.3/buildkit-v0.10.4.linux-amd64.tar.gz
	tar -zxf buildkit-v0.10.4.linux-amd64.tar.gz
	mv /buildkit/bin/* /usr/bin/
	```

2. 设置BuildKit为系统服务
	``` bash
	cat /lib/systemd/system/buildkit.socket
	#######################################
	[Unit]
	Description=BuildKit
	Documentation=https://github.com/moby/buildkit
	
	[Socket]
	ListenStream=%t/buildkit/buildkitd.sock
	SocketMode=0660
	
	[Install]
	WantedBy=sockets.target
	
	
	cat /lib/systemd/system/buildkit.service
	#########################################
	[Unit]
	Description=BuildKit
	Requires=buildkit.socket
	After=buildkit.socket
	Documentation=https://github.com/moby/buildkit
	
	[Service]
	ExecStart=/usr/local/bin/buildkitd --oci-worker=false --containerd-worker=true
	
	[Install]
	WantedBy=multi-user.target
	```

3. 启动服务
	``` bash
	systemctl enable buildkit.service
	systemctl start buildkit.service
	```