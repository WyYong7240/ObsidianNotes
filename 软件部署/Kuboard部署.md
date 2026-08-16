---
tags:
  - kuboard
---

# 获取kuboard的yaml文件

> 安装版本为kuboard v3

~~~bash
kubectl apply -f https://addons.kuboard.cn/kuboard/kuboard-v3.yaml
# 您也可以使用下面的指令，唯一的区别是，该指令使用华为云的镜像仓库替代 docker hub 分发 Kuboard 所需要的镜像
# kubectl apply -f https://addons.kuboard.cn/kuboard/kuboard-v3-swr.yaml
~~~

# 修改Kuboard-v3.yaml配置

> 这是为了让kuboard部署在master上

修改Deployment配置

~~~yaml
..........
---
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations: {}
  labels:
    k8s.kuboard.cn/name: kuboard-v3
  name: kuboard-v3
  namespace: kuboard
spec:
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      k8s.kuboard.cn/name: kuboard-v3
  template:
    metadata:
      labels:
        k8s.kuboard.cn/name: kuboard-v3
    spec:
      nodeName: k8s-master #######增加这个为k8s得主节点node名称
      affinity:
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - preference:
                matchExpressions:
                  - key: node-role.kubernetes.io/master
                    operator: Exists
              weight: 100
            - preference:
                matchExpressions:
                  - key: node-role.kubernetes.io/control-plane
                    operator: Exists
              weight: 100
  ..........
~~~

# 部署kuboard

~~~BASH
kubectl apply -f kuboard-v3.yaml
~~~

# 查看创建情况

~~~BASH
[root@master ~]# kubectl get pods -n kuboard
NAME                               READY   STATUS    RESTARTS   AGE
kuboard-agent-2-65bc84c86c-r7tc4   1/1     Running   2          28s
kuboard-agent-78d594567-cgfp4      1/1     Running   2          28s
kuboard-etcd-fh9rp                 1/1     Running   0          67s
kuboard-etcd-nrtkr                 1/1     Running   0          67s
kuboard-etcd-ader3                 1/1     Running   0          67s
kuboard-v3-645bdffbf6-sbdxb        1/1     Running   0          67s

~~~

# 访问UI界面

* URL：http://masterIP:30080

  如果是公网，记得安全组开放30080、30081两个端口

* 账号：admin

* 密码：Kuboard123

* ![image.png](https://typora2icture.oss-cn-beijing.aliyuncs.com/img2/20250522151001.png)




# 常见问题

1. 如果结果中没有出现 `kuboard-etcd-xxxxx` 的容器，

   缺少 Master Role

   - 可能缺少 Master Role 的情况有：

     - 当您在 ***阿里云、腾讯云（以及其他云）托管*** 的 K8S 集群中以此方式安装 Kuboard 时，您执行 `kubectl get nodes` 将 ***看不到 master 节点***；

     - 当您的集群是通过二进制方式安装时，您的集群中可能缺少 Master Role，或者当您删除了 Master 节点的 `node-role.kubernetes.io/master=` 标签时，此时执行 `kubectl get nodes`，结果如下所示：

       ~~~BASH
       [root@k8s-19-master-01 ~]# kubectl get nodes
       NAME               STATUS   ROLES    AGE   VERSION
       k8s-19-master-01   Ready    <none>   19d   v1.19.11
       k8s-19-node-01     Ready    <none>   19d   v1.19.11
       k8s-19-node-02     Ready    <none>   19d   v1.19.11
       k8s-19-node-03     Ready    <none>   19d   v1.19.11
       ~~~

   - 在集群中缺少 Master Role 节点时，您也可以为一个或者三个 worker 节点添加 `k8s.kuboard.cn/role=etcd` 的标签，来增加 kuboard-etcd 的实例数量；

     - 执行如下指令，可以为 `your-node-name` 节点添加所需要的标签

       ~~~BASH
       kubectl label nodes your-node-name k8s.kuboard.cn/role=etcd
       ~~~

       > 此处建议如果只有一个集群的话，就把master设置上这个标签就行

2. 如果更改完后，没有`kuboard-agent-xxxx`容器

   建议多等一会，然后就是，只设置一个master具有k8s.kuboard.cn/role=etcd的标签

   然后通过edit kuboard-agent的deployment，避免其调度到边节点上

   1. 获取deployment

      ~~~BASH
      ➜  kuboard kubectl get deployment -n kuboard
      NAME              READY   UP-TO-DATE   AVAILABLE   AGE
      kuboard-agent     0/1     1            0           10m
      kuboard-agent-2   1/1     1            1           10m
      kuboard-v3        1/1     1            1           15m
      ~~~

   2. 编辑deployment

      ~~~BASH
      ➜  kuboard kubectl edit deployment kuboard-agent -n kuboard
      
      ###
       spec:
            affinity:
              nodeAffinity:
                preferredDuringSchedulingIgnoredDuringExecution:
                - preference:
                    matchExpressions:
                    - key: node-role.kubernetes.io/master
                      operator: Exists
                    - key: node-role.kubernetes.io/edge	### 新增内容
                      operator: DoesNotExist
                      
      
      ➜  kuboard kubectl edit deployment kuboard-agent-2 -n kuboard
      
      
      ###
       spec:
            affinity:
              nodeAffinity:
                preferredDuringSchedulingIgnoredDuringExecution:
                - preference:
                    matchExpressions:
                    - key: node-role.kubernetes.io/master
                      operator: Exists
                    - key: node-role.kubernetes.io/edge	### 新增内容
                      operator: DoesNotExist
      ~~~

   

   

   # 参考文档

   1. [安装 Kuboard v3 - kubernetes](https://kuboard.cn/install/v3/install-in-k8s.html#%E5%AE%89%E8%A3%85)

   
