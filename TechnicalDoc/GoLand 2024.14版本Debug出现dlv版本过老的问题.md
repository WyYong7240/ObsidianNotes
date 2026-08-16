---
tags:
  - Goland
  - Debugger
  - dlv
---

# 问题描述与原因

* 在debug时，console控制台输出形如如下的提示，并且Goland在debug时，变量无法有效查看

  ~~~shell
  warning: undefined behavior - version of Delve is too old for Go version 1.20.1 (maximum supported version 1.19)
  ~~~

* 原因：

  Goland IDE 使用的是Goland下载时自带的delve调试器，自带的调试器版本过低无法调式版本较新的Goland编译出来的程序

# 解决办法

> 1. 等待Jetbrians官方发布更新修复包
> 2. IDE中配置Goland使用的delve可执行文件路径

## 安装新版本Delve

~~~shell
go install github.com/go-delve/delve/cmd/dlv@latest
~~~

安装完成后，一般在`GOPATH/bin`下



## 配置IDE的delve的路径
<img src="https://typora2icture.oss-cn-beijing.aliyuncs.com/img2/202507281510085.png" alt="image.png" style="zoom:80%;" />

编辑路径：

1. Windows

   ~~~properties
   dlv.path=C:\\Users\\25189\\go\\bin\\dlv.exe
   ~~~

2. Linux

   ~~~properties
   dlv.path=$GOPATH/bin/dlv
   ~~~



# 参考文档：

1. https://muwaii.com/posts/upgrade-your-golang-debugger-delve-in-goland
