---
tags:
  - chaosblade
  - 异常注入
---

# 环境准备

1. 需要Kubernetes集群的版本在v1.16+

2. Helm V3

   版本检查`helm version`

   ~~~shell
   root@master:~/work/locust# helm version
   version.BuildInfo{Version:"v3.14.2", GitCommit:"c309b6f0ff63856811846ce18f3bdc93d2b4d54b", GitTreeState:"clean", GoVersion:"go1.21.7"}
   ~~~

> 除此以外，本文档安装chaosblade时，
>
> Ubuntu 24.04
>
> Kubernetes V1.28.15
>
> helm v3.14.2

# Chaosblade-box平台安装

* 先在k8s集群中创建一个chaosblade的namespace

  ~~~shell
  kubectl create ns chaosblade
  ~~~

* https://chaosblade.io/docs/getting-started/installation-and-deployment/platform-box-install-and-uninstall

  有一个问题就是，部署完成后，如果是在本地的k8s集群上部署，是无法满足Chaosblade-box Service的Loadbalance要求的，如下

  ~~~bash
  (locust-env) root@master:~/work/locust# kubectl get svc -n chaosblade
  NAME                        TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)           AGE
  chaosblade-box              LoadBalancer   10.111.1.227    <pending>     7001:31575/TCP    39h
  chaosblade-box-mysql        ClusterIP      10.96.185.217   <none>        3306/TCP          39h
  ~~~

  可见`chaosblade-box`的EXTERNAL-IP状态是pending

  有两种解决方案

  1. 编辑`chaosblade-box`Service,将其变为NodePort类型

     ~~~bash
     kubectl edit svc chaosblade-box -n chaosblade
     ~~~

     ~~~yaml
     ...
     spec:
       clusterIP: 10.111.1.227
       clusterIPs:
       - 10.111.1.227
       externalTrafficPolicy: Cluster
       internalTrafficPolicy: Cluster
       ipFamilies:
       - IPv4
       ipFamilyPolicy: SingleStack
       ports:
       - name: http
         nodePort: 31575
         port: 7001
         protocol: TCP
         targetPort: 7001
       selector:
         app: chaosblade-box
       sessionAffinity: None
       type: NodePort	#将原来的LoadBalancer类型变为NodePort类型
     status:
       loadBalancer: {}
     ~~~

  2. 安装MetaLB（未试验）

     https://www.lixueduan.com/posts/cloudnative/01-metallb/

  然后访问chaosblade-box的服务，
  ![image.png](https://typora2icture.oss-cn-beijing.aliyuncs.com/img2/20250612160706.png)
  
  ==注意，这里是需要自己去注册之后再登录的==

# Chaosblade-box 探针安装

* 探针主要作为平台端建联、命令下发通道和数据收集等功能，所以如果需要对目标集群或主机进行演练，需要在端侧的目标集群或主机上安装探针，以便将平台编排好的演练转化成命令，下发到目标机器上。

* https://chaosblade.io/docs/getting-started/installation-and-deployment/agent-install

  同样存在问题，如下

  ~~~bash
  root@master:~/work/locust# kubectl get svc -n chaosblade
  NAME                        TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)           AGE
  chaos-agent                 LoadBalancer   10.102.25.95    <pending>     19527:32262/TCP   43h
  chaosblade-box              NodePort       10.111.1.227    <none>        7001:31575/TCP    44h
  chaosblade-box-mysql        ClusterIP      10.96.185.217   <none>        3306/TCP          44h
  ~~~

  可见，chaos-agent的Service也是出于LoadBalance不可用的状态，处理方法同上，本次也采用将LoadBalance改为NodePort的方式

# ChaosBlade安装

* https://chaosblade.io/docs/getting-started/installation-and-deployment/tool-chaosblade-install-and-uninstall

* 安装完成后，会有

  ~~~bash
  NAME                                    READY   STATUS    RESTARTS   AGE
  chaosblade-operator-688568959-lcwgb     1/1     Running   0          6s
  chaosblade-tool-c9xjd                   1/1     Running   0          6s
  chaosblade-tool-hvqcv                   1/1     Running   0          6s
  chaosblade-tool-q8jjd                   1/1     Running   0          6s
  ~~~

  本机上结果为：

  ~~~bash
  (locust-env) root@master:~/work/locust# kubectl get pod -n chaosblade -o wide
  NAME                                    READY   STATUS    RESTARTS   AGE   IP              NODE     NOMINATED NODE   READINESS GATES
  chaos-agent-84c54cdbd6-pd9zw            1/1     Running   0          44h   192.168.3.226   master   <none>           <none>
  chaosblade-box-6749f6d4dd-4v29g         1/1     Running   0          44h   10.244.2.9      node2    <none>           <none>
  chaosblade-box-mysql-5955f6f49b-h6j5n   1/1     Running   0          44h   10.244.3.22     node3    <none>           <none>
  chaosblade-operator-59d4968db9-fqfmg    1/1     Running   0          44h   10.244.3.21     node3    <none>           <none>
  chaosblade-tool-d94cl                   1/1     Running   0          44h   192.168.3.228   node3    <none>           <none>
  chaosblade-tool-h6xrc                   1/1     Running   0          44h   192.168.3.229   node1    <none>           <none>
  chaosblade-tool-h9865                   1/1     Running   0          44h   192.168.3.224   node2    <none>           <none>
  chaosblade-tool-tl7sp                   1/1     Running   0          44h   192.168.3.226   master   <none>           <none>
  ~~~

  ~~~bash
  (locust-env) root@master:~/work/locust# kubectl get svc -n chaosblade
  NAME                        TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)           AGE
  chaos-agent                 NodePort    10.102.25.95    <none>        19527:32262/TCP   43h
  chaosblade-box              NodePort    10.111.1.227    <none>        7001:31575/TCP    44h
  chaosblade-box-mysql        ClusterIP   10.96.185.217   <none>        3306/TCP          44h
  chaosblade-webhook-server   ClusterIP   10.102.220.37   <none>        443/TCP           44h
  ~~~

* 关于chaosblade的使用，以给`k8s-learn`命名空间的`my-custom-metrics-app-deployment-d9bf8c9c4-jxlsc`注入异常为例

  ~~~bash
  (locust-env) root@master:~/work/locust# kubectl get pod -n k8s-learn -o wide
  NAME                                               READY   STATUS    RESTARTS   AGE    IP            NODE    NOMINATED NODE   READINESS GATES
  my-custom-metrics-app-deployment-d9bf8c9c4-jxlsc   1/1     Running   0          20h    10.244.3.23   node3   <none>           <none>
  nginx-deployment-7c79c4bf97-2qwvz                  1/1     Running   0          7d1h   10.244.3.10   node3   <none>           <none>
  nginx-deployment-7c79c4bf97-5rrkc                  1/1     Running   0          7d1h   10.244.1.7    node1   <none>           <none>
  ~~~

  其中，`my-custom-metrics-app-deployment-d9bf8c9c4-jxlsc`的yaml配置为

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
  
  ---
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

* 首先注入时延异常

  * 创建对应的yaml

    ~~~yaml
    apiVersion: chaosblade.io/v1alpha1
    kind: ChaosBlade
    metadata:
      name: custom-metrics-network-delay
      namespace: k8s-learn
    spec:
      experiments:
      - scope: pod
        target: network
        action: delay
        desc: "delay pod network"
        matchers:
        - name: names
          value:
          - "my-custom-metrics-app-deployment-d9bf8c9c4-dlmv9"
        - name: namespace
          value:
          - "k8s-learn"
        - name: local-port
          value: ["8000"]
        - name: interface
          value: ["eth0"]
        - name: time
          value: ["3000"]
        - name: offset
          value: ["1000"]
    ~~~

    然后执行

    ~~~bash
    kubectl apply -f custom-metrics-network-delay.yaml
    ~~~

    结果查看

    ~~~bash
    (locust-env) root@master:~/work/locust/chaoses# kubectl get blade -n k8s-learn
    NAME                           AGE
    custom-metrics-network-delay   14s
    (locust-env) root@master:~/work/locust/chaoses# kubectl get blade custom-metrics-network-delay -n k8s-learn -o json
    {
        "apiVersion": "chaosblade.io/v1alpha1",
        "kind": "ChaosBlade",
        "metadata": {
            "annotations": {
                "kubectl.kubernetes.io/last-applied-configuration": "{\"apiVersion\":\"chaosblade.io/v1alpha1\",\"kind\":\"ChaosBlade\",\"metadata\":{\"annotations\":{},\"name\":\"custom-metrics-network-delay\"},\"spec\":{\"experiments\":[{\"action\":\"delay\",\"desc\":\"delay pod network\",\"matchers\":[{\"name\":\"names\",\"value\":[\"my-custom-metrics-app-deployment-d9bf8c9c4-jxlsc\"]},{\"name\":\"namespace\",\"value\":[\"k8s-learn\"]},{\"name\":\"local-port\",\"value\":[\"8000\"]},{\"name\":\"interface\",\"value\":[\"eth0\"]},{\"name\":\"time\",\"value\":[\"3000\"]},{\"name\":\"offset\",\"value\":[\"1000\"]}],\"scope\":\"pod\",\"target\":\"network\"}]}}\n"
            },
            "creationTimestamp": "2025-06-12T08:21:31Z",
            "finalizers": [
                "finalizer.chaosblade.io"
            ],
            "generation": 1,
            "name": "custom-metrics-network-delay",
            "resourceVersion": "1169923",
            "uid": "3eaa15b9-b7f9-4b24-a07e-fc452fd9c03e"
        },
        "spec": {
            "experiments": [
                {
                    "action": "delay",
                    "desc": "delay pod network",
                    "matchers": [
                        {
                            "name": "names",
                            "value": [
                                "my-custom-metrics-app-deployment-d9bf8c9c4-jxlsc"
                            ]
                        },
                        {
                            "name": "namespace",
                            "value": [
                                "k8s-learn"
                            ]
                        },
                        {
                            "name": "local-port",
                            "value": [
                                "8000"
                            ]
                        },
                        {
                            "name": "interface",
                            "value": [
                                "eth0"
                            ]
                        },
                        {
                            "name": "time",
                            "value": [
                                "3000"
                            ]
                        },
                        {
                            "name": "offset",
                            "value": [
                                "1000"
                            ]
                        }
                    ],
                    "scope": "pod",
                    "target": "network"
                }
            ]
        },
        "status": {
            "expStatuses": [
                {
                    "action": "delay",
                    "resStatuses": [
                        {
                            "id": "e850b7bb523ffd54",
                            "identifier": "k8s-learn/node3/my-custom-metrics-app-deployment-d9bf8c9c4-jxlsc/my-custom-metrics-app/b55157efc93593d0112b5ad8d865e961c9afdb59b7c415950559db01665d64cd/containerd",
                            "kind": "pod",
                            "state": "Success",
                            "success": true
                        }
                    ],
                    "scope": "pod",
                    "state": "Success",
                    "success": true,
                    "target": "network"
                }
            ],
            "phase": "Running"
        }
    }
    ~~~

    ==注意chaosBlade的yaml配置要和被注入的Pod的配置对应==

    效果体现，可以使用curl访问对应的服务，或者使用locust对对应服务进行压测，可以看到ResponseTime会有较大的变化提升

* 再给出镜像拉取异常

  ~~~yaml
  apiVersion: chaosblade.io/v1alpha1
  kind: ChaosBlade
  metadata:
    name: custom-metrics-pod-image-failure
    namespace: k8s-learn
  spec:
    experiments:
    - scope: pod
      target: pod
      action: fail
      desc: "inject pod image failure"
      matchers:
      - name: labels
        value:
        - "app=my-custom-metrics-app"
      - name: namespace
        value:
        - "k8s-learn"
  ~~~

* 网络丢包异常

  ~~~yaml
  apiVersion: chaosblade.io/v1alpha1
  kind: ChaosBlade
  metadata:
    name: custom-metrics-network-loss
    namespace: k8s-learn
  spec:
    experiments:
    - scope: pod
      target: network
      action: loss
      desc: "delay pod network"
      matchers:
      - name: names
        value:
        - "my-custom-metrics-app-deployment-d9bf8c9c4-dlmv9"
      - name: namespace
        value:
        - "k8s-learn"
      - name: local-port
        value: ["8000"]
      - name: interface
        value: ["eth0"]
      - name: percent
        value: ["50"]
  ~~~

  
