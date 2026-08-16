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

有关Ingress-nginx的部署和测试，详见[[Ingress-Nginx部署]]



# 第8章 Kubernetes调度与资源管理

## 8.1 Kubernetes的资源模型与资源管理

* 在Kubernetes里Pod是最小的原子调度单位。这就意味着，所有跟调度和资源管理相关的属性都应该是屈千Pod对象的字段

* 资源类型：

  * 在Kubernetes中，像CPU这样的资源被称作**可压缩资源**(compressible resources )。可压缩资源的典型特点是，**当它不足时， Pod 只会”饥饿＂，不会退出**。
  * 像内存这样的资源则被称作**不可压缩资源**( incompressible resources)。**当不可压缩资源不足时， Pod 就会因为OOM (out of memory ) 被内核结束。**

* 由于Pod可以由多个Container组成，**因此CPU和内存资源的限额是要配置在每个Container的定义上的**，Pod的整体资源配置就是这些Container配置值累加得到。

  * 配置方式：

    1. CPU

       Kubernetes中CPU设置的单位是“CPU的个数”，**具体“1个CPU”在宿主机上的解释，取决于宿主机CPU的实现方式**，类型有：1个CPU核心、1个vCPU、1个CPU的超线程；

       除此以外，Kubernetes允许将CPU限额设置为分数，例如CPU limits设置为500m，500m指的是500millicpu，即0.5个CPU，这样Pod就会被分配1个CPU一半的计算能力

    2. 内存

       内存资源的单位就是byte，Kubernetes支持使用Ei、Pi、Ti、Gi、Mi、Ki（E、P、T、G、M、K）作为byte的值，注意分清MiB和MB（1Mi=1024\*1024）

  * 实现方式

    1. CPU

       资源分配还分为limits和requests两种情况，区别在于，==在调度时，kube-scheduler只会按照requests的值进行计算，但是在设置真正的Cgroups限制时，按照limits值设置==。

       ==当指定了requests.cpu=250m后，相当于设置了Cgroups的cpu.shares的值为(250/1000)\*1000，如果没有设置cpu.requests，其默认值为1024==；以此方式实现对CPU的按时间比例分配。

       ==如果指定limits.cpu=500m，相当于设置cpu.cfs_quota_us为(500/1000)\*100ms，而cpu.cfs_period_us的值始终是100ms，这样Kubernetes设置了这个容器只能使用到CPU的50%==

       > 关于cpu.shares、cpu.cfs_quota_us、cpu.cfs_period_us三个字段
       >
       > 1. **cpu.shares**：
       >    - 这个参数定义了在CPU资源竞争情况下，cgroup内的进程相对于系统中其他cgroup中的进程所获得的CPU使用比例。
       >    - **它采用的是相对权重的方式，默认值通常是1024。数值越大，表示这个cgroup中的进程在竞争CPU资源时能获取更多的CPU时间。**
       >    - 例如，如果一个cgroup设置为2048，而另一个设置为1024，则前者在CPU资源紧张时会获得两倍于后者的CPU时间。
       > 2. **cpu.cfs_quota_us**：
       >    - 这个参数用于限制cgroup内所有进程在一个周期(`cpu.cfs_period_us`)内**可以使用的CPU时间总量（以微秒为单位）。**
       >    - 如果设置为-1（默认值），则表示没有限制。
       >    - 例如，如果设置为50000微秒，意味着在这个cgroup中的所有进程在一个周期内最多只能使用50毫秒的CPU时间。
       > 3. **cpu.cfs_period_us**：
       >    - 这个参数定义了时间周期长度（以微秒为单位），在此周期内评估`cpu.cfs_quota_us`的限制。
       >    - 默认值通常是100000微秒（即100毫秒）。这意味着每100毫秒重新计算一次cgroup可以使用的CPU时间。
       >    - 结合`cpu.cfs_quota_us`使用，可以有效地限制cgroup内的进程在每个周期内可消耗的CPU资源量。

    2. 内存

       当指定limits.memory=128Mi后，相当于设置Cgroups的memory.limit_in_bytes设置为128\*1024\*1024，但是在调度时，调度器只会用requests.memory=64Mi进行判断。

  * 样例yaml

    ~~~ yaml
    apiversion: v1
    kind: Pod
    metadata:
      name: myapp-pod
    spec:
      containers:
      - name: myapp-container
        image: busybox
        env:
        - name: MY_ENV_VAR
          value: my_value
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
      - name: wp
        image: wordpress
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
    ~~~

    

  > 调度系统不是必须严格遵循容器化作业在提交时设置的资源边界，这是因为在实际场景中，大多数作业用到的资源其实远少千它所请求的资源限额。
  >
  > 当作业资源使用量增加到一定阈值，Borg通过快速恢复过程还原作业原始的资源限额，防止出现异常。
  >
  > Kubernetes的requests+limits方式就是Borg思路的简化版。

### 三种Qos级别

在Kubernetes中，不同的requests和limits的设置方式其实会将这个Pod划分到不同的QoS级

* 三种Qos类型定义

  * **Guaranteed**

    Pod中每个Container都同时设置了request、limits并且requests和limits值相等，那么这个Pod属于Guaranteed类别；**如果Pod仅设置了limits但是没有设置requests，Kubernetes会自动为其设置与limits相同的requests值，因此也属于Guaranteed情况。**

    此类Pod创建后，其qosClass字段自动设置为Guaranteed；

  * **Burstable**

    当不满足Guaranteed类别，但是至少有一个Container设置了requests，这个Pod就属于Burstable类别

  * BestEffort

    Pod的requests、limits都没有设置

* 分Qos类型的作用

  当宿主机资源紧张时，kubelet对Pod进行Eviction资源回收时用到。具体来说，就是当宿主机不可压缩资源短缺时，就可能触发Eviction

  Kubernetes默认的阈值如下：

  ~~~yaml
  memory.available<100Mi
  nodefs.available<10%
  nodefs.inodesFree<5%
  imagefs.available<15%
  ~~~

  这些都是可配置的

  ~~~yaml
  kubelet
  -- eviction-hard=imagefs.available<l0% ,memory.availabl e<500Mi ,nodefs.available<5%,
  nodefs.inodesFree<5%  --eviction-sof t=imagefs.available<30%，nodefs.available<l0%
  - -eviction-soft-grace-period=imagefs.available=2m, nodefs.availabl e=2m
  --eviction-max-pod-grace-period=600
  ~~~

  * Eviction分为Soft和Hard模式

    Soft模式允许为Eviction过程设置一段优雅时间，==比如上面例子中的imagefs.available=2m,就意味着当达到imagefs不足的阙值超过2分钟，kubelet才会开始Eviction的过程。而在Hard Eviction 模式下，Eviction 过程会在达到阈值之后立刻开始。==

  * Kubemetes 计算Eviction阔值的数据来源，主要依赖从Cgroups 读取的值以及使用cAdvisor监控到的数据。
  
  * 当宿主机的Eviction阈值达到后，就会进入MemoryPressure 或者DiskPressure 状态，从而避免新的Pod 被调度到这台宿主机上。
  
  * **当Eviction 发生时， kubelet 具体会挑选哪些Pod 进行删除，就需要参考这些Pod 的QoS类别了。**
    1. **首先是BestEffort类别的Pod**
    
    2. **其次是Burstable类别，并且发生饥饿的资源使用量已经超出requests的pod**
    
       此处的“饥饿”的资源使用量超过requests的含义是：指某个容器或Pod尝试使用的资源（如CPU或内存）超出了其在部署时所声明的requests值，但还没有达到limits
    
    3. **最后是Guaranteed类别，并且保证只有当Guaranteed类别的Pod资源使用量超过其limits限制，或者宿主机处于Memory Pressure状态时，Guaranteed类别Pod才会被选中进行Eviction操作。**
    
    > 对于同Qos类别的Pod来说，Kubernetes还会根据Pod的优先级进一步排序和选择
  



### CPUset特性

* 使用容器时，可以通过CPUset特性把容器绑定到某个CPU核上，而不是像cpushare一样共享CPU的计算能力。

  此种情况下，由于操作系统在CPU之间进行上下文切换的次数大大减少，因此容器里应用的性能会大幅提升。事实上，cpuset方式是生产环境中部署在线应用类型的Pod的一种常用方式。

* 实现方式

  1. **首先，Pod的Qos类别必须是Guaranteed类型**
  2. **将Pod的CPU资源的requests和limits设置为相同的整数值**

  例如：

  ~~~yaml
  spec:
    containers:
    - name: test
      image: nginx:latest
      resources:
        limits:
          memory: "200Mi"
          cpu: "2" 
        requests:
          memory: "200Mi"
          cpu: "2"
  ~~~

## 8.2 Kubernetes默认调度器

Kubernetes默认调度器的主要职责就是为新创建出来的Pod寻找一个最合适的节点。最合适含义包含两层：

1. **从集群中所有节点中根据调度算法选出所有可以运行该Pod的节点**（预选）
2. **从第一部的结果中，再根据调度算法挑选一个最符合条件的节点作为最终结果**（优选）

在具体的调度流程中，步骤概述如下：

1. 默认调度其调用叫做Predicate的调度算法检查每个节点
2. 再调用叫做Priority的调度算法，给上一步得到的结果里的每个节点打分
3. 得分最高的节点就是最终调度结果

**调度器对一个Pod调度成功，就是将其`spec.nodeName`字段填上调度结果的名字**

### Kubernetes调度机制详解

<img src="https://typora2icture.oss-cn-beijing.aliyuncs.com/img2/image-20250508110457170.png" alt="image-20250508110457170" style="zoom: 67%;" />

Kubernetes调度器的核心就是两个相互独立的控制循环。

* **Informer Path控制循环**

  1. 主要目的是启动一系列Informer，用于监听etcd中的Pod、Node、Service等与调度相关的API对象的变化。
  2. ==例如当一个待调度Pod被创建出来后，其nodeName字段为空，调度器会通过Pod Informer的Handler将这个待调度Pod加入调度队列。==
  3. 默认情况下的Kubernetes调度队列是一个PriorityQueue，并且当集群的某些信息发生变化时，调度器还会对调度队列中的内容进行特殊操作，方便==调度优先级和抢占的实现==。
  4. ==Kubernetes的默认调度器还负责更新调度器缓存scheduler cache。Kubernetes调度优化性能的根本原则就是尽可能将集群信息缓存化，提升Predicate和Priority算法执行效率。==

* **Scheduling Path控制循环**

  1. Scheduling Path是负责Pod调度的主循环，==其逻辑是不断从调度队列中出队一个Pod，然后调用Predicate算法进行过滤，得到一组可以运行这个Pod的节点，Predicate算法所需的信息都是从Scheduler Cache中获取的。==

  2. ==调度器再调用Priority算法为过滤得到的节点打分，从0-10，得分最高的节点作为调度的结果。==

  3. 调度算法完成后，调度器将Pod的nodeName字段修改为上述节点的名称，这个操作被称为Bind

     > **但是为了不在关键调度路径中远程访问API Server，Kubernetes默认调度器在Bind阶段只会更新Scheduler Cache中的Pod和Node信息。**
     >
     > **这种基于乐观假设的API对象更新方式，称为Assume。**
     >
     > **Assume完成后，调度器会创建一个Goroutine来异步地向API Server发起更新Pod的请求，来完成真正的Bind操作。如果异步Bind过程失败了，只要等Scheduler Cache同步后就可以了。**
     >
     > 
     >
     > 由于乐观绑定的设计，当一个pod完成调度需要在某个节点上运行之前，**该节点的kubelet还会通过一个叫做Admit的操作来再次验证该Pod是否可以在该节点上运行。**
     >
     > 这一步Admit操作就是把一组叫做GeneralPredicates的最基本的调度算法（资源是否可用、端口是否冲突）再执行一遍，作为kubelet端的二次确认。

### Kubernetes调度器的无锁化设计

1. 在Scheduling Path上，**调度器会启动多个Goroutine以节点为粒度并发执行Predicate算法，从而提高该阶段执行效率**。

2. **Priority算法也会以MapReduce的方式并行计算然后汇总。**

   并在所有需要并发的路径上，调度器会避免设置任何全局资源的竞争，免除使用锁进行同步而产生的巨大性能损耗。

3. 由于此设计，Kubernetes调度器只有对调度队列和Scheduler Cache进行操作时，才需要加锁，而这两部分操作都不在Scheduling Path的算法执行路径上。

### Kubernetes默认调度器的可扩展性设计


<img src="https://typora2icture.oss-cn-beijing.aliyuncs.com/img2/20250510142026.png" alt="image.png" style="zoom:67%;" />

* Kubernetes中默认调度器的可扩展机制叫做**Scheduler Framework**

  主要目的就是在调度器生命周期的各个关键点上，向用于暴露可以进行扩展和实现的接口，详见[[Kubernetes调度框架设计与Scheduler Plugins开发部署示例]]

* Scheduler Framework的缺陷

  1. 一旦插入点的接口设计不合理，就会导致整个生态无法充分利用该插件机制。
  2. 接口的变更耗时费力

## 8.3 Kubernetes默认调度器调度策略解析

### Predicates预选策略

预选在调度过程中的作用可以理解为Filter，按照调度策略从当前集群中的所有节点中过滤出一系列复合条件的节点。

默认的的预选策略有如下4种

1. **GeneralPredicates**

   * 这一组过滤规则负责最基础的调度策略，例如PodFitsResource计算的是组主机的CPU、内存资源是否够用（检查的只是Pod的requests字段）。

     > Kubernetes调度器并没有为GPU等硬件资源定义具体的资源类型，而是==统一使用ExtendedResource的Key-value格式扩展字段来描述==，例如：
     >
     > ~~~yaml
     > ·apiVersion: v1
     > kind: Pod
     > metadata:
     >   name: myapp-pod
     > spec:
     >   containers:
     >   - name: myapp-container
     >     image: busybox
     >     resources:
     >       requests:
     >         requests.nvidia.com/gpu: 2
     >       limits:
     >         limits.nvidia.com/gpu: 2
     > ~~~
     >
     > 这个pod通过`requests.nvidia.com/gpu: 2`这样的方式声明使用了两个NVIDIA类型的GPU
     >
     > 但是在PodFitsResource种，调度器并不知道这个字段Key的含义是GPU，而是==直接使用后面的value进行计算==；当然，在Node的Capacity字段里也要加上这台宿主机上GPU的总数。==这是Device Plugin==的内容。

   * PodFitsHost检查宿主机的名字是否与Pod的Spec.nodeName字段一致（**因为存在用户手动指定spec.nodeName的情况**）

   * PodFitsHostPort见擦汗Pod申请的宿主机端口`spec.nodePort`是否与已被使用的端口有冲突

   * PodMatchNodeSelector检查Pod的nodeSelector或者nodeAffinity指定的节点与当前节点是否匹配等等。

2. **与Volume相关的过滤规则**：负责跟容器PV相关的调度策略

   * **NoDiskConflict检查多个Pod声明挂载的PV是否有冲突**

     > 比如AWS、EBS类型的Volume不允许被两个Pod同时使用，所以当一个名叫A的EBS volume已经挂载在某个节点上时，另一个同样声明使用这个A volume的Pod就不能调度到这个节点上了

   * MaxPDVolumeCountPredicate检查一个节点上某个类型的PV是否超过了一定数目，若是，则声明使用该类型PV的Pod不能再调度到该节点上

   * VolumeZonePredicate检查PV的Zone（高可用域）标签是否与当前节点的Zone标签匹配

   * VolumeBindingPredicates规则检查该Pod对应的PV的nodeAffinity字段是否跟某个节点的标签相匹配

     > 之前的内容说明，**LocalPV必须使用nodaAffinity来跟某个具体的节点绑定，这也就说明，在Predicate节点，Kubernetes必须能够根据Pod的Volume属性进行调度**
     >
     > 此外，**如果该Pod的PVC还没跟具体的PV绑定，调度器还负责检查所有待绑定的PV，当有可用的PV存在并该PV的nodeAffinity与当前节点一致时**，才返回成功，例如：
     >
     > ~~~yaml
     > apiVersion: v1
     > kind: PersistentVolume
     > metadata:
     >   name: my-pv
     > spec:
     >   capacity:
     >     storage: 1Gi
     >   accessModes:
     >     - ReadWriteOnce
     >   PersistentVolumeReclaimPolicy: Retain
     >   storageClassName: my-storage-class
     >   local:
     >     path: /mnt/disks/vol1
     >   nodeAffinity:
     >     required:
     >       nodeSelectorTerms:
     >       - matchExpressions:
     >         - key: kubernetes.io/hostname
     >           operator: In
     >           values:
     >           - my-node
     > ~~~
     >
     > ==该PV对应的持久化目录只会出现在名叫my-node的节点上。所以任何一个通过PVC使用这个PV的Pod，必须被调度到my-node上才可以正常工作==，VolumeBinding-Predicate就是在调度其中完成该决策的位置。

3. **宿主机相关过滤规则**：主要考察Pod是否满足节点本身的某些条件

   * PodToleratesNodeTaints检查节点污染机制，只有Pod的Toleration字段与Node的Taint字段匹配时，这个Pod才可以调度到这个节点上

   * NodeMemoryPressurePredicate检查当前节点内存是否不够，不够则不能调度

     > 这个检查与GeneralPredicates中关于内存的检查的区别在于？
     >
     > 1. **检查的角度和目的不同**：
     >    - `NodeMemoryPressurePredicate` 主要是从节点的压力角度来考虑的。它关注的是节点是否处于内存压力状态（即节点的内存使用率已经很高，继续在该节点上分配更多的内存可能会导致系统性能下降或者其他问题）。如果一个节点处于内存压力之下，Kubernetes 调度器会避免将新的 Pod 调度到这个节点上，以防止进一步加重内存压力。
     >    - `GeneralPredicates` 包括一系列基本的谓词检查，其中关于内存的检查主要是验证节点是否有足够的可分配资源（包括内存）来满足新 Pod 的需求。这是从资源可用性的角度进行的直接比较。
     > 2. **作用范围不同**：
     >    - `NodeMemoryPressurePredicate` 更侧重于节点的整体健康状况和稳定性，特别是当节点因为内存使用过高而可能影响其上运行的所有容器的性能时。
     >    - `GeneralPredicates` 中的内存检查则更加具体地针对即将被调度的 Pod 所需的资源量与节点剩余可用资源之间的匹配程度。
     > 3. **触发条件不同**：
     >    - `NodeMemoryPressurePredicate` 通常在节点报告了内存压力信号时触发，这可能由 kubelet 根据节点的内存使用情况自动上报。
     >    - `GeneralPredicates` 中的内存检查是在每次尝试为 Pod 寻找合适的节点时都会执行的标准流程的一部分。

4. **Pod相关过滤规则**：大多与GeneralPredicates重合

   * **PodAffinityPredicate检查待调度Pod与节点上已有Pod之间的亲密和反亲密关系**

     > 例如：
     >
     > ~~~yaml
     > apiVersion: v1
     > kind: Pod
     > metadata:
     >   name: with-pod-anti-affinity
     > spec:
     >   affinity:
     >     podAntiAffinity:
     >       requiredDuringSchedulingIgnoredDuringExecution:
     >       - weight: 100
     >         podAntiAffinityTerm:
     >           labelSelector:
     >             matchExpressions:
     >             - key: security
     >               operator: In
     >               values:
     >               - S2
     >             topologyKey: "kubernetes.io/hostname"
     >   containers:
     >   - name: nginx
     >     image: nginx:1.14.2
     > ~~~
     >
     > ==这个例子中的podAntiAffinity规则指定该Pod不希望与任何具有security=S2标签的Pod存在于同一个节点上。==
     >
     > PodAntiAffinityPredicate是有作用域的，==上述这个例子的规则，仅对具有KEY是`kubernetes.io/hostname`标签的节点有效，这也是topologyKey的作用==

   * podAffinity于PodAntiAffinity相反

     > 例如：
     >
     > ~~~yaml
     > apiVersion: v1
     > kind: Pod
     > metadata:
     >   name: with-pod-affinity
     > spec:
     >   affinity:
     >     podAffinity:
     >       requiredDuringSchedulingIgnoredDuringExecution:
     >       - labelSelector:
     >           matchExpressions:
     >           - key: security
     >             operator: In
     >             values:
     >             - S1
     >         topologyKey: failure-domain.beta.kubernetes.io/zone
     >   containers:
     >   - name: nginx
     >     image: nginx:1.14.2
     > ~~~
     >
     > 这个例子中的pod只会被调度到已经有携带security=S1标签的Pod运行的节点上。作用域是所有携带Key是`failure-domain.beta.kubernetes.io/zone`标签的节点

   上述两个例子中的`requiredDuringSchedulingIgnoredDuringExecution`字段的含义是，==这条规则必须在Pod调度时进行检查，但是如果是已经在运行的Pod发生变化，使得该Pod不再适合在该节点上运行时，Kubernetes不进行主动修正==

5. **关于预选过程**

   在具体执行时，当开始调度一个Pod，kubernetes调度器会同时启动16个Goroutine，并发地为集群中的所有节点计算预选策略，最后返回可以运行这个Pod的宿主机列表。

   在为每个节点执行预选时，调度器会按照固定的顺序进行检查，顺序是按照预选的含义来确定的。例如，宿主机相关的预选会优先进行检查。否则，在一台资源已经严重不足的宿主机上，一开始就执行PodAffinityPredicates是没有意义的。

   > * 关于为什么是16个Goroutine
   >
   >   **这个Goroutine的数量是固定的，16个是经验值，能够充分利用多核CPU的并发能力，同时避免多线程竞争和上下文切换的开销。**

### Priority优选策略

* 优选策略最常使用的打分规则就是LeastRequestedPriority，其计算方法简单总结为如下公式

  $score = (cpu((capacity-sum(requested))\times 10/capacity) \quad+\quad memory((capacity-sum(requested))\times 10/capacity))/2 $

  可以看到，**算法实际上就是选择空闲资源最多的宿主机**

* BalanceResourceAllocation于LeastRequestedPriority一起发挥作用，其计算公式如下：

  $score = 10 - variance(cpuFraction,memoryFraction,volumeFraction)*10$

  其中每种资源的Fraction的定义是，`Pod请求的资源/节点上可用的资源`。**Variance算法的作用则是计算每两种资源Fraction之间的距离，最终选择资源Fraction差距最小的节点**

  **所以BalanceResourceAllocation选择的是调度完成后，所有节点里各种资源分配最均衡的节点**

* 其他优选策略：NodeAffinityPriority、TaintTolerationPriority、InterPodAffinityPriority

  这三种策略与之前的三种PodMatchNodeSelector、PodToleratesNodeTaints、PodAffinityPredicate三种预选策略的含义与计算方法类似

  **但是作为优选策略，一个节点满足上述规则的字段数目越多，其得分越高**

### 关于ImageLocalityPriority策略

> 这也是默认优选策略

* 这是在Kubernetes v1.12里开启的新调度规则

  如果待调度的Pod需要使用很大的镜像，并且已经存在于某些节点上，那么这些节点的得分就会较高

* 为了避免该算法引起调度堆叠，调度器在计算得分时，会根据镜像的分布进行优化

  即如果大镜像分布的节点数目很少，那么这些节点的权重就会被降低，从而“对冲”引起调度堆叠的风险。

  > 1. 关于调度堆叠：
  >
  >    * **调度堆叠** 指的是 Kubernetes 调度器在分配 Pod 时，由于某些条件限制（如资源需求、镜像分布、节点亲和性等），导致大量 Pod 被集中调度到 **少数节点** 上，从而引发资源竞争或性能瓶颈的现象。
  >
  >    * **典型场景**：
  >
  >      - **镜像分布不均**：如果某个大镜像（例如 5GB 的数据库镜像）只存在于集群中的少数节点上，而多个 Pod 需要使用该镜像，调度器会优先将这些 Pod 调度到镜像所在的节点。
  >
  >      - **结果**：这些节点可能因负载过高（如 CPU、内存、网络 I/O）而成为瓶颈，甚至导致节点崩溃。
  >
  > 2. 大镜像分布的节点数目少，为什么会导致调度堆叠？
  >
  >    * **问题根源：**
  >      镜像拉取依赖：Pod 启动前需要从镜像仓库拉取所需的镜像。**如果镜像未提前缓存到节点上，Pod 会被调度到有镜像的节点（或等待镜像拉取完成）**。
  >      节点镜像缓存差异：**如果某个大镜像只存在于少数节点上（例如 A、B 节点），而其他节点没有缓存，则调度器会优先将 Pod 调度到 A、B 节点。**
  >    * **后果：**
  >      资源集中：A、B 节点的负载迅速增加，可能超出其资源容量（CPU、内存、磁盘 I/O）。
  >      性能瓶颈：镜像拉取和容器启动时间变长，导致整体调度效率下降。
  >      故障风险：如果 A、B 节点发生故障，所有依赖该镜像的 Pod 都会受到影响。
  >
  > 3.  **权重调低如何“对冲”调度堆叠风险？**
  >
  >    调度器通过动态调整节点的 **权重（weight）** 来平衡负载，避免资源集中在少数节点上。
  >
  >    *  **具体机制**：
  >      1. **权重计算**：
  >         - 调度器会评估每个节点的资源利用率（CPU、内存）、镜像分布情况、亲和性策略等。
  >         - 如果某个节点的镜像分布较少（如大镜像仅存在于少数节点），调度器会降低这些节点的权重。
  >      2. **动态调整**：
  >         - **降低权重**：调度器会减少这些节点被选中的概率，从而分散 Pod 到更多节点。
  >         - **示例**：假设 A 节点是唯一有大镜像的节点，调度器会逐步降低其权重，引导部分 Pod 调度到其他节点（即使需要重新拉取镜像）。
  >      3. **“对冲”风险**：
  >         - **避免资源集中**：通过权重调整，减少对少数节点的依赖，降低节点过载或故障的影响。
  >         - **平衡负载**：即使需要重新拉取镜像，也能通过分散负载避免单点故障。
  >
  > 4. **实际案例**
  >
  >    假设一个 Kubernetes 集群有 10 个节点，其中只有 2 个节点（Node-A 和 Node-B）缓存了大镜像 `my-big-image:1.0`。此时有 10 个 Pod 需要使用该镜像：
  >
  >    * **默认行为**：
  >
  >      - 调度器会优先将 10 个 Pod 调度到 Node-A 和 Node-B。
  >
  >      - 结果：Node-A 和 Node-B 的 CPU/内存被耗尽，Pod 可能因资源不足被驱逐。
  >
  >    * **权重调低后的行为**：
  >
  >      - 调度器检测到 Node-A 和 Node-B 的镜像分布较少，降低它们的权重。
  >
  >      - 部分 Pod 被调度到其他节点，即使需要重新拉取镜像。
  >
  >      - 结果：负载分散到更多节点，Node-A 和 Node-B 的压力减少，整体稳定性提高。



## 8.4 Kubernetes默认调度器的优先级和抢占机制

### 优先级机制

* 优先级和抢占机制解决的是Pod调度失败的问题。**正常情况下，当一个Pod调度失败后，其会被暂时搁置，直到Pod被更新或者集群的状态发生变化，调度器才会对这个Pod进行重新调度。**

  ==优先级和抢占机制，使得Pod调度失败后，不会被搁置，而是挤走某个节点上一些低优先级的Pod，以此保证这个高优先级的Pod调度成功。==

  **优先级和抢占机制需要使用PriorityClass**

  > * 定义：
  >
  >   ~~~yaml
  >   apiVersion: scheduling.k8s.io/v1
  >   kind: PriorityClass
  >   metadata:
  >     name: high-priority
  >   value: 1000000
  >   globalDefault: false
  >   description: "This priority class should be used for high-priority tasks."
  >   ~~~
  >
  >   这个yaml定义了一个名叫high-priority的PriorityClass，其中value是1000000
  >
  > 1. 优先级是一个32bit整数，最大值不超过1 000 000 000，并且值越大，优先级越高。
  >
  >    **超出10亿的值，是Kubernetes分配给系统Pod使用的，旨在避免系统Pod被用户抢占。**
  >
  > 2. ==yaml文件中的globalDefault被设置为true，意味着这个PriorityClass的值会成为系统的默认值，如果是false，表示只希望声明使用该PriorityClass的Pod拥有值为100000的优先级==
  >
  >    对于没有声明Priorityclass的Pod来说，其优先级为0
  >
  > * 使用
  >
  >   ~~~yaml
  >   apiVersion: v1
  >   kind: Pod
  >   metadata:
  >     name: high-priority-pod
  >     labels:
  >       app: high-priority-app
  >   spec:
  >     priorityClassName: high-priority
  >     containers:
  >     - name: high-priority-container
  >       image: busybox
  >   ~~~
  >
  >   当这个Pod被提交给Kubernetes后，PriorityAdmissionController会自动将这个Pod的spec.priority字段设置为100000
  >
  >   PriorityClassName的使用没有同命名空间要求；
  >
  >   **PriorityClass是要给集群作用域资源，Pod引用PriorityClass不需要同命名空间；如果创建了RBAC，确保Pod创建者有权限读取集群范围内的PriorityClass资源**

* 优先级的体现：

  当Pod拥有优先级后，高优先级的Pod可能比低优先级的Pod提前出队，从而尽早完成调度过程。

### 抢占机制

* 抢占的体现

  当高优先级的Pod调度失败时，调度器的抢占能力就会触发。调度器会试图从当前集群中找一个节点：

  **当该节点上的一个或多个低优先级Pod被删除后，待调度的高优先级Pod可以被调度到该节点上。**

* **抢占过程概述**：称高优先级Pod为抢占者（preemptor）

  1. 抢占过程发生时，抢占者并不会立刻被调度到被抢占节点上。

     调度器只会将抢占者的spec.nominatedNodeName字段设置为被抢占节点的名字，然后抢占者会重新进入下一个调度周期，在新的调度周期里决定是否要在被抢占的节点上运行。
  
     > ==由于调度器只会通过标准的DELETE API来删除被抢占的Pod，所以这些Pod必然有一定的退出时间，而在此期间，其他节点也有可能变成可调度的，或者有新节点加入集群，所以集群的可调度性可能会发生变化==，因此将抢占者交给下一个调度周期是合理的。
  
  2. 抢占者等待被调度的过程中，若有其他优先级更高的Pod要抢占同一个节点，调度器就会清空原有抢占者的spec.nominatedNodeName字段，从而允许优先级更高的抢占者抢占节点。
  
     这也使得原抢占者有机会重新抢占其他节点。
  
     > **不会出现抢占者要抢占的节点被另一个低优先级的Pod抢占的情况**
     >
     > 因为抢占者进入activeQ后，由于优先级机制，会被调度器优先调度。
  
* **抢占机制的实现：**调度队列的实现中，使用了两个不同的队列

  1. activeQ：凡是activeQ中的Pod，都是下一个调度周期需要调度的对象。

     当在Kubernetes集群中新建一个Pod时，调度器会将该Pod入队到activeQ中。调度器不断从队列中出队Pod进行调度也是从activeQ中出队的。

  2. unschedulableQ：专门用于存放调度失败的Pod。

     当一个unschedulableQ中的Pod更新后，调度器会自动将这个Pod移动到activeQ中。

* **抢占过程详细阐述：**

  1. 调度失败后，抢占者进入unschedulableQ，触发调度器为抢占者寻找牺牲者流程。

  2. 调度器检查事件的失败原因，以确认是否能帮助抢占者找到一个新节点。

     > **因为很多Predicates的失败不是能通过抢占来解决的**。例如PodFitHost算法，除非节点的名字发生变化，否则删除再多的Pod，抢占者也不可能调度成功。

  3. **如果确定可以抢占，调度器会把自己缓存的所有节点信息复制一份，然后使用这个副本模拟抢占过程。**

     调度器会检查缓存副本中的每个节点，然后从一个节点上优先级最低的Pod开始，逐一删除这些Pod。每删除一个低优先级Pod，调度器都会检查抢占者是否能在该节点上运行。

     如果能够运行，调度器就记录下该节点的名字和被删除Pod的列表。

  4. 当遍历完所有节点后，调度器从上述模拟产生的所有抢占结果中选出最佳结果，判断原则就是**尽量减少抢占对整个系统的影响。比如需要抢占的Pod越少越好，需要抢占的Pod优先级越低越好等。**

  5. 得到最佳抢占结果后，调度器开始操作

     1. 调度器见擦汗牺牲者列表，清理这些Pod所携带的nominatedNodeName字段

        > * 问：这里提到的nominatedNodeName字段，是每个Pod被调度成功后都会设置的字段吗？不然为什么要清理这些被删除Pod的该字段呢？
        >
        > * 答：**nominatedNodeName字段并不是所有Pod被调度成功后自动写入的，仅在涉及抢占时才会被设置。**
        >
        >   当高优先级Pod抢占节点后，调度器会清空牺牲者Pod的nominatedNodeName字段（如果存在），==是为了防止牺牲者Pod在重新调度时错误地尝试抢占已经被提名的节点。==

     2. 调度器将抢占者的nominatedNodeName设置为被抢占者节点的名字

     3. 调度器遍历牺牲者列表，向API Server发起请求，逐一删除牺牲者

     4. 对抢占者Pod的更新操作会触发让抢占者Pod进入activeQ，从而在下一个调度周期重新调度的流程。

  > **在为某一对Pod和节点执行Predicates算法时，如果待检查节点是即将被抢占的节点，即调度队列中存在nominatedNodeName字段值是该节点的Pod，那么调度器就会对该节点运行两次同样的Predicate算法。**
  >
  > 1. 第一遍，调度器假设上述潜在抢占者已经在该节点上运行，然后执行预选算法
  >
  >    第一遍是因为InterPodAntiAffinity规则，调度器必须考虑抢占者如果存在于该节点上，待调度Pod是否能调度成功。
  >
  >    **这一步，只用考虑优先级高于待调度Pod的高优先级抢占者Pod，对于其他较低优先级的抢占者Pod，待调度Pod可以通过抢占机制在别的节点上运行。**
  >
  > 2. 第二遍，调度器正常执行预选算法，不考虑任何潜在抢占者
  >
  >    第二遍是因为潜在抢占者不一定会在该节点上运行。



## 8.5 Kubernetes GPU管理与Device Plugin机制

* 对于云用户在GPU的支持上，其最基本诉求为：

  在Pod的yaml中声明某容器需要的GPU个数，Kubernetes为其创建的容器中就应该出现对应的GPU设备即驱动目录。

  **以NVIDIA的GPU为例，上述需求意味着当用户容器被创建后，容器中必须出现如下设备和目录：**

  1. GPU设备，例如/dev/nvidia0
  2. GPU驱动目录，比如/usr/local/nvidia/*

  GPU设备路径正是容器启动时的Device参数，驱动目录是容器启动时的Volume参数。

  **在Kubernetes的GPU实现中，kubelet实际上就是将上述两部分内容设置在了创建容器的CRI参数中。**

* **Kubernetes中没有API对象为GPU专门设置一个资源类型字段，而是使用Extended Resource来传递GPU信息**

  ~~~yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: cuda-vector-add
  spec:
    restartPolicy: OnFailure
    containers:
    - name: cuda-vector-add
      image: k8s.gcr.io/cuda-vector-add:v0.1
      resources:
        limits:
          nvidia.com/gpu: 1 # requesting 1 GPU
  ~~~

  上述Pod的limits字段中，这个资源的名称是`nvidia.com/gpu`，其值是1，说明这个Pod声明了使用一个NVIDIA类型GPU；

  **但是kube-scheduler中，并不关心字段具体含义，而是将调度器中保存的该类型资源的可用量直接减去Pod声明数值，Extended Resource是Kubernetes的自定义资源支持**

* **为了让调度器直到这个自定义类型资源在每台宿主机上的可用量，宿主机节点必须能够向API Server汇报该类型资源的可用量。**

  Kubernetes中各种资源的可用量是Node对象Status字段内容

  ~~~bash
  ➜  ~ kubectl describe node master
  Name:               master
  Roles:              control-plane
  Labels:             beta.kubernetes.io/arch=amd64
                      beta.kubernetes.io/os=linux
  .......
  Addresses:
    InternalIP:  172.23.192.199
    Hostname:    master
  Capacity:
    cpu:                8
    ephemeral-storage:  40901312Ki
    hugepages-1Gi:      0
    hugepages-2Mi:      0
    memory:             15980972Ki
    pods:               110
  Allocatable:
    cpu:                8
    ephemeral-storage:  37694649077
    hugepages-1Gi:      0
    hugepages-2Mi:      0
    memory:             15878572Ki
    pods:               110
    ........
  ~~~

  使用PATCH API在上述Capacity字段中添加自定义资源数据并更新这个Node，如下是使用curl简单操作PATCH操作

  ~~~bash
  # 启动Kubernetes的客户端proxy，这样就可以使用curl来跟API Server进行交互
  $ kubectl proxy
  # 执行PATCH操作
  $ curl --header "Contend-Type: application/json-patch+json" \
  --request PATCH \
  --data '[{"op": "add", "path": "/status/capacity/nvidia.com/gpu", "value": "1"}]' \
  http://localhost:8001/api/v1/nodes/<your-node-name>/status
  ~~~

  PATCH操作完成后，Node是Capacity变化
  <img src="https://typora2icture.oss-cn-beijing.aliyuncs.com/img2/20250512171021.png" alt="image.png" style="zoom: 80%;" />
  
  **但是在Kubernetes的GPU支持方案中，用户不需要做上述Extended Resource这些操作，对所有硬件加速设备进行管理的功能，由一种叫做Device Plugin的插件负责，其中包括对该硬件的Extended Resource进行汇报的逻辑。**

### Device Plugin机制
<img src="https://typora2icture.oss-cn-beijing.aliyuncs.com/img2/20250512171332.png" alt="image.png" style="zoom:80%;" />

* 每种硬件设备都需要由其对应的Device Plugin进行管理

  这些Device Plugin都通过gRPC方式同kubelet连接；Device Plugin会通过ListAndWatch的API，定期向kubelet汇报该节点上GPU的列表（图中例子就有GPU0、1、2三个GPU）。

  kubelet在拿到列表后，可以直接在向API Server发送的心跳中，以Extended Resource方式加上这些GPU的数量，例如`nvidia.com/gpu=3`

  > **ListAndWatch向上汇报的信息，只有本机上GPU的ID列表，没有关于GPU设备本身的信息。**
  >
  > Kubernetes中对硬件设备的管理，只能处理“设备个数”这种清空，一旦设备是异构的，不能简单的使用数目描述具体的使用需求时，Device Plugin就不能处理了
  >
  > 同时，如果硬件设备属性比较复杂，Pod也关心硬件属性的话，Device Plugin也是无法支持的。
  >
  > ==其实说白了就是无法具体描述其他设备的属性信息，只能够抽象描述GPU为例的个数的信息就是，如果集群中，一个节点是AMD的GPU，一个是NVIDIA的GPU，都是使用GPU数量来描述的不能具体分辨型号==
  >
  > **Kubelet向API Server汇报时，只会汇报该GPU对应的Extended Resource的数量；但是Kubelet本身会将这个GPU的ID列表保存在自己的内存中，并通过ListAndWatch定时更新。**

* 当一个Kubelet发现这个Pod容器请求一个GPU时，kubelet从自己持有的GPU列表中，为该容器分配一个GPU

  1. **此时，kubelet就会向本机的Device Plugin发起一个Allocate()请求，携带的参数就是即将分配给这个容器的设备ID列表。**

  2. **当Device Plugin接收到Allocate请求后，其会根据kubelet传递的设备ID列表从Device Plugin中找到这些设备对应的设备路径和驱动目录。**

     > 设备路径、驱动目录都是Device Plugin定期从本机查询到的

  3. 被分配GPU对应的设备路径和驱动目录信息返回给kubelet后，kubelet就完成了为一个容器分配GPU的工作

     **kubelet会把这些信息追加到创建该容器对应的CRI请求中，当这个CRI请求发送给Docker后，Docker创建出来的容器就会出现这个GPU设备。**

* 其他类型的硬件，也要遵循Device Plugin的流程来实现Allocate和ListAndWatch API

  ~~~go
  service DevicePlugin {
      rpc ListAndWatch(Empty) returns (stream ListAndWatchResponse) {}
      rpc Allocate(AllocateRequest) returns (AllocateResponse) {}
  }
  ~~~




# 第9章 容器运行时

## 9.1 SIG-Node与CRI

> 在Kubernetes中，与kubelet和容器运行时管理相关的内容都属于SIG-Node的范畴
>
> 是K8s这样容器编排与管理系统跟容器打交道的主要场所

* Kubelet本身也是按照控制器模式工作的，根据下图，kubelet的工作核心是一个控制循环`SyncLoop`，驱动该控制循环的运行事件包括4种

  1. Pod更新事件
  2. Pod生命周期发生变化
  3. kubelet本身设置的执行周期
  4. 定时的清理事件

  <img src="https://typora2icture.oss-cn-beijing.aliyuncs.com/img2/20250522120438.png" alt="image.png" style="zoom: 67%;" />
  
  1. kubelet启动时首先设置Listers，注册其关心的各种事件的Informer，这些Informer是SyncLoop需要处理的数据来源
  
  2. kubelet还维护很多其他子控制循环，例如Volume Manager、Image Manager、Node Status Manager
  
     这些控制循环就是通过控制器模式完成kubelet的某项具体任务
  
     * Node Status Manager负责响应节点状态变化、收集节点状态信息，上报给API Server
     * CPU Manager维护节点的CPU核信息，方便Pod通过cpuset请求CPU核时，能正确管理
  
     > * 能否针对CPU开发此类Manager？
     >
     >   在Kubernetes中，kubelet确实负责许多子系统控制循环的运行，例如Volume Manager用于管理存储卷的挂载和卸载，CPU Manager用于提供更细粒度的CPU资源分配策略。对于GPU资源，Kubernetes已经提供了相应的支持机制，虽然它不是以“GPU Manager”这样的名字直接存在，但功能上实现了类似的目标。
     >
     >   Kubernetes通过设备插件（Device Plugins）框架支持对GPU等硬件加速器的管理。NVIDIA GPU是目前最常见的一种被Kubernetes集群使用的加速器类型。为了利用这些设备，集群管理员通常会部署NVIDIA设备插件，这是一个实现Kubernetes设备插件API的DaemonSet，它允许节点向kubelet报告其可用的NVIDIA GPU资源。然后，kubelet将这些资源公开给Kubernetes调度器，使用户能够请求这些资源用于他们的Pods。
     >
     >   此外，Kubernetes还支持其他类型的扩展来增强对特定硬件的支持，比如可以通过准入控制器（Admission Controllers）、自定义调度器或Operator模式来实现更加复杂精细的资源管理和调度策略。
     >
     >   所以，虽然没有明确称为“GPU Manager”的组件，但通过Kubernetes现有的设计和扩展机制，完全可以针对GPU开发出符合需求的管理功能。这包括但不限于使用设备插件暴露GPU资源、使用拓扑管理器优化容器内设备的分配、或者通过自定义调度器满足特定的调度需求。
  
  3. kubelet也是通过Watch机制，监听与自己相关的Pod对象的变化，过滤条件是该Pod的nodeName字段与自己的相同，并且会把相关的Pod信息存在内存中
  
     当一个Pod完成调度与一个节点绑定后，Pod的变化会触发kubelet在控制循环中注册的Handler，即图中的HandlePods部分
  
     **kubelet通过检查内存中的状态，判断这个Pod是新调度Pod，从而触发Handler中的ADD事件处理逻辑**，具体来说，kubelet启动叫做PodUpdate Worker的Goroutine完成对Pod的处理工作
  
     > 1. 如果是ADD事件，kubelet为这个新的Pod生成对应的Pod Status，检查Pod需要的 Volume是否准备好，调用下层的Docker创建这个Pod定义的容器
     > 2. 如果是是Update事件，kubelet根据Pod对象具体变更的情况，调用Docker对容器进行重建
     >
     > ==但是kubelet调用下层的容器运行时不会直接调用Docker的API，而是通过CRI的gRPC接口间接执行==，这是为了屏蔽下层容器运行时的差异，但是1.6版本之前的k8s是直接调用Docker的API的

* **关于虚拟化容器、Linux容器与基于虚拟化技术的强隔离容器**
  
  * **虚拟化容器和Linux容器很多情况下指代相同的技术**，即基于操作系统级别的虚拟化技术创建的隔离环境
  
  * 基于虚拟化技术的强隔离容器与标准化容器是不一样的，**主要体现在隔离性和实现机制**
  
    1. 标准容器通过namespace和cgroup提供隔离，所有容器共享宿主机内核
  
       强隔离容器采用包括硬件辅助的虚拟化技术、Kata Containers、gVisor这样的解决方案为每个容器提供虚拟机级别的隔离
  
    2. **强隔离容器拥有与自己的内核实例**，减少容器间、容器对宿主机的威胁
  
    3. 相比标准容器，强隔离容器有更高的资源消耗和较慢的启动时间

### CRI

> 前情是k8s具有多样的底层容器运行时的实现，但是kubelet维护不同容器运行时的调用API是很复杂的，因此产生了CRI

**kubelet将对容器的操作统一抽象为接口，具体的容器项目需要提供该接口的实现，然后对kubelet暴露gRPC服务**

* 如下是具有CRI接口的k8s与kubelet架构图

  <img src="https://typora2icture.oss-cn-beijing.aliyuncs.com/img2/20250526112702.png" alt="image.png" style="zoom: 67%;" />

  在接收到新来的Pod后，kubelet会通过SyncLoop判断需要执行的具体操作，然后调用GenericRuntime的组件，发起Pod的CRI请求。

  CRI请求的响应，根据使用的容器运行时不同而不同，docker的是dockershim组件，其将CRI请求中的内容取出，组装成Docker API发给Docker Daemon

## 9.2 解读CRI和容器运行时

* CRI机制的核心在于每个容器项目都可以实现一个自己的CRI shim，自行对CRI请求进行处理

  **除了dockershim，其他的容器运行时的CRI shim都要额外部署在宿主机上**，以containerd为例
  <img src="https://typora2icture.oss-cn-beijing.aliyuncs.com/img2/20250526115128.png" alt="image.png" style="zoom:67%;" />
  
  **containerd接收来自kubelet的CRI 请求，并将这些CRI 请求转换为对底层容器运行时runC的调用，而runC才是真正的底层容器运行时**
  
  **但是containerd也不是直接调用runC，而是通过containerd-shim中间层来管理容器生命周期，shim的作用是解耦containerd和容器进程，确保containerd重启时容器不受影响，runC仅作为创建容器的工具，由shim调用**
  
  containerd中非containerd-shim的部分的作用：
  
  1. 提供gRPC API与CRI实现
  2. 镜像管理
  3. 存储管理
  4. 容器运行时管理
  5. 元数据存储
  6. 事件与监控
  
* CRI可以分为两组

  1. **RuntimeService**：提供容器相关操作，比如创建、启动、删除容器、执行exec命令

     CRI中有一组叫做RunPodSandbox的接口，对应的不是k8s中的Pod对象，只是抽取了Pod中的一部分与容器运行时相关的字段，例如Hostname、DnsConfig、CgroupParent等

     作为具体的容器项目，要自己决定如何使用这些字段实现一个k8s期望的Pod模型
     <img src="https://typora2icture.oss-cn-beijing.aliyuncs.com/img2/20250526143057.png" alt="image.png" style="zoom:67%;" />

     例如上图中，创建一个包含A、B两个容器的Pod，Pod的信息到达kubelet之后，kubelet就会按顺序调用CRI接口

     具体的CRI shim中，接口的实现完全不同，如果是Dockers项目，docker shim会创建出名为foo的`Infra（pause）`容器用于hold整个Pod的NetworkNamespace
  
     在RunPodSandbox这个接口的实现中，还要调用networkPlugin.SetUpPod()来为这个Sandbox设置网络，而SetUpPod实际上就是在执行CNI插件中的add方法，为Pod创建网络，并把Infra容器加入网络中。
  
     
  
     CRI shim还要实现exec、logs等接口，这两个接口与前面的区别在于，exec、logs调用期间，kubelet需要跟容器项目维护一个长连接来传输数据，这种API称为 Streaming API
     
     <img src="https://typora2icture.oss-cn-beijing.aliyuncs.com/img2/20250526151746.png" alt="image.png" style="zoom:67%;" />
     
     当对一个容器执行exec指令，这个请求会先较为API Server，然后API Server会调用kubelet 的Exec API，然后kubelet调用CRI 的Exec接口，负责响应这个接口的就是CRI shim
     
     但是CRI shim不会直接调用后端容器运行时处理，会返回一个URL给kubelet，这个URL就是Streaming Server的地址和端口。kubelet获取到这个URL后，将其以Redirect方式返回给API Server，然后API Server通过重定向向Streaming Server发起exec请求，并与其建立长连接。
     
     这个Streaming Server本身需要使用SIG-Node维护的Streaming API库来实现，并且Streaming Server会与CRI shim同时启动；
     
     除此以外，Streaming Server的具体实现由CRIshim的维护者决定。
  
  
  
  
  2. **ImageService**：提供容器镜像相关操作，比如拉取、删除镜像，较为简单，不再讲解





# 第10章 Kubernetes监控与日志

