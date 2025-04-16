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



## 2.1 等待调度过程

### 2.1.1 PreEnqueue

这阶段的Pod处于ready for scheduling阶段，详见其[内部原理](https://github.com/kubernetes/community/blob/f03b6d5692bd979f07dd472e7b6836b2dad0fd9b/contributors/devel/sig-scheduling/scheduler_queues.md)，这一步不通过，就不会进入调度队列，更不会进入调度流程

### 2.1.2 QueueSort

这一步是对调度队列Scheduling Queue中的Pod进行排序，决定先调度哪些Pods

## 2.2 调度阶段（Scheduling cycle）

### 2.2.1 PreFilter：pod预处理和检查，不符合预期就提前结束调度

这里的插件可以对Pod进行预处理，或者条件检查，函数签名如下：

``` go
// https://github.com/kubernetes/kubernetes/blob/v1.28.4/pkg/scheduler/framework/interface.go#L349-L367

// PreFilterPlugin is an interface that must be implemented by "PreFilter" plugins.
// These plugins are called at the beginning of the scheduling cycle.
type PreFilterPlugin interface {
    // PreFilter is called at the beginning of the scheduling cycle. All PreFilter
    // plugins must return success or the pod will be rejected. PreFilter could optionally
    // return a PreFilterResult to influence which nodes to evaluate downstream. This is useful
    // for cases where it is possible to determine the subset of nodes to process in O(1) time.
    // When it returns Skip status, returned PreFilterResult and other fields in status are just ignored,
    // and coupled Filter plugin/PreFilterExtensions() will be skipped in this scheduling cycle.
    PreFilter(ctx , state *CycleState, p *v1.Pod) (*PreFilterResult, *Status)

    // PreFilterExtensions returns a PreFilterExtensions interface if the plugin implements one,
    // or nil if it does not. A Pre-filter plugin can provide extensions to incrementally
    // modify its pre-processed info. The framework guarantees that the extensions
    // AddPod/RemovePod will only be called after PreFilter, possibly on a cloned
    // CycleState, and may call those functions more than once before calling
    // Filter again on a specific node.
    PreFilterExtensions() PreFilterExtensions
}
```

* 输入：
  * `p *v1.pod`是==待调度的pod==
  * 第二参数state用于存放一些状态信息，然后在后面的扩展点使用
* 输出：
  * 只要有任何一个plugin返回失败，这个pod的调度就失败了
  * 所有已经注册的PreFilter plugins都成功后，pod才会进入到下一个环节

### 2.2.2 Filter：排除所有不符合要求的Node

这里的插件可以==过滤掉不满足要求的Node==

* 针对每个Node，调度器会配置顺序，依次执行Filter Plugins

* 任何一个插件返回失败，这个Node就被排除

* 函数签名

  ``` go
  // https://github.com/kubernetes/kubernetes/blob/v1.28.4/pkg/scheduler/framework/interface.go#L349C1-L367C2
  
  // FilterPlugin is an interface for Filter plugins. These plugins are called at the
  // filter extension point for filtering out hosts that cannot run a pod.
  // This concept used to be called 'predicate' in the original scheduler.
  // These plugins should return "Success", "Unschedulable" or "Error" in Status.code.
  // However, the scheduler accepts other valid codes as well.
  // Anything other than "Success" will lead to exclusion of the given host from running the pod.
  type FilterPlugin interface {
      Plugin
      // Filter is called by the scheduling framework.
      // All FilterPlugins should return "Success" to declare that
      // the given node fits the pod. If Filter doesn't return "Success",
      // it will return "Unschedulable", "UnschedulableAndUnresolvable" or "Error".
      // For the node being evaluated, Filter plugins should look at the passed
      // nodeInfo reference for this particular node's information (e.g., pods
      // considered to be running on the node) instead of looking it up in the
      // NodeInfoSnapshot because we don't guarantee that they will be the same.
      // For example, during preemption, we may pass a copy of the original
      // nodeInfo object that has some pods removed from it to evaluate the
      // possibility of preempting them to schedule the target pod.
      Filter(ctx , state *CycleState, pod *v1.Pod, nodeInfo *NodeInfo) *Status
  }
  ```

  * 输入：
    1. state、pod不再介绍，见上一条参数解释
    2. **nodeInfo是当前给定的Node信息，Filter程序判断这个Node是否符合要求**
  * 输出：
    * 状态，即放行或者拒绝这个Node

**对于给定的Node，如果所有的Filter plugins都返回成功，这个Node才算通过筛选，成为备选Node之一**

### 2.2.3 PostFilter：Filter之后没有Node剩下，补救阶段

如果Filter阶段之后，所有的Node都被筛选掉了，才会执行这个阶段

函数签名：

``` go
// https://github.com/kubernetes/kubernetes/blob/v1.28.4/pkg/scheduler/framework/interface.go#L392C1-L407C2

// PostFilterPlugin is an interface for "PostFilter" plugins. These plugins are called after a pod cannot be scheduled.
type PostFilterPlugin interface {
    // A PostFilter plugin should return one of the following statuses:
    // - Unschedulable: the plugin gets executed successfully but the pod cannot be made schedulable.
    // - Success: the plugin gets executed successfully and the pod can be made schedulable.
    // - Error: the plugin aborts due to some internal error.
    //
    // Informational plugins should be configured ahead of other ones, and always return Unschedulable status.
    // Optionally, a non-nil PostFilterResult may be returned along with a Success status. For example,
    // a preemption plugin may choose to return nominatedNodeName, so that framework can reuse that to update the
    // preemptor pod's .spec.status.nominatedNodeName field.
    PostFilter(ctx , state *CycleState, pod *v1.Pod, filteredNodeStatusMap NodeToStatusMap) (*PostFilterResult, *Status)
}
```

* 按照plugin顺序依次执行，任何一个插件将Node标记为Schedulable就算成功，不再执行剩下的PostFilter plugins

  比如：preemptiontoleration，Filter之后已经没有可用Node了，在这个阶段挑选一个pod/node，抢占其资源。

### 2.2.4 PreScore

`PreScore/Score/NormalizeScore` 都是给 node 打分的，以最终选出一个最合适的 node。这里就不展开了， 函数签名也在上面给到的源文件路径中，这里就不贴了。

### 2.2.5 Score

针对每个Node依次调用scoring plugins，得到一个分数

如果要是

### 2.2.6 NormalizeScore

### 2.2.7 Reserve：Informational，维护plugin状态信息

* 函数签名

  ~~~go
  // https://github.com/kubernetes/kubernetes/blob/v1.28.4/pkg/scheduler/framework/interface.go#L444C1-L462C2
  
  // ReservePlugin is an interface for plugins with Reserve and Unreserve
  // methods. These are meant to update the state of the plugin. This concept
  // used to be called 'assume' in the original scheduler. These plugins should
  // return only Success or Error in Status.code. However, the scheduler accepts
  // other valid codes as well. Anything other than Success will lead to
  // rejection of the pod.
  type ReservePlugin interface {
      // Reserve is called by the scheduling framework when the scheduler cache is
      // updated. If this method returns a failed Status, the scheduler will call
      // the Unreserve method for all enabled ReservePlugins.
      Reserve(ctx , state *CycleState, p *v1.Pod, nodeName string) *Status
      // Unreserve is called by the scheduling framework when a reserved pod was
      // rejected, an error occurred during reservation of subsequent plugins, or
      // in a later phase. The Unreserve method implementation must be idempotent
      // and may be called by the scheduler even if the corresponding Reserve
      // method for the same plugin was not called.
      Unreserve(ctx , state *CycleState, p *v1.Pod, nodeName string)
  }
  ~~~

  这里的两个方法都是Informational的，不影响调度决策，维护了runtime state，可以通过这两个方法接收scheduler传递的信息

  1. Reserve

     用来避免scheduler等待bind操作结束期间，因为race condition导致的错误。只有当所有Reserve plugins都成功后，才会进入下一阶段，否则scheduling cycle就终止调度

     > 意思是这个插件的运行和bind并行运行？

  2. Unreserve

     调度失败，这个阶段回滚时执行。**Unreserve()必须幂等，且不能fail**

### 2.2.8 Permit：允许/拒绝/等待进入Binding cycle

这是scheduling cycle的最后一个扩展点，可以阻止或者延迟将一个pod binding到candidate node

* 函数签名

  ~~~go
  // PermitPlugin is an interface that must be implemented by "Permit" plugins.
  // These plugins are called before a pod is bound to a node.
  type PermitPlugin interface {
      // Permit is called before binding a pod (and before prebind plugins). Permit
      // plugins are used to prevent or delay the binding of a Pod. A permit plugin
      // must return success or wait with timeout duration, or the pod will be rejected.
      // The pod will also be rejected if the wait timeout or the pod is rejected while
      // waiting. Note that if the plugin returns "wait", the framework will wait only
      // after running the remaining plugins given that no other plugin rejects the pod.
      Permit(ctx , state *CycleState, p *v1.Pod, nodeName string) (*Status, time.Duration)
  }
  ~~~

  这个函数有三种结果

  1. approve：所有Permit plugins都approve后，这个pod就进入后面的binding阶段
  2. deny：任何一个Permit plugin deny后，pod就无法进入binding阶段，这会触发Reserve plugins的Unreserve()方法
  3. wait（with a timeout）：如果有Permit plugins返回wait，这个pod就会进入一个interval waiting Pod list

## 2.3 绑定阶段（binding cycle）

  ### 2.3.1 PreBind：Bind之前的预处理，例如到Node上去挂载volume

  就好比，将一个pod调度到一个node之前，需要先给这个pod在调度到的node上挂载一个network volume

  * 函数签名：

    ~~~go
    // PreBindPlugin is an interface that must be implemented by "PreBind" plugins.
    // These plugins are called before a pod being scheduled.
    type PreBindPlugin interface {
        // PreBind is called before binding a pod. All prebind plugins must return
        // success or the pod will be rejected and won't be sent for binding.
        PreBind(ctx , state *CycleState, p *v1.Pod, nodeName string) *Status
    }
    ~~~

    任何一个PreBind Plugin失败，都会让Pod被拒绝，进入到Reserve plugins的Unreserve()方法

  ### 2.3.2 Bind：将Pod关联到Node

  所有PreBind都完成后，才会进入Bind

  * 函数签名

    ~~~go
    // https://github.com/kubernetes/kubernetes/blob/v1.28.4/pkg/scheduler/framework/interface.go#L497
    
    // Bind plugins are used to bind a pod to a Node.
    type BindPlugin interface {
        // Bind plugins will not be called until all pre-bind plugins have completed. Each
        // bind plugin is called in the configured order. A bind plugin may choose whether
        // or not to handle the given Pod. If a bind plugin chooses to handle a Pod, the
        // remaining bind plugins are skipped. When a bind plugin does not handle a pod,
        // it must return Skip in its Status code. If a bind plugin returns an Error, the
        // pod is rejected and will not be bound.
        Bind(ctx , state *CycleState, p *v1.Pod, nodeName string) *Status
    }
    ~~~

    * 所有plugin按配置顺序依次执行

    * 每个plugin可以选择是否要处理一个给定的Pod

    * 如果选择处理，后面剩下的plugins会跳过这个Pod，也就是最多只有一个Bind plugin会执行


### 2.3.3 PostBind：Informational，可选，执行清理操作

* 只有Bind操作成功的Pod才会进入这个阶段

* 作为Binding cycle的最后一个阶段，一般用于清理一些相关资源

* 函数签名

  ~~~go
  // https://github.com/kubernetes/kubernetes/blob/v1.28.4/pkg/scheduler/framework/interface.go#L473
  
  // PostBindPlugin is an interface that must be implemented by "PostBind" plugins.
  // These plugins are called after a pod is successfully bound to a node.
  type PostBindPlugin interface {
      // PostBind is called after a pod is successfully bound. These plugins are informational.
      // A common application of this extension point is for cleaning
      // up. If a plugin needs to clean-up its state after a pod is scheduled and
      // bound, PostBind is the extension point that it should register.
      PostBind(ctx , state *CycleState, p *v1.Pod, nodeName string)
  }
  ~~~

  

# 参考文章

[K8s 调度框架设计与 scheduler plugins 开发部署示例（2024）](https://arthurchiao.art/blog/k8s-scheduling-plugins-zh/#223-postfilterfilter-%E4%B9%8B%E5%90%8E%E6%B2%A1%E6%9C%89-node-%E5%89%A9%E4%B8%8B%E8%A1%A5%E6%95%91%E9%98%B6%E6%AE%B5)

[K8s 自定义调度器 Part2：通过 Scheduler Framework 实现自定义调度逻辑](https://www.lixueduan.com/posts/kubernetes/34-custom-scheduker-by-scheduler-framework/)

