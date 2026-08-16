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



# Schedule-plugins实现与安装

> 环境：
>
> 1. Kubernetes v1.28.15
> 2. ubuntu-Server 24.04 LTS

## 获取基础项目代码与实现自定义调度插件

1. 在获取项目基础代码之前，应该在Ubuntu上具有完整的Golang的环境，例如GOROOT、GOPATH、GOPROXY等

   关于这部分，详见[[配置Go工作环境]]

2. 接下来就是将基础项目代码clone到对应的路径中，如[参考文档1](https://github.com/kubernetes-sigs/scheduler-plugins/blob/release-1.28/doc/develop.md)中所说，需要将项目基础代码clone到`$GOPATH/src/sigs.k8s.io`下

   ~~~bash
   # 创建相应路径
   mkdir -p $GOPATH/src/sigs.k8s.io
   cd $GOPATH/src/sigs.k8s.io
   # clone项目代码
   git clone https://github.com/kubernetes-sigs/scheduler-plugins.git
   ~~~

   > 注意切换到当前使用是Kubernetes集群兼容的版本，命令为
   >
   > ~~~sh
   > git branch -a
   > git checkout XXXX
   > ~~~

3. 接下来就是实现自定义调度插件，在`pkg`文件夹下新建我们自定义调度插件的文件夹`customschedulingplugins`

   ~~~bash
   cd scheduler-plugins
   mkdir -p pkg/customschedulingplugins
   ~~~

   > 注意，这里需要将clone下来的文件夹改名为scheduler-plugins，这是为了后续编译插件方便

   在自定义调度插件文件夹下编辑自定义调度插件`customschedulingplugins.go`文件

   ~~~go
   package customschedulingplugins
   
   import (
           "context"
           "fmt"
           v1 "k8s.io/api/core/v1"
           "k8s.io/apimachinery/pkg/runtime"
           "k8s.io/kubernetes/pkg/scheduler/framework"
           "k8s.io/kubernetes/pkg/scheduler/framework/plugins/helper"
           "log"
           "strconv"
   )
   
   const Name = "wuyong_schedule_plugins"  // 自定义调度器的名字
   const Label = "wuyong.schedule.plugins" // Pod使用该调度器将调度到的节点，必须带有该Label
   
   type CustomScheduler struct {
           handle framework.Handle
   }
   
   // 此处相当于声明要实现的扩展点
   var _ framework.FilterPlugin = &CustomScheduler{}
   var _ framework.ScorePlugin = &CustomScheduler{}
   
   //var _ framework.PostFilterPlugin = &CustomScheduler{}
   
   // 新建并初始化一个新的插件，并返回，这里是在将插件注册到cmd/schedule/main.go中使用的
   // 0.28.9版本的scheduling plugins使用的是该版本的New函数
   //func New(_ context.Context, _ runtime.Object, h framework.Handle) (framework.Plugin, error) {
   //      return &CustomScheduler{handle: h}, nil
   //}
   
   func New(_ runtime.Object, h framework.Handle) (framework.Plugin, error) {
           return &CustomScheduler{handle: h}, nil
   }
   
   // 返回插件名称
   func (cs *CustomScheduler) Name() string {
           return Name
   }
   
   // 实现自定义调度框架的Filter扩展点
   func (cs *CustomScheduler) Filter(ctx context.Context, state *framework.CycleState, pod *v1.Pod, nodeInfo *framework.NodeInfo) *framework.Status {
           log.Printf("filter pod: %v, node: %v", pod.Name, nodeInfo)
           log.Print(state)
   
           //只调度到携带指定Label的节点上
           if _, ok := nodeInfo.Node().Labels[Label]; !ok {
                   // 如果没有定义的标签，表示节点不可用，返回Unschedulable，该节点无法通过过滤
                   return framework.NewStatus(framework.Unschedulable, fmt.Sprintf("Node:%s dose not have label %s", "Node:"+nodeInfo.Node().Name, Label))
           }
           //节点通过过滤
           return framework.NewStatus(framework.Success, "Node:"+nodeInfo.Node().Name)
   }
   
   //func (cs *CustomScheduler) PostFilter(ctx context.Context, state *framework.CycleState, pod *v1.Pod, filteredNodeStatusMap framework.NodeToStatusMap) (*framework.PostFilterResult, *framework.Status) {
   //      return framework.NewStatus(framework.Success, "Node:"+nodeInfo.Node().Name)
   //}
   
   // 是调度器的评分阶段逻辑，作用是根据节点的标签值计算节点的优先级分数
   func (cs *CustomScheduler) Score(ctx context.Context, state *framework.CycleState, pod *v1.Pod, nodeName string) (int64, *framework.Status) {
           nodeInfo, err := cs.handle.SnapshotSharedLister().NodeInfos().Get(nodeName) //  获取节点信息
           if err != nil {
                   // 无法获取节点信息
                   return 0, framework.NewStatus(framework.Error, fmt.Sprintf("getting node %q from Snapshot: %v", nodeName, err))
           }
           //获取Node上的Label作为分数
           csStr, ok := nodeInfo.Node().Labels[Label] // 检查是否带有自定义的标志
           if !ok {
                   // 标签值无效
                   return 0, framework.NewStatus(framework.Error, fmt.Sprintf("node %q dose not have label %s", nodeName, Label))
           }
           customSchedule, err := strconv.Atoi(csStr) // 将标签值解析为优先级分数
           if err != nil {
                   return 0, framework.NewStatus(framework.Error, fmt.Sprintf("node %q has priority %s is invalid", nodeName, csStr))
           }
           return int64(customSchedule), framework.NewStatus(framework.Success, "") // 返回分数和状态
   }
   
   // Score plugin的 ScoreExtensions
   // 该方法返回ScoreExtensions接口实现，这里是自定义调度器本身，
   // 该方法使得插件可以支持扩展功能
   func (cs *CustomScheduler) ScoreExtensions() framework.ScoreExtensions {
           return cs
   }
   
   // NormalizeScore 在对所有节点打分后激活
   // NormalizeScore 方法是对所有节点的分数进行归一化处理，调用了DefaultNormalizeScore方法，将分数归一化到[0,MaxNodeScore]范围内
   func (cs *CustomScheduler) NormalizeScore(ctx context.Context, state *framework.CycleState, pod *v1.Pod, scores framework.NodeScoreList) *framework.Status {
           // false表示不反转分数排序，scores是所有节点的分数列表
           return helper.DefaultNormalizeScore(framework.MaxNodeScore, false, scores)
   }
   ~~~

4. 然后在`scheduler-plugins/cmd/scheduler/main.go`文件中添加我们自定义调度插件的编译选项

   ~~~go
   /*
   Copyright 2020 The Kubernetes Authors.
   
   Licensed under the Apache License, Version 2.0 (the "License");
   you may not use this file except in compliance with the License.
   You may obtain a copy of the License at
   
       http://www.apache.org/licenses/LICENSE-2.0
   
   Unless required by applicable law or agreed to in writing, software
   distributed under the License is distributed on an "AS IS" BASIS,
   WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
   See the License for the specific language governing permissions and
   limitations under the License.
   */
   
   package main
   
   import (
           "os"
           customschedulingplugins "sigs.k8s.io/scheduler-plugins/pkg/customschedulingplugins"
   
           "k8s.io/component-base/cli"
           _ "k8s.io/component-base/metrics/prometheus/clientgo" // for rest client metric registration
           _ "k8s.io/component-base/metrics/prometheus/version"  // for version metric registration
           "k8s.io/kubernetes/cmd/kube-scheduler/app"
   
           "sigs.k8s.io/scheduler-plugins/pkg/capacityscheduling"
           "sigs.k8s.io/scheduler-plugins/pkg/coscheduling"
           "sigs.k8s.io/scheduler-plugins/pkg/networkaware/networkoverhead"
           "sigs.k8s.io/scheduler-plugins/pkg/networkaware/topologicalsort"
           "sigs.k8s.io/scheduler-plugins/pkg/noderesources"
           "sigs.k8s.io/scheduler-plugins/pkg/noderesourcetopology"
           "sigs.k8s.io/scheduler-plugins/pkg/podstate"
           "sigs.k8s.io/scheduler-plugins/pkg/preemptiontoleration"
           "sigs.k8s.io/scheduler-plugins/pkg/qos"
           "sigs.k8s.io/scheduler-plugins/pkg/sysched"
           "sigs.k8s.io/scheduler-plugins/pkg/trimaran/loadvariationriskbalancing"
           "sigs.k8s.io/scheduler-plugins/pkg/trimaran/lowriskovercommitment"
           "sigs.k8s.io/scheduler-plugins/pkg/trimaran/targetloadpacking"
   
           // Ensure scheme package is initialized.
           _ "sigs.k8s.io/scheduler-plugins/apis/config/scheme"
   )
   
   func main() {
           // Register custom plugins to the scheduler framework.
           // Later they can consist of scheduler profile(s) and hence
           // used by various kinds of workloads.
           command := app.NewSchedulerCommand(
                   app.WithPlugin(capacityscheduling.Name, capacityscheduling.New),
                   app.WithPlugin(coscheduling.Name, coscheduling.New),
                   app.WithPlugin(loadvariationriskbalancing.Name, loadvariationriskbalancing.New),
                   app.WithPlugin(networkoverhead.Name, networkoverhead.New),
                   app.WithPlugin(topologicalsort.Name, topologicalsort.New),
                   app.WithPlugin(noderesources.AllocatableName, noderesources.NewAllocatable),
                   app.WithPlugin(noderesourcetopology.Name, noderesourcetopology.New),
                   app.WithPlugin(preemptiontoleration.Name, preemptiontoleration.New),
                   app.WithPlugin(targetloadpacking.Name, targetloadpacking.New),
                   app.WithPlugin(lowriskovercommitment.Name, lowriskovercommitment.New),
                   app.WithPlugin(sysched.Name, sysched.New),
                   // Sample plugins below.
                   // app.WithPlugin(crossnodepreemption.Name, crossnodepreemption.New),
                   app.WithPlugin(podstate.Name, podstate.New),
                   app.WithPlugin(qos.Name, qos.New),
   
                   app.WithPlugin(customschedulingplugins.Name, customschedulingplugins.New),
           )
   
           code := cli.Run(command)
           os.Exit(code)
   }
   ~~~

5. 给自定义调度插件定义自定义名称

   在项目目录`/scheduler-plugins/hack/build-images.sh`中，可以自定义镜像的仓库名称、镜像名称还有版本，其文件内容如下

   ~~~sh
   #!/usr/bin/env bash
   
   # Copyright 2023 The Kubernetes Authors.
   #
   # Licensed under the Apache License, Version 2.0 (the "License");
   # you may not use this file except in compliance with the License.
   # You may obtain a copy of the License at
   #
   #     http://www.apache.org/licenses/LICENSE-2.0
   #
   # Unless required by applicable law or agreed to in writing, software
   # distributed under the License is distributed on an "AS IS" BASIS,
   # WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
   # See the License for the specific language governing permissions and
   # limitations under the License.
   
   set -o errexit
   set -o nounset
   set -o pipefail
   
   SCRIPT_ROOT=$(realpath $(dirname "${BASH_SOURCE[@]}")/..)
   
   SCHEDULER_DIR="${SCRIPT_ROOT}"/build/scheduler
   CONTROLLER_DIR="${SCRIPT_ROOT}"/build/controller
   
   REGISTRY=${REGISTRY:-"wuyong7240/scheduler-plugins"}
   IMAGE=${IMAGE:-"neilats-refactor-scheduler:v2.1"}
   CONTROLLER_IMAGE=${CONTROLLER_IMAGE:-"neilats-refactor-controller:v2.1"}
   RELEASE_VERSION=${RELEASE_VERSION:-"v0.0.0"}
   
   BUILDER=${BUILDER:-"docker"}
   
   if ! command -v ${BUILDER} && command -v nerdctl >/dev/null; then
     BUILDER=nerdctl
   fi
   
   ARCH=${ARCH:-$(go env GOARCH)}
   if [[ "${ARCH}" == "arm64" ]]; then
     ARCH="arm64v8"
   fi
   
   GO_BASE_IMAGE=${GO_BASE_IMAGE:-"golang"}
   ALPINE_BASE_IMAGE=${ALPINE_BASE_IMAGE:-"$ARCH/alpine"}
   
   cd "${SCRIPT_ROOT}"
   
   ${BUILDER} build \
              -f ${SCHEDULER_DIR}/Dockerfile \
              --build-arg ARCH=${ARCH} \
              --build-arg RELEASE_VERSION=${RELEASE_VERSION} \
              --build-arg GO_BASE_IMAGE=${GO_BASE_IMAGE} \
              --build-arg ALPINE_BASE_IMAGE=${ALPINE_BASE_IMAGE} \
              -t ${REGISTRY}/${IMAGE} .
   ${BUILDER} build \
              -f ${CONTROLLER_DIR}/Dockerfile \
              --build-arg ARCH=${ARCH} \
              --build-arg RELEASE_VERSION=${RELEASE_VERSION} \
              --build-arg GO_BASE_IMAGE=${GO_BASE_IMAGE} \
              --build-arg ALPINE_BASE_IMAGE=${ALPINE_BASE_IMAGE} \
              -t ${REGISTRY}/${CONTROLLER_IMAGE} .
   ~~~

6. 编译自定义调度插件，获取镜像

   根据[文档](https://github.com/kubernetes-sigs/scheduler-plugins/blob/release-1.28/doc/develop.md)，回到`scheduler-plugins`根目录，使用命令

   ~~~bash
   make local-image
   ~~~

   等待编译完成，会发现多出两个镜像

   ~~~bash
   (base) root@master:~/Golang/code/go/src/sigs.k8s.io/scheduler-plugins/cmd/scheduler# nerdctl images
   REPOSITORY                                           TAG       IMAGE ID        CREATED        PLATFORM       SIZE       BLOB SIZE
   localhost:5000/scheduler-plugins/controller          latest    7aa439bbe6d5    3 days ago     linux/amd64    48.53MB    15.33MB
   localhost:5000/scheduler-plugins/kube-scheduler      latest    06422a44f220    3 days ago     linux/amd64    76.31MB    22.89MB
   os-backend-app                                       v3.0      25cdd2577bf7    10 days ago    linux/amd64    1.15GB     391MB
   ghcr.io/cloudflare/ebpf_exporter                     latest    2cd341e1f860    12 days ago    linux/amd64    59.04MB    26.19MB
   my-custom-metrics-app                                v1.1      0563a94dccf4    5 weeks ago    linux/amd64    20.74MB    10.21MB
   golang                                               1.23.2    ad5c126b5cf5    5 weeks ago    linux/amd64    926.9MB    304.3MB
   alpine                                               latest    8a1f59ffb675    5 weeks ago    linux/amd64    8.978MB    3.798MB
   ~~~

## 自定义调度插件的部署与测试

1. 首先，在Kubernetes v1.28.15只有contained的情况下，将nerdctl默认命名空间中的这两个镜像改名并加入`k8s.io`命名空间

   * 改名

     ~~~bash
     nerdctl tag localhost:5000/scheduler-plugins/controller:latest wuyong20724/scheduler-plugins/test-controller:v0.1
     nerdctl tag localhost:5000/scheduler-plugins/kube-scheduler:latest wuyong20724/scheduler-plugins/test-kube-scheduler:v0.1
     ~~~

   * 转移命名空间

     ~~~bash
     nerdctl save wuyong20724/scheduler-plugins/test-controller:v0.1 -o temp.tar.gz
     nerdctl -n k8s.io load -i temp.tar.gz
     nerdctl save wuyong20724/scheduler-plugins/test-kube-scheduler:v0.1 -o temp2.tar.gz
     nerdctl -n k8s.io load -i temp2.tar.gz
     ~~~

   * 结果：

     ~~~bash
     (base) root@master:~/Golang/code/go/src/sigs.k8s.io/scheduler-plugins/cmd/scheduler# nerdctl -n k8s.io images
     REPOSITORY                                                         TAG                IMAGE ID        CREATED        PLATFORM       SIZE       BLOB SIZE
     <none>                                                             <none>             7aa439bbe6d5    8 hours ago    linux/amd64    48.53MB    15.33MB
     wuyong20724/scheduler-plugins/test-controller                      v0.1               7aa439bbe6d5    8 hours ago    linux/amd64    48.53MB    15.33MB
     <none>                                                             <none>             06422a44f220    8 hours ago    linux/amd64    76.31MB    22.89MB
     wuyong20724/scheduler-plugins/test-kube-scheduler                  v0.1               06422a44f220    8 hours ago    linux/amd64    76.31MB    22.89MB
     ~~~

2. 安装自定义调度插件

   其实有两种安装方式，这里采用**作为第二个调度程序**的方式安装，[参考文档](https://github.com/kubernetes-sigs/scheduler-plugins/blob/release-1.28/manifests/install/charts/as-a-second-scheduler/README.md)

   进入安装文件夹，使用helm安装

   ~~~bash
   cd $GOPATH/src/sigs.k8s.io/scheduler-plugins/manifests/install/charts/as-a-second-scheduler
   ~~~

   编辑`values.yaml`

   ~~~yaml
   # Default values for scheduler-plugins-as-a-second-scheduler.
   # This is a YAML-formatted file.
   # Declare variables to be passed into your templates.
   
   scheduler:
     name: scheduler-plugins-scheduler	# 这是我们自定义调度插件的调度器的名字，可以更改，会在调度过程中使用到
       #image: registry.k8s.io/scheduler-plugins/kube-scheduler:v0.28.9
     image: wuyong20724/scheduler-plugins/test-kube-scheduler:v0.1
     replicaCount: 1
     leaderElect: false
     nodeSelector: {}
     affinity: {}
     tolerations: []
   
   controller:
     name: scheduler-plugins-controller
       #image: registry.k8s.io/scheduler-plugins/controller:v0.28.9
     image: wuyong20724/scheduler-plugins/test-controller:v0.1
     replicaCount: 1
     nodeSelector: {}
     affinity: {}
     tolerations: []
   
   # LoadVariationRiskBalancing and TargetLoadPacking are not enabled by default
   # as they need extra RBAC privileges on metrics.k8s.io.
   
   plugins:
     enabled: ["Coscheduling","CapacityScheduling","NodeResourceTopologyMatch","NodeResourcesAllocatable"]
     disabled: ["PrioritySort"] # only in-tree plugins need to be defined here
   
   # Customize the enabled plugins' config.
   # Refer to the "pluginConfig" section of manifests/<plugin>/scheduler-config.yaml.
   # For example, for Coscheduling plugin, you want to customize the permit waiting timeout to 10 seconds:
   pluginConfig:
   - name: Coscheduling
     args:
       permitWaitingTimeSeconds: 10 # default is 60
   # Or, customize the other plugins
   # - name: NodeResourceTopologyMatch
   #   args:
   #     scoringStrategy:
   #       type: MostAllocated # default is LeastAllocated
   #- name: SySched
   #  args:
   #    defaultProfileNamespace: "default"
   #    defaultProfileName: "full-seccomp"
   ~~~

   这里将两个镜像改成咱们的自定义调度插件的镜像了，并且，还需要在`plugins.enbaled`中，**添加我们自定义调度插件的名称，用以让调度器在运行时加载咱们自定义插件**

   **关于自定义调度插件的配置参数传入**，请参考如下部分（自定义调度插件的配置参数名称是有要求的，一般为`XXXXArgs`，在设计实现自定义调度插件时注意此问题）
   [[neilats-refactor-scheduler-plugins#修改4]]至[[neilats-refactor-scheduler-plugins#修改7]]

   返回上级目录并安装

   ~~~bash
   cd ../
   helm install scheduler-plugins as-a-second-scheduler/ --create-namespace --namespace scheduler-plugins
   ~~~

   然后可以发现

   ~~~bash
   $ kubectl get deploy -n scheduler-plugins
   NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
   scheduler-plugins-controller   1/1     1            1           7s
   scheduler-plugins-scheduler    1/1     1            1           7s
   ~~~

   > 这里注意，其他几个节点并没有我们的自定义调度插件的镜像，可能会出现拉取镜像错误

3. 自定义调度插件测试

   部署如下测试yaml

   ~~~yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: test
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: test
     template:
       metadata:
         labels:
           app: test
       spec:
         schedulerName: scheduler-plugins-scheduler
         containers:
         - image: busybox:1.36
           name: nginx
           command: ["sleep"]         
           args: ["99999"]
   ~~~

   这里的自定义调度器的名字，参考`$GOPATH/src/sigs.k8s.io/scheduler-plugins/manifests/install/charts/as-a-second-scheduler/values.yaml`中的`scheduler.name`选项

   

# 参考文章

[K8s 调度框架设计与 scheduler plugins 开发部署示例（2024）](https://arthurchiao.art/blog/k8s-scheduling-plugins-zh/#223-postfilterfilter-%E4%B9%8B%E5%90%8E%E6%B2%A1%E6%9C%89-node-%E5%89%A9%E4%B8%8B%E8%A1%A5%E6%95%91%E9%98%B6%E6%AE%B5)

[K8s 自定义调度器 Part2：通过 Scheduler Framework 实现自定义调度逻辑](https://www.lixueduan.com/posts/kubernetes/34-custom-scheduker-by-scheduler-framework/)

