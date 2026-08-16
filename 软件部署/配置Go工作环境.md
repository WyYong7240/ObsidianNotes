---
tags:
  - Go
  - GOROOT
  - GOPATH
---

# Ubuntu配置Go工作环境

## Go的安装详见

https://golang.google.cn/doc/install

##  Go环境变量与工作环境配置

1. 创建一个Go的工作目录

   ~~~bash
   mkdir -p ~/Golang/
   cd ~/Golang/
   pwd
   /root/Golang/
   ~~~

2. 配置环境变量

   ~~~bash
   vim ~/.bashrc
   # 尾部写入
   export GOROOT=/usr/local/go
   export GOPATH=/root/Golang/
   export GOPROXY=https://goproxy.cn,direct
   export PATH=$PATH:$GOROOT/bin
   # 保存退出
   ~~~

   配置了`GOPATH`后，GO相关的项目就可以在GOPATH下运行，并且相关GO项目依赖的GO包，默认也会下载到GOPATH下

3. 配置生效

   ~~~bash
   source ~/.bashrc
   ~~~

   

