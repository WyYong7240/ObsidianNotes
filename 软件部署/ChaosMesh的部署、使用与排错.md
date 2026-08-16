---
tags:
  - ChaosMesh
  - 异常注入
---

# 前提准备

1. Kubernetes集群：v1.28.15
2. Helm：v3.14.2



# 使用Helm安装ChaosMesh

1. 添加ChaosMesh仓库

   ~~~bash
   helm repo add chaos-mesh https://charts.chaos-mesh.org
   ~~~

2. 查看可以安装的ChaosMesh Charts版本

   ~~~bash
   helm search repo chaos-mesh
   ~~~

   > 上述命令会输出最新发布的 chart，如需安装历史版本，请执行如下命令查看所有的版本：
   >
   > ~~~bash
   > helm search repo chaos-mesh -l
   > ~~~

3. 创建ChaosMesh的命名空间

   ~~~bash
   kubectl create ns chaos-mesh
   ~~~

4. 在不同的环境下安装ChaosMesh

   * 容器运行时为Docker的Kubernetes集群：

     ~~~bash
     # Default to /var/run/docker.sock
     helm install chaos-mesh chaos-mesh/chaos-mesh -n=chaos-mesh --version 2.7.2
     ~~~

   * 容器运行时为containerd的Kubernetes集群：

     ~~~bash
     helm install chaos-mesh chaos-mesh/chaos-mesh -n=chaos-mesh --set chaosDaemon.runtime=containerd --set chaosDaemon.socketPath=/run/containerd/containerd.sock --version 2.7.2
     ~~~

   > 1. 此处暂不考虑MicroK8s、K3s、CRI-O
   >
   > 2. 如需安装特定版本的 Chaos Mesh，请在 `helm install` 后添加 `--version x.y.z` 参数，如 `helm install chaos-mesh chaos-mesh/chaos-mesh -n=chaos-mesh --version 2.1.0`。
   >
   > 3. 为了保证高可用性，Chaos Mesh 默认开启了 `leader-election` 特性。如果不需要这个特性，请通过 `--set controllerManager.leaderElection.enabled=false` 手动关闭该特性。
   >
   >    例如：
   >
   >    ~~~bash
   >    helm install chaos-mesh chaos-mesh/chaos-mesh -n=chaos-mesh --set chaosDaemon.runtime=containerd --set chaosDaemon.socketPath=/run/containerd/containerd.sock --version 2.7.2 --set controllerManager.leaderElection.enabled=false
   >    ~~~
   >
   >    
   >
   >    > 如果版本 `<2.6.1`，你仍然需要设置 `--set controllerManager.replicaCount=1` 来将控制器管理器减少到一个副本。

5. 验证安装

   ~~~bash
   (base) root@master:~/work/locust# kubectl get pod -n chaos-mesh -o wide
   NAME                                        READY   STATUS    RESTARTS       AGE     IP             NODE    NOMINATED NODE   READINESS GATES
   chaos-controller-manager-84d6dd87b7-2wmvh   1/1     Running   0              4h44m   10.244.3.177   node3   <none>           <none>
   chaos-controller-manager-84d6dd87b7-nk4v8   1/1     Running   0              4h44m   10.244.1.154   node1   <none>           <none>
   chaos-controller-manager-84d6dd87b7-ztc4w   1/1     Running   0              4h44m   10.244.2.156   node2   <none>           <none>
   chaos-daemon-dtxwf                          1/1     Running   0              4h44m   10.244.3.176   node3   <none>           <none>
   chaos-daemon-nsspm                          1/1     Running   0              4h44m   10.244.2.157   node2   <none>           <none>
   chaos-daemon-zgqpg                          1/1     Running   0              4h44m   10.244.1.153   node1   <none>           <none>
   chaos-dashboard-cdb6f54-68869               1/1     Running   0              4h57m   10.244.2.153   node2   <none>           <none>
   chaos-dns-server-864ccc6f4d-8zjpr           1/1     Running   2 (5d8h ago)   18d     10.244.2.63    node2   <none>           <none>
   
   (base) root@master:~/work/locust# kubectl get svc -n chaos-mesh
   NAME                            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                                 AGE
   chaos-daemon                    ClusterIP   None            <none>        31767/TCP,31766/TCP                     18d
   chaos-dashboard                 NodePort    10.101.41.111   <none>        2333:31255/TCP,2334:30843/TCP           18d
   chaos-mesh-controller-manager   ClusterIP   10.97.21.197    <none>        443/TCP,10081/TCP,10082/TCP,10080/TCP   18d
   chaos-mesh-dns-server           ClusterIP   10.101.1.245    <none>        53/UDP,53/TCP,9153/TCP,9288/TCP         18d
   ~~~

   访问chaos-dashboard后，如下：
   ![image.png](https://typora2icture.oss-cn-beijing.aliyuncs.com/img2/202507020024972.png)

   至于后面的RBAC鉴权登录，详见页面中的向导吧

# 升级、更改ChaosMesh初始设定

* 升级命令：

  ~~~bash
  helm upgrade chaos-mesh chaos-mesh/chaos-mesh
  ~~~

  > 如需升级至特定版本的 Chaos Mesh，请在 `helm upgrade` 后添加 `--version x.y.z` 参数，如 `helm upgrade chaos-mesh chaos-mesh/chaos-mesh -n=chaos-mesh --version 2.1.0`。
  >
  > 再例如：
  >
  > ~~~bash
  > helm upgrade chaos-mesh chaos-mesh/chaos-mesh -n chaos-mesh --set chaosDaemon.runtime=containerd --set chaosDaemon.socketPath=/run/containerd/containerd.sock --set chaosDaemon.privileged=true
  > ~~~

# 卸载ChaosMesh

* 卸载命令：

  ~~~bash
  helm uninstall chaos-mesh -n chaos-mesh
  ~~~



# ChaosMesh排错

## NetworkDelay时异常注入出现错误

* 问题描述

  模拟网络延迟故障时，chaos-mesh前端报错：`Failed to apply chaos: failed to apply for pod tt/ts-admin-route-service-7b957b9877-dhrkz: unable to flush ip sets for pod ts-admin-route-service-7b957b9877-dhrkz`

  并且之前创建的网络延迟异常注入无法删除
  
  ![image.png](https://typora2icture.oss-cn-beijing.aliyuncs.com/img2/202507020031586.png)
  
* 问题解释：

  * DeepSeek做如下猜测：

    在ChaosMesh中注入NetworkDelay时出现 `unable to flush ip sets for pod` 错误，通常与网络规则操作失败有关。以下是可能的原因及解决方案：

    1. **权限不足**

       - **问题**：ChaosMesh的Daemon Pod（`chaos-daemon`）需要特权权限修改目标Pod的网络命名空间（如操作iptables/ipset）。
       - **解决**：
         - 确保`chaos-daemon` Pod以特权模式运行：
           ```yaml
           # chaos-daemon.yaml (部分配置)
           securityContext:
             privileged: true  # 必须为true
             capabilities:
               add:
                 - NET_ADMIN   # 必须添加NET_ADMIN权限
           ```
         - 检查ServiceAccount权限：确认安装ChaosMesh时启用了`--set chaosDaemon.runtime=containerd --set chaosDaemon.privileged=true`（Helm安装参数）。

    2. **目标Pod状态异常**

       - **问题**：目标Pod非`Running`状态（如`Pending`/`CrashLoopBackOff`），导致无法注入。
       - **解决**：
         - 检查目标Pod状态：
           ```bash
           kubectl get pod -n tt ts-admin-route-service-7b957b9877-dhrkz -o wide
           ```
         - 确保Pod处于`Running`状态且Ready（非0/1）。

    3. **内核模块缺失**

       - **问题**：节点缺少`ipset`或`xt_set`内核模块，导致无法操作网络规则。
       - **解决**：
         - 在目标节点加载内核模块：
           ```bash
           sudo modprobe ipset    # 加载ipset模块
           sudo modprobe xt_set   # 加载xt_set模块
           ```
         - 持久化模块（防止重启失效）：
           ```bash
           echo "ipset" >> /etc/modules-load.d/chaos.conf
           echo "xt_set" >> /etc/modules-load.d/chaos.conf
           ```

    4. **NetworkChaos CR配置错误**

       - **问题**：NetworkChaos YAML中目标选择器（selector）配置错误。
       - **解决**：
         - 检查NetworkChaos配置，确保`selector`匹配目标Pod：
           ```yaml
           apiVersion: chaos-mesh.org/v1alpha1
           kind: NetworkChaos
           metadata:
             name: network-delay
             namespace: chaos-testing
           spec:
             action: delay
             mode: one           # 或指定Pod
             selector:
               namespaces: [tt]
               pods:
                 tt: [ts-admin-route-service-7b957b9877-dhrkz]  # 精确匹配Pod
             delay:
               latency: "10ms"
           ```

    5. **节点资源问题**

       - **问题**：节点资源不足（如可用网络命名空间耗尽）。
       - **解决**：
         - 检查节点资源：
           ```bash
           kubectl describe node <目标节点>
           ```
         - 重启目标节点释放资源（谨慎操作）。

  * 搜寻博客2、3，博客让我猜测可能是因为容器运行时的问题

    <img src="https://typora2icture.oss-cn-beijing.aliyuncs.com/img2/202507020037722.png" alt="image.png" style="zoom: 80%;" />
    
    <img src="https://typora2icture.oss-cn-beijing.aliyuncs.com/img2/202507020036999.png" alt="image.png" style="zoom:80%;" />
  
* 问题解决：

  综合上述两种可能原因，针对ChaosMesh做出如下的升级、更改配置

  ~~~bash
  helm upgrade chaos-mesh chaos-mesh/chaos-mesh -n chaos-mesh --set chaosDaemon.runtime=containerd --set chaosDaemon.socketPath=/run/containerd/containerd.sock --set chaosDaemon.privileged=true
  ~~~

  1. `chaosDaemon.runtime=containerd`:指定容器运行时为containerd
  2. `chaosDaemon.socketPath=/run/containerd/containerd.sock`:配置socketPath
  3. `chaosDaemon.privileged=true`:开启权限

* 验证问题是否解决

  由于每次`helm upgrade`都会让chaos-mesh的各个组件重启，因此需要查看更改配置后组件容器是否正常运行

  ~~~bash
  (base) root@master:~/work/locust# kubectl get pod -n chaos-mesh -o wide
  NAME                                        READY   STATUS    RESTARTS       AGE     IP             NODE    NOMINATED NODE   READINESS GATES
  chaos-controller-manager-84d6dd87b7-2wmvh   1/1     Running   0              5h4m    10.244.3.177   node3   <none>           <none>
  chaos-controller-manager-84d6dd87b7-nk4v8   1/1     Running   0              5h4m    10.244.1.154   node1   <none>           <none>
  chaos-controller-manager-84d6dd87b7-ztc4w   1/1     Running   0              5h4m    10.244.2.156   node2   <none>           <none>
  chaos-daemon-dtxwf                          1/1     Running   0              5h4m    10.244.3.176   node3   <none>           <none>
  chaos-daemon-nsspm                          1/1     Running   0              5h4m    10.244.2.157   node2   <none>           <none>
  chaos-daemon-zgqpg                          1/1     Running   0              5h4m    10.244.1.153   node1   <none>           <none>
  chaos-dashboard-cdb6f54-68869               1/1     Running   0              5h17m   10.244.2.153   node2   <none>           <none>
  chaos-dns-server-864ccc6f4d-8zjpr           1/1     Running   2 (5d8h ago)   18d     10.244.2.63    node2   <none>           <none>
  ~~~

  经过再次创建NetworkDelay实验，发现问题已经解决



# 参考文章

1. https://chaos-mesh.org/zh/docs/production-installation-using-helm/
2. https://www.jianshu.com/p/4d6cda60afe2
3. https://github.com/chaos-mesh/chaos-mesh/issues/2300
