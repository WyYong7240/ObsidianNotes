---
tags:
  - kube-prometheus
  - Add-target
  - custom-metrics
  - ebpf-exporter
  - Endpoints
---

# 添加监控目标准备

> 部署监控目标，首先得有被监控目标提供的待监控的服务及其端口
>
> 这里使用go代码，自行构建了一个计算端口被访问次数的镜像，并将其运行为Pod，具体见[[构建容器镜像#构建镜像代码]]
>
> * Kubernetes：v1.28.15
> * Kube-prometheus:v1.12.1

* 部署待监控应用`deployment.yaml`

  ~~~yaml
  # ================= Deployment =================
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: my-custom-metrics-app-deployment
    namespace: k8s-learn
    labels:
      app: my-custom-metrics-app
  spec:
    replicas: 1
    selector:
      matchLabels:
        app: my-custom-metrics-app
    template:
      metadata:
        labels:
          app: my-custom-metrics-app
      spec:
        containers:
        - name: my-custom-metrics-app
          image: my-custom-metrics-app:v1.1
          ports:
          - containerPort: 8000
  ~~~

* 部署待监控应用的service `service.yaml`

  ~~~yaml
  # ================= Service =================
  apiVersion: v1
  kind: Service
  metadata:
    name: my-custom-metrics-service
    namespace: k8s-learn
    labels:
      app: my-custom-metrics-service
  spec:
    type: NodePort
    selector:
      app: my-custom-metrics-app
    ports:
      - name: metrics
        protocol: TCP
        port: 8000
        targetPort: 8000
  ~~~

  * 详述`spec.ports`字段

    1. `name`字段，为端口起了个名字，用于后面定义ServiceMonitor时使用

    2. `protocol`字段，指定端口使用的协议，包括TCP、UDP、SCTP，默认为TCP

    3. `port`字段，Service在集群内部暴露的端口，其他Pod可以通过`clusterIP:port`访问该服务

    4. `targetPort`字段，后端Pod上，接收流量的端口，Service会将请求转发到匹配标签Pod的targetPort上

       可以是数字，也可以是Pod容器端口的名字

       > 这里的后台应用Pod，提供了端口8000，并且提供了访问路径`IP:8000/metrics`这个访问路径的实现也见其他文件[[构建容器镜像#构建镜像代码]]
       >
       > 因为有这个访问路径，Prometheus可以监听到`/metrics`中提供的Prometheus格式的监控数据

# 为Kube-prometheus授权

> Kube-prometheus使用的是名为`prometheus-k8s`的ServiceAccount，属于`monitoring`命名空间，但是这个ServiceAccount缺少在其他命名空间下的资源访问权限
>
> 但是`endpoints`、`services`是Prometheus抓取监控时必需的，尤其是使用ServiceMonitor来自动发现服务
>
> 因此，需要为Prometheus添加RBAC权限

## 针对特定命名空间授权

1. 在特定命令空间下创建Role(以`k8s-learn`命名空间为例)

   ~~~yaml
   apiVersion: rbac.authorization.k8s.io/v1
   kind: Role
   metadata:
     namespace: k8s-learn
     name: prometheus-namespaced-access
   rules:
   - apiGroups: [""]
     resources: ["services", "endpoints", "pods"]
     verbs: ["get", "list", "watch"]
   ~~~

2. 创建RoleBinding，以绑定到Prometheus的ServiceAccount

   ~~~yaml
   apiVersion: rbac.authorization.k8s.io/v1
   kind: RoleBinding
   metadata:
     name: prometheus-rolebinding
     namespace: k8s-learn
   subjects:
   - kind: ServiceAccount
     name: prometheus-k8s
     namespace: monitoring
   roleRef:
     kind: Role
     name: prometheus-namespaced-access
     apiGroup: rbac.authorization.k8s.io
   ~~~

3. 然后将两个资源`kubectl apply`一下

## 针对集群范围访问

1. 创建ClusterRole

   ~~~yaml
   apiVersion: rbac.authorization.k8s.io/v1
   kind: ClusterRole
   metadata:
     name: prometheus-cluster-access
   rules:
   - apiGroups: [""]
     resources: ["services", "endpoints", "pods"]
     verbs: ["get", "list", "watch"]
   ~~~

2. 创建ClusterRoleBinding

   ~~~yaml
   apiVersion: rbac.authorization.k8s.io/v1
   kind: ClusterRoleBinding
   metadata:
     name: prometheus-clusterbinding
   subjects:
   - kind: ServiceAccount
     name: prometheus-k8s
     namespace: monitoring
   roleRef:
     kind: ClusterRole
     name: prometheus-cluster-access
     apiGroup: rbac.authorization.k8s.io
   ~~~

   1. 将上述两个资源`kubectl apply`一下

# 创建特定服务的ServiceMonitor进行监控

* ServiceMonitor与待监控的Service未跨命名空间情况

  ~~~yaml
  apiVersion: monitoring.coreos.com/v1
  kind: ServiceMonitor
  metadata:
    name: my-custom-metrics-svcmot
    namespace: k8s-learn
    labels:
      app: my-custom-metrics-svcmot
  spec:
    selector:	# 这是用于选定待监控Service
      matchLabels:
        app: my-custom-metrics-service	# 这个应该是待监控Service有的标签
    endpoints:
    - port: metrics # 指定要抓取的端口名，必须与Service中定义的端口名一致
      path: /metrics # 访问路径
      interval: 5s # 抓取时间间隔
    namespaceSelector:	# 待监控Service所在命名空间选定
      matchNames:
      - k8s-learn
  ~~~

  这个ServiceMonitor所在命名空间与待监控的Service在同一个命名空间，因此RBAC授权，只需要授权其所在命名空间

* ServiceMonitor与待监控的Service跨命名空间情况

  ~~~yaml
  apiVersion: monitoring.coreos.com/v1
  kind: ServiceMonitor
  metadata:
    name: ingress-metrics-svcmot
    namespace: k8s-learn
    labels:
      app: ingress-metrics-svcmot
  spec:
    selector:
      matchLabels:
        app: ingress-metrics
    endpoints:
    - port: metrics # 假设服务端口名称为http-metrics
      path: /metrics
      interval: 5s
    namespaceSelector:
      matchNames:
      - ingress-nginx
  ~~~

  这个ServiceMonitor所在命名空间与待监控的Service不在同一个命名空间，因此RBAC授权，需要授权ServiceMonitor、待监控Service所在命名空间`k8s-learn`、`ingress-nginx`

# 效果展示

> 这里以第一个`my-custom-metrics-svcmot`展示

* 访问Prometheus的界面中的target
  ![image.png](https://typora2icture.oss-cn-beijing.aliyuncs.com/img2/20250605193605.png)

* 在该待监控的Service中，自定义了两个指标`myapp_requests_total`、`myapp_request_duration_seconds`

  可以在Prometheus的界面中的Graph进行查询
  ![image.png](https://typora2icture.oss-cn-beijing.aliyuncs.com/img2/20250605193750.png)
  ![image.png](https://typora2icture.oss-cn-beijing.aliyuncs.com/img2/20250605193838.png)
  



# 非k8s本地服务的监控目标

> 该部分目的是将没有以Pod方式运行在k8s集群上的服务的metrics指标添加到kube-Prometheus中
>
> 以docker容器方式运行的ebpf-export为例

## 部署ebpf-exporter

~~~bash
git clone git@github.com:cloudflare/ebpf_exporter.git
cd ebpf-exporter
~~~

运行镜像示例

~~~bash
docker run --rm -it --privileged -p 9435:9435 \
  ghcr.io/cloudflare/ebpf_exporter --config.dir=examples --config.names=timers
~~~

`-it`表示这是以交互式方式运行的容器，服务提供的容器端口为`9435`，ebpf-exporter运行的示例程序是上述仓库下`/example`文件夹下的timers程序，还有`biolatency`,见下面的以`-detach`方式运行的命令

~~~bash
docker run --rm -d --privileged -p 9435:9435 \
  ghcr.io/cloudflare/ebpf_exporter --config.dir=examples --config.names=biolantency
~~~

效果如下
<img src="https://typora2icture.oss-cn-beijing.aliyuncs.com/img2/20250609154735.png" alt="image.png" style="zoom:80%;" />
<img src="https://typora2icture.oss-cn-beijing.aliyuncs.com/img2/20250609154751.png" alt="image.png" style="zoom:80%;" />




## 为该metrics建立Endpoints与Service

1. 在默认命名空间创建Endpoints

   ~~~yaml
   apiVersion: v1
   kind: Endpoints
   metadata:
     name: ebpf-exporter-metrics
   subsets:
   - addresses:
     - ip: 192.168.3.221
     ports:
     - name: metrics
       port: 9435
   ~~~

2. 同命名空间下建立Service

   ~~~yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: ebpf-exporter-metrics
     labels:
       app: ebpf-exporter-metrics
   spec:
     ports:
     - name: metrics
       port: 9435
       targetPort: 9435
   ~~~

> 注意，service的name字段必须与endpoint相同，并且下面port的name也需要相同。另外，需要为这个service配置一个label，以便后面servicemonitor进行筛选。
>
> 创建后，执行kubectl describe，看看service的endpoint字段是否已经显示为[宿主机ip]:9435。执行curl [service的ip]:9435，看看有没有监测数据的返回值。

## 为该服务建立ServiceMonitor

~~~yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: ebpf-exporter-metrics
  labels:
    app: ebpf-exporter-metrics
spec:
  endpoints:
  - port: metrics
    path: /metrics
    interval: 10s
  selector:
    matchLabels:
      app: ebpf-exporter-metrics
  namespaceSelector:
    matchNames:
    - default
~~~

效果如下
![image.png](https://typora2icture.oss-cn-beijing.aliyuncs.com/img2/20250609154829.png)







# 参考文档

1. https://www.51cto.com/article/807115.html
2. https://www.langxw.com/2022/04/19/Kube-prometheus%E6%B7%BB%E5%8A%A0%E8%87%AA%E5%AE%9A%E4%B9%89%E7%9B%91%E6%8E%A7%E9%A1%B9/
3. https://cloud.tencent.com/developer/article/1725761
4. https://realtiger.github.io/k8s-prometheus-add-targets/
5. https://www.cnblogs.com/00986014w/p/12655031.html
6. https://github.com/cloudflare/ebpf_exporter?tab=readme-ov-file#building-examples
7. https://blog.csdn.net/qq_34258344/article/details/115532126
8. https://github.com/cloudflare/ebpf_exporter
9. https://grafana.com/grafana/dashboards/18612-kernel-ebpf-hook/
