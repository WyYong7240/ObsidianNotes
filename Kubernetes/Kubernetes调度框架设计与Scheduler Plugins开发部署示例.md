---
tags:
  - Kubernetes
  - 调度框架
  - 调度
  - 调度器开发
  - Scheduler
---
# 1 引言
> Pod使用调度器的方式
> - 如果Pod的spec里没有指定==schedulerName==字段，则使用默认调度器
> - 如果指定了，就会使用相应的调度器/调度插件

## 1.1 调度框架（Scheduling FrameWork）扩展点
Kubernetes在调度过程中提供了一些扩展点，如下图
![image.png](https://typora2icture.oss-cn-beijing.aliyuncs.com/img2/20250414185500.png)

每个扩展点上，一般会有多个Plugins，按照注册顺序依次执行
根据扩展点是否可以影响调度决策，可以分为两类

- 大部分扩展点是==影响调度决策的==
	- 扩展点的函数的返回值中包括一个成功/失败字段，决定了允许还是拒绝这个Pod进入下一个阶段
	- 任何一个扩展点失败了，Pod的调度就失败了
- 少数的扩展点是Informational的
	- 这些函数==没有返回值==，不能影响调度决策
	- 但是，在这里面==可以修改pod/node等信息==，或者执行清理操作
## 1.2 调度插件分类
根据是否维护在K8s代码仓库本身，分为两类
- `in-tree` plugins
	- 这类插件维护在kubernetes代码目录[`pkg/scheduler/framework/plugins`](https://github.com/kubernetes/kubernetes/tree/master/pkg/scheduler/framework/plugins)中，跟==内置调度器一起编译==，大部分都是常用的
		```bash
		$ ll pkg/scheduler/framework/plugins
		defaultbinder/
		defaultpreemption/
		dynamicresources/
		feature/
		imagelocality/
		interpodaffinity/
		names/
		nodeaffinity/
		nodename/
		nodeports/
		noderesources/
		nodeunschedulable/
		nodevolumelimits/
		podtopologyspread/
		queuesort/
		schedulinggates/
		selectorspread/
		tainttoleration/
		volumebinding/
		volumerestrictions/
		volumezone/
		```
		但是in-tree方式每次添加新插件或者修改原有插件，都要修改kube-scheduler代码，然后重新部署kube-scheduler，比较重量级
- `out-of-tree` plugins
	
	- 此类插件由==用户自己编写维护，独立部署==，不需要对K8s做任何代码配置或改动
	- 其用法有两种
		1. 跟现有的调度器并i选哪个部署，只管理特定的某些Pods
		2. 取代现有的调度器，因为其功能也是完全的
	
	> 本质上 out-of-tree plugins 也是跟 kube-scheduler 代码一起编译的，不过 kube-scheduler 相关代码已经抽出来作为一个独立项目 [github.com/kubernetes-sigs/scheduler-plugins](https://github.com/kubernetes-sigs/scheduler-plugins)。 用户只需要引用这个包，编写自己的调度器插件，然后以普通 pod 方式部署就行（其他部署方式也行，比如 binary 方式部署）。 编译之后是个包含**默认调度器和所有 out-of-tree 插件**的总调度器程序，
	>
	> - 它有内置调度器的功能；
	> - 也包括了 out-of-tree 调度器的功能；

## 1.3 每个扩展点上的内置插件

具体的内置插件分别工作在哪些扩展点，详见[官方文档](https://kubernetes.io/docs/reference/scheduling/config/#scheduling-plugins)，比如

- node select 和 node affinity用到了==NodeAffinity Plugin==
- tain/toleration 用到了 ==TaintToleration Plugin==

# 2 Pod调度过程

Pod的调度过程可以分为两个阶段：

1. ==Scheduling cycle==

   为Pod选择一个Node，类似于数据库查询和筛选

2. ==Binding cycle==

   落实上述选择，类似于处理各种关联的东西，并将结果写回数据库

> 例如，虽然Scheduling cycle为Pod选择了一个Node，但是在Binding cycle的过程中，在这个Node上为这个Pod创建PV 失败了，那么整个调度过程也算是失败的。

两个过程合起来称为==scheduling contex==

除此以外，在进入另一个scheduling context之前，还有一个调度队列，可以编写自己的算法对队列内的Pods进行排序，决定哪些pods先进入调度流程。如下是总流程图：
![image.png](https://typora2icture.oss-cn-beijing.aliyuncs.com/img2/20250414185415.png)

