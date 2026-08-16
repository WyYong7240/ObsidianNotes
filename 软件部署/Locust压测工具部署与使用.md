---
tags:
  - locust
  - 压测工具
---

# 前期准备与locust安装

> 由于Locust是基于python的，需要安装python环境
>
> 而安装python环境，可以基于conda创建虚拟环境，也可以不使用conda，直接使用python
>
> 但是由于本人更倾向于使用Anaconda管理虚拟环境，因此建议先安装Anaconda
>
> 本文档使用locust环境：Ubuntu 24.04.02 Server

* Anaconda安装: [[一些软件的部署#Anaconda部署]]
  
* Anaconda创建一个Locust的虚拟环境

  ~~~bash
  conda create -n locust-env python=3.12
  ~~~

* 激活`locust-env`虚拟环境

  ~~~bash
  conda activate locust-env
  ~~~

* 安装python3-pip

  ~~~bash
  apt-get install python3-pip -y
  ~~~

* 安装locust包

  ~~~bash
  pip install locust
  ~~~

# Locust使用

> 在进入上述创建好的的虚拟环境，并且安装了locust包之后，我们开始使用locust

1. 创建如下文件`locust-test.py`

   ~~~py
   from locust import HttpUser, task, between
   
   class ServiceUser(HttpUser):
       wait_time = between(1, 3)   # 每次任务的等待时间，随机在1，3秒之间
       @task
       def load_product_page(self):
           # 向被压测界面发送GET请求
           self.client.get("/")
   ~~~

2. 创建如下启动脚本

   ~~~sh
   #!/bin/bash
   
   # 查找占用 8089 端口的进程 PID
   PID=$(netstat -tulnp | grep :8089 | awk '{print $7}' | cut -d'/' -f1)
   
   # 检查是否找到了进程
   if [ -n "$PID" ]; then
       echo "Killing process on port 8089 with PID: $PID"
       kill $PID
       echo "Process killed."
   else
       echo "No process found on port 8089."
   fi
   
   # 启动 locust
   echo "Starting locust..."
   export TZ="Asia/Shanghai"
   nohup locust -f locust-test.py --host=http://192.168.3.226:32241 &
   echo "Locust started."
   
   ~~~

   1. 由于locust的web服务默认占用的是8089端口，因此先检查宿主机8089端口有没有已经在占用的进程，并将其杀死
   2. `nohup`是要让该locust压测服务在后台运行，防止shell关闭后，压测服务也跟随关闭
   3. 也可以使用命令`locust -f locust-test.py`直接启动locust
   4. `--host:http://192.168.3.226:32241`是指定locsut压测的默认地址，也就是被压测的服务的地址

3. 启动后，访问`http://localhost:8089`
    ![image.png](https://typora2icture.oss-cn-beijing.aliyuncs.com/img2/20250612111407.png)

  即可以开始压测



# 参考文档

1. https://blog.csdn.net/NHB234567/article/details/139422460
2. https://locust.io/#install

