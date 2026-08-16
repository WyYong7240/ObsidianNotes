---
tags:
  - Prometheus-adapter
  - custom-metrics
  - kube-apiservice
  - HPA
---

# 前期准备

> 本文的步骤，要求已经能够在Prometheus的web控制台查询到自己想要使用的自定义指标，例如我想使用文章 [[构建容器镜像#构建镜像代码]]中代码暴露的自定义指标`myapp_requests_total`,以 **每30s访问该服务的平均访问次数** 为HPA的自动扩展指标
>
> 总体流程图：
>[[Prometheus自定义指标的实现方案图解]]
> <img src="https://typora2icture.oss-cn-beijing.aliyuncs.com/img2/202506171459662.png" alt="image.png" style="zoom:50%;" />

>
> 总体步骤如下：

1. 编写相对应的用于暴露自定义指标的代码，并将其打包为容器镜像，部署在集群中
  [[构建容器镜像]]
2. 将该暴露出来的自定义指标，添加到Prometheus的监控列表中
  [[KubePrometheus添加监控目标target]]
3. 使用Prometheus-adapter将该自定义指标聚合到Kubernetes集群的本身的apiservice中，让HPA能够查到
  [[使用Prometheus-adapter提供custom-metrics实现自定义指标HPA#配置Prometheus-adapter与APIService]]
4. 使用HPA进行自动扩展
  [[HPA部署与使用]]

# 配置Prometheus-adapter与APIService

> 1. Kubernetes v1.28.15：[[搭建Kubernetes集群ECS]]
> 2. kube-prometheus v0.13.0：[[一些软件的部署#Prometheus部署]]

## Prometheus-adapter配置

* 以kube-prometheus v0.13.0为例，其安装不再详述，详情见[github](https://github.com/prometheus-operator/kube-prometheus/tree/release-0.13)

  1. 进入部署文件夹

     ~~~bash
     (base) root@master:~# cd software/kube-prometheus-0.13.0/manifests/
     (base) root@master:~/software/kube-prometheus-0.13.0/manifests# ls
     alertmanager-alertmanager.yaml            grafana-dashboardDefinitions.yaml                                kubeStateMetrics-networkPolicy.yaml         prometheusOperator-clusterRole.yaml
     alertmanager-networkPolicy.yaml           grafana-dashboardSources.yaml                                    kubeStateMetrics-prometheusRule.yaml        prometheusOperator-deployment.yaml
     alertmanager-podDisruptionBudget.yaml     grafana-deployment.yaml                                          kubeStateMetrics-serviceAccount.yaml        prometheusOperator-networkPolicy.yaml
     alertmanager-prometheusRule.yaml          grafana-networkPolicy.yaml                                       kubeStateMetrics-serviceMonitor.yaml        prometheusOperator-prometheusRule.yaml
     alertmanager-secret.yaml                  grafana-prometheusRule.yaml                                      kubeStateMetrics-service.yaml               prometheusOperator-serviceAccount.yaml
     alertmanager-serviceAccount.yaml          grafana-serviceAccount.yaml                                      nodeExporter-clusterRoleBinding.yaml        prometheusOperator-serviceMonitor.yaml
     alertmanager-serviceMonitor.yaml          grafana-serviceMonitor.yaml                                      nodeExporter-clusterRole.yaml               prometheusOperator-service.yaml
     alertmanager-service.yaml                 grafana-service.yaml                                             nodeExporter-daemonset.yaml                 prometheus-podDisruptionBudget.yaml
     blackboxExporter-clusterRoleBinding.yaml  kubePrometheus-prometheusRule.yaml                               nodeExporter-networkPolicy.yaml             prometheus-prometheusRule.yaml
     blackboxExporter-clusterRole.yaml         kubernetesControlPlane-prometheusRule.yaml                       nodeExporter-prometheusRule.yaml            prometheus-prometheus.yaml
     blackboxExporter-configuration.yaml       kubernetesControlPlane-serviceMonitorApiserver.yaml              nodeExporter-serviceAccount.yaml            prometheus-roleBindingConfig.yaml
     blackboxExporter-deployment.yaml          kubernetesControlPlane-serviceMonitorCoreDNS.yaml                nodeExporter-serviceMonitor.yaml            prometheus-roleBindingSpecificNamespaces.yaml
     blackboxExporter-networkPolicy.yaml       kubernetesControlPlane-serviceMonitorKubeControllerManager.yaml  nodeExporter-service.yaml                   prometheus-roleConfig.yaml
     blackboxExporter-serviceAccount.yaml      kubernetesControlPlane-serviceMonitorKubelet.yaml                prometheus-adapter                          prometheus-roleSpecificNamespaces.yaml
     blackboxExporter-serviceMonitor.yaml      kubernetesControlPlane-serviceMonitorKubeScheduler.yaml          prometheus-clusterRoleBinding.yaml          prometheus-serviceAccount.yaml
     blackboxExporter-service.yaml             kubeStateMetrics-clusterRoleBinding.yaml                         prometheus-clusterRole.yaml                 prometheus-serviceMonitor.yaml
     grafana-config.yaml                       kubeStateMetrics-clusterRole.yaml                                prometheus-networkPolicy.yaml               prometheus-service.yaml
     grafana-dashboardDatasources.yaml         kubeStateMetrics-deployment.yaml                                 prometheusOperator-clusterRoleBinding.yaml  setup
     
     ~~~

     > 这里提前将所有关于PrometheusAdapter的文件都放入了prometheus-adapter文件夹中了

     ~~~bash
     (base) root@master:~/software/kube-prometheus-0.13.0/manifests# cd prometheus-adapter/
     (base) root@master:~/software/kube-prometheus-0.13.0/manifests/prometheus-adapter# ls
     prometheusAdapter-apiService.yaml                          prometheusAdapter-clusterRoleServerResources.yaml  prometheusAdapter-networkPolicy.yaml          prometheusAdapter-serviceMonitor.yaml
     prometheusAdapter-clusterRoleAggregatedMetricsReader.yaml  prometheusAdapter-clusterRole.yaml                 prometheusAdapter-podDisruptionBudget.yaml    prometheusAdapter-service.yaml
     prometheusAdapter-clusterRoleBindingDelegator.yaml         prometheusAdapter-configMap.yaml                   prometheusAdapter-roleBindingAuthReader.yaml
     prometheusAdapter-clusterRoleBinding.yaml                  prometheusAdapter-deployment.yaml                  prometheusAdapter-serviceAccount.yaml
     ~~~

  2. 编辑`prometheusAdapter-configMap.yaml`

     这里注意configMap中，Prometheus-adapter的配置格式，这里给出[github上给的样例配置格式](https://github.com/kubernetes-sigs/prometheus-adapter/blob/master/docs/walkthrough.md)

     `prom-adapter.config.yaml`

     ~~~yaml
     apiVersion: v1
     kind: ConfigMap
     metadata:
       name: adapter-config
       namespace: monitoring
     data:
       config.yaml: |-
         "rules":
         - "seriesQuery": |
              {namespace!="",__name__!~"^container_.*"}
           "resources":
             "template": "<<.Resource>>"
           "name":
             "matches": "^(.*)_total"
             "as": ""
           "metricsQuery": |
             sum by (<<.GroupBy>>) (
               irate (
                 <<.Series>>{<<.LabelMatchers>>}[1m]
               )
             )
     ~~~

     如下是题主编辑完自定义指标后的`prometheusAdapter-configmap.yaml`的完整文件内容

     ~~~yaml
     apiVersion: v1
     data:
       config.yaml: |-
         "rules":
         - "seriesQuery": |
              myapp_requests_total{namespace="k8s-learn",service="my-custom-metrics-service",container="my-custom-metrics-app"}
           "resources":
             "overrides":
               "namespace":
                 "resource": "namespace"
               "node":
                 "resource": "node"
               "service":
                 "resource": "service"
               "pod":
                 "resource": "pod"
           "name":
             "matches": "^(.*)_total"
             "as": "service_requests_per_30s"
           "metricsQuery": |
             sum(increase(<<.Series>>{<<.LabelMatchers>>}[30s])) by (service, namespace)
         "resourceRules":
           "cpu":
             "containerLabel": "container"
             "containerQuery": |
               sum by (<<.GroupBy>>) (
                 irate (
                     container_cpu_usage_seconds_total{<<.LabelMatchers>>,container!="",pod!=""}[120s]
                 )
               )
             "nodeQuery": |
               sum by (<<.GroupBy>>) (
                 1 - irate(
                   node_cpu_seconds_total{mode="idle"}[60s]
                 )
                 * on(namespace, pod) group_left(node) (
                   node_namespace_pod:kube_pod_info:{<<.LabelMatchers>>}
                 )
               )
               or sum by (<<.GroupBy>>) (
                 1 - irate(
                   windows_cpu_time_total{mode="idle", job="windows-exporter",<<.LabelMatchers>>}[4m]
                 )
               )
             "resources":
               "overrides":
                 "namespace":
                   "resource": "namespace"
                 "node":
                   "resource": "node"
                 "pod":
                   "resource": "pod"
           "memory":
             "containerLabel": "container"
             "containerQuery": |
               sum by (<<.GroupBy>>) (
                 container_memory_working_set_bytes{<<.LabelMatchers>>,container!="",pod!=""}
               )
             "nodeQuery": |
               sum by (<<.GroupBy>>) (
                 node_memory_MemTotal_bytes{job="node-exporter",<<.LabelMatchers>>}
                 -
                 node_memory_MemAvailable_bytes{job="node-exporter",<<.LabelMatchers>>}
               )
               or sum by (<<.GroupBy>>) (
                 windows_cs_physical_memory_bytes{job="windows-exporter",<<.LabelMatchers>>}
                 -
                 windows_memory_available_bytes{job="windows-exporter",<<.LabelMatchers>>}
               )
             "resources":
               "overrides":
                 "instance":
                   "resource": "node"
                 "namespace":
                   "resource": "namespace"
                 "pod":
                   "resource": "pod"
           "window": "5m"
     kind: ConfigMap
     metadata:
       labels:
         app.kubernetes.io/component: metrics-adapter
         app.kubernetes.io/name: prometheus-adapter
         app.kubernetes.io/part-of: kube-prometheus
         app.kubernetes.io/version: 0.11.1
       name: adapter-config
       namespace: monitoring
     ~~~

  3. 应用新配置文件，并重启Prometheus-adapter
  
     * 应用新配置文件
  
       ~~~bash
       kubectl apply -f prometheusAdapter-configMap.yaml
       ~~~
  
     * 重启Prometheus-adapter
  
       ~~~bash
       kubectl rollout restart deploy prometheus-adapter -n monitoring
       ~~~
  
       1. 如果此处发生了Prometheus-adapter Pod不断重启进入`CrashLoopOff`状态，说明`prometheusAdapter-configmap.yaml`新增加的配置部分格式不正确，尝试重新配置应用后再重启Prometheus-adapter deployment
  
       2. 经过实验，Prometheus-adapter deployment仅识别名为**adapter-config**且在与Prometheus-adapter同命名空间下的config-map，所以，在同命名空间下重建一个新的名为其他名字的config-map是不能被Prometheus-adapter识别的
  
       3. ==最重要的一条，只有当Prometheus-adapter配置的自定义规则和指标，例如上述配置文件中的`rules`规则，后面的APIServcie配置完成后，该custom-metrics APIService才能从Prometheus-adapter的服务中找到对应的服务端点==
  
          否则会出现如图的错误：
          ![4dc347e29c10b7aa54c7e4885b02b72.png](https://typora2icture.oss-cn-beijing.aliyuncs.com/img2/202506171305768.png)

          详细描述为：
          ![image.png](https://typora2icture.oss-cn-beijing.aliyuncs.com/img2/202506171307908.png)


### 注意点

* 不同形式的PrometheusAdapter-configmap配置形式

  1. 另一种配置形式

     ~~~yaml
         rules:
         - seriesQuery: 'myapp_requests_total{namespace="k8s-learn",service="my-custom-metrics-service",container="my-custom-metrics-app"}'
           resources:
             overrides:
               namespace: {resource: "namespace"}
               service: {resource: "service"}
               pod: {resource: "pod"}
           name:
             as: "service_requests_per_30s"  # 自定义指标名称
           metricsQuery: |
             sum(increase(<<.Series>>{<<.LabelMatchers>>}[30s])) by (service, namespace)
     ~~~

  2. deepseek解释

     你提到的两种配置形式在功能上是完全等价的，只是 YAML 的写法不同。这不是不同 Prometheus-adapter 版本的格式要求差异，而是 YAML 语法允许的两种不同表达方式。以下是具体分析：

     第一种形式（多行字符串风格）

     ```yaml
     rules:
         - "seriesQuery": |
             myapp_requests_total{...}
           "resources":
             "overrides":
               "namespace":
                 "resource": "namespace"
               "node":
                 "resource": "node"
           "name":
             "matches": "^(.*)_total"
             "as": "service_requests_per_30s"
           "metricsQuery": |
             sum(increase(...))
     ```

     第二种形式（紧凑风格）

     ```yaml
     rules:
         - seriesQuery: 'myapp_requests_total{...}'
           resources:
             overrides:
               namespace: {resource: "namespace"}
               service: {resource: "service"}
           name:
             as: "service_requests_per_30s"
           metricsQuery: |
             sum(increase(...))
     ```

     主要区别点

     1. **引号使用**：
       
        - 第一种显式给所有字段名加了双引号（JSON 风格）
        - 第二种省略了大多数引号（标准 YAML 风格）
        
     2. **多行字符串**：
       
        - 第一种用 `|` 和缩进表示多行字符串
        - 第二种对短字符串直接使用单引号
        
     3. **结构表示**：
       
        - 第一种用完整嵌套形式：
          ```yaml
          "namespace":
            "resource": "namespace"
          ```
        - 第二种用紧凑形式：
          ```yaml
          namespace: {resource: "namespace"}
          ```
        
     4. **正则匹配**：
        - 第一种包含 `name.matches` 字段
        - 第二种直接通过 `name.as` 重命名

     **本质上是相同的**

     这两种配置：
     1. 最终生成的 JSON 结构完全相同
     2. 都会被 Prometheus-adapter 正确解析
     3. 实现的功能完全一致

     **为什么会有不同写法？**

     1. **历史原因**：早期 Prometheus-adapter 示例多用第一种风格
     2. **个人偏好**：第二种更符合现代 YAML 的简洁写法
     3. **工具生成**：不同工具生成的 YAML 风格可能不同

     **最佳实践建议**

     推荐使用第二种（紧凑风格），因为：
     - 更易读和维护
     - 减少不必要的引号和缩进
     - 社区新示例普遍采用这种风格

     唯一需要注意：当字符串包含特殊字符（如 `:` 或 `{ }`）时，需要用引号包裹。

  ==但是经过试验，我还是没有使用该种方法成功实现Prometheus-adapter的自定义指标配置，退而求其次，先使用这种繁琐的配置格式==

* PrometheusAdapter-configmap中自定义规则各个字段的含义

  以如下字段为例：

  ~~~yaml
      "rules":
      - "seriesQuery": |
           myapp_requests_total{namespace="k8s-learn",service="my-custom-metrics-service",container="my-custom-metrics-app"}
        "resources":
          "overrides":
            "namespace":
              "resource": "namespace"
            "node":
              "resource": "node"
            "service":
              "resource": "service"
            "pod":
              "resource": "pod"
        "name":
          "matches": "^(.*)_total"
          "as": "service_requests_per_30s"
        "metricsQuery": |
          sum(increase(<<.Series>>{<<.LabelMatchers>>}[30s])) by (service, namespace)
  ~~~

  Prometheus Adapter 配置解释

  这个配置是用于 Kubernetes 自定义指标 API (custom metrics API) 的 Prometheus Adapter 的规则定义，用于将 Prometheus 指标暴露给 Kubernetes 的 Horizontal Pod Autoscaler (HPA) 使用。下面是各个字段的详细解释：

  1. `seriesQuery`

     ~~~yaml
     "seriesQuery": |
       myapp_requests_total{namespace="k8s-learn",service="my-custom-metrics-service",container="my-custom-metrics-app"}
     ~~~

  - 这是 PromQL 查询，用于发现要处理的指标系列
  - 这里查询的是 `myapp_requests_total` 指标，且带有特定标签值：
    - `namespace="k8s-learn"`
    - `service="my-custom-metrics-service"`
    - `container="my-custom-metrics-app"`

  2. `resources`

     ~~~yaml
     "resources":
       "overrides":
         "namespace":
           "resource": "namespace"
         "node":
           "resource": "node"
         "service":
           "resource": "service"
         "pod":
           "resource": "pod"
     ~~~

  - 定义 Prometheus 标签到 Kubernetes 资源的映射关系
  - 当 Prometheus 指标中的标签匹配左侧名称时，会被映射为右侧的 Kubernetes 资源
  - 例如：
    - 指标中的 `namespace` 标签对应 Kubernetes 的 `namespace` 资源
    - 指标中的 `node` 标签对应 Kubernetes 的 `node` 资源
    - 以此类推

  3. `name`

     ~~~yaml
     "name":
       "matches": "^(.*)_total"
       "as": "service_requests_per_30s"
     ~~~

  - 定义如何转换指标名称
  - `matches`: 正则表达式匹配原始指标名 (`myapp_requests_total`)
  - `as`: 转换后的指标名，这里会将 `myapp_requests_total` 转换为 `service_requests_per_30s`
  - 正则捕获组 `(.*)` 匹配 `myapp_requests` 部分

  4. `metricsQuery`

     ~~~yaml
     "metricsQuery": |
       sum(increase(<<.Series>>{<<.LabelMatchers>>}[30s])) by (service, namespace)
     ~~~

  - 定义实际查询 Prometheus 的指标查询语句
  - `<<.Series>>`: 会被替换为 `seriesQuery` 中找到的指标名 (`myapp_requests_total`)
  - `<<.LabelMatchers>>`: 会被替换为基于资源映射生成的标签匹配器
  - 这个查询计算过去30秒内请求数的增量，并按 `service` 和 `namespace` 分组求和

  5. 整体功能

     这个配置的作用是：

     1. 发现 `myapp_requests_total` 指标
     2. 将其转换为 Kubernetes 自定义指标 `service_requests_per_30s`
     3. 提供查询过去30秒内请求增量的功能
     4. 建立 Prometheus 标签与 Kubernetes 资源的关联

     这样，HPA 就可以使用 `service_requests_per_30s` 指标来自动扩展 Pod 数量。

## APIService配置

* 根据官网给定的配置`api-service.yaml`

  ~~~yaml
  apiVersion: apiregistration.k8s.io/v1
  kind: APIService
  metadata:
    name: v1beta2.custom.metrics.k8s.io
  spec:
    group: custom.metrics.k8s.io
    groupPriorityMinimum: 100
    insecureSkipTLSVerify: true
    service:
      name: prometheus-adapter
      namespace: monitoring
    version: v1beta2
    versionPriority: 100
  ~~~

  ~~~bash
  kubectl apply -f custom-metrics-v1beta2.yaml
  ~~~

* 应用后，可以发现
  ![image.png](https://typora2icture.oss-cn-beijing.aliyuncs.com/img2/202506171317772.png)

  可以看见有三个以Prometheus-adapter为服务端点的apiservice

  1. `v1beta1.custom.metrics.k8s.io`:是另外一种apiservice的配置，其配置文件如下
  
     ~~~yaml
     apiVersion: apiregistration.k8s.io/v1
     kind: APIService
     metadata:
       name: v1beta1.custom.metrics.k8s.io
     spec:
       service:
         name: prometheus-adapter
         namespace: monitoring
       group: custom.metrics.k8s.io
       version: v1beta1
       insecureSkipTLSVerify: true
       groupPriorityMinimum: 100
       versionPriority: 100
     ~~~
  
  2. `v1beta1.metrics.k8s.io`:是kube-prometheus自带的一个apiservcie，实现了对cpu、mem等等指标的监控，也就是Prometheus默认的监控指标
  
  3. `v1beta2.custom.metrics.k8s.io`:这个就是上面的yaml配置的apiservice
  
  > 有一个注意点就是，`spec`下面定义的就是`v1beta1.custom.metrics.k8s.io`这个自定义指标获取的端点，这里指定的是`monitoring`命名空间下面的`prometheus-adapter`这个service
  
* 测试自定义指标

  指令

  ~~~yaml
  kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta2/namespaces/k8s-learn/service/*/service_requests_per_30s" | jq .
  ~~~

  ~~~bash
  (base) root@master:~/work/custom-metircs# kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta2/namespaces/k8s-learn/service/*/service_requests_per_30s" | jq .
  {
    "kind": "MetricValueList",
    "apiVersion": "custom.metrics.k8s.io/v1beta2",
    "metadata": {},
    "items": [
      {
        "describedObject": {
          "kind": "Service",
          "namespace": "k8s-learn",
          "name": "my-custom-metrics-service",
          "apiVersion": "/v1"
        },
        "metric": {
          "name": "service_requests_per_30s",
          "selector": null
        },
        "timestamp": "2025-06-17T05:29:36Z",
        "value": "29273m"
      }
    ]
  }
  ~~~

  另一个版本，指令：

  ~~~bash
  kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1/namespaces/k8s-learn/service/*/service_requests_per_30s" | jq .
  ~~~

  ~~~bash
  (base) root@master:~/work/custom-metircs# kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1/namespaces/k8s-learn/service/*/service_requests_per_30s" | jq .
  {
    "kind": "MetricValueList",
    "apiVersion": "custom.metrics.k8s.io/v1beta1",
    "metadata": {},
    "items": [
      {
        "describedObject": {
          "kind": "Service",
          "namespace": "k8s-learn",
          "name": "my-custom-metrics-service",
          "apiVersion": "/v1"
        },
        "metricName": "service_requests_per_30s",
        "timestamp": "2025-06-17T05:29:28Z",
        "value": "37498m",
        "selector": null
      }
    ]
  }
  ~~~

### 注意点

* 指标数值和Prometheus控制台中显示的不一样
  ![image.png](https://typora2icture.oss-cn-beijing.aliyuncs.com/img2/202506171439867.png)

  可以发现，上述指标中显示的值是`29273m`、`37498m`，这是因为

  - **Adapter 返回值**：`28601m` 表示 **28.601**（Kubernetes 标准毫单位表示法）
  - **Prometheus UI 值**：通常显示原始值（如 `28.601`）

  这并不影响，可以通过使用locust对服务进行压测，查看apiservice的返回值、Prometheus控制台的值，对比发现就是同一个指标

* `/apis/custom.metrics.k8s.io/v1beta1/namespaces/k8s-learn/service/*/service_requests_per_30s`

  其实还没有搞明白为什么`/apis/custom.metrics.k8s.io/v1beta1/namespaces/k8s-learn`后面得是使用`service`才能查到对应的指标

  而不能使用`pod`或者其他，但是根据AI描述，这个地方与PromQL`sum(increase(myapp_requests_total{namespace="k8s-learn",service="my-custom-metrics-service",container="my-custom-metrics-app"}[30s])) by (service,namespace)`,即adapter-config中的**seriesQuery、metricsQuery**两部分组合起来有关

  有待学习

* 可以使用如下指令，查询自定义指标API中是否包含我们自定义的指标

  ~~~bash
  kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1" | jq .
  ~~~

  ~~~bash
  (base) root@master:~/work/custom-metircs# kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1" | jq .
  {
    "kind": "APIResourceList",
    "apiVersion": "v1",
    "groupVersion": "custom.metrics.k8s.io/v1beta1",
    "resources": [
      {
        "name": "namespaces/service_requests_per_30s",
        "singularName": "",
        "namespaced": false,
        "kind": "MetricValueList",
        "verbs": [
          "get"
        ]
      },
      {
        "name": "pods/service_requests_per_30s",
        "singularName": "",
        "namespaced": true,
        "kind": "MetricValueList",
        "verbs": [
          "get"
        ]
      },
      {
        "name": "services/service_requests_per_30s",
        "singularName": "",
        "namespaced": true,
        "kind": "MetricValueList",
        "verbs": [
          "get"
        ]
      }
    ]
  }
  ~~~

  该部分显示的name后面跟的类型，有namespace、pods、services，这与adapter-config配置中的`resources.overrides`下面配置的资源映射项有关，基本上应该是进行了资源映射的，都能够在指标中显示





# HPA自动扩展测试

1. HPA准备

   使用的Kubernetes集群版本是v1.28.15，HPA是Kubernetes自带的资源，无需安装，可通过如下命令进行检验

   ~~~bash
   kubectl get hpa -A
   ~~~

   如果有输出，或者输出的是`No resources found`即具有HPA能力

2. 针对前几章给的自定义指标编辑相应指标的hpa的yaml

   ~~~yaml
   apiVersion: autoscaling/v2
   kind: HorizontalPodAutoscaler
   metadata:
     name: custom-metrics-hpa
     namespace: k8s-learn
   spec:
     scaleTargetRef:
       apiVersion: apps/v1
       kind: Deployment
       name: my-custom-metrics-app-deployment
     minReplicas: 2
     maxReplicas: 10
     metrics:
     - type: Object
       object:
         describedObject:
           apiversion: /v1
           kind: Service
           name: my-custom-metrics-service
         metric:
           name: service_requests_per_30s
         target:
           type: value
           value: 50
   ~~~

   这里经过博客对比，`metrics`下面的type类型，根据自定义资源的type不同而不同，如果使用的是metrics-server来进行HPA的话，好像又不一样，使用的是Resource，
   metrics-server的安装[[一些软件的部署#Metricc-server部署]]

   > 使用metrics-server来进行HPA的yaml示例如下
   >
   > ~~~yaml
   > apiVersion: autoscaling/v2
   > kind: HorizontalPodAutoscaler
   > metadata:
   >   name: my-app-hpa
   >   namespace: default
   > spec:
   >   scaleTargetRef:
   >     apiVersion: apps/v1
   >     kind: Deployment
   >     name: my-app-deployment
   >   minReplicas: 2
   >   maxReplicas: 10
   >   metrics:
   >   - type: Resource
   >     resource:
   >       name: cpu
   >       target:
   >         type: Utilization
   >         averageUtilization: 50  # 当 CPU 平均使用率超过 50% 时开始扩容
   > ~~~
   >
   > 并且，如果使用metrics-server进行hpa的话，好像要对进行hpa的应用的Deployment编辑，使之具有如下栏目`resources.requests`

3. 检验部署情况

   * hpa的部署情况
     <img src="https://typora2icture.oss-cn-beijing.aliyuncs.com/img2/202507141112392.png" alt="image.png"  />
   * Pod的扩展情况 
<img src="https://typora2icture.oss-cn-beijing.aliyuncs.com/img2/202507141113263.png" alt="image.png"  />

4. 使用locust对服务进行加压，查看自动扩展情况

  * locust加压情况

    ![image.png](https://typora2icture.oss-cn-beijing.aliyuncs.com/img2/202507141114812.png)

  * hpa自动扩展情况

    ![image.png](https://typora2icture.oss-cn-beijing.aliyuncs.com/img2/202507141115572.png)



   

   

# 参考文章和链接

* https://blog.csdn.net/weixin_43391291/article/details/142212854
* https://blog.csdn.net/m0_57949696/article/details/143066976
* https://blog.csdn.net/qq_39458487/article/details/125409811
* https://blog.csdn.net/qq_31977125/article/details/130822675
* https://www.cnblogs.com/zhangmingcheng/p/15773348.html
* https://www.cnblogs.com/wangxu01/articles/11676621.html
* https://kubeservice.cn/2024/10/25/k8s-custom-metrics-hpa/
* https://github.com/kubernetes-sigs/prometheus-adapter/blob/master/docs/walkthrough.md
* https://github.com/kubernetes-sigs/prometheus-adapter/blob/master/docs/config-walkthrough.md
* https://github.com/prometheus-operator/kube-prometheus/tree/release-0.13
* https://github.com/kubernetes-sigs/prometheus-adapter/issues/626

# 相关文章链接

* Ingress-nginx controller + Prometheus + Grafana：https://blog.csdn.net/qq_44930876/article/details/142848211
* Custom-metrics-apiservice：https://github.com/kubernetes-sigs/custom-metrics-apiserver?tab=readme-ov-file
* 事件驱动弹性伸缩KEDA：https://imroc.cc/kubernetes/best-practices/autoscaling/keda/overview





