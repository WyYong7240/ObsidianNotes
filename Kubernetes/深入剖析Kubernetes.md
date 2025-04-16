---
tags:
  - k8s
  - Kubernets
  - 剖析
---

# 第7章 Kubernetes网络原理

## 7.7 从外界连通Service与Service 调试三板斧

* 由于Service的访问入口，其实就是每台宿主机上由kube-proxy 生成的iptables 规则，以及由kube－如s 生成的DNS 记录。而一旦离开了这个集群，这些信息对用户来说自然也就没有作用了。
* 使用Kubernetes的Service，从外部访问的集群中的Service的方式

### NodePort

* 示例yaml

  ```yaml
  apiVersion: v1
  kind: Service
  metadata:
    name: my-nginx
    labels:
      run: my-nginx
  spec:
    type: NodePort
    ports:
    - name: http
      nodePort: 30080
      port: 80
      targetPort: 80
      protocol: TCP
    - name: https
      nodePort: 30443
      port: 443
      protocol: TCP
    selector:
      run: my-nginx
  ```

  声明Service的类型**type=NodePort**,Ports字段声明Service的30080端口代理Pod的80端口，30443端口代理Pod的443端口

  如果不显式声明nodePort字段，Kubernetes会随机分配可用端口，范围默认是30000-32767，可以通过kube-apiserver的–service-node-port-range参数修改此范围

  访问这个service，`curl <k8s集群中任意主机IP>:8080`

  > - nodePort是集群节点上暴露的服务端口，用于外部服务访问
  >
  > - port是Kubernetes Service对外暴露的虚拟端口
  >
  >   当你创建一个 `Service` 对象时，Kubernetes 会为该服务分配一个稳定的 IP 地址，这通常被称为 `ClusterIP`。这个 IP 地址是虚拟的，并且只在集群内部可达。通过这个 `ClusterIP`，其他的服务或 Pod 可以访问你的服务。
  >
  >   - **`port`** 是服务暴露的一个端口号，用于接收来自集群内部的请求。它不是VIP本身，但它与VIP一起工作，确保请求能够被正确路由到服务。
  >   - **`ClusterIP`**（可以理解为VIP的一种形式）是服务的虚拟IP地址，它与`port`结合使用，允许集群内的其他组件通过 `<ClusterIP>:<port>` 访问服务。
  >
  > - targetPort是实际运行在后端Pod上的容器端口
  >
  > 外部访问路径：`<NodeIP>:nodePort` → `Service:port` → `Pod:targetPort`

* 原理：

  NodePort模式中，kube-proxy在每台宿主机上生成一条iptables规则：

  `-A KUBE-NODEPORTS -p tcp -m comment --comment "default /my-nginx : nodePort" -m tcp --dport 30080 KUBE-SVC-67RL4FN6JRUPOJYM`

  而KUBE-SVC-67RL4FN6JRUPOJYM就是上一节那种随机模式的iptables规则，接下来的流程和Cluster IP模式完全一致

* 需要注意：

  在NodePort方式下，Kubernetes会在IP包离开宿主机发往目的Pod时，对该IP包进行一次SNAT操作

  `-A KUBE- POSTROUTING -m comment - -comment "kubernetes service traffic requiring SNAT" -m mark - -mark Ox4000/0x4000 -j MASQUERADE`

  这条规则设置在POSTROUTING检查点，它对即将离开这台主机的IP 包进行了一次SNAT 操作，**将这个IP 包的源地址替换成了这台宿主机上的CNI 网桥地址，或者宿主机本身的IP 地址（ 若CNI 网桥不存在）**。

  并且SNAT操作只需要对Service转发出来的IP包进行SNAT操作，依据就是查看该IP包是否包含IP包被执行DNAT操作时留下的0x4000的标志

  * 为什么要进行SNAT操作？

    外部访问图解：[[ServiceNodePort访问图解-SNAT]]
    <img src="https://typora2icture.oss-cn-beijing.aliyuncs.com/img2/20250416171355.png" alt="image.png" style="zoom:30%;" />

  
    原本访问流程：
  
    1. client通过Node2的地址访问Service
  
    2. Node2上的负载均衡规则将IP包发给Node1上的Pod
  
    3. Node1上的Pod处理完请求，**按照IP包的源地址发出回复**
  
       **如果没有进行SNAT操作，此时IP包的源地址就是IP地址，所以Pod会将回复直接发给Client，但是Client的请求发给Node2，却收到了Node1的回复，会出错**
  
    SNAT访问流程：
  
    1. client通过Node2的地址访问Service
    2. Node2上的负载均衡规则将IP包发给Node1上的Pod，**同时IP包的源IP地址被改为node2的CNI网桥地址或Node2的地址**
    3. Node1上的Pod处理完请求，将回复发给Node2
    4. Node2将回复发给Client
  
  * 如果Pod需要知道真正的请求来自何处，也就是知道Client地址，可以将Service的`spec.externalTrafficPolicy`字段设置为local
  
    原理就是**宿主机上的iptables规则设置为只将IP包转发给在本宿主机上运行的Pod，这样Pod就可以使用源地址发出回复包，不用SNAT**
  
    但也意味着**无法从Node2上访问这个Service**
  
    此情况下的流程图：[[ServiceNodePort访问图解NoSNAT]]
  <img src="https://typora2icture.oss-cn-beijing.aliyuncs.com/img2/20250416171538.png" alt="image.png" style="zoom: 50%;" />


### LoadBalance

此种方式适用于公有云上的Kubernetes服务，在云上可以指定一个LoadBalance类型的Service，其yaml如下所示

```yaml
kind: Service
apiVersion: v1
metadata:
  name: expamle-service
spec:
  ports:
  - port: 8765
    targetPort: 9376
  selector:
    app: example-app
  type: LoadBalancer
```

* 公有云提供的Kubernetes服务中，使用了CloudProvider的转接层来跟公有云本身的API进行对接。在上述LoadBalance类型的Service提交之后，Kubernetes就会调用CloudProvider 在公有云上创建一个负载均衡服务，并且把被代理的Pod的IP地址配置给负载均衡服务作为后端

### ExternalName

> 此特性在v1.7之后支持

* 示例yaml

  ~~~yaml
  kind: Service
  apiVersion: v1
  metadata:
    name: my-service
  spec:
    type: ExternalName
    externalName: my.database.example.com
  ~~~

  可以注意到，这个yaml不需要指定selector

  当通过service的DNS名字访问它时（比如访问my-service.default.svc.cluster.local），Kubernetes返回的就是my.database.example.com

  所以，ExternalName类型的Service其实是在Kube-Dns中添加了一条CNAME记录

* Kubernetes的Service还允许为Service分配公有IP地址

  ~~~yaml
  kind: Service
  apiVersion: v1
  metadata:
    name: my-service
  spec:
    selector:
      app: my-app
    ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 9736
    externalIPs:
    - 80.11.12.10
  ~~~

  上述Service指定了externalIPs=80.11.12.10，这样就可以通过80.11.12.10:80访问到被代理的Pod了，**不过此处要求externalIPs至少能够路由到一个Kubernetes节点**



### Service的Debug

当Service无法通过DNS访问时，要区分是Service本身的配置问题，还是集群的DNS出了问题

* 检查Kubernetes自己的Master节点的ServiceDNS是否正常

  ~~~bash
  root@master:~# kubectl exec prometheus-k8s-0 -n monitoring -it -- /bin/sh
  /prometheus $ cat /etc/resolv.conf
  search monitoring.svc.cluster.local svc.cluster.local cluster.local
  nameserver 10.96.0.10
  options ndots:5
  /prometheus $
  /prometheus $
  /prometheus $ nslookup kubernetes.default
  Server:         10.96.0.10
  Address:        10.96.0.10#53
  
  Name:           kubernetes.default
  Address:        10.96.0.1
  ~~~

  如果Kubernetes.default返回值都有问题，就要检查kube-dns的运行状态了

  > resolve.conf中配置了Pod的DNS，用于解析集群内部服务名称，这个文件内容通常由kubernetes自动设置，并指向集群的DNS服务（kube-dns或coreDNS）
  >
  > 示例典型内容：
  >
  > nameserver 10.96.0.10 
  >
  > search default.svc.cluster.local svc.cluster.local cluster.local 
  >
  > options ndots:5

* 如果Service无法通过ClusterIP访问到，首先检查这个Service是否有EndPoints

  ~~~bash
  root@master:~# kubectl get endpoints -A
  NAMESPACE     NAME                    ENDPOINTS                                                                    AGE
  default       kubernetes              39.98.76.224:6443                                                            15d
  kube-system   kube-dns                10.244.2.13:53,10.244.2.15:53,10.244.2.13:53 + 3 more...                     15d
  kube-system   kubelet                 192.168.3.227:10250,192.168.3.226:10250,172.23.192.199:10250 + 12 more...    15d
  kubeedge      cloudcore               172.23.192.199:10000,172.23.192.199:10003,172.23.192.199:10004 + 2 more...   15d
  monitoring    alertmanager-main       10.244.1.43:8080,10.244.1.44:8080,10.244.2.27:8080 + 3 more...               8d
  monitoring    alertmanager-operated   10.244.1.43:9094,10.244.1.44:9094,10.244.2.27:9094 + 6 more...               8d
  monitoring    blackbox-exporter       10.244.2.21:9115,10.244.2.21:19115                                           8d
  monitoring    grafana                 10.244.2.26:3000                                                             8d
  monitoring    kube-state-metrics      10.244.1.41:8443,10.244.1.41:9443                                            8d
  monitoring    node-exporter           172.23.192.199:9100,172.23.192.200:9100,172.23.192.201:9100                  8d
  monitoring    prometheus-adapter      10.244.1.34:6443,10.244.2.22:6443                                            8d
  monitoring    prometheus-k8s          10.244.1.37:8080,10.244.2.23:8080,10.244.1.37:9090 + 1 more...               8d
  monitoring    prometheus-operated     10.244.1.37:9090,10.244.2.23:9090                                            8d
  monitoring    prometheus-operator     10.244.1.35:8443                                                             8d
  
  root@master:~# kubectl get endpoints prometheus-k8s -n monitoring
  NAME             ENDPOINTS                                                        AGE
  prometheus-k8s   10.244.1.37:8080,10.244.2.23:8080,10.244.1.37:9090 + 1 more...   8d
  ~~~

  注意，如果Pod的readniessProbe没通过，它也不会出现在Endpoints列表中

* 如果Endpoints正常，就要确认kube-proxy是否正常运行，查看kube-proxy日志

  ~~~bash
  root@master:~# kubectl logs -f kube-proxy-psf4q -n kube-system
  I0409 03:23:52.260550       1 server_others.go:69] "Using iptables proxy"
  I0409 03:23:53.009778       1 node.go:141] Successfully retrieved node IP: 172.23.192.199
  I0409 03:23:53.014446       1 conntrack.go:52] "Setting nf_conntrack_max" nfConntrackMax=262144
  I0409 03:23:53.014591       1 conntrack.go:100] "Set sysctl" entry="net/netfilter/nf_conntrack_tcp_timeout_close_wait" value=3600
  I0409 03:23:53.092998       1 server.go:632] "kube-proxy running in dual-stack mode" primary ipFamily="IPv4"
  I0409 03:23:53.096818       1 server_others.go:152] "Using iptables Proxier"
  I0409 03:23:53.096874       1 server_others.go:421] "Detect-local-mode set to ClusterCIDR, but no cluster CIDR for family" ipFamily="IPv6"
  I0409 03:23:53.096882       1 server_others.go:438] "Defaulting to no-op detect-local"
  I0409 03:23:53.096924       1 proxier.go:250] "Setting route_localnet=1 to allow node-ports on localhost; to change this either disable iptables.localhostNodePorts (--iptables-localhost-nodeports) or set nodePortAddresses (--nodeport-addresses) to filter loopback addresses"
  I0409 03:23:53.097264       1 server.go:846] "Version info" version="v1.28.15"
  I0409 03:23:53.097279       1 server.go:848] "Golang settings" GOGC="" GOMAXPROCS="" GOTRACEBACK=""
  I0409 03:23:53.098444       1 config.go:188] "Starting service config controller"
  I0409 03:23:53.098481       1 shared_informer.go:311] Waiting for caches to sync for service config
  I0409 03:23:53.098561       1 config.go:97] "Starting endpoint slice config controller"
  I0409 03:23:53.098577       1 shared_informer.go:311] Waiting for caches to sync for endpoint slice config
  I0409 03:23:53.098587       1 config.go:315] "Starting node config controller"
  I0409 03:23:53.098612       1 shared_informer.go:311] Waiting for caches to sync for node config
  I0409 03:23:53.199064       1 shared_informer.go:318] Caches are synced for node config
  I0409 03:23:54.598960       1 shared_informer.go:318] Caches are synced for endpoint slice config
  I0409 03:23:54.598962       1 shared_informer.go:318] Caches are synced for service config
  ~~~

* 如果kube-proxy一切正常，就要查看宿主机上的iptables。iptables模式的Service对应的所有规则如下

  1. KUBE-SERVICES 或者KUBE-NODEPORTS 规则对应的Service 的入口链，这些规则应该与VIP（ClusterIP）和Service 端口一一对应；
  2. KUBE-SEP-(hash)规则对应的DNAT 链，这些规则应该与Endpoints一一对应；
  3. KUBE-SVC-(hash)规则对应的负载均衡链，这些规则的数目应该与Endpoints 数目一致
  4. 如果是NodePort 模式，还涉及POSTROUTING 处的SNAT链。

  > 除此以外，还有别的链

* 如果是Pod无法通过Service访问自己

  1. 注意kubelet的hairpin-mode有没有正确设置
  2. 可以将kubelet的hairpin-mode设置为hairpin-veth或者promiscuous-birdge

## 7.8 Kubernetes中的Ingress对象

* 产生原因:

  由于每个Service都要有一个负载均衡服务，浪费资源同时成本较高。

  Kubernetes中的Ingress服务，可以实现全局的、代理不同后端Service的负载均衡服务。其实就是Service的Service

  > 假如有一个站点： https://cafe.example.com, 其中https://cafe.example.com/coffee 对应”咖啡点餐系统”，而https://cafe.example.com/tea 对应“茶水点餐系统” 。这两个系统分别由名叫coffee 和tea 的两个Deployment 来提供服务

* Ingress示例yaml

  ~~~yaml
  apiVersion: extensions/v1beta1
  kind: Ingress
  metadata:
    name: cafe-ingress
  spec:
    tls:
    - hosts:
      - cafe.example.com
      secretName: cafe-secret
    rules:
    - host: cafe.example.com
      http:
        paths:
        - path: /tea
          backend:
            serviceName: tea-svc
            servicePort: 80
        - path: /coffee
          backend:
            serviceName: coffee-svc
            servicePort: 80
  ~~~

  最重要的是rules字段，在Kubernetes中叫做IngressRule，其Key叫做host，必须是标准的域名格式字符串

  * host字段定义这个Ingress的入口

    当用户访问cafe.example.com时，实际上访问到这个Ingress对象，然后Kubernetes使用IngressRule对请求进行转发

  * IngressRule规则定义依赖path字段

    每个path都对应一个后端Service

* Ingress对象，其实就是Kubernetes对反向代理的一种抽象

  实际使用过程中，只用选择一个Ingress Controller，将其部署到Kubernetes集群中，然后这个Ingress Controller就会根据定义的Ingress对象提供对应的代理能力

  目前常用的反向代理项目，比如Nginx、HAProxy、Envoy、Traefik等，都为kubernetes专门维护了Ingress Controller
