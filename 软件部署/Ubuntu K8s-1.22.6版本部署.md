---
tags:
  - Kubernets
  - k8s
  - docker
  - kubeedge1-10-1
---
[[搭建Kubernetes集群ECS]]
# Ubuntu K8s-1.22.6版本部署

> docker版本：docker-ce: 20.10.13
>
> kubernetes版本：Kubernetes: 1.22.6
>
> kubeedge版本：kubeedge：v1.10.0

## 安装docker

> 三个节点的docker版本要一致

* 安装依赖

  ~~~bash
  $ sudo apt-get update
  
  $ sudo apt-get install \
      apt-transport-https \
      ca-certificates \
      curl \
      gnupg-agent \
      software-properties-common
  ~~~

* 添加 Docker 官方的 GPG 密钥（为了确认所下载软件包的合法性，需要添加软件源的 GPG 密钥）

  ~~~bash
  （官方）$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
  （国内）$ curl -fsSL https://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo apt-key add -
  ~~~

  **注意：可以两个都添加，也可以只添加一个，但是要和下面的apt仓库地址对应**

* 设置稳定版本的apt仓库地址

  ~~~bash
  （官方）
  $ sudo add-apt-repository \
     "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
     $(lsb_release -cs) \
     stable"
     
  （国内）
  $ sudo add-apt-repository \
       "deb [arch=amd64] https://mirrors.aliyun.com/docker-ce/linux/ubuntu \
       $(lsb_release -cs) \
       stable"
  ~~~

  **注意：可以两个都添加，也可以只添加一个，但是要和上面的Docker官方的GPG密钥对应**

* 刷新源

  ~~~bash
  $ sudo apt-get update
  ~~~

* 查看可安装的docker版本

  ~~~bash
  $ sudo apt-cache madison docker-ce
  ~~~

* 安装docker-ce: 20.10.13

  ~~~bash
  （最新版本）$ sudo apt-get install docker-ce docker-ce-cli containerd.io
  （指定版本）$ sudo apt-get install docker-ce=5:20.10.13~3-0~ubuntu-bionic docker-ce-cli=5:20.10.13~3-0~ubuntu-bionic containerd.io
  ~~~

  **注意：指定版本中“=”后面的版本信息要根据apt-cache madison docker-ce输出的版本信息对应一致**

* 验证

  ~~~bash
  $ sudo docker version
  ~~~

* 配置docker镜像加速仓库

  ~~~bash
  $ sudo mkdir /etc/docker
  $ vim /etc/docker/daemon.json
  ~~~

  在daemon.json文件中输入以下内容(`以下镜像加速地址截至2024.11.11有效`)

  ~~~json
  {
      "exec-opts":["native.cgroupdriver=systemd"],
      "registry-mirrors": [
          "http://hub-mirror.c.163.com",
          "https://mirror.baidubce.com",
          "https://docker.mirrors.ustc.edu.cn",
          "https://registry.docker-cn.com",
          "https://dockerproxy.com",
          "https://ccr.ccs.tencentyun.com",
          "https://registry.cn-hangzhou.aliyuncs.com"
      ]
  }
  ~~~

  让配置生效重启docker，并让docker自启动

  ~~~bash
  $ systemctl enable docker && systemctl restart docker
  ~~~

## 安装Kubernets

### 每个节点都要有的操作

* 关闭防火墙和selinux

  ~~~bash
  systemctl stop firewalld && systemctl disable firewalld && sed -i 's/SELINUX=permissive/SELINUX=disabled/' /etc/sysconfig/selinux
  ~~~

* 关闭交换分区

  ~~~bash
  sed -i 's/.*swap.*/#&/' /etc/fstab
  ~~~

* 开启路由转发

  ~~~bash
  vim /etc/sysctl.conf
  #添加如下内容：
  net.ipv4.ip_forward = 1
  net.bridge.bridge-nf-call-ip6tables = 1
  net.bridge.bridge-nf-call-iptables = 1
  #使配置生效
  sysctl -p
  ~~~

* 设置本地地址解析（根据自己的集群地址和主机名设置）

  ~~~bash
  vim /etc/hosts
  192.168.23.100 master
  192.168.23.101 node1
  192.168.23.102 node2   
  ~~~

* 为k8s安装基础环境

  ~~~bash
  # 安装基础环境
  apt-get install -y ca-certificates curl software-properties-common apt-transport-https curl
  curl -s https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | sudo apt-key add -
  # 执行配置k8s阿里云源  
  echo 'deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main' >>/etc/apt/sources.list.d/kubernetes.list
  # 执行更新
  apt-get update -y
  ~~~

* 查看可安装的k8s版本，并且安装指定版本

  ~~~bash
  # 查看可安装的kubeadm、kubectl、kubelet版本
  apt-cache madison kubeadm
  # 安装kubeadm、kubectl、kubelet  
  apt-get install -y kubelet=1.22.6-00 kubeadm=1.22.6-00 kubectl=1.22.6-00
  ~~~

  注意：kubelet=“版本号”，中版本号要和apt-cache madison kubeadm输出的版本号一致

* 启用kubelet

  ~~~bash
  systemctl enable kubelet && systemctl restart kubelet
  ~~~

* 查看看此版本k8s需要的镜像

  ~~~bash
  kubeadm config images list
  ~~~

* 镜像拉取

  * 从国内镜像仓库拉取并改名（推荐）

    由于k8s工作节点上最终也要部署其中的一些容器，所以也需要其中的一些镜像，所以推荐以此方式在每个节点上都拉取所有镜像

    ~~~bash
    #镜像拉取
    docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:v1.22.17
    docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager:v1.22.17
    docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler:v1.22.17
    docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.22.17
    docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.5
    docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:3.5.0-0
    docker pull coredns/coredns:1.8.4
    #镜像改名
    docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:v1.22.17 k8s.gcr.io/kube-apiserver:v1.22.17
    docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager:v1.22.17 k8s.gcr.io/kube-controller-manager:v1.22.17
    docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler:v1.22.17 k8s.gcr.io/kube-scheduler:v1.22.17
    docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.22.17 k8s.gcr.io/kube-proxy:v1.22.17
    docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.5 k8s.gcr.io/pause:3.5
    docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:3.5.0-0 k8s.gcr.io/etcd:3.5.0-0
    docker tag coredns/coredns:1.8.4 k8s.gcr.io/coredns/coredns:v1.8.4
    ~~~

  * 直接在kubeadm init处，设置镜像拉取仓库（见下一步，kubeadm init）

### master节点上的操作

* 初始化kubeadm

  * 不指定拉取仓库版本

    ~~~~bash
    kubeadm init --apiserver-advertise-address=192.168.23.100 --apiserver-bind-port=6443 --kubernetes-version=v1.22.6 --pod-network-cidr=10.244.0.0/16 --service-cidr=10.96.0.0/12 --token-ttl=0 --cgroup-driver=systemd
    ~~~~

  * 指定拉取仓库版本

    ~~~bash
    kubeadm init --apiserver-advertise-address=192.168.23.100 --apiserver-bind-port=6443 --kubernetes-version=v1.22.6 --pod-network-cidr=10.244.0.0/16 --service-cidr=10.96.0.0/12 --image-repository=registry.cn-hangzhou.aliyuncs.com/google_containers --token-ttl=0 --cgroup-driver=systemd
    ~~~

  * 公网部署（使用云服务器）版本

    公网部署（使用云服务器）如果按照上述步骤进行初始化，会卡在![](https://typora2icture.oss-cn-beijing.aliyuncs.com/img2/8a8d0f221ae6d019bb154e31b45f52fd.png)

    因为 etcd 绑定端口的时候使用外网 IP，而云服务器外网 IP 并不是本机的网卡，而是网关分配的一个供外部访问的 IP，从而导致初始化进程一直重试绑定，长时间卡住后失败。

    使用如下方法，初始化成功：

    ~~~bash
    vim kubeadm-init.yaml
    ~~~

    ~~~yaml
    apiVersion: kubeadm.k8s.io/v1beta3
    bootstrapTokens:
    - groups:
      - system:bootstrappers:kubeadm:default-node-token
      token: abcdef.0123456789abcdef
      ttl: 24h0m0s
      usages:
      - signing
      - authentication
    kind: InitConfiguration
    localAPIEndpoint:
      advertiseAddress: {$public_ip} #你的公网ip，即k8s master 节点的公网ip
      bindPort: 6443
    ---
    apiServer:
      timeoutForControlPlane: 4m0s
    apiVersion: kubeadm.k8s.io/v1beta3
    certificatesDir: /etc/kubernetes/pki
    clusterName: kubernetes
    controllerManager: {}
    dns: {}
    etcd:
      local:
        dataDir: /var/lib/etcd
        extraArgs:
          listen-client-urls: https://0.0.0.0:2379 #直接在配置文件指定，这是公网部署的关键
          listen-peer-urls: https://0.0.0.0:2380 #直接在配置文件指定
    kind: ClusterConfiguration
    kubernetesVersion: 1.22.6
    networking:
      dnsDomain: cluster.local
      podSubnet: "10.244.0.0/16"
      serviceSubnet: 10.96.0.0/12
    imageRepository: "registry.aliyuncs.com/google_containers"
    controlPlaneEndPoint: "{$public_ip}:6443"	#这里还有一个，这个不用管，1.28.15版本不识别此参数
    scheduler: {}
    ---
    apiVersion: kubeadm.k8s.io/v1beta2
    kind: UserConfig
    metadata:
      name: admin
    users:
    - name: admin
      user:
        client-certificate: /etc/kubernetes/pki/admin.crt
        client-key: /etc/kubernetes/pki/admin.key
        embed-certs: true
        certificate-authority: /etc/kubernetes/pki/ca.crt
    ~~~

    启动初始化

    ~~~bashkubeadm init --config=kubeadm-init.yaml
    kubeadm init --config=kubeadm-init.yaml
    ~~~

  * 如果初始化出错，执行以下命令重置

    ~~~bash
    kubeadm reset
    ~~~

* 初始化完成后，根据初始化输出，创建.kube/config，获取加入集群的命令与token

  * 创建.kube/config(推荐根据初始化输出执行命令，而不是根据此教程)

    ~~~bash
    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config
    ~~~

  * 获取加入集群命令与token(推荐根据初始化输出执行命令，而不是根据此教程)

    ~~~bash
    kubeadm join 192.168.23.100:6443 --token ikx47e.4knl5pwe9fenrwds --discovery-token-ca-cert-hash sha256:9d783c63bf9a20ce9634788b96905f369668d9f439e444e271907561455b8779 
    ~~~

    接下来只要在其他node节点上执行上述命令即可加入集群

### 部署flannel

* 在master节点部署flannel

  * kube-flannel.yaml

    ~~~yaml
    apiVersion: v1
    kind: Namespace
    metadata:
      labels:
        k8s-app: flannel
        pod-security.kubernetes.io/enforce: privileged
      name: kube-flannel
    ---
    apiVersion: v1
    kind: ServiceAccount
    metadata:
      labels:
        k8s-app: flannel
      name: flannel
      namespace: kube-flannel
    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRole
    metadata:
      labels:
        k8s-app: flannel
      name: flannel
    rules:
    - apiGroups:
      - ""
      resources:
      - pods
      verbs:
      - get
    - apiGroups:
      - ""
      resources:
      - nodes
      verbs:
      - get
      - list
      - watch
    - apiGroups:
      - ""
      resources:
      - nodes/status
      verbs:
      - patch
    - apiGroups:
      - networking.k8s.io
      resources:
      - clustercidrs
      verbs:
      - list
      - watch
    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRoleBinding
    metadata:
      labels:
        k8s-app: flannel
      name: flannel
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: ClusterRole
      name: flannel
    subjects:
    - kind: ServiceAccount
      name: flannel
      namespace: kube-flannel
    ---
    apiVersion: v1
    data:
      cni-conf.json: |
        {
          "name": "cbr0",
          "cniVersion": "0.3.1",
          "plugins": [
            {
              "type": "flannel",
              "delegate": {
                "hairpinMode": true,
                "isDefaultGateway": true
              }
            },
            {
              "type": "portmap",
              "capabilities": {
                "portMappings": true
              }
            }
          ]
        }
      net-conf.json: |
        {
          "Network": "10.244.0.0/16",
          "Backend": {
            "Type": "vxlan"
          }
        }
    kind: ConfigMap
    metadata:
      labels:
        app: flannel
        k8s-app: flannel
        tier: node
      name: kube-flannel-cfg
      namespace: kube-flannel
    ---
    apiVersion: apps/v1
    kind: DaemonSet
    metadata:
      labels:
        app: flannel
        k8s-app: flannel
        tier: node
      name: kube-flannel-ds
      namespace: kube-flannel
    spec:
      selector:
        matchLabels:
          app: flannel
          k8s-app: flannel
      template:
        metadata:
          labels:
            app: flannel
            k8s-app: flannel
            tier: node
        spec:
          affinity:
            nodeAffinity:
              requiredDuringSchedulingIgnoredDuringExecution:
                nodeSelectorTerms:
                - matchExpressions:
                  - key: kubernetes.io/os
                    operator: In
                    values:
                    - linux
          containers:
          - args:
            - --ip-masq
            - --kube-subnet-mgr
            command:
            - /opt/bin/flanneld
            env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
            - name: EVENT_QUEUE_DEPTH
              value: "5000"
            image: docker.io/flannel/flannel:v0.25.4
            name: kube-flannel
            resources:
              requests:
                cpu: 100m
                memory: 50Mi
            securityContext:
              capabilities:
                add:
                - NET_ADMIN
                - NET_RAW
              privileged: false
            volumeMounts:
            - mountPath: /run/flannel
              name: run
            - mountPath: /etc/kube-flannel/
              name: flannel-cfg
            - mountPath: /run/xtables.lock
              name: xtables-lock
          hostNetwork: true
          initContainers:
          - args:
            - -f
            - /flannel
            - /opt/cni/bin/flannel
            command:
            - cp
            image: docker.io/flannel/flannel-cni-plugin:v1.4.1-flannel1
            name: install-cni-plugin
            volumeMounts:
            - mountPath: /opt/cni/bin
              name: cni-plugin
          - args:
            - -f
            - /etc/kube-flannel/cni-conf.json
            - /etc/cni/net.d/10-flannel.conflist
            command:
            - cp
            image: docker.io/flannel/flannel:v0.25.4
            name: install-cni
            volumeMounts:
            - mountPath: /etc/cni/net.d
              name: cni
            - mountPath: /etc/kube-flannel/
              name: flannel-cfg
          priorityClassName: system-node-critical
          serviceAccountName: flannel
          tolerations:
          - effect: NoSchedule
            operator: Exists
          volumes:
          - hostPath:
              path: /run/flannel
            name: run
          - hostPath:
              path: /opt/cni/bin
            name: cni-plugin
          - hostPath:
              path: /etc/cni/net.d
            name: cni
          - configMap:
              name: kube-flannel-cfg
            name: flannel-cfg
          - hostPath:
              path: /run/xtables.lock
              type: FileOrCreate
            name: xtables-lock
    ~~~

  * 关于flannel镜像，可以在本机上从dockerhub中获取指定版本的flannel镜像，本地下载后通过ssh工具导入集群中，在集群中使用docker导入镜像并改名

  * 执行flannel部署

    ~~~BASH
    kubectl apply -f kube-flannel.yaml
    ~~~

  * 当所有节点的flannel插件运行正常，所有节点ready，完成部署

# Kubeedge部署

> kubeedge v1.10.1

### 云核心部署

* 文件准备

  为了避免安装过程中，下载过慢，网络墙的干扰，先在本机上获取如下软件包，并且将这些软件包上传至k8s集群和边缘节点edge

  1. checksum_kubeedge-v1.10.0-linux-amd64.tar.gz.txt
  2. keadm-v1.10.0-linux-amd64.tar.gz
  3. kubeedge-1.10.0.tar.gz
  4. kubeedge-v1.10.0-linux-amd64.tar.gz

* 安装keadm

  ~~~bash
  root@ubuntu:# tar -zxf keadm-v1.10.0-linux-amd64.tar.gz
  root@ubuntu:# cp keadm-v1.10.0-linux-amd64/keadm/keadm /usr/bin/
  
  #检验是否安装成功
  root@ubuntu:# keadm version
  version: version.Info{Major:"1", Minor:"10", GitVersion:"v1.10.0", GitCommit:"3803951602f938d9d90d74957eb0fbc238142101", GitTreeState:"clean", BuildDate:"2022-03-14T02:30:42Z", GoVersion:"go1.16.15", Compiler:"gc", Platform:"linux/amd64"}
  ~~~

* 将所有文件准备好，提前放在该在的位置

  ~~~bash
  mkdir -p /etc/kubeedge
  cp checksum_kubeedge-v1.10.0-linux-amd64.tar.gz.txt /etc/kubeedge
  cp kubeedge-v1.10.0-linux-amd64.tar.gz /etc/kubeedge
  tar -zxf kubeedge-1.10.0.tar.gz
  cd kubeedge-1.10.0/build/
  cp tools/*.service /etc/kubeedge/
  cp -r crds/ /etc/kubeedge/
  ~~~

* 初始化云核心

  ~~~bash
  keadm init deprecated --kubeedge-version=1.10.0 --advertise-address 192.168.23.100
  ~~~

  其中–advertise-address是云核心，即k8s的master节点的ip

  检查云核心启动情况

  ~~~bash
  root@ubuntu:~# netstat -ntpl
  Active Internet connections (only servers)
  Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
  tcp        0      0 127.0.0.1:39499         0.0.0.0:*               LISTEN      11718/kubelet       
  tcp        0      0 192.168.5.64:2379       0.0.0.0:*               LISTEN      11302/etcd          
  tcp        0      0 127.0.0.1:2379          0.0.0.0:*               LISTEN      11302/etcd          
  tcp        0      0 192.168.5.64:2380       0.0.0.0:*               LISTEN      11302/etcd          
  tcp        0      0 127.0.0.1:2381          0.0.0.0:*               LISTEN      11302/etcd          
  tcp        0      0 127.0.0.1:10257         0.0.0.0:*               LISTEN      11343/kube-controll 
  tcp        0      0 127.0.0.1:10259         0.0.0.0:*               LISTEN      11314/kube-schedule 
  tcp        0      0 127.0.0.53:53           0.0.0.0:*               LISTEN      760/systemd-resolve 
  tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      1305/sshd           
  tcp        0      0 127.0.0.1:10248         0.0.0.0:*               LISTEN      11718/kubelet       
  tcp        0      0 127.0.0.1:10249         0.0.0.0:*               LISTEN      12134/kube-proxy    
  tcp6       0      0 :::10250                :::*                    LISTEN      11718/kubelet       
  tcp6       0      0 :::6443                 :::*                    LISTEN      11286/kube-apiserve 
  tcp6       0      0 :::10000                :::*                    LISTEN      29206/cloudcore     
  tcp6       0      0 :::10256                :::*                    LISTEN      12134/kube-proxy    
  tcp6       0      0 :::10002                :::*                    LISTEN      29206/cloudcore     
  tcp6       0      0 :::22                   :::*                    LISTEN      1305/sshd   
  ~~~

  可以发现，10000，10002端口启动

* 开启cloudstream

  ~~~bash
  cd kubeedge-1.10.0/build/tools
  cp certgen.sh /etc/kubeedge
  export CLOUDCOREIPS="10.196.110.64"
  ./certgen.sh stream
   
  vi /etc/kubeedge/config/cloudcore.yaml
    cloudStream:
      enable: true  #修改
      streamPort: 10003
      tlsStreamCAFile: /etc/kubeedge/ca/streamCA.crt
      tlsStreamCertFile: /etc/kubeedge/certs/stream.crt
      tlsStreamPrivateKeyFile: /etc/kubeedge/certs/stream.key
      tlsTunnelCAFile: /etc/kubeedge/ca/rootCA.crt
      tlsTunnelCertFile: /etc/kubeedge/certs/server.crt
      tlsTunnelPrivateKeyFile: /etc/kubeedge/certs/server.key
      tunnelPort: 10004
  ~~~

  就是修改cloudcore的配置文件，上面的`CLOUDCOREIPS=“10.196.100.64”`经过验证，可以不用修改

* 重启cloudcore并检查端口开启情况

  ~~~bash
  systemctl enable cloudcore.service && systemctl restart cloudcore.service
  netstat -ntpl
  #开启了10000 - 10004 端口
  tcp6       0      0 :::4443                 :::*                    LISTEN      9418/metrics-server 
  tcp6       0      0 :::10250                :::*                    LISTEN      7351/kubelet        
  tcp6       0      0 :::6443                 :::*                    LISTEN      6915/kube-apiserver 
  tcp6       0      0 :::111                  :::*                    LISTEN      622/rpcbind         
  tcp6       0      0 :::10000                :::*                    LISTEN      25064/cloudcore     
  tcp6       0      0 :::80                   :::*                    LISTEN      19023/docker-proxy  
  tcp6       0      0 :::10256                :::*                    LISTEN      8244/kube-proxy     
  tcp6       0      0 :::10257                :::*                    LISTEN      6913/kube-controlle 
  tcp6       0      0 :::10002                :::*                    LISTEN      25064/cloudcore     
  tcp6       0      0 :::10003                :::*                    LISTEN      25064/cloudcore     
  tcp6       0      0 :::10259                :::*                    LISTEN      6900/kube-scheduler 
  tcp6       0      0 :::10004                :::*                    LISTEN      25064/cloudcore   
  ~~~

  如果发现没有的，可以尝试重启一下，或许就有了

  如果找不到cloudcore服务，将`/etc/kubeedge`的service文件移动到`/etc/systemd/system`目录下

  ~~~bash
  cp cloudcore.service /etc/systemd/system/service
  systemctl enable cloudcore.service && systemctl restart cloudcore.service
  ~~~



### 边核心部署

* 安装keadm

  ~~~bash
  root@ubuntu:# tar -zxf keadm-v1.10.0-linux-amd64.tar.gz
  root@ubuntu:# cp keadm-v1.10.0-linux-amd64/keadm/keadm /usr/bin/
  
  #检验是否安装成功
  root@ubuntu:# keadm version
  version: version.Info{Major:"1", Minor:"10", GitVersion:"v1.10.0", GitCommit:"3803951602f938d9d90d74957eb0fbc238142101", GitTreeState:"clean", BuildDate:"2022-03-14T02:30:42Z", GoVersion:"go1.16.15", Compiler:"gc", Platform:"linux/amd64"}
  ~~~

* 将所有文件准备好，提前放在该在的位置

  ~~~bash
  mkdir -p /etc/kubeedge
  cp checksum_kubeedge-v1.10.0-linux-amd64.tar.gz.txt /etc/kubeedge
  cp kubeedge-v1.10.0-linux-amd64.tar.gz /etc/kubeedge
  ~~~

* 安装cJson

  * 方法1：

    ~~~bash
    sudo apt-get install libjson-c-dev
    ~~~

  * 方法2：

    下载cJson源码包，源码编译安装

    ~~~bash
    #安装编译工具
    apt-get install gcc g++ make openssl libssl-dev
    #cJson源码编译安装
    unzip cJSON-master.zip 
    cd cJSON-master
    make && make install 
    ~~~

* 安装并启动mosquito

  * 方式1：

    ~~~bash
    sudo apt-add-repository ppa:mosquitto-dev/mosquitto-ppa
    sudo apt-get update
    sudo apt-get install mosquitto
    ~~~

  * 方式2：

    下载mosquitto源码包，源码编译安装

    > mosquitto源码包网址：https://mosquitto.org/files/source/

    ~~~bash
    tar xf mosquitto-2.0.14.tar.gz 
    make && make install
    ~~~

  * 启动mosquito

    ~~~bash
    mosquitto -d
    #查看mosquito运行情况
    netstat -ntpl
    
    Active Internet connections (only servers)
    Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
    tcp        0      0 127.0.0.53:53           0.0.0.0:*               LISTEN      779/systemd-resolve 
    tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      1290/sshd           
    tcp        0      0 127.0.0.1:1883          0.0.0.0:*               LISTEN      10406/mosquitto     
    tcp6       0      0 :::22                   :::*                    LISTEN      1290/sshd           
    tcp6       0      0 ::1:1883                :::*                    LISTEN      10406/mosquitto 
    ~~~

* 由于keadm的edge加入集群初始化需要访问raw.githubusercontent.com，为了避免DNS找不到的问题，现在hosts中添加

  ~~~bash
  vim /etc/hosts
  
  185.199.108.133 raw.githubusercontent.com
  ~~~

* 从云核心获取token

  ~~~bash
  sudo keadm gettoken
  3f80c76990436a31f4a22f723a6b5db44fdd9d3427ebcade0ad1638d0f59798a.eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE3MzE5ODEzNzJ9.Wby2awueAdEySI4Tzijk8thl01lYU1ZXYLki1iFb5CU
  
  ~~~

* 边缘节点加入（概率失败，由于连接raw.githubusercontent.com问题，多试几次）

  ~~~bash
  keadm join --cloudcore-ipport=192.168.23.100:10000 --edgenode-name=edge1 --kubeedge-version=1.10.0 --cgroupdriver systemd --token=3f80c76990436a31f4a22f723a6b5db44fdd9d3427ebcade0ad1638d0f59798a.eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE3MzE5ODEzNzJ9.Wby2awueAdEySI4Tzijk8thl01lYU1ZXYLki1iFb5CU
  ~~~

  其中，`–-cloudcore-ipport=192.168.23.100：10000`是master云核心所在节点的ip，端口号不用改，注意，需要指定安装kubeedge安装版本`--kubeedge-version=1.10.0`

  * 注意:

  成功加入后会有：

  ~~~bash
  ...
  kubeedge-v1.10.0-linux-amd64/edge/edgecore
  kubeedge-v1.10.0-linux-amd64/version
  
  KubeEdge edgecore is running, For logs visit: journalctl -u edgecore.service -xe
  ~~~

  



# 参考文档

* 无坑点ubuntu 部署k8s集群保姆教程（保证一次成功：https://blog.csdn.net/Tiger_lin1/article/details/126980166

* K8s kubeadm添加新节点详细操作：https://blog.csdn.net/qq_26129413/article/details/115179285

* KubeEdge1.10从零开始详细搭建教程：

  https://blog.csdn.net/u010549795/article/details/125609761

* kubeedge学习(2) 二进制离线搭建kubeedge 1.10.0：

  https://blog.csdn.net/qihan1124/article/details/131469212

* [Ubuntu 22.04] 安装docker，并设置镜像加速：

  https://blog.csdn.net/IOT_AI/article/details/131911915

* ubuntu安装指定版本docker(包含官方/国内安装方法)：

  https://blog.csdn.net/qq_37843943/article/details/107245223

* 公网k8s部署（无坑小白版）：

  https://cloud.tencent.com/developer/article/2274059