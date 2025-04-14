---
tags:
  - ECS
  - K8s1-28-15
  - containerd
  - ubunut
  - kubeedge1-18-3
---

# 快速搭建K8s集群
> 使用的配置：
>
> - 操作系统：Ubuntu 24.04 Server
> - K8s 版本：1.28.15

## 1. 主机安装

首先要先规划出各个主机的 IP 地址、硬件资源等信息。例如：

| 作用   | IP地址          | 操作系统            | 配置                     |
| ------ | --------------- | ------------------- | ------------------------ |
| Master | 192.168.100.140 | Ubuntu 24.04 Server | 2颗CPU  2G内存   50G硬盘 |
| Node1  | 192.168.100.141 | Ubuntu 24.04 Server | 2颗CPU  2G内存   50G硬盘 |
| Node2  | 192.168.100.142 | Ubuntu 24.04 Server | 2颗CPU  2G内存   50G硬盘 |

并且，在安装操作系统（例如 Centos 7.5）的时候，「软件选择」选择「基础设施服务器」。

「网络配置」像下面这样配置：

```bash
网络地址：192.168.100.140  （每台主机都不一样  分别为140、141、142）
子网掩码：255.255.255.0
默认网关：192.168.100.2
DNS：    8.8.8.8
```

三台主机的「主机名」按照下面这样设置：

```bash
master节点： master
node节点：   node1
node节点：   node2
```

> 也可以不指定主机的IP，但是名字建议还是指定一下，只要知晓主机IP，并且三台主机可以互相连通即可

## 2. 环境初始化

1. IP 地址修改与主机名解析。

    为了方便后续集群节点之间的直接调用，在这里要配置一下主机名解析。

    如果是在云ECS弹性服务器上部署，一定要配置/etc/hosts

    ```bash
    vim /etc/sysconfig/network-scripts/ifcfg-ens33
    # 然后修改 IP 地址
    systemctl restart network
    
    # 分别设置主机域名
    hostnamectl set-hostname master/node1/node2
    
    vim /etc/hosts
    # 加入下面的内容
    --------------------------
    192.168.100.140  master
    192.168.100.141  node1
    192.168.100.142  node2
    185.199.108.133 raw.githubusercontent.com
    --------------------------
    ```

2. 时间同步。

    K8s 要求集群中的节点时间必须准确一致。这里直接使用 chronyd 服务从网络同步时间。

    ```bash
    # 启动 chronyd 服务
    systemctl start chronyd
    # 设置 chronyd 服务开机自启动
    systemctl enable chronyd
    # 服务启动之后稍微等几秒钟，就可以使用 date 命令验证时间了
    date
    ```

    Ubuntu没有chronyd服务可以使用 ntpdate 来同步时间：

    ```bash
    # 安装 ntpdate
    apt-get install ntpdate
    # 写入 crontab 计划任务
    cat << EOF >> /etc/crontab
    0 */1 * * * ntpdate time1.aliyun.com
    EOF
    # 使用 date 命令验证时间
    date
    ```

3. 禁用 iptables 和 firewalld 服务

    K8s 和 docker 在运行中会产生大量的 iptables 规则，为了不让系统规则跟它们混淆，直接关闭系统的规则。

    ```bash
    # 关闭 firewalld 服务
    systemctl stop firewalld
    systemctl disable firewalld
    firewall-cmd --state
    # 关闭 iptables 服务（Centos7.5 中不用做这个操作）
    systemctl stop iptables
    systemctl disable iptables
    ```

    > Ubuntu 系统默认好像是没有开启这两个服务的，有的话还是关闭一下

4. 禁用 SELinux。

    SELinux 是 linux 系统下的一个安全服务，如果不关闭它，在安装集群中会产生各种各样的奇葩问题。

    ```bash
    vim /etc/selinux/config
    # 修改其中的 SELINUX 这一项为 disabled
    -----------
    SELINUX=disabled
    -----------
    ```

    > 这一步Ubuntu系统也可以不用做，没有相应的配置

5. 禁用 swap 分区。

    swap 分区指的是虚拟内存分区，它的作用是在物理内存使用完之后，将磁盘空间虚拟成内存来使用。

    启用 swap 设备会对系统的性能产生非常负面的影响，因此 K8s 要求每个节点都要禁用 swap 设备。

    ```bash
    vim /etc/fstab
    # 接下来注释掉 swap 分区这一行内容
    # 一般就是 fstab 的最后一行
    ```

6. 修改 linux 的内核参数。

    通过修改 linux 内核参数，添加网桥过滤和地址转发功能。

    ```bash
    cat << EOF >> /etc/sysctl.d/kubernetes.conf
    net.bridge.bridge-nf-call-ip6tables = 1
    net.bridge.bridge-nf-call-iptables = 1
    net.ipv4.ip_forward = 1
    EOF
    
    # 重新加载配置
    sysctl -p
    
    # 加载网桥过滤模块
    # 将加载模块的配置写入文件中，这样可以永久生效
    echo "br_netfilter" | sudo tee /etc/modules-load.d/br_netfilter.conf && sudo modprobe br_netfilter
    
    # 查看网桥过滤模块是否加载成功
    lsmod | grep br_netfilter
    ```
    
    > 这一步一定要配置，特别是br_netfilter，在后续的flannel版本中，没有了对br_netfilter模块启用的检查，但是还是依赖于该模块的
    
7. 配置 ipvs 功能。

    在 K8s 中 Service 有两种代理模型，一种是基于 iptables 的，一种是基于 ipvs 的。

    两者比较的话，ipvs 的性能明显要高一些，但是如果要使用它，需要手动加载 ipvs 模块。

    ```bash
    # 安装 ipset 和 ipvsadm
    yum install ipset ipvsadm -y
    
    # 添加需要加载的模块写入脚本文件
    # ！！！如果是比较新的 Linux 内核中，可能就没有 nf_conntrack_ipv4 这个模块。因为这个模块被合并到 nf_conntrack 中了。
    cat <<EOF >  /etc/sysconfig/modules/ipvs.modules
    #!/bin/bash
    modprobe -- ip_vs
    modprobe -- ip_vs_rr
    modprobe -- ip_vs_wrr
    modprobe -- ip_vs_sh
    modprobe -- nf_conntrack_ipv4
    EOF
    
    # 为脚本文件添加执行权限
    chmod +x /etc/sysconfig/modules/ipvs.modules
    
    # 执行脚本文件
    /bin/bash /etc/sysconfig/modules/ipvs.modules
    
    # 查看对应的模块是否加载成功
    lsmod | grep -e ip_vs -e nf_conntrack
    ```

    > 这一步无所谓，Ubuntu系统可以不做

8. 重启服务器。

    上面的步骤完成之后，需要重启 linux 操作系统。

    ```bash
    reboot
    ```

## 3. 安装 Containerd

新版的 K8s 已经不支持 Docker-shim 了，所以现在我们就不安装 Docker 了，而是使用 Containerd。

### 3.1 Containerd 的下载

```bash
# 下载 Containerd
wget https://github.com/containerd/containerd/releases/download/v1.7.22/containerd-1.7.22-linux-amd64.tar.gz
# 这个压缩包里面就是一个 bin 目录，里面装着一些可执行文件，例如 containerd、ctr 之类的
# 所以我们可以直接将压缩包解压到 /usr/local，然后这些可执行文件就会自动加载到 /usr/local/bin 里面
tar -zxvf containerd-1.7.22-linux-amd64.tar.gz -C /usr/local

# 通过 systemd 启动 containerd
# 写入下面的内容（内容来自于 https://github.com/containerd/containerd/blob/main/containerd.service）
cat << EOF > /etc/systemd/system/containerd.service
[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target local-fs.target

[Service]
ExecStartPre=-/sbin/modprobe overlay
ExecStart=/usr/local/bin/containerd  # 确认路径是否正确
Type=notify
Delegate=yes
KillMode=process
Restart=always
RestartSec=5
LimitNPROC=infinity
LimitCORE=infinity
# TasksMax=infinity  # 移除这一行，避免弃用警告
OOMScoreAdjust=-999

[Install]
WantedBy=multi-user.target
EOF

# 加载配置、启动
systemctl daemon-reload
systemctl enable containerd

```

### 3.2 Containerd 初始化配置

```bash
# 生成 Containerd 初始化配置
mkdir /etc/containerd
containerd config default > /etc/containerd/config.toml

# 修改配置文件，需要修改三个地方
vim /etc/containerd/config.toml
---------------------------------------------------------------------------------
# 如果要配置 K8s 的话，就要修改这里
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
  SystemdCgroup = true
# 如果容器启动不起来，爆出 sandbox_image 之类的东西，可以修改这里
  sandbox_image = "registry.aliyuncs.com/google_containers/pause:3.9"
# 修改镜像加速文件地址
  config_path = "/etc/containerd/certs.d"
---------------------------------------------------------------------------------
```

### 3.3 Containerd 镜像加速地址配置

```bash
# docker hub镜像加速
mkdir -p /etc/containerd/certs.d/docker.io
cat > /etc/containerd/certs.d/docker.io/hosts.toml << EOF
server = "https://docker.io"
# 此处的华为云的镜像加速服务是免费的，可以在华为云官网中找到SWR容器镜像加速服务，输入自己的加速链接
[host."https://5fa0b842a2ab4dc4839e3d999fd45600.mirror.swr.myhuaweicloud.com"]
  capabilities = ["pull", "resolve"]

[host."https://le2c3l3b.mirror.aliyuncs.com"]
  capabilities = ["pull", "resolve"]

[host."https://docker.m.daocloud.io"]
  capabilities = ["pull", "resolve"]

[host."https://reg-mirror.qiniu.com"]
  capabilities = ["pull", "resolve"]

[host."https://registry.docker-cn.com"]
  capabilities = ["pull", "resolve"]

[host."http://hub-mirror.c.163.com"]
  capabilities = ["pull", "resolve"]

EOF

# registry.k8s.io镜像加速
mkdir -p /etc/containerd/certs.d/registry.k8s.io
tee /etc/containerd/certs.d/registry.k8s.io/hosts.toml << 'EOF'
server = "https://registry.k8s.io"

[host."https://k8s.m.daocloud.io"]
  capabilities = ["pull", "resolve", "push"]
EOF

# docker.elastic.co镜像加速
mkdir -p /etc/containerd/certs.d/docker.elastic.co
tee /etc/containerd/certs.d/docker.elastic.co/hosts.toml << 'EOF'
server = "https://docker.elastic.co"

[host."https://elastic.m.daocloud.io"]
  capabilities = ["pull", "resolve", "push"]
EOF

# gcr.io镜像加速
mkdir -p /etc/containerd/certs.d/gcr.io
tee /etc/containerd/certs.d/gcr.io/hosts.toml << 'EOF'
server = "https://gcr.io"

[host."https://gcr.m.daocloud.io"]
  capabilities = ["pull", "resolve", "push"]
EOF

# ghcr.io镜像加速
mkdir -p /etc/containerd/certs.d/ghcr.io
tee /etc/containerd/certs.d/ghcr.io/hosts.toml << 'EOF'
server = "https://ghcr.io"

[host."https://ghcr.m.daocloud.io"]
  capabilities = ["pull", "resolve", "push"]
EOF

# k8s.gcr.io镜像加速
mkdir -p /etc/containerd/certs.d/k8s.gcr.io
tee /etc/containerd/certs.d/k8s.gcr.io/hosts.toml << 'EOF'
server = "https://k8s.gcr.io"

[host."https://k8s-gcr.m.daocloud.io"]
  capabilities = ["pull", "resolve", "push"]
EOF

# mcr.m.daocloud.io镜像加速
mkdir -p /etc/containerd/certs.d/mcr.microsoft.com
tee /etc/containerd/certs.d/mcr.microsoft.com/hosts.toml << 'EOF'
server = "https://mcr.microsoft.com"

[host."https://mcr.m.daocloud.io"]
  capabilities = ["pull", "resolve", "push"]
EOF

# nvcr.io镜像加速
mkdir -p /etc/containerd/certs.d/nvcr.io
tee /etc/containerd/certs.d/nvcr.io/hosts.toml << 'EOF'
server = "https://nvcr.io"

[host."https://nvcr.m.daocloud.io"]
  capabilities = ["pull", "resolve", "push"]
EOF

# quay.io镜像加速
mkdir -p /etc/containerd/certs.d/quay.io
tee /etc/containerd/certs.d/quay.io/hosts.toml << 'EOF'
server = "https://quay.io"

[host."https://quay.m.daocloud.io"]
  capabilities = ["pull", "resolve", "push"]
EOF

# registry.jujucharms.com镜像加速
mkdir -p /etc/containerd/certs.d/registry.jujucharms.com
tee /etc/containerd/certs.d/registry.jujucharms.com/hosts.toml << 'EOF'
server = "https://registry.jujucharms.com"

[host."https://jujucharms.m.daocloud.io"]
  capabilities = ["pull", "resolve", "push"]
EOF

# rocks.canonical.com镜像加速
mkdir -p /etc/containerd/certs.d/rocks.canonical.com
tee /etc/containerd/certs.d/rocks.canonical.com/hosts.toml << 'EOF'
server = "https://rocks.canonical.com"

[host."https://rocks-canonical.m.daocloud.io"]
  capabilities = ["pull", "resolve", "push"]
EOF
```

### 3.4 验证安装结果

```bash
# 重启 Containerd
systemctl restart containerd
# 验证
ctr version
```

### 3.5 安装 nerdctl（非必须）

安装 nerdctl 的目的是改变 Containerd 的命令行操作方式。因为 ctr 命令太难用了，换成 nerdctl，操作的命令就与 Docker 的命令特别相似。（但是如果单纯操作 K8s，也可以不安装 nerdctl）。

```bash
# 下载 nerdctl
wget https://github.com/containerd/nerdctl/releases/download/v2.0.0/nerdctl-2.0.0-linux-amd64.tar.gz

# 解压，只需要把其中的 nerdctl 可执行文件放到 /usr/local/bin 就行了
tar -zxvf nerdctl-2.0.0-linux-amd64.tar.gz nerdctl
mv nerdctl /usr/local/bin

# 验证
nerdctl version
```

### 3.6 安装 runc

`runc`  是一个用来运行容器的命令行工具。它是一个底层的容器运行时，可以被像 Containerd 这样的高级运行时调用。

Containerd 自身并不会直接运行容器，它是通过调用 `runc` 来执行容器的低级操作，例如启动、停止、删除等。也就是说，`runc` 是 Containerd 的核心部分，它负责把容器镜像创建为真正运行的容器实例。

```bash
# 从 github 下载 runc
wget https://github.com/opencontainers/runc/releases/download/v1.1.15/runc.amd64

# 把 runc 移动到 /usr/local/sbin 目录下，并命名为 runc
install runc.amd64 -m 755 /usr/local/sbin/runc
```

注：把 `runc`  放到 `/usr/local/sbin` 而不是 `/usr/local/bin` 是因为它是一个系统级的工具，需要管理员权限来运行。一般情况下，普通用户不需要也不应该直接使用它，所以将其放在 `/usr/local/sbin` 中，可以更好地反映它的用途和权限需求。

### 3.7 安装 cni-plugins

基础的 cni 插件可以实现单主机的多个容器之间的网络通信。但是如果要部署 K8s 集群，也就是说如果要进行多个主机之间容器的通信的话，就需要使用 Flannel、Calico 这些网络插件。

```bash
# 下载 cni-plugins
wget https://github.com/containernetworking/plugins/releases/download/v1.5.1/cni-plugins-linux-amd64-v1.5.1.tgz

# 安装 cni-plugins 到 /opt
mkdir -p /opt/cni/bin
tar zxvf cni-plugins-linux-amd64-v1.5.1.tgz -C /opt/cni/bin
```

### 3.8 安装 crictl

`crictl` 并不是必须安装的，但是强烈建议安装它，因为它能为容器的管理和调试提供便利。

在 K8s 环境当中，最好同时安装 `nerdctl` 和 `crictl`。这样可以根据需求选择最合适的工具：

- 使用 `crictl` 来调试和排查 K8s 集群问题。
- 使用 `nerdctl` 来执行容器的管理和构建任务。

```bash
# 下载 crictl
wget https://github.com/kubernetes-sigs/cri-tools/releases/download/v1.31.1/crictl-v1.31.1-linux-amd64.tar.gz

# 安装 crictl
tar zxvf crictl-v1.31.1-linux-amd64.tar.gz -C /usr/loca/bin

# 配置 crictl
cat >> /etc/crictl.yaml << EOF
runtime-endpoint: unix:///var/run/containerd/containerd.sock
image-endpoint: unix:///var/run/containerd/containerd.sock
timeout: 10
debug: true
EOF

# 重启 containerd
systemctl restart containerd
```

## 4. 安装 K8s 组件

> 如下是笔记原版中centos的组件下载方式
>
> ~~~ bash
> # 由于 K8s 的镜像源在国外，速度比较慢，这里换成国内的镜像源
> cat << EOF > /etc/yum.repos.d/kubernetes.repo
> [kubernetes]
> name=Kubernetes
> baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
> enabled=1
> gpgcheck=0
> repo_gpgcheck=0
> gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
>        http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
> EOF 
> # 如果是 arm64 架构的机器，要将上面的 x86_64 修改为 aarch64
> 
> # 安装 kubeadm、kubelet 和 kubectl
> # 这里安装的是 kubelet 1.28.2 版本
> yum install kubeadm-1.28.2-0 kubelet-1.28.2-0 kubectl-1.28.2-0 -y
> 
> # 编辑 kubelet 的 cgroup
> cat << EOF > /etc/sysconfig/kubelet
> KUBELET_CGROUP_ARGS="--cgroup-driver=systemd"
> KUBE_PROXY_MODE="ipvs"
> EOF
> 
> # 设置 kubelet 开机自启动
> systemctl start kubelet
> systemctl enable kubelet
> ~~~
>
> 

```bash
#在所有节点上添加Kubernetes的阿里云源
apt-get update && apt-get install -y apt-transport-https
curl -fsSL https://mirrors.aliyun.com/kubernetes-new/core/stable/v1.28/deb/Release.key |
    gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://mirrors.aliyun.com/kubernetes-new/core/stable/v1.28/deb/ /" |
    tee /etc/apt/sources.list.d/kubernetes.list
apt-get update

#安装kubeadm、kubelet、kubectl
apt install -y kubeadm=1.28.15-1.1 kubelet=1.28.15-1.1 kubectl=1.28.15-1.1

#安装完成后，检查各个组件版本
kubeadm version
kubectl version
kubelet --version
```

* 提前在各个节点上拉取Kubernetes需要的镜像

  ~~~bash
  # 首先查看 kubeadm 的版本
  kubeadm version
  # 这里就会输出我的版本是 v1.28.15 的
  
  # 查看此版本k8s需要的镜像
  kubeadm config images list
  
  # 使用ctr命令直接拉取各个镜像
  ctr -n k8s.io images pull registry.k8s.io/kube-apiserver:v1.28.15 --hosts-dir="/etc/containerd/certs.d"
  ctr -n k8s.io images pull registry.k8s.io/kube-controller-manager:v1.28.15 --hosts-dir="/etc/containerd/certs.d"
  ctr -n k8s.io images pull registry.k8s.io/kube-scheduler:v1.28.15 --hosts-dir="/etc/containerd/certs.d"
  ctr -n k8s.io images pull registry.k8s.io/kube-proxy:v1.28.15 --hosts-dir="/etc/containerd/certs.d"
  ctr -n k8s.io images pull registry.k8s.io/pause:3.9 --hosts-dir="/etc/containerd/certs.d"
  ctr -n k8s.io images pull registry.k8s.io/etcd:3.5.15-0 --hosts-dir="/etc/containerd/certs.d"
  ctr -n k8s.io images pull registry.k8s.io/coredns/coredns:v1.10.1 --hosts-dir="/etc/containerd/certs.d"
  ~~~

## 5. 集群初始化

==**下面的操作都是只需要在 master 上面完成就可以！**==

对 kubeadm 进行配置：

> 下面配置文件中，如果是在ECS云服务器上部署Kubernetes，一定要加入那两行配置
>
> 因为由于在ECS服务器上，给定的IP地址其实不是云主机的网卡的IP，所以在kubeadm初始化的时候，可能会卡死

```bash
# 生成默认配置文件
kubeadm config print init-defaults > kubeadm-init-config.yaml

# 编辑配置文件
vim kubeadm.yaml
----------------------------------------------------------------------
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
  advertiseAddress: 39.98.76.224  # 修改为宿主机ip
  bindPort: 6443
nodeRegistration:
  criSocket: unix:///var/run/containerd/containerd.sock
  imagePullPolicy: IfNotPresent
  name: master   # 修改为宿主机名
  taints: null
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
      listen-client-urls: https://0.0.0.0:2379 #直接在配置文件指定，用于在ECS服务器上部署Kubernetes
      listen-peer-urls: https://0.0.0.0:2380 #直接在配置文件指定，用于在ECS服务器上部署Kubernetes
imageRepository: registry.aliyuncs.com/google_containers # 修改为阿里镜像
kind: ClusterConfiguration
kubernetesVersion: 1.28.15  # kubeadm的版本为多少这里就修改为多少
networking:
  dnsDomain: cluster.local
  serviceSubnet: 10.96.0.0/12
  podSubnet: 10.244.0.0/16   # 设置pod网段
scheduler: {}

###添加内容：配置kubelet的CGroup为systemd
---
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
cgroupDriver: systemd
----------------------------------------------------------------------
```

然后下载相应的镜像，并进行初始化：

```bash
# 下载镜像，由于之前提前下载过了，就不用这步了
# kubeadm config images pull --image-repository=registry.aliyuncs.com/google_containers  --kubernetes-version=v1.28.2

# 集群初始化modprobe br_netfilter
kubeadm init --config kubeadm.yaml

# 初始化完成之后，就根据提示对 master 和 node 做对应的操作就行
# master:
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
# node:
kubeadm join 192.168.100.140:6443 --token abcdef.0123456789abcdef \
        --discovery-token-ca-cert-hash sha256:0644a865914a18da57dcef71860fe4efa69908e7f2ec30094ed04784b58cb0f1


# 查看集群状态，此时的集群状态为 NotReady，这是因为还没有配置网络插件
[root@master ~]# kubectl get nodes
NAME     STATUS     ROLES           AGE     VERSION
master   NotReady   control-plane   2m43s   v1.28.2
node1    NotReady   <none>          18s     v1.28.2
node2    NotReady   <none>          12s     v1.28.2
```

## 6. 安装网络插件

kubernetes支持多种网络插件，比如flannel、calico、canal等等，任选一种使用即可，本次选择flannel。

==**下面的操作都只需要在 master 上面执行！**==

```bash
# 下载 flannel
wget https://raw.githubusercontent.com/flannel-io/flannel/refs/heads/master/Documentation/kube-flannel.yml

# 加载 flannel
kubectl apply -f kube-flannel.yml
```

> 由于flannel默认使用vxlan方式
>
> vxlan默认使用端口：
>
> **4789/udp**：这是VXLAN协议的默认IANA分配的UDP端口号，用于VXLAN封装和传输的数据包

## 7. 验证

```bash
# 部署nginx
kubectl create deployment nginx --image=nginx:latest

# 暴露端口
kubectl expose deployment nginx --port=80 --type=NodePort

# 查看服务状态
kubectl get pods,service
------------------------------------------------------------------------------------------
NAME                         READY   STATUS    RESTARTS   AGE
pod/nginx-56fcf95486-2nz72   1/1     Running   0          30s

NAME                 TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
service/kubernetes   ClusterIP   10.96.0.1        <none>        443/TCP        61m
service/nginx        NodePort    10.100.226.160   <none>        80:31729/TCP   20s
------------------------------------------------------------------------------------------

# 访问 nginx 服务
# 用主机上的浏览器访问：192.168.100.140:31729
```



# 关于Kubeedge

* 关于Kubeedge的安装，参考教程：https://github.com/TobyIcetea/LearningNotes/blob/main/KubeEdge/KubeEdge%E6%90%AD%E5%BB%BA%EF%BC%882%EF%BC%89.md

  建议在进行init和join之前，先拉好镜像

  ~~~bash
  ctr -n k8s.io images pull docker.io/kubeedge/iptables-manager:v1.18.3 --hosts-dir="/etc/containerd/certs.d"
  ctr -n k8s.io images pull docker.io/kubeedge/controller-manager:v1.18.3 --hosts-dir="/etc/containerd/certs.d"
  ctr -n k8s.io images pull docker.io/kubeedge/cloudcore:v1.18.3 --hosts-dir="/etc/containerd/certs.d"
  ctr -n k8s.io images pull docker.io/kubeedge/installation-package:v1.18.3 --hosts-dir="/etc/containerd/certs.d"
  ctr -n k8s.io images pull docker.io/kubeedge/admission:v1.18.3 --hosts-dir="/etc/containerd/certs.d"
  ~~~

* 如果边缘端出现flannel不断重启的情况，请检查如下

  1. /opt/cni/bin中是否有flannel插件
  2. /etc/cni/net.d/中是否有flannel生成的默认配置文件10-flannel.conflist
  3. /run/flannel/下是否有subnet.env子网分配文件

  如果没有，可以手动编写一个subnet.env、10-flannel.conflist文件

  关于subnet.env，示例文件如下：

  ~~~bash
  FLANNEL_NETWORK=10.244.0.0/16
  FLANNEL_SUBNET=10.244.0.1/24	# 这是该节点的pod的子网网段，一般来说，每个节点的子网网段都是不一样的，比如master的是.0.1，node1的是.1.1，node2的是.2.1
  FLANNEL_MTU=1450
  FLANNEL_IPMASQ=true
  ~~~







