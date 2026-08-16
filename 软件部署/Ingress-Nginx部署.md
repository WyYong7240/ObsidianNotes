---
tags:
  - ingress-nginx
  - ingress
---

# Ingress-Nginx部署

> Kubernetes：v1.22.6
>
> Ingress-nginx:v1.4.0

* 根据Ingress-Nginx官网的版本适应性，下载合适自己Kubernetes版本的Ingress-Nginx controller包

  ~~~ bash
  wget https://github.com/kubernetes/ingress-nginx/archive/refs/tags/controller-v1.4.0.tar.gz
  ~~~

* 解压缩，并部署Ingress-nginx的yaml

  ~~~ bash
  tar -zxf ingress-nginx-controller-v1.4.0.tar.gz
  cd ingress-nginx-controller-v1.4.0/deploy/static/provider/baremetal/
  kubectl apply -f deploy.yaml
  ~~~

  此处注意镜像，好像1.4.0版本的镜像已经从官方网站拉取不到了

  我用的是

  ~~~bash
  docker pull dyrnq/controller:v1.4.0
  docker pull dyrnq/kube-webhook-certgen:v20220916-gd32f8c343
  ~~~

  然后改名为对应的镜像

  > 本次是在自己本地的集群中部署的，因此进入的是baremetal目录部署的deploy.yaml，如果是使用的云服务器ECS部署，应该是要进入cloud目录部署deploy.yaml
  >
  > 至于这几种部署方式的区别，暂且没有了解过-2025.4.17

* 查看运行情况

  ~~~bash
  (base) [root@master software]# kubectl get pod -n ingress-nginx -o wide
  NAME                                        READY   STATUS      RESTARTS   AGE    IP             NODE     NOMINATED NODE   READINESS GATES
  ingress-nginx-admission-create--1-4rknn     0/1     Completed   0          135m   10.244.1.253   node1    <none>           <none>
  ingress-nginx-admission-patch--1-lbwzv      0/1     Completed   0          135m   10.244.2.243   node2    <none>           <none>
  ingress-nginx-controller-5444f8b96b-drrlb   1/1     Running     0          135m   10.244.0.151   master   <none>           <none>
  
  (base) [root@master software]# kubectl get svc -n ingress-nginx
  NAME                                    TYPE           CLUSTER-IP     EXTERNAL-IP                                                         PORT(S)                      AGE
  ingress-nginx-controller                NodePort       10.105.25.16   <none>                                                              80:31921/TCP,443:31301/TCP   136m
  ingress-nginx-controller-admission      ClusterIP      10.96.34.152   <none>                                                              443/TCP                      136m
  ~~~

  此时，如果访问`http://<集群节点IP>:31921`或者`http://<集群节点IP>:31301`,会出现
  <img src="https://typora2icture.oss-cn-beijing.aliyuncs.com/img2/20250417173152.png" alt="image.png" style="zoom:50%;" />

  那就部署成功了

  > 31921和31301两个端口的区别在于，一个是http、一个是https

# Ingress 测试案例1：普通情况

* 测试yaml：Deployment和Service:(cafe.yaml)

  ~~~yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: coffee
  spec:
    replicas: 1
    selector:
      matchLabels:
        app: coffee
    template:
      metadata:
        labels:
          app: coffee
      spec:
        containers:
        - name: coffee
          image: nginxdemos/hello:plain-text
          ports:
          - containerPort: 80
  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: coffee-svc
  spec:
    type: NodePort
    ports:
    - port: 80
      targetPort: 80
      protocol: TCP
      name: http
    selector:
      app: coffee
  ---
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: tea
  spec:
    replicas: 1
    selector:
      matchLabels:
        app: tea
    template:
      metadata:
        labels:
          app: tea
      spec:
        containers:
        - name: tea
          image: nginxdemos/hello:plain-text
          ports:
          - containerPort: 80
  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: tea-svc
    labels:
  spec:
    type: NodePort
    ports:
    - port: 80
      targetPort: 80
      protocol: TCP
      name: http
    selector:
      app: tea
  ~~~

* 测试yaml：Ingress:(cafe-ingress.yaml)

  ~~~yaml
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    name: cafe-ingress
    annotations:
      kubernetes.io/ingress.class: "nginx"  # 明确指定 Ingress Class
  spec:
    rules:
    - host: cafe.example.com
      http:
        paths:
        - path: /tea
          pathType: Prefix
          backend:
            service:
              name: tea-svc
              port:
                number: 80
        - path: /coffee
          pathType: Prefix
          backend:
            service:
              name: coffee-svc
              port:
                number: 80
  ~~~

  > 由于版本不同，在2025年的1.12.1版本的ingress-nginx、k8s版本为1.28.15中，样例ingress定义为
  >
  > ~~~yaml
  > # ingress.yaml
  > apiVersion: networking.k8s.io/v1
  > kind: Ingress
  > metadata:
  >   name: my-ingress
  >   annotations:
  >     nginx.ingress.kubernetes.io/enable-access-log: "true"
  > spec:
  >   ingressClassName: nginx
  >   rules:
  >   - host: my-service.example.com
  >     http:
  >       paths:
  >       - path: /
  >         pathType: Prefix
  >         backend:
  >           service:
  >             name: my-service
  >             port:
  >               number: 80
  > ~~~
  >
  > 可见，没了` kubernetes.io/ingress.class: "nginx"`这样的注解
  >
  > 在k8s v1.18+中，推荐使用`spec.ingressClassName`字段来指定Ingress类，而不是使用注解

* 服务部署成功：

  ~~~bash
  (base) [root@master cafeExample]# kubectl get svc
  NAME            TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
  coffee-svc      NodePort    10.97.203.59     <none>        80:32753/TCP   133m
  tea-svc         NodePort    10.103.142.115   <none>        80:30542/TCP   133m
  ~~~

* Ingress部署成功

  ~~~bash
  (base) [root@master cafeExample]# kubectl get ingress
  NAME                 CLASS    HOSTS              ADDRESS          PORTS   AGE
  cafe-ingress         <none>   cafe.example.com   192.168.23.100   80      75m
  ~~~

  > 注意到，不论是service还是ingress都是在default命名空间的
  >
  > 注意，注解`kubernetes.io/ingress.class: "nginx" `是一定要加的
  >
  > **该注解指定由哪个 Ingress Controller 来处理该 Ingress 资源**：
  >
  > - 当集群中存在多个 Ingress Controller 时（如：nginx、traefik、istio 等）
  > - 通过此注解告诉 Kubernetes 应该由哪个 Controller 来接管这个 Ingress 配置
  > - 没有此注解或注解值不匹配的 Ingress 会被对应的 Controller 忽略

* 在主机中hosts设置DNS解析并访问

  ~~~bash
  192.168.23.100 cafe.example.com
  ~~~

  访问：`http://cafe.example.com:31921/tea`和`http://cafe.example.com:31921/coffee`、
  
  <img src="https://typora2icture.oss-cn-beijing.aliyuncs.com/img2/20250417201005.png" alt="image.png" style="zoom: 67%;" />
  
  <img src="https://typora2icture.oss-cn-beijing.aliyuncs.com/img2/20250417201813.png" alt="image.png" style="zoom: 80%;" />
  
  <img src="https://typora2icture.oss-cn-beijing.aliyuncs.com/img2/20250417201310.png" alt="image.png" style="zoom:80%;" />
  
  > 这里采用了两种方式访问
  >
  > 但是不知道为什么，第一种方式访问tea时，就会出现无法访问的情况，测试发现，可能是浏览器的问题
  >
  > 但是第二种方式访问成功说明没有问题
  
  

# Ingress 测试案例2：跨命名空间转发

> 关于跨命名空间转发，查到了一种方法，但是经过测试，该方法不可行，但是网上很多方法都是这样，先记录一下

* 这里的Service以上面的测试案例的Service为例

  ~~~yaml
  apiVersion: v1
  kind: Service
  metadata:
    name: tea-svc
    labels:
  spec:
    type: NodePort
    ports:
    - port: 80
      targetPort: 80
      protocol: TCP
      name: http
    selector:
      app: tea
  ~~~

* 这里的Service是在default命名空间的，而我们创建的Ingress要在Ingress nginx命名空间，所以先为default命名空间的Service创建一个在Ingress nginx命名空间的Externalname类型的Service，其定义如下：

  ~~~yaml
  apiVersion: v1
  kind: Service
  metadata:
    name: tea-service-externalname      #service的名字
    namespace: ingress-nginx   #ingress-controller所在的namespace
  spec:
    type: ExternalName     
    sessionAffinity: None
    externalName: tea-svc.default.svc.cluster.local 
    #servicename.namespacename.scv.cluster.local
  
  ~~~

* 然后在Ingress nginx命名空间下创建Ingress

  ~~~yaml
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    name: tea-svc-externalname-ingress    #ingress名称
    namespace: ingress-nginx       #这里写前端服务运行所在的命名空间名称
    annotations:
      kubernetes.io/ingress.class: "nginx"
  spec:
    rules:
    - host: tea-svc.cn       #设置域名
      http:
        paths:
        - path: /
          pathType: Prefix    # 前缀匹配
          backend:
            service:
              name: tea-service-externalname     #填写上一步service的名称
              port:
                number: 80            #填写服务暴漏的端口
  ~~~

* apply之后会发现ingress-nginx找不到服务的Endpoint，经过查找后，发现网上说问题原因如下：

  可能是Externalname代理的Service本身只是个CNAME条例，并没有Endpoint，但是Ingress nginx是需要找到一个Endpoint的，所以会代理失败。

  其他相关文档和链接可待探索：

  [请问k8s中，ingress可以实现跨namespace代理service吗？](https://www.zhihu.com/question/494808166)

# Ingress 测试案例3：使用重写注解转发

* 测试yaml

  ~~~yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: nginx-deployment
    labels:
      app: nginx
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
          image: nginx:1.25.3 # 使用具体的 Nginx 镜像版本
          ports:
          - containerPort: 80
  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: nginx-service
    labels:
      app: nginx
  spec:
    type: NodePort # 可以根据需要更改为 NodePort 或 ClusterIP
    ports:
    - port: 80
      targetPort: 80
      protocol: TCP
    selector:
      app: nginx
  
  ---
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    name: nginx-demo-ingress
    annotations:
      kubernetes.io/ingress.class: "nginx"
      nginx.ingress.kubernetes.io/rewrite-target: /  # 添加路径重写
  spec:
    rules:
    - host: nginx-demo.com
      http:
        paths:
        - path: /tea
          pathType: Prefix
          backend:
            service:
              name: nginx-service
              port:
                number: 80
  ~~~

  > 此处`nginx.ingress.kubernetes.io/rewrite-target: /`为重写注解，其将下面path中所有访问路径，传递给后台服务的路径都是/，而没有path后面的tea之类的字符

* 服务部署成功：

  ~~~bash
  (base) [root@master cafeExample]# kubectl get svc
  NAME            TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
  nginx-service   NodePort    10.108.104.125   <none>        80:31345/TCP   28m
  ~~~

* Ingress部署成功

  ~~~bash
  (base) [root@master cafeExample]# kubectl get ingress
  NAME                 CLASS    HOSTS              ADDRESS          PORTS   AGE
  nginx-demo-ingress   <none>   nginx-demo.com     192.168.23.100   80      30m
  ~~~

  > 注意到，不论是service还是ingress都是在default命名空间的

* 在主机中hosts设置DNS解析并访问

  ~~~bash
  192.168.23.100 nginx-demo.com
  ~~~

  访问：`http://nginx-demo.com:31921/tea`
  
  <img src="https://typora2icture.oss-cn-beijing.aliyuncs.com/img2/20250417200732.png" alt="image.png" style="zoom:50%;" />

  <img src="https://typora2icture.oss-cn-beijing.aliyuncs.com/img2/20250417201328.png" alt="image.png" style="zoom: 80%;" />
  
  
  > 两种方式访问，都是有效的


# 参考文章

[nginx-ingress部署+跨命名空间转发](https://blog.csdn.net/baidu_35848778/article/details/129277139)

https://github.com/resouer/kubernetes-ingress/tree/master/examples/complete-example

https://developer.aliyun.com/article/1111110

https://github.com/kubernetes/ingress-nginx

https://blog.csdn.net/justlpf/article/details/132227856
