---
tags:
  - kube-prometheus
  - prometheus
  - blackbox-exporter
  - 网站监测
---

# 前期准备

1. kube-prometheus的安装
  [[一些软件的部署#Prometheus部署]]

2. 部署完成后，可以看到对应的Pod、Service

  ~~~bash
  (base) root@master:~/work/locust# kubectl get pod -n monitoring -o wide
  NAME                                   READY   STATUS    RESTARTS       AGE   IP              NODE     NOMINATED NODE   READINESS GATES
  alertmanager-main-0                    2/2     Running   4 (5d8h ago)   26d   10.244.2.56     node2    <none>           <none>
  alertmanager-main-1                    2/2     Running   4 (5d8h ago)   26d   10.244.3.75     node3    <none>           <none>
  alertmanager-main-2                    2/2     Running   4 (5d8h ago)   26d   10.244.1.54     node1    <none>           <none>
  blackbox-exporter-6cfc4bffb6-wn9gw     3/3     Running   6 (5d8h ago)   26d   10.244.2.60     node2    <none>           <none>
  grafana-748964b847-sv972               1/1     Running   2 (5d8h ago)   26d   10.244.1.57     node1    <none>           <none>
  kube-state-metrics-6b4d48dcb4-42czx    3/3     Running   6 (5d8h ago)   26d   10.244.3.79     node3    <none>           <none>
  node-exporter-2pc2w                    2/2     Running   4 (5d8h ago)   26d   192.168.3.226   master   <none>           <none>
  node-exporter-dwk9m                    2/2     Running   4 (5d8h ago)   26d   192.168.3.229   node1    <none>           <none>
  node-exporter-jvkpc                    2/2     Running   4 (5d8h ago)   26d   192.168.3.228   node3    <none>           <none>
  node-exporter-z525p                    2/2     Running   4 (5d8h ago)   26d   192.168.3.224   node2    <none>           <none>
  prometheus-adapter-796986659f-lkfst    1/1     Running   2 (5d8h ago)   14d   10.244.1.55     node1    <none>           <none>
  prometheus-adapter-796986659f-p4vnw    1/1     Running   2 (5d8h ago)   14d   10.244.3.77     node3    <none>           <none>
  prometheus-k8s-0                       2/2     Running   4 (5d8h ago)   26d   10.244.2.58     node2    <none>           <none>
  prometheus-k8s-1                       2/2     Running   4 (5d8h ago)   26d   10.244.1.53     node1    <none>           <none>
  prometheus-operator-68f6c79f9d-wldmc   2/2     Running   4 (5d8h ago)   26d   10.244.2.65     node2    <none>           <none>
  
  (base) root@master:~/work/locust# kubectl get svc -n monitoring
  NAME                    TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                          AGE
  alertmanager-main       ClusterIP   10.111.115.93    <none>        9093/TCP,8080/TCP                26d
  alertmanager-operated   ClusterIP   None             <none>        9093/TCP,9094/TCP,9094/UDP       26d
  blackbox-exporter       NodePort    10.97.201.27     <none>        9115:32098/TCP,19115:31154/TCP   26d
  grafana                 NodePort    10.104.230.62    <none>        3000:31228/TCP                   26d
  kube-state-metrics      ClusterIP   None             <none>        8443/TCP,9443/TCP                26d
  node-exporter           ClusterIP   None             <none>        9100/TCP                         26d
  prometheus-adapter      ClusterIP   10.96.239.127    <none>        443/TCP                          26d
  prometheus-k8s          NodePort    10.100.239.146   <none>        9090:31739/TCP,8080:30136/TCP    26d
  prometheus-operated     ClusterIP   None             <none>        9090/TCP                         26d
  prometheus-operator     ClusterIP   None             <none>        8443/TCP                         26d
  ~~~

  注：Prometheus-k8s、blackbox-exporter、grafana这三个Service的类型是NodePort是后来手动编辑对应配置文件改的，并不是一开始安装就是这样



# 部署Probe

> 其实应该是ServiceMonitor也可以的，但是找到的ServiceMonitor的配置并没有起作用，所以暂时先使用Probe方式
>
> 当然，找到的ServiceMonitor的配置也会贴在后面

* `blackbox-exporter-probe.yaml`

  ~~~yaml
  apiVersion: monitoring.coreos.com/v1V
  kind: Probe
  metadata:
    name: test-blackbox-monitoring
    namespace: monitoring
  spec:
    interval: 5s
    module: http_2xx
    prober:
      url: blackbox-exporter.monitoring.svc.cluster.local:19115
    targets:
      staticConfig:
        static:
          - http://192.168.3.226:32241
          - http://192.168.3.226:32677
          - http://192.168.3.226:32677/index.html
          - http://192.168.3.226:32677/client_login.html
          - http://192.168.3.226:32677/client_order_list.html
          - http://192.168.3.226:32677/client_consign_list.html
          - http://192.168.3.226:32677/client_adsearch.html
          - http://192.168.3.226:32677/client_ticket_collect.html
          - http://192.168.3.226:32677/client_enter_station.htmj
          - http://192.168.3.226:32677/upload_avatar.html
  ~~~

* 应用该配置

  ~~~bash
  kubectl apply -f blackbox-exporter-probe.yaml
  ~~~

* 等待片刻，在Prometheus的控制界面上，查看targets
  ![image.png](https://typora2icture.oss-cn-beijing.aliyuncs.com/img2/202507020000092.png)

  可以看到，Prometheus成功获取到了对应的监控指标
  
* blackbox-exporter提供的指标详解

  blackbox-exporter提供了如下指标
  ![image.png](https://typora2icture.oss-cn-beijing.aliyuncs.com/img2/202507020006701.png)

  1. **probe_dns_lookup_time_seconds**：表示DNS查询的时间（以秒为单位），即从发起DNS请求到收到响应所花费的时间。
  
  2. **probe_duration_seconds**：表示整个探测过程的总时长（以秒为单位），包括DNS解析、TCP连接建立、HTTP请求发送和接收响应等所有步骤。
  
  3. **probe_failed_due_to_regex**：当使用正则表达式检查HTTP响应内容时，如果匹配失败，则该指标值为1，否则为0。
  
  4. **probe_http_content_length**：表示HTTP响应的内容长度（字节数）。
  
  5. **probe_http_duration_seconds**：表示HTTP请求的处理时间（以秒为单位），从发送HTTP请求到接收到完整响应的时间。
  
  6. **probe_http_redirects**：表示在获取最终响应过程中经历的重定向次数。
  
  7. **probe_http_ssl**：表示是否使用了HTTPS协议进行通信，值为1表示使用了SSL/TLS加密，值为0表示未使用。
  
  8. **probe_http_status_code**：表示HTTP响应的状态码，如200表示成功，404表示未找到资源等。
  
  9. **probe_http_uncompressed_body_length**：表示HTTP响应体解压缩后的长度（字节数）。
  
  10. **probe_http_version**：表示HTTP协议版本，如“1.1”或“2”。
  
  11. **probe_ip_addr_hash**：表示目标IP地址的哈希值，用于唯一标识被探测的目标。
  
  12. **probe_ip_protocol**：表示使用的网络协议类型，如IPv4或IPv6。
  
  13. **probe_success**：表示探测是否成功，值为1表示成功，值为0表示失败。
  
  14. **prober_probe_duration_seconds_bucket**、**prober_probe_duration_seconds_count**、**prober_probe_duration_seconds_sum**：这三个指标与`probe_duration_seconds`相关，分别表示探测时长的直方图桶、计数和总和，用于统计分析探测时长的分布情况。
  



# ServiceMonitor方式

* 示例ServiceMonitor配置：

  ~~~yaml
  apiVersion: monitoring.coreos.com/vV
  kind: ServiceMonitor
  metadata:
    name: blackbox-http-monitoring
    namespace: monitoring
    labels:
      release: prometheus-stack
  spec:
    namespaceSelector:
      matchNames:
        - monitoring
    selector:
      matchLabels:
        app.kubernetes.io/name: blackbox-exporter
    jobLabel: service-probe
    endpoints:
      - params:
          module:
            - http_2xx
          target:
            - http://192.168.3.226:3000
            - http://192.168.3.226:32677
          path: /probe
          port: probe
  ~~~

  



# 参考文章

* https://blog.csdn.net/qq_40907977/article/details/113241723
* https://www.modb.pro/db/423353
* https://blog.csdn.net/weixin_37708418/article/details/125842458
* https://blog.csdn.net/weixin_37708418/article/details/125842458
