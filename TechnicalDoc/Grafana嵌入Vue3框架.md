# 安装Grafana

> 由于将Grafana的监控面板嵌入Vue框架需要更改Grafana的配置文件`/etc/grafana.ini`
>
> kube-prometheus自带的grafana并没有找到该配置文件，所以选用了本地部署的Grafana

Ubuntu版本：24.04

1. 获取最新版key

   ~~~bash
   wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
   ~~~

2. 添加最新稳定版仓库

   ~~~bash
   echo "deb https://packages.grafana.com/enterprise/deb stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list
   ~~~

3. 安装grafana

   ~~~bash
   sudo apt-get update
   sudo apt-get install grafana -y
   ~~~

# 配置Grafana

1. 为Grafana配置数据源、登录账号等等在此不再详述

2. 配置`grafana.ini`

   * 配置可以匿名访问

     ~~~ini
     #################################### Anonymous Auth ######################
     [auth.anonymous]
     # enable anonymous access
     enabled = true
     
     # specify organization name that should be used for unauthenticated users
     org_name = Main Org.
     
     # specify role for unauthenticated users
     org_role = Viewer
     ~~~

   * 配置允许面板嵌入

     ~~~ini
     # set to true if you want to allow browsers to render Grafana in a <frame>, <iframe>, <embed> or <object>. default is false.
     allow_embedding = true
     ~~~

   * 更改初始服务配置

     ~~~ini
     [server]
     domain = your_server_ip  # 改为Grafana主机的真实IP或域名
     root_url = http://your_server_ip:3000/  # 端口需与实际一致
     ~~~

3. 重启grafana

   ~~~BASH
   systemctl restart grafana-server.service
   ~~~

# 将grafana嵌入Vue

> 方式有两种
>
> 1. 将整个grafana嵌入vue，使用vue中的\<iframe\>标签，使用的是上述的匿名访问，如果不配置，会陷入登录循环
> 2. 将grafana已经创建好的数据面板嵌入vue，但是好像只能嵌入小面板，大面板不知道为什么还是会出现`Invalid Panel ID`的错误

## 将整个Grafana嵌入Vue

* 如下是vue的一个component

  ~~~vue
  <script setup>
  const grafanaUrl = "http://192.168.3.226:3000"
  </script>
  
  <template>
    <iframe :src="grafanaUrl"
            width="100%"
            height="100%"
            frameborder="0"></iframe>
  </template>
  
  <style scoped>
  
  </style>
  ~~~

  效果如图
  ![image.png](https://typora2icture.oss-cn-beijing.aliyuncs.com/img2/202506271809237.png)

  可以发现，右上角显示的是`sign in`即未登录状态，说明是以匿名访问的方式进行嵌入的

## 将小面板进行嵌入

* 进入grafana已经创建好的监控面板，选中需要嵌入的小面板
  ![](https://typora2icture.oss-cn-beijing.aliyuncs.com/img2/202506271811038.png)

* 点开后获取嵌入的\<iframe\>标签
![image.png](https://typora2icture.oss-cn-beijing.aliyuncs.com/img2/202506271813425.png)

* 将获取的标签放入vue的component组件中

  ~~~vue
  <script setup>
      
  </script>
  
  <template>
    <iframe src="http://192.168.3.226:3000/d-solo/rYdddlPWk/node-exporter-full?orgId=1&from=1750911134726&to=1750934582513&timezone=browser&var-DS_PROMETHEUS=feq0hzd7uy0aof&var-job=node-exporter&var-nodename=master&var-node=master&var-diskdevices=%5Ba-z%5D%2B%7Cnvme%5B0-9%5D%2Bn%5B0-9%5D%2B%7Cmmcblk%5B0-9%5D%2B&refresh=1m&panelId=74&__feature.dashboardSceneSolo" width="450" height="200" frameborder="0"></iframe>
  </template>
  
  <style scoped>
  
  </style>
  ~~~
  
  效果如图：
   ![image.png](https://typora2icture.oss-cn-beijing.aliyuncs.com/img2/202506271815617.png)



## 将整个面板嵌入的情况（未成功）

* 进入要嵌入的大面板，并选中分享![5c082fff70881e83b21f14c726aa45bd.png](https://typora2icture.oss-cn-beijing.aliyuncs.com/img2/202506271817600.png)![image.png](https://typora2icture.oss-cn-beijing.aliyuncs.com/img2/202506271817362.png)

* 将连接放入\<iframe\>的src中

  ~~~vue
  <script setup>
  const grafanaUrl = "http://192.168.3.226:3000/public-dashboards/412edbbf441b420787a0e58403dccecd"
  </script>
  
  <template>
    <iframe :src="grafanaUrl"
            width="100%"
            height="100%"
            frameborder="0"></iframe>
  </template>
  
  <style scoped>
  
  </style>
  ~~~
  
  效果如图
  ![image.png](https://typora2icture.oss-cn-beijing.aliyuncs.com/img2/202506271821212.png)
  
  出现了`Invalid Panel ID`的错误
