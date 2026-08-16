---
tags:
  - rook-ceph
---

# 部署的前提条件

* 原始设备，即无分区未格式化的文件系统
* 原始分五，即未格式化的文件系统
* LVM逻辑卷，即未格式化的文件系统
* 加密设备，即未格式化的文件系统
* 多路设备，即未格式化的文件系统
* 从存储类中以block模式可用的持久卷

> 其实就是得有个没有格式化的分区或者一块硬盘，是为了让rook-ceph使用的

* k8s：1.28.15
* rook-ceph：1.17.2

> 由于是在裸机上部署，即非云环境中，或者测试环境，所以至少需要三个工作节点，master、node1、node2、node3

# 部署rook-ceph

* 获取rook-ceph

  ~~~bash
  git clone --single-branch --branch v1.17.2 https://github.com/rook/rook.git
  ~~~

* 部署operator

  ~~~bash
  cd rook/deploy/examples
  kubectl create -f crds.yaml -f common.yaml -f operator.yaml
  ~~~

  部署完成后：

  ~~~bash
  $ kubectl -n rook-ceph get pod
  NAME                                                 READY   STATUS      RESTARTS   AGE
  rook-ceph-operator-85f5b946bd-s8grz                  1/1     Running     0          92m
  ~~~

* 部署rook-ceph Cluster

  * 如果需要指定部署的节点和设备，则在`cluster.yaml`中的storage.nodes指定，格式如下

    ~~~yaml
    storage:
      nodes:
        - name: "node1"
          devices: # specific devices to use for storage can be specified for each node
            - name: "sdb"
        - name: "node2"
          devices: # specific devices to use for storage can be specified for each node
            - name: "sdb"
        - name: "node3"
          devices: # specific devices to use for storage can be specified for each node
            - name: "sdb"
    ~~~

    其中，`devices.name`是各个节点中，未格式化的磁盘设备名称

    在Ubuntu中，使用如下命令查看

    ~~~bash
    lsblk -o NAME,SIZE,TYPE,MOUNTPOINT
    ~~~

    输入示例如下：

    ~~~bash
    NAME    SIZE TYPE MOUNTPOINT
    sda     50G  disk
    ├─sda1  1G   part /boot/efi
    ├─sda2  2G   part /boot
    └─sda3 47G   part /
    sdb    100G  disk
    vda    200G  disk
    ~~~

  * 如果不指定部署的节点和设备，默认配置是

    ~~~yaml
      storage: # cluster level storage configuration and selection
        useAllNodes: true
        useAllDevices: true
        nodes:
        #deviceFilter:
    ......
    ~~~
    
    这样的话就会使用所有可用节点上的所有可用设备
    
  * 部署完成后：
  
    ~~~bash
    root@master:~/software/rook/deploy/examples# kubectl get pod -n rook-ceph -o wide
    NAME                                              READY   STATUS      RESTARTS      AGE   IP              NODE    NOMINATED NODE   READINESS GATES
    csi-cephfsplugin-cltv5                            3/3     Running     1 (22m ago)   22m   192.168.3.224   node2   <none>           <none>
    csi-cephfsplugin-jtd9v                            3/3     Running     1 (22m ago)   22m   192.168.3.229   node1   <none>           <none>
    csi-cephfsplugin-provisioner-79bcdb8bdf-662v4     6/6     Running     1 (21m ago)   22m   10.244.3.3      node3   <none>           <none>
    csi-cephfsplugin-provisioner-79bcdb8bdf-xn9pz     6/6     Running     2 (16m ago)   22m   10.244.2.3      node2   <none>           <none>
    csi-cephfsplugin-wtnf4                            3/3     Running     1 (22m ago)   22m   192.168.3.228   node3   <none>           <none>
    csi-rbdplugin-4dd5x                               3/3     Running     1 (22m ago)   22m   192.168.3.224   node2   <none>           <none>
    csi-rbdplugin-fp6wt                               3/3     Running     1 (22m ago)   22m   192.168.3.229   node1   <none>           <none>
    csi-rbdplugin-provisioner-57f48bdc79-klqxf        6/6     Running     1 (21m ago)   22m   10.244.3.2      node3   <none>           <none>
    csi-rbdplugin-provisioner-57f48bdc79-p5wl4        6/6     Running     2 (16m ago)   22m   10.244.1.2      node1   <none>           <none>
    csi-rbdplugin-qpf6q                               3/3     Running     1 (22m ago)   22m   192.168.3.228   node3   <none>           <none>
    rook-ceph-crashcollector-node1-647575f746-jg5xh   1/1     Running     0             15m   10.244.1.11     node1   <none>           <none>
    rook-ceph-crashcollector-node2-569989ddfc-4w8gd   1/1     Running     0             15m   10.244.2.11     node2   <none>           <none>
    rook-ceph-crashcollector-node3-f6957ddd8-4sggc    1/1     Running     0             15m   10.244.3.6      node3   <none>           <none>
    rook-ceph-exporter-node1-67894c88c6-hvtr7         1/1     Running     0             15m   10.244.1.12     node1   <none>           <none>
    rook-ceph-exporter-node2-859ffc4d7f-m85q7         1/1     Running     0             15m   10.244.2.12     node2   <none>           <none>
    rook-ceph-exporter-node3-7f58fb56bb-2njsz         1/1     Running     0             15m   10.244.3.7      node3   <none>           <none>
    rook-ceph-mgr-a-84bfcd4d9d-5zdtr                  3/3     Running     0             15m   10.244.1.6      node1   <none>           <none>
    rook-ceph-mgr-b-646f8bc8c-pnhlw                   3/3     Running     0             15m   10.244.2.6      node2   <none>           <none>
    rook-ceph-mon-a-c6fb4cd4c-gzvpp                   2/2     Running     0             17m   10.244.1.5      node1   <none>           <none>
    rook-ceph-mon-b-7545dcf99f-8ktgl                  2/2     Running     0             16m   10.244.2.5      node2   <none>           <none>
    rook-ceph-mon-c-66ddfffc47-k4tvs                  2/2     Running     0             16m   10.244.3.5      node3   <none>           <none>
    rook-ceph-operator-5fc87d4cc8-hk5xg               1/1     Running     0             28m   10.244.2.2      node2   <none>           <none>
    rook-ceph-osd-0-676656bd75-rmph6                  2/2     Running     0             15m   10.244.1.10     node1   <none>           <none>
    rook-ceph-osd-1-7d5ddd778d-f28c9                  2/2     Running     0             15m   10.244.2.10     node2   <none>           <none>
    rook-ceph-osd-2-797b599f4f-wq78z                  2/2     Running     0             15m   10.244.3.9      node3   <none>           <none>
    rook-ceph-osd-prepare-node1-8rs87                 0/1     Completed   0             15m   10.244.1.9      node1   <none>           <none>
    rook-ceph-osd-prepare-node2-6n5lz                 0/1     Completed   0             15m   10.244.2.9      node2   <none>           <none>
    rook-ceph-osd-prepare-node3-wphm6                 0/1     Completed   0             15m   10.244.3.8      node3   <none>           <none>
    ~~~
    
    > 如果这里部署过程中，一开始出现`rook-ceph-mon-a-canry`、`rook-ceph-mon-a`之类的Pod出现了Pending的情况的话，那就是因为集群中没有三个工作节点，即不包含master节点还得有三个节点，rook-ceph的Pod好像都是不部署在master节点上的，如果想要解决这种情况，强制使用只有两个工作节点的集群的话
    >
    > 1. 获取类Pod的deployment
    >
    >    ~~~bash
    >    kubectl get deployment -n rook-ceph
    >    ~~~
    >
    > 2. 编辑deployment
    >
    >    ~~~bash
    >    kubectl edit deployment rook-ceph-mon-a -n rook-ceph
    >    ~~~
    >
    >    然后在对应的地方加上容忍
    >
    >    ~~~bash
    >          tolerations:
    >          - key: "node-role.kubernetes.io/control-plane"
    >            operator: "Exists"
    >            effect: "NoSchedule"
    >    ~~~
    >
    >    使得此类Pod可以在master节点上运行
    >
    > 但是注意，就算此类Pod成功运行了，在运行完毕后，只有两个工作节点的集群，如果每个工作节点上只预留了一个未使用磁盘的话，最终也只会有两个OSD Pod，
    >
    > 这样其实是不符合rook-ceph集群的要求的
  
* 部署dashboard的访问

  由于1.17.2版本默认开启的dashboard的部署，在svc中可以看到mgr-dashboard

  ~~~bash
  root@master:~/software/rook/deploy/examples# kubectl get svc -n rook-ceph
  NAME                                     TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)             AGE
  rook-ceph-exporter                       ClusterIP   10.99.112.86     <none>        9926/TCP            19m
  rook-ceph-mgr                            ClusterIP   10.108.169.32    <none>        9283/TCP            19m
  rook-ceph-mgr-dashboard                  ClusterIP   10.110.219.54    <none>        8443/TCP            19m
  rook-ceph-mon-a                          ClusterIP   10.106.155.199   <none>        6789/TCP,3300/TCP   21m
  rook-ceph-mon-b                          ClusterIP   10.109.54.25     <none>        6789/TCP,3300/TCP   20m
  rook-ceph-mon-c                          ClusterIP   10.110.216.173   <none>        6789/TCP,3300/TCP   20m
  ~~~

  默认设置为了clusterIP，且不可改变，因此部署exampls文件夹下的external-https服务

  ~~~bash
  kubectl apply -f dashboard-external-https.yaml
  ~~~

  这样就变为

  ~~~bash
  root@master:~/software/rook/deploy/examples# kubectl get svc -n rook-ceph
  NAME                                     TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)             AGE
  rook-ceph-exporter                       ClusterIP   10.99.112.86     <none>        9926/TCP            19m
  rook-ceph-mgr                            ClusterIP   10.108.169.32    <none>        9283/TCP            19m
  rook-ceph-mgr-dashboard                  ClusterIP   10.110.219.54    <none>        8443/TCP            19m
  rook-ceph-mgr-dashboard-external-https   NodePort    10.108.195.91    <none>        8443:31558/TCP      17m
  rook-ceph-mon-a                          ClusterIP   10.106.155.199   <none>        6789/TCP,3300/TCP   21m
  rook-ceph-mon-b                          ClusterIP   10.109.54.25     <none>        6789/TCP,3300/TCP   20m
  rook-ceph-mon-c                          ClusterIP   10.110.216.173   <none>        6789/TCP,3300/TCP   20m
  ~~~

  然后访问`https://<hostIP>:31558`,用户名为`admin`,密码使用如下命令获取：

  ~~~bash
  kubectl -n rook-ceph get secret rook-ceph-dashboard-password -o jsonpath="{['data']['password']}" | base64 --decode && echo
  ~~~

  最终在界面中为：
  <img src="https://typora2icture.oss-cn-beijing.aliyuncs.com/img2/20250528145251.png" alt="image.png" style="zoom:67%;" />
  
  可以看到我们余留的三个工作节点中的未格式化磁盘在使用中
  
* toolbox部署

  ~~~bash
  kubectl apply -f deploy/examples/toolbox.yaml
  ~~~

  pod中增加：

  ~~~bash
  rook-ceph-tools-56fbc74755-jb6wp                  1/1     Running     0             12m   10.244.1.13     node1   <none>           <none>
  ~~~

  使用如下命令进入toolbox

  ~~~bash
  kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash
  ~~~

  进入后，使用命令`ceph status`查看rook-ceph集群使用情况

  ~~~bash
  bash-5.1$ ceph status
    cluster:
      id:     0a3001db-00de-4b1f-bf64-ad8c4fd4d733
      health: HEALTH_WARN
              clock skew detected on mon.c
  
    services:
      mon: 3 daemons, quorum a,b,c (age 27m)
      mgr: a(active, since 26m), standbys: b
      osd: 3 osds: 3 up (since 26m), 3 in (since 26m)
  
    data:
      pools:   2 pools, 33 pgs
      objects: 4 objects, 577 KiB
      usage:   81 MiB used, 30 GiB / 30 GiB avail
      pgs:     33 active+clean
  ~~~

# 部署storage-class并使用rook-ceph

* 部署storage-class.yaml

  ~~~bash
  kubectl create -f deploy/examples/csi/rbd/storageclass.yaml
  ~~~

  其中storage-class.yaml中的

  ~~~yaml
  apiVersion: ceph.rook.io/v1
  kind: CephBlockPool
  metadata:
    name: replicapool
    namespace: rook-ceph
  spec:
    failureDomain: host
    replicated:
      size: 3
  ~~~

  `replicated.size`的数目，要小于等于osd的个数，也就是小于等于我们工作节点中预留的磁盘的数量

  完成后，可以看到

  ~~~bash
  root@master:~/software/rook/deploy/examples/csi/rbd# kubectl get sc -A
  NAME              PROVISIONER                  RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
  rook-ceph-block   rook-ceph.rbd.csi.ceph.com   Delete          Immediate           true                   88s
  ~~~

  

* Pod使用rook-ceph以word-press.yaml为例

  第二部分创建PVC的`spec.storageClassName`中使用了我们创建的`rook-ceph-block`，然后在后面的Pod使用中，在volumes中声明使用的第二部分定义的PVC

  ~~~yaml
  apiVersion: v1
  kind: Service
  metadata:
    name: wordpress
    labels:
      app: wordpress
  spec:
    ports:
      - port: 80
    selector:
      app: wordpress
      tier: frontend
    #type: LoadBalancer
    type: NodePort  #为了方便测试，我这里改成NodePort模式
  ---
  apiVersion: v1
  kind: PersistentVolumeClaim
  metadata:
    name: wp-pv-claim  #创建pvc
    labels:
      app: wordpress
  spec:
    storageClassName: rook-ceph-block  #声明为我们之前创建的块存储
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: 20Gi
  ---
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: wordpress
    labels:
      app: wordpress
      tier: frontend
  spec:
    selector:
      matchLabels:
        app: wordpress
        tier: frontend
    strategy:
      type: Recreate
    template:
      metadata:
        labels:
          app: wordpress
          tier: frontend
      spec:
        containers:
          - image: wordpress:4.6.1-apache
            name: wordpress
            env:
              - name: WORDPRESS_DB_HOST
                value: wordpress-mysql
              - name: WORDPRESS_DB_PASSWORD
                value: changeme
            ports:
              - containerPort: 80
                name: wordpress
            volumeMounts:
              - name: wordpress-persistent-storage
                mountPath: /var/www/html
        volumes:
          - name: wordpress-persistent-storage
            persistentVolumeClaim:
              claimName: wp-pv-claim
  ~~~

  

