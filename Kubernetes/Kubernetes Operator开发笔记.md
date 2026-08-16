---
tags:
  - Operator
  - Admission-Webhook
  - CRD开发
---

# Chapter2：开始Operator开发

> 使用的软件与版本
>
> 1. Kubernetes：v1.28.15
>
> 2. kubebuilder：v4.8
>
> 3. nerdctl
>
>    ~~~sh
>    (base) root@master:~# nerdctl version
>    Client:
>     Version:       v2.1.1
>     OS/Arch:       linux/amd64
>     Git commit:    d16f34059f5599a58d1ea7aad4e43790f4d27118
>     buildctl:
>      Version:      v0.21.1
>      GitCommit:    66735c67040bc80e6ed104f451683e094030a4e1
>                                                                               
>    Server:
>     containerd:
>      Version:      v2.1.0
>      GitCommit:    061792f0ecf3684fb30a3a0eb006799b8c6638a7
>     runc:
>      Version:      1.1.15
>      GitCommit:    v1.1.15-0-gbc20cb44
>    ~~~

## 2.5.4 CRD部署

* 问题描述

  p40页在对`api/v1/application_types.go`中的`ApplicationSpec`结构体编辑完后，其结构体如下

  ~~~go
  type ApplicationSpec struct {
  	// INSERT ADDITIONAL SPEC FIELDS - desired state of cluster
  	// Important: Run "make" to regenerate code after modifying this file
  	// The following markers will use OpenAPI v3 schema to validate the value
  	// More info: https://book.kubebuilder.io/reference/markers/crd-validation.html
  
  	// foo is an example field of Application. Edit application_types.go to remove/update
  	// +optional
  	Replicas int32 `json:"replicas,omitempty"`
  	Template corev1.PodTemplateSpec `json:"template,omitempty"`
  }
  ~~~

  但是，在经过`make manifests`和`make install`后，`make install`发生错误，信息如下：

  ~~~sh
  (base) root@master:~/Golang/code/go/src/sigs.k8s.io/MyOperatorProjects/application-operator# make install
  /root/Golang/code/go/src/sigs.k8s.io/MyOperatorProjects/application-operator/bin/controller-gen rbac:roleName=manager-role crd webhook paths="./..." output:crd:artifacts:config=config/crd/bases
  /root/Golang/code/go/src/sigs.k8s.io/MyOperatorProjects/application-operator/bin/kustomize build config/crd | kubectl apply -f -
  The CustomResourceDefinition "applications.apps.daniehu.cn" is invalid: metadata.annotations: Too long: must have at most 262144 bytes
  make: *** [Makefile:157: install] Error 1
  ~~~

* 原因分析

  这是因为CRD的annotations总大小超过了256KB的限制，在Kubernetes中，所有对象的`metadata.annotations`字段都有个上限，目前是256KB

  在Operator开发时，常见的原因是：

  1. 使用了 **Kubebuilder/Operator SDK**，自动生成的 `controller-gen` CRD YAML 文件里包含了很大的 `openAPIV3Schema`，而且这个 schema 被塞进了 `metadata.annotations["kubectl.kubernetes.io/last-applied-configuration"]` 或者 `controller-gen.kubebuilder.io/...` 等注解中。
  2. CRD 里定义了很多字段或嵌套结构，导致生成的 schema 特别庞大，从而引起 annotation 过大。

  将定义的CRD结构体分析，给出结论，是**Template corev1.PodTemplateSpec \`json:"template,omitempty"`**这一行导致annotation过大

  `corev1.PodTemplateSpec` → 展开后会包含 **整个 Pod 的 OpenAPI Schema**：

  - metadata、labels、annotations
  - spec（里面有 containers、volumes、probes、env、resources、affinity、tolerations …）
  - 每个字段都有详细的 validation

  当 `controller-gen` 生成 CRD 时，它会把完整的 Pod Schema 展开到 CRD 的 `openAPIV3Schema` 里。这个 Schema 会非常大（几十 KB 甚至上百 KB）。再加上 `kubectl.kubernetes.io/last-applied-configuration` 注解保存一份 YAML，就可能超过 256KB 限制。

* 解决方法

  此处采用的解决方法，是使用 `x-kubernetes-preserve-unknown-fields`,告诉生成器不要展开 `PodTemplateSpec` 的 schema，而是直接“保留原样”，这样生成的 CRD 文件会大大缩小。

  ~~~go
  type ApplicationSpec struct {
  	// INSERT ADDITIONAL SPEC FIELDS - desired state of cluster
  	// Important: Run "make" to regenerate code after modifying this file
  	// The following markers will use OpenAPI v3 schema to validate the value
  	// More info: https://book.kubebuilder.io/reference/markers/crd-validation.html
  
  	// foo is an example field of Application. Edit application_types.go to remove/update
  	// +optional
  	Replicas int32 `json:"replicas,omitempty"`
  
  	// +kubebuilder:pruning:PreserveUnknownFields
  	// +kubebuilder:validation:Schemaless
  	Template corev1.PodTemplateSpec `json:"template,omitempty"`
  }
  ~~~

  这样 `controller-gen` 会把 `template` 当作一个 **任意结构体** 字段，不去展开整个 Pod 的 OpenAPI schema。

  > **缺点**：失去了对 `template` 内部字段的 CRD 层级校验（只能在 admission webhook 或运行时再校验）。



## 2.5.8 部署Controller

* 问题描述

  其实并没有发生问题，只是有个注意点，由于此处咱们使用的镜像构建工具是nerdctl，而不是docker，所以需要对Makefile文件做一些修改，让其适配nerdctl

* 解决方法

  查看make docker-build的.PONY

  ~~~makefile
  .PHONY: docker-build
  docker-build: ## Build docker image with the manager.
  	$(CONTAINER_TOOL) build -t ${IMG} .
  ~~~

  其中`$(CONTAINER_TOOL)`默认是docker

  ~~~makefile
  CONTAINER_TOOL ?= docker
  ~~~

  咱们只需要将docker改为nerdctl即可，因为两者的镜像构建命令基本是一致的




# Chapter4：理解Client-go

## 4.2.1 client-go集群内认证配置

* 问题描述

  p58页贴出了当在集群内部时，认证配置的样例，是将特定命名空间的Pod打印到控制台，其代码如下

  ~~~go
  package in_cluster_configuration
  
  import (
  	"context"
  	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
  	"k8s.io/client-go/kubernetes"
  	"k8s.io/client-go/rest"
  	"log"
  	"time"
  )
  
  func main() {
  
  	//Pod创建时会自动把ServiceAccount token挂载到容器内，所以InClusterConfig就是基于这个原理读取了认证所需的token与ca.crt文件
  	config, err := rest.InClusterConfig()
  	if err != nil {
  		log.Fatal(err)
  	}
  
  	//通过config初始化clientSet，可以实现各种资源的CURD操作
  	clientSet, err := kubernetes.NewForConfig(config)
  	if err != nil {
  		log.Fatal(err)
  	}
  
  	for {
  		pods, err := clientSet.CoreV1().Pods("k8s-learn").List(context.TODO(), metav1.ListOptions{})
  		if err != nil {
  			log.Fatal(err)
  		}
  		log.Printf("There are %d pods in the cluster\n", len(pods.Items))
  		for i, pod := range pods.Items {
  			log.Printf("%d -> %s/%s", i+1, pod.Namespace, pod.Name)
  		}
  		<-time.Tick(5 * time.Second)
  	}
  }
  ~~~

  但是书本中，使用的是`k8s.io/client-go@v0.20.1`与`k8s.io/apimachinery@v0.20.1`两个依赖包

  但是如果使用的是v0.28.0版本的，会发现无法解析`pods.Items`

* 原因分析

  发现是还缺少了依赖包`k8s.io/api@v0.28.0`，关于`pod.Items`的解析在api包中

* 解决方法

  执行如下命令

  ~~~go
  go get k8s.io/api@v0.28.0
  ~~~

  而后即可正常解析`pods.Items`

## 4.2.2 client-go集群外认证配置

* 问题描述

  p61贴出了当在集群外部时，认证配置的样例，是将特定命名空间的Pod打印到控制台，其代码如下

  ~~~go
  package out-of-cluster-configuration
  
  import (
  	"context"
  	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
  	"k8s.io/client-go/kubernetes"
  	"k8s.io/client-go/tools/clientcmd"
  	"k8s.io/client-go/util/homedir"
  	"log"
  	"path/filepath"
  	"time"
  )
  
  func main() {
  	// 获取用户家目录
  	homePath := homedir.HomeDir()
  	if homePath == "" {
  		log.Fatal("failed to get the home directory")
  	}
  
  	// 拼接获得config文件所在路径
  	kubeConfig := filepath.Join(homePath, ".kube", "config")
  
  	// 第一个参数是Kubernetes API的地址，但是由于默认kubeconfig文件中包含API Server的连接信息，所以这里就没有提供第一个参数
  	config, err := clientcmd.BuildConfigFromFlags("", kubeConfig)
  	if err != nil {
  		log.Fatal(err)
  	}
  
  	clientSet, err := kubernetes.NewForConfig(config)
  	if err != nil {
  		log.Fatal(err)
  	}
  
  	for {
  		pods, err := clientSet.CoreV1().Pods("k8s-learn").List(context.TODO(), metav1.ListOptions{})
  		if err != nil {
  			log.Fatal(err)
  		}
  		log.Printf("there are %d pods in the cluster\n", len(pods.Items))
  		for i, pod := range pods.Items {
  			log.Printf("%d -> %s/%s", i+1, pod.Namespace, pod.Name)
  		}
  		<-time.Tick(5 * time.Second)
  	}
  }
  
  ~~~

  在执行代码编译命令时，发现无法编译

  ~~~go
  (base) root@master:~/Golang/code/go/src/sigs.k8s.io/MyOperatorProjects/client-go-examples/out-of-cluster-configuration# go build -o out-of-cluster
  go: downloading k8s.io/api v0.28.0
  ../../../../../pkg/mod/k8s.io/apimachinery@v0.28.0/pkg/apis/meta/v1/generated.pb.go:27:2: missing go.sum entry for module providing package github.com/gogo/protobuf/proto (imported by k8s.io/apimachinery/pkg/apis/meta/v1); to add:
          go get k8s.io/apimachinery/pkg/apis/meta/v1@v0.28.0
  ../../../../../pkg/mod/k8s.io/apimachinery@v0.28.0/pkg/apis/meta/v1/generated.pb.go:28:2: missing go.sum entry for module providing package github.com/gogo/protobuf/sortkeys (imported by k8s.io/apimachinery/pkg/apis/meta/v1); to add:
  
  ...........
  
  (base) root@master:~/Golang/code/go/src/sigs.k8s.io/MyOperatorProjects/client-go-examples/out-of-cluster-configuration# ls
  main.go
  ~~~

  可以看到，编译并没有生成可执行文件`out-of-cluster`

* 原因分析

  **根本原因不是代码错误，而是 Go Module 的依赖完整性问题** —— 你的 `go.sum` 文件缺失了许多间接依赖的校验条目，导致 `go build` 拒绝编译。

  * ❌ 为什么没生成可执行文件？

    因为 `go build` **在编译前会检查模块完整性**。你虽然通过 `go mod download` 下载了 `k8s.io/apimachinery@v0.28.0` 等模块，但：

  * `go.sum` 文件中缺少这些模块所依赖的第三方库（如 `gogo/protobuf`, `golang.org/x/net`, `sigs.k8s.io/yaml` 等）的 checksum。

  * Go 认为模块状态不完整或可能被篡改，因此 **拒绝构建**。

* 解决方法

  在执行编译之前，执行`go mod tidy`

* 问题描述2

  编译生成`out-of-cluster`后，发现其并不是可执行文件

  ~~~go
  (base) root@master:~/Golang/code/go/src/sigs.k8s.io/MyOperatorProjects/client-go-examples/out-of-cluster-configuration# ./out-of-cluster 
  ./out-of-cluster: line 1: syntax error near unexpected token `newline'
  ./out-of-cluster: line 1: `!<arch>'
  ~~~

* 原因分析2

  给出代码后，给出原因，

  *  **根本原因：你的 `main.go` 文件的包名是 `package out_of_cluster_configuration`，而不是 `package main`**

    Go 语言规定：**只有 `package main` 且包含 `func main()` 的文件，才能被编译为可执行程序。**

* 解决方法

  将package 改为main，更改后的代码为：

  ~~~go
  package main
  
  import (
  	"context"
  	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
  	"k8s.io/client-go/kubernetes"
  	"k8s.io/client-go/tools/clientcmd"
  	"k8s.io/client-go/util/homedir"
  	"log"
  	"path/filepath"
  	"time"
  )
  
  func main() {
  	// 获取用户家目录
  	homePath := homedir.HomeDir()
  	if homePath == "" {
  		log.Fatal("failed to get the home directory")
  	}
  
  	// 拼接获得config文件所在路径
  	kubeConfig := filepath.Join(homePath, ".kube", "config")
  
  	// 第一个参数是Kubernetes API的地址，但是由于默认kubeconfig文件中包含API Server的连接信息，所以这里就没有提供第一个参数
  	config, err := clientcmd.BuildConfigFromFlags("", kubeConfig)
  	if err != nil {
  		log.Fatal(err)
  	}
  
  	clientSet, err := kubernetes.NewForConfig(config)
  	if err != nil {
  		log.Fatal(err)
  	}
  
  	for {
  		pods, err := clientSet.CoreV1().Pods("k8s-learn").List(context.TODO(), metav1.ListOptions{})
  		if err != nil {
  			log.Fatal(err)
  		}
  		log.Printf("there are %d pods in the cluster\n", len(pods.Items))
  		for i, pod := range pods.Items {
  			log.Printf("%d -> %s/%s", i+1, pod.Namespace, pod.Name)
  		}
  		<-time.Tick(5 * time.Second)
  	}
  }
  ~~~



# Chapter5：client-go源码分析

## 5.1.2 clien-go模块概览

![image-20250904170942810](https://typora2icture.oss-cn-beijing.aliyuncs.com/img2/202509041709130.png)

这里就不详细介绍每个组件的作用了详见p70页，这里介绍上图整个流程

1. ListAndWatch

   Reflector 向 Kubernetes API Server 发起 `ListAndWatch` 请求，获取指定命名空间中所有 Pod 的初始列表，并开始监听后续的变更事件。

2. Add Obj To FIFO

   API Server 返回的初始 Pod 列表中的每个 Pod 对象都会被添加到 DeltaFIFO 队列中

3. Pop Obj from FIFO

   Informer从DeltaFIFO中弹出相应对象，先通过Indexer将Pod对象和索引`namespace/podName`放到本地的Thread Safe Store中（步骤4）

   然后根据弹出的Pod的状态，触发相应的事件处理函数 Resouce Event Handlers（步骤5）

4. Store Obj & Key

   其实就是Indexer起到的作用，提供根据一定条件检索Pod对象的能力，通过Thread Safe Store来存储

5. Dispatch Event

   Infomer根据Pod的变化类型，调用相应的事件处理器（OnAdd、OnUpdata、OnDelete）

6. Enqueue

   事件处理器将需要处理的 Pod的键添加到工作队列workQueue中

7. Get key from WorkQueue

   Worker从工作队列中取出Pod的键

8. Get Obj from Key

   Worker使用Pod的键从Thread Safe Store中取出相应的Pod对象

9. Get/Updata Objs

   Worker执行实际的业务逻辑，例如更新Pod的状态，触发其他操作等

10. CURD

    如果需要，Worker可以通过ClientSet对Pod进行CRUD操作，例如更新Pod的标签，删除Pod等

> * 请问是每个事件处理器都各自 对应一个WorkQueue和Worker吗?如果不是，那么事件处理器的作用是？为什么Informer不直接将需要处理的Pod的key给到对应不同操作的Worker呢？
>
>   事件处理器并不各自对应一个WorkQueue和Worker，**所有事件处理器共享一个WorkerQueue**，由**一组**Worker从队列厚葬消费并处理
>
>   事件处理器只需要把需要处理的对象塞进WorkQueue，然后立刻返回，不执行任何实际业务逻辑
>
>   **为什么不用多个 WorkQueue / 多个 Worker 分别处理 Add/Update/Delete？**
>
>   1. **同一个对象可能经历多种事件**
>      - 比如一个 Pod：`Add` → `Update` → `Update` → `Delete`
>      - 如果用不同队列，逻辑分散，难以保证顺序和一致性
>   2. **最终目标是“处理对象的当前状态”**
>      - 无论是 Add 还是 Update，你最终都要：
>        - 获取最新 Pod
>        - 检查它的标签、状态
>        - 决定是否要创建 Service、更新 Deployment 等
>      - 所以**处理逻辑是基于对象本身，而不是事件类型**
>   3. **去重需要统一队列**
>      - 如果 `Add` 和 `Update` 发生在短时间内，你只想处理一次最新状态
>      - 多个队列无法共享去重机制
>
>   事件类型信息也没有丢失，虽然入同一个队列，但是任然可以在处理时知道事件类型，**最好还是从Store中获取对象的当前状态来决策**
>
> * 那其实就是，你说的一组worker，每个worker的功能都是一样的吗？然后每个worker处理的key，都通过从Thread Safe Store中获取Pod的当前事件状态来处理这个Pod？如果当前事件处理完毕了，而这个Pod有多个事件，那么worker是一次将这个Pod的所有事件都处理完还是处理完一个事件后下一轮Pod再显示新 的状态需要worker处理？
>
>   1. 一组worker，每个worker的功能都是一样的吗？
>
>      **是的，完全一样。**
>
>      - 所有 Worker 都运行**相同的处理逻辑函数**（比如 `handle(key string)`）。
>      - 它们从**同一个 WorkQueue** 中并发取任务（key）。
>      - 它们彼此独立，互不通信，就像“流水线上的工人”。
>
>   2. 每个worker是如何处理一个key的？
>
>      是的，Worker 会：
>
>      1. 从 WorkQueue 中取出一个 key（如 `"default/my-pod"`）
>      2. 通过 `informer.GetStore().GetByKey(key)` 从 **Thread Safe Store** 中获取该对象的**最新状态**
>      3. 执行业务逻辑（reconcile）
>
>   3. 如果一个Pod有多个事件，worker会一次性处理完所有事件吗？
>
>      不会
>
>      假设一个 Pod 经历了以下变化：
>
>      | 时间 | 事件       | 说明                         |
>      | ---- | ---------- | ---------------------------- |
>      | T0   | Pod 创建   | Reflector 收到 `Add` 事件    |
>      | T1   | 标签更新   | Reflector 收到 `Update` 事件 |
>      | T2   | 副本数变更 | Reflector 收到 `Update` 事件 |
>
>      发生了什么？
>
>      1. **每个事件都会触发 `workqueue.Add(key)`**
>      2. 但由于 WorkQueue 支持**去重（deduplication）**，最终队列中可能只保留一个 `default/my-pod`（除非你禁用了去重）
>      3. 一个 Worker 取出这个 key
>      4. 它从 **Store 中获取 Pod 的最新状态（T2 时刻的状态）**
>      5. 它基于**最终状态**执行一次 reconcile
>      6. 处理完成，返回成功
>
>      ✅ **结果**：虽然发生了 3 次事件，但控制器**只处理了一次**，而且处理的是**最新的状态**。
>
>   4. 那之前的中间的状态会被跳过吗？安全吗？
>
>      **是的，中间状态被跳过，但这是设计意图，是安全的。**
>
>      为什么这是正确的？
>
>      因为 Kubernetes 控制器是 **“状态驱动”**，而不是 **“事件驱动”**。
>
>      - 你关心的不是“发生了什么”，而是“**现在是什么样，应该变成什么样**”。
>      - 即使你错过了中间状态，只要最终状态被处理，系统就会达到预期。
>
>      这叫做 **Reconciliation Loop（调谐循环）**：
>
>      ~~~txt
>      当前状态 → 期望状态 → 执行操作 → 达成一致
>      ~~~



## 5.2.2 延时队列DelayingQueue的实现

> 这里着重分析AddAfter和waiting Loop两个函数的实现和延时队列的概念

### 延时队列简述

* 延时队列概念

  延时队列就是**进入该队列的消息会被延迟消费**的队列；一般的队列，消息一旦入队了之后就被被消费者马上消费

* 延时队列工作场景

  1. 延时消费

     * 用户生成订单后，需要过一段时间校验订单的支付状态，如果订单仍未支付则需要及时关闭订单
     * 用户注册成后，需要过一段时间比如一周后校验用户的使用情况，如果用户活跃度较低，则发送邮件或者短信来提醒用户使用

  2. 延时重试

     消费者从队列里消费消息时失败了，但是想要延时一段时间之后自动重试

  **如果不使用延时队列，只能通过一个轮询扫描程序完成，但是这种方案既不优雅也不方便做成统一的服务方便开发人员使用**

* 延时队列结构

  这部分是自己的补充，从宏观上看，延时队列由两个队列构成，**一个是主消费队列，由不同队列构成；另一个是堆结构，用于定时将元素加入主消费队列**

### Client-go中的延时队列实现

> 在正式说明waitingLoop的实现之前，先了解waitFor结构体

* waitFor结构体

  ~~~go
  type waitFor struct {
  	data    t         // 准备添加到队列中的数据
  	readyAt time.Time // 应该被加入主队列的时间
  	// index in the priority queue (heap)
  	index int // 在heap中的索引
  }
  ~~~

  这个是延时队列中的元素结构

* waitForPriorityQueue

  ~~~go
  type waitForPriorityQueue []*waitFor
  
  func (pq waitForPriorityQueue) Len() int {
  	return len(pq)
  }
  func (pq waitForPriorityQueue) Less(i, j int) bool {
  	return pq[i].readyAt.Before(pq[j].readyAt)
  }
  func (pq waitForPriorityQueue) Swap(i, j int) {
  	pq[i], pq[j] = pq[j], pq[i]
  	pq[i].index = i
  	pq[j].index = j
  }
  
  // Push adds an item to the queue. Push should not be called directly; instead,
  // use `heap.Push`.
  func (pq *waitForPriorityQueue) Push(x interface{}) {
  	n := len(*pq)
  	item := x.(*waitFor)
  	item.index = n
  	*pq = append(*pq, item)
  }
  
  // Pop removes an item from the queue. Pop should not be called directly;
  // instead, use `heap.Pop`.
  func (pq *waitForPriorityQueue) Pop() interface{} {
  	n := len(*pq)
  	item := (*pq)[n-1]
  	item.index = -1
  	*pq = (*pq)[0:(n - 1)]
  	return item
  }
  
  // Peek returns the item at the beginning of the queue, without removing the
  // item or otherwise mutating the queue. It is safe to call directly.
  func (pq waitForPriorityQueue) Peek() interface{} {
  	return pq[0]
  }
  ~~~

  这个切片其实就是client-go中，延时队列的堆部分，名为waitForPriorityQueue，其实现了`heap.Interface`的接口，包含`Len、Less、Swap、Pop、Push`等函数接口

  这也使得`waitForPriorityQueue`可以进行`heap.Init`、`heap.Pop`等堆才可以进行的操作

  **这个堆的实现，其实就是将堆中元素按照时间早晚进行排序的一个小顶堆，最先到加入队列时间的元素在堆顶**

  > 在go中，堆好像就是这样的实现方式，只要传入的结构体实现了`heap.Interface`接口，就可以对该类型的切片进行堆相关的操作

* waitingLoop函数实现

  ~~~go
  // waitingLoop runs until the workqueue is shutdown and keeps a check on the list of items to be added.
  func (q *delayingType) waitingLoop() {
      // 防止goroutine因panic崩溃，崩溃时会由utilruntime捕获并打印堆栈
  	defer utilruntime.HandleCrash()
  
  	// Make a placeholder channel to use when there are no items in our list
      // 创建一个永远不触发的channel，用于在select中占位，当没有等待项的时，nextReadyAt指向never，这样‘<-nextReadyAt’永远不会触发
  	never := make(<-chan time.Time)
  
  	// Make a timer that expires when the item at the head of the waiting queue is ready
      // 用于定时唤醒goroutine，检查堆中最早的那个等待项是否已经就绪
  	var nextReadyAtTimer clock.Timer
  
  	waitingForQueue := &waitForPriorityQueue{}
  	heap.Init(waitingForQueue)
  
      // 用于快速查找某个data是否已经在等待队列中，避免重复插入
  	waitingEntryByData := map[t]*waitFor{}
  
  	for {
          // 如果队列正在关闭，直接退出循环
  		if q.Interface.ShuttingDown() {
  			return
  		}
  
          // 获取当前时间
  		now := q.clock.Now()
  
  		// Add ready entries
          // 处理已经就绪的任务，即批量出堆；由于是一个无限for循环，所以可以批量处理已经到时间的等待项
  		for waitingForQueue.Len() > 0 {
              // 查看堆顶元素，如果堆顶元素的时间在现在时间之后，则break
  			entry := waitingForQueue.Peek().(*waitFor)
  			if entry.readyAt.After(now) {
  				break
  			}
  
              // 说明堆顶元素已经到时间了，则使用heap.Pop获取堆顶元素，并将其加入主队列，并在map中删除对应项
  			entry = heap.Pop(waitingForQueue).(*waitFor)
  			q.Add(entry.data)
  			delete(waitingEntryByData, entry.data)
  		}
  
  		// Set up a wait for the first item's readyAt (if one exists)
          // 设置下一次唤醒时间，即定时器，用于及时检查下一批到时间的等待项
          // 如果等待队列中是空的，则将定时器设置为永不唤醒
  		nextReadyAt := never
  		if waitingForQueue.Len() > 0 {
              // 如果等待队列不为空，并且定时器也不为空的话，需要将旧的定时器停止，防止资源泄露，一切以最新的等待队列的状态为准
  			if nextReadyAtTimer != nil {
  				nextReadyAtTimer.Stop()
  			}
              // 查看等待队列中最先到时间的等待项
  			entry := waitingForQueue.Peek().(*waitFor)
              // 设置定时器在entry.readyAt - now之后时间触发
  			nextReadyAtTimer = q.clock.NewTimer(entry.readyAt.Sub(now))
              // 将nextReadyAt设为这个定时器的通道，用于在select中监听这个定时器
  			nextReadyAt = nextReadyAtTimer.C()
  		}
  
          // 多路监听
  		select {
              // 停止信号：收到停止信号直接退出goroutine
  		case <-q.stopCh:
  			return
  
              // 心跳触发：即使没有任务就绪，也会定期唤醒，防止错过就绪项，唤醒后会重新进入循环，检查是否有新的就绪项
  		case <-q.heartbeat.C():
  			// continue the loop, which will add ready items
  
              // 定时器触发：最近一个等待项已经就绪或即将就绪，出发后循环重新开始，进入批量处理就绪项阶段
  		case <-nextReadyAt:
  			// continue the loop, which will add ready items
  
              // 新增等待项：有新的延时任务被加入到延时队列
  		case waitEntry := <-q.waitingForAddCh:
              // 先判断新加入的等待项是否已经到时间，如果没到时间，先将其插入等待队列和map中；如果到时间则直接加入主队列
  			if waitEntry.readyAt.After(q.clock.Now()) {
  				insert(waitingForQueue, waitingEntryByData, waitEntry)
  			} else {
  				q.Add(waitEntry.data)
  			}
  
              //批量排空：上面针对新到来的等待项，只能处理一个，不希望等待项在channel中堆积，所以堆channel中剩余的等待项进行处理
  			drained := false
  			for !drained {
  				select {
                      // 针对每个新到来的等待想的逻辑同上
  				case waitEntry := <-q.waitingForAddCh:
  					if waitEntry.readyAt.After(q.clock.Now()) {
  						insert(waitingForQueue, waitingEntryByData, waitEntry)
  					} else {
  						q.Add(waitEntry.data)
  					}
                      // 直到q.waitingForAddCh没有触发，也就是channel中没有等待项，drained为true，也就是channel排空了
  				default:
  					drained = true
  				}
  			}
  		}
  	}
  }
  
  // insert adds the entry to the priority queue, or updates the readyAt if it already exists in the queue
  func insert(q *waitForPriorityQueue, knownEntries map[t]*waitFor, entry *waitFor) {
  	// if the entry already exists, update the time only if it would cause the item to be queued sooner
  	existing, exists := knownEntries[entry.data]
      // 如果map中已经存在了新的等待项，并且新等待项的时间早于已经存在的等待项的时间，则以新等待项的时间为准
  	if exists {
  		if existing.readyAt.After(entry.readyAt) {
  			existing.readyAt = entry.readyAt
  			heap.Fix(q, existing.index)
  		}
  
  		return
  	}
  
      // 如果等待队列中没有这个新等待项，则直接加入
  	heap.Push(q, entry)
  	knownEntries[entry.data] = entry
  }
  ~~~

  这个函数的作用是，管理一个等待队列，其中的元素只在指定时间`readyAt`之后才能真正加入主队列进行处理，其实现了延迟添加、高效唤醒、批量处理ready项，非阻塞接收新等待项

  其逻辑图如下
[[client-go延时队列工作流程]]
![image.png](https://typora2icture.oss-cn-beijing.aliyuncs.com/img2/202509101434946.png)


* AddAfter方法

  ~~~go
  // AddAfter adds the given item to the work queue after the given delay
  func (q *delayingType) AddAfter(item interface{}, duration time.Duration) {
  	// don't add if we're already shutting down
  	if q.ShuttingDown() {
  		return
  	}
  
  	q.metrics.retry()
  
  	// immediately add things with no delay
      // 如果等待项已经到时间了，则直接加入主队列
  	if duration <= 0 {
  		q.Add(item)
  		return
  	}
  
  	select {
  	case <-q.stopCh:
  		// unblock if ShutDown() is called
          // 否则创建一个waitFor类型元素，并通过channel加入等待队列
  	case q.waitingForAddCh <- &waitFor{data: item, readyAt: q.clock.Now().Add(duration)}:
  	}
  }
  ~~~

## 5.2.3 限速队列RateLimitingQueue的实现

### 限速队列简述

限速队列的作用是，控制从队列中取出任务处理的频率，防止对后端系统API Server造成过大压力

在client-go的工作流程中，就是控制worker从work queue中取出任务处理的频率，如果一个item任务在一次处理中失败了，那么根据其失败的次数和配置的限速器确定其再次重试的时间间隔

再次重试的方法就是将其按照给定的时间间隔放入延时队列中

流程如下：
![[限速队列工作流程.svg]]

* 限速队列的实现

  ~~~go
  type rateLimitingType struct {
      // 延时队列
  	DelayingInterface
  
      // 限速器
  	rateLimiter RateLimiter
  }
  
  // AddRateLimited AddAfter's the item based on the time when the rate limiter says it's ok
  // 使用限速器的When方法给出的时间作为让这个item等待的duration，重新加入延时队列
  func (q *rateLimitingType) AddRateLimited(item interface{}) {
  	q.DelayingInterface.AddAfter(item, q.rateLimiter.When(item))
  }
  
  func (q *rateLimitingType) NumRequeues(item interface{}) int {
  	return q.rateLimiter.NumRequeues(item)
  }
  
  func (q *rateLimitingType) Forget(item interface{}) {
  	q.rateLimiter.Forget(item)
  }
  ~~~

### 限速器的几种实现

> 限速器接口定义：
>
> ~~~go
> type RateLimiter interface {
>     
> 	// When gets an item and gets to decide how long that item should wait
>     // 返回获取一个元素需要等待的时长
> 	When(item interface{}) time.Duration
>     
> 	// Forget indicates that an item is finished being retried.  Doesn't matter whether it's for failing
> 	// or for success, we'll stop tracking it
>     // 标记一个元素结束重试
> 	Forget(item interface{})
>     
> 	// NumRequeues returns back how many failures the item has had
>     // 标记这个元素被处理了多少次了
> 	NumRequeues(item interface{}) int
> }
> ~~~

* BucketRateLimiter

  ~~~go
  func DefaultControllerRateLimiter() RateLimiter {
  	return NewMaxOfRateLimiter(
  		NewItemExponentialFailureRateLimiter(5*time.Millisecond, 1000*time.Second),
  		// 10 qps, 100 bucket size.  This is only for retry speed and its only the overall factor (not per item)
          // 配置为10 QPS，100桶大小；平均每秒最多处理10个任务、允许突发最多100个任务、超过限制的任务会被延迟处理（通过Delay返回非零值）
  		&BucketRateLimiter{Limiter: rate.NewLimiter(rate.Limit(10), 100)},
  	)
  }
  
  // BucketRateLimiter adapts a standard bucket to the workqueue ratelimiter API
  type BucketRateLimiter struct {
  	*rate.Limiter
  }
  
  // BucketRateLimiter实现了RateLimiter接口
  var _ RateLimiter = &BucketRateLimiter{}
  
  func (r *BucketRateLimiter) When(item interface{}) time.Duration {
      // Reserve尝试预留一个令牌；Delay返回需要多久才能拿到这个令牌
      // 返回值是一个time.Duration，告诉队列，这个任务要等多久才能被处理，用于从work Queue中取出任务
  	return r.Limiter.Reserve().Delay()
  }
  
  // 因为其是全局限速器，所以不记录每个item重试次数，所以返回0
  func (r *BucketRateLimiter) NumRequeues(item interface{}) int {
  	return 0
  }
  
  // 因为不跟踪item，无需清理
  func (r *BucketRateLimiter) Forget(item interface{}) {
  }
  ~~~

  - 它是对 Go 标准库 `golang.org/x/time/rate.Limiter` 的封装。

  - 使用的是经典的 **令牌桶算法（Token Bucket）**。

    想象一个桶：

    - 每秒往桶里放 `r` 个令牌（比如 10 个）；
    - 桶最多能存 `b` 个令牌（比如 100 个）；
    - 每次你想执行一个操作（如处理一个任务），必须从桶里拿一个令牌；
    - 如果没有令牌了，你就得等。

    这就实现了 **限流**：平均速率不超过 `r`，但允许短时间突发（最多 `b` 次）。

* ItemExponentialFailureRateLimiter

  ~~~go
  type ItemExponentialFailureRateLimiter struct {
  	failuresLock sync.Mutex	// 保护failures字典的并发访问，线程安全
  	failures     map[interface{}]int	// 记录每个item当前已经失败的次数
  
  	baseDelay time.Duration	// 初始延时时间（第一次失败后等待多久重试）
  	maxDelay  time.Duration	// 最大延时时间（防止无限增长）
  }
  
  // ItemExponentialFailureRateLimiter实现了RateLimiter
  var _ RateLimiter = &ItemExponentialFailureRateLimiter{}
  
  // 初始化一个限速器，按照自定义的初始延时时间和最大延时时间
  func NewItemExponentialFailureRateLimiter(baseDelay time.Duration, maxDelay time.Duration) RateLimiter {
  	return &ItemExponentialFailureRateLimiter{
          // 初始化一个failures映射
  		failures:  map[interface{}]int{},
  		baseDelay: baseDelay,
  		maxDelay:  maxDelay,
  	}
  }
  
  // 默认的ItemExponentialFailuresRateLimiter限速器
  func DefaultItemBasedRateLimiter() RateLimiter {
  	return NewItemExponentialFailureRateLimiter(time.Millisecond, 1000*time.Second)
  }
  
  // 决定延时多久
  func (r *ItemExponentialFailureRateLimiter) When(item interface{}) time.Duration {
      // 获取锁，针对failures Map的互斥访问锁
  	r.failuresLock.Lock()
  	defer r.failuresLock.Unlock()
  
  	exp := r.failures[item]	// 获取当前失败次数（作为指数）
  	r.failures[item] = r.failures[item] + 1	// 失败次数+1
  
  	// The backoff is capped such that 'calculated' value never overflows.、
      // 计算延迟：baseDelay * 2^exp
      // baseDelay转成纳秒参与计算，避免精度丢失
  	backoff := float64(r.baseDelay.Nanoseconds()) * math.Pow(2, float64(exp))
      // 防止浮点数溢出导致duration越界
  	if backoff > math.MaxInt64 {
  		return r.maxDelay
  	}
  
      // 截断到最大延迟：即使没溢出，也要限制在合理范围；比如最多延迟1000秒，不能再多了
      calculated := time.Duration(backoff)
  	if calculated > r.maxDelay {
  		return r.maxDelay
  	}
  
  	return calculated
  }
  
  // 返回该item被重试的次数，即失败次数；被Forget调用，这个计数会清零
  func (r *ItemExponentialFailureRateLimiter) NumRequeues(item interface{}) int {
  	r.failuresLock.Lock()
  	defer r.failuresLock.Unlock()
  
  	return r.failures[item]
  }
  
  // 当某个item被成功处理完成后，应该调用queue.Forget(item)；这会触发limiter.Forget(item)，清除其失败计数
  // 如果它再次进入队列，就从baseDelay开始重新计算；这是非常重要的清理机制，避免长期积累错误状态
  func (r *ItemExponentialFailureRateLimiter) Forget(item interface{}) {
  	r.failuresLock.Lock()
  	defer r.failuresLock.Unlock()
  
  	delete(r.failures, item)
  }
  ~~~

  该限速器，为每个独立的任务项item实现**指数退避重试策略**，防止某个失败对象被无限高频重试，从而保护系统稳定性

  * 优点：每个item独立计数，互不影响，防止热点对象刷屏重试，可控的最大延时，避免无限等待
  * 缺点：使用`map[interface{}]int`，若item不可比较或者未重引用可能导致内存泄漏，需要手动调用`forget`，否则状态不会清除

  该限速器与全局限速器（如BucketRateLimiter）配合使用，构成Kubernetes控制器中最常见的双重保护机制
  
* ItemFastSlowRateLimiter快慢限速器

  ~~~go
  // ItemFastSlowRateLimiter does a quick retry for a certain number of attempts, then a slow retry after that
  type ItemFastSlowRateLimiter struct {
  	failuresLock sync.Mutex	// map的互斥访问锁，与上面的限速器一样
  	failures     map[interface{}]int	// 用于记录每个item失败的次数
  
  	maxFastAttempts int	// 快速重试的次数上限
  	fastDelay       time.Duration	// 快重试间隔
  	slowDelay       time.Duration	// 慢重试间隔
  }
  
  // ItemFastSlowRateLimiter实现了RateLimiter的接口
  var _ RateLimiter = &ItemFastSlowRateLimiter{}
  
  func NewItemFastSlowRateLimiter(fastDelay, slowDelay time.Duration, maxFastAttempts int) RateLimiter {
  	return &ItemFastSlowRateLimiter{
  		failures:        map[interface{}]int{},
  		fastDelay:       fastDelay,
  		slowDelay:       slowDelay,
  		maxFastAttempts: maxFastAttempts,
  	}
  }
  
  func (r *ItemFastSlowRateLimiter) When(item interface{}) time.Duration {
  	r.failuresLock.Lock()
  	defer r.failuresLock.Unlock()
      
      // 当前item失败次数+1
  	r.failures[item] = r.failures[item] + 1
  
      // 如果当前失败次数小于快速重试次数上限，返回快速重试时间间隔，否则返回慢速重试间隔
  	if r.failures[item] <= r.maxFastAttempts {
  		return r.fastDelay
  	}
  
  	return r.slowDelay
  }
  
  func (r *ItemFastSlowRateLimiter) NumRequeues(item interface{}) int {
  	r.failuresLock.Lock()
  	defer r.failuresLock.Unlock()
  
  	return r.failures[item]
  }
  
  func (r *ItemFastSlowRateLimiter) Forget(item interface{}) {
  	r.failuresLock.Lock()
  	defer r.failuresLock.Unlock()
  
  	delete(r.failures, item)
  }
  ~~~

* MaxOfRateLimiter

  ~~~go
  // MaxOfRateLimiter calls every RateLimiter and returns the worst case response
  // When used with a token bucket limiter, the burst could be apparently exceeded in cases where particular items
  // were separately delayed a longer time.
  type MaxOfRateLimiter struct {
      // 限速器切片，可以传入任意多个限速器；所有操作都会广播给所有子限速器，并取最严格的结果
  	limiters []RateLimiter
  }
  
  func (r *MaxOfRateLimiter) When(item interface{}) time.Duration {
  	ret := time.Duration(0)
      // 取所有限速器中给出的最大时延
  	for _, limiter := range r.limiters {
  		curr := limiter.When(item)
  		if curr > ret {
  			ret = curr
  		}
  	}
  
  	return ret
  }
  
  func NewMaxOfRateLimiter(limiters ...RateLimiter) RateLimiter {
  	return &MaxOfRateLimiter{limiters: limiters}
  }
  
  func (r *MaxOfRateLimiter) NumRequeues(item interface{}) int {
  	ret := 0
      // 同理，重试次数也取最大重试次数
  	for _, limiter := range r.limiters {
  		curr := limiter.NumRequeues(item)
  		if curr > ret {
  			ret = curr
  		}
  	}
  
  	return ret
  }
  
  func (r *MaxOfRateLimiter) Forget(item interface{}) {
      // 逐个限速器调用Forget函数
  	for _, limiter := range r.limiters {
  		limiter.Forget(item)
  	}
  }
  ~~~

  该限速器**同时应用多个限速器，并返回它们中延迟最长的那个结果**，这实现了多重保护机制，既防止整体过载，也防止单个对象频繁失败

* WithMaxWaitRateLimiter

  ~~~go
  // WithMaxWaitRateLimiter have maxDelay which avoids waiting too long
  type WithMaxWaitRateLimiter struct {
  	limiter  RateLimiter	// 其他限速器
  	maxDelay time.Duration	// 最大时延
  }
  
  func NewWithMaxWaitRateLimiter(limiter RateLimiter, maxDelay time.Duration) RateLimiter {
  	return &WithMaxWaitRateLimiter{limiter: limiter, maxDelay: maxDelay}
  }
  
  func (w WithMaxWaitRateLimiter) When(item interface{}) time.Duration {
  	delay := w.limiter.When(item)
      // 如果其他限速器给定的时延大于给定的最大时延，则直接返回最大时延
  	if delay > w.maxDelay {
  		return w.maxDelay
  	}
  
  	return delay
  }
  
  func (w WithMaxWaitRateLimiter) Forget(item interface{}) {
  	w.limiter.Forget(item)
  }
  
  func (w WithMaxWaitRateLimiter) NumRequeues(item interface{}) int {
  	return w.limiter.NumRequeues(item)
  }
  ~~~

  就是在其他限速器上包装一个最大延迟属性，如果到了最大延时，则直接返回

## 5.3.1 Queue接口与DeltaFIFO的实现

> DeltaFIFO是client-go中从Reflector到Informer流程中的核心数据结构，其存储的不是原始对象，而是对象的变更记录
>
> 缓存对象的变更历史，并保证消费者按顺序处理每个对象的所有变更

### Queue接口和Store接口

* 首先是Queue接口定义

  ~~~go
  type Queue interface {
  	Store
  
  	// Pop blocks until there is at least one key to process or the
  	// Queue is closed.  In the latter case Pop returns with an error.
  	// In the former case Pop atomically picks one key to process,
  	// removes that (key, accumulator) association from the Store, and
  	// processes the accumulator.  Pop returns the accumulator that
  	// was processed and the result of processing.  The PopProcessFunc
  	// may return an ErrRequeue{inner} and in this case Pop will (a)
  	// return that (key, accumulator) association to the Queue as part
  	// of the atomic processing and (b) return the inner error from
  	// Pop.
      // 会阻塞，直到有一个元素可以Pop出来或者队列关闭
  	Pop(PopProcessFunc) (interface{}, error)
  
  	// AddIfNotPresent puts the given accumulator into the Queue (in
  	// association with the accumulator's key) if and only if that key
  	// is not already associated with a non-empty accumulator.
  	AddIfNotPresent(interface{}) error
  
  	// HasSynced returns true if the first batch of keys have all been
  	// popped.  The first batch of keys are those of the first Replace
  	// operation if that happened before any Add, AddIfNotPresent,
  	// Update, or Delete; otherwise the first batch is empty.
  	HasSynced() bool
  
  	// Close the queue
  	Close()
  }
  ~~~

* 其中嵌套的Store接口

  ~~~go
  type Store interface {
  
  	// Add adds the given object to the accumulator associated with the given object's key
  	Add(obj interface{}) error
  
  	// Update updates the given object in the accumulator associated with the given object's key
  	Update(obj interface{}) error
  
  	// Delete deletes the given object from the accumulator associated with the given object's key
  	Delete(obj interface{}) error
  
  	// List returns a list of all the currently non-empty accumulators
  	List() []interface{}
  
  	// ListKeys returns a list of all the keys currently associated with non-empty accumulators
  	ListKeys() []string
  
  	// Get returns the accumulator associated with the given object's key
  	Get(obj interface{}) (item interface{}, exists bool, err error)
  
  	// GetByKey returns the accumulator associated with the given key
  	GetByKey(key string) (item interface{}, exists bool, err error)
  
  	// Replace will delete the contents of the store, using instead the
  	// given list. Store takes ownership of the list, you should not reference
  	// it after calling this function.
  	Replace([]interface{}, string) error
  
  	// Resync is meaningless in the terms appearing here but has
  	// meaning in some implementations that have non-trivial
  	// additional behavior (e.g., DeltaFIFO).
  	Resync() error
  }
  ~~~

  就是一些基础的方法

### DeltaFIFO关于Queue接口的实现

> 需要注意：经过了解，在go语言中，像client-go这种大型库中，并不需要使用`var _ XXXInterface = &MyStruct{}`这种接口断言来强制实现接口。这种写法是一种可选的静态检查技巧，并非Go语言的语法要求
>
> 只要一个结构体实现了接口中定义的所有方法，就会被自动认为是该接口的实现，无需显式声明

* DeltaFIFO结构体定义

  ~~~go
  type DeltaFIFO struct {
  	// lock/cond protects access to 'items' and 'queue'.
  	lock sync.RWMutex
  	cond sync.Cond
  
  	// `items` maps a key to a Deltas.
  	// Each such Deltas has at least one Delta.
  	items map[string]Deltas
  
  	// `queue` maintains FIFO order of keys for consumption in Pop().
  	// There are no duplicates in `queue`.
  	// A key is in `queue` if and only if it is in `items`.
      // queue中是没有重复元素的，与items中的key保持一致
  	queue []string
  
  	// populated is true if the first batch of items inserted by Replace() has been populated
  	// or Delete/Add/Update/AddIfNotPresent was called first.
  	populated bool
  	// initialPopulationCount is the number of items inserted by the first call of Replace()
  	initialPopulationCount int
  
  	// keyFunc is used to make the key used for queued item
  	// insertion and retrieval, and should be deterministic.
  	keyFunc KeyFunc
  
  	// knownObjects list keys that are "known" --- affecting Delete(),
  	// Replace(), and Resync()
  	knownObjects KeyListerGetter
  
  	// Used to indicate a queue is closed so a control loop can exit when a queue is empty.
  	// Currently, not used to gate any of CRUD operations.
  	closed bool
  
  	// emitDeltaTypeReplaced is whether to emit the Replaced or Sync
  	// DeltaType when Replace() is called (to preserve backwards compat).
  	emitDeltaTypeReplaced bool
  
  	// Called with every object if non-nil.
  	transformer TransformFunc
  }
  ~~~

  最重要的就是`items`和`queue`两个变量，其中`Delats`类型的定义如下：

  ~~~go
  type Delta struct {
  	Type   DeltaType
  	Object interface{}
  }
  
  // Deltas is a list of one or more 'Delta's to an individual object.
  // The oldest delta is at index 0, the newest delta is the last one.
  type Deltas []Delta
  ~~~
  
  可见，是一个Delta类型的切片，而Delta类型是包含了Type和Object的结构体，而DeltaType类型则是自定义的
  
  ~~~go
  // DeltaType is the type of a change (addition, deletion, etc)
  type DeltaType string
  
  // Change type definition
  const (
  	Added   DeltaType = "Added"
  	Updated DeltaType = "Updated"
  	Deleted DeltaType = "Deleted"
  	// Replaced is emitted when we encountered watch errors and had to do a
  	// relist. We don't know if the replaced object has changed.
  	//
  	// NOTE: Previous versions of DeltaFIFO would use Sync for Replace events
  	// as well. Hence, Replaced is only emitted when the option
  	// EmitDeltaTypeReplaced is true.
  	Replaced DeltaType = "Replaced"
  	// Sync is for synthetic events during a periodic resync.
  	Sync DeltaType = "Sync"
  )
  ~~~
  
  而**Object则是就是这个Delta对应的对象，比如某个具体的Pod**；说白了**Delta类型就是将对象与对象做的改变声明结合起来，变为一个变更记录的结构体类型**
  
  最终，DeltaFIFO存储结构大致如下：
  ![image.png](https://typora2icture.oss-cn-beijing.aliyuncs.com/img2/202509101650542.png)

  **即items存储的是一个对象的一系列的变更记录**，其Key的构造，是由其成员函数`keyFunc`完成的，默认是由`MetaNamespaceKeyFunc`函数完成，在其New函数中可见
## 5.3.2 queueActionLocked方法的逻辑
* DeltaFIFO实现了Queue的接口，也就实现了`Add()`、`Update()`、`Delete()`等等方法，可以在源代码中看到，`Add()`、`Update()`、`Delete()`方法的实现过程都用到了一个函数`queueActionLocked()`

  ~~~go
  func (f *DeltaFIFO) queueActionLocked(actionType DeltaType, obj interface{}) error {
      // 获取对象的名称，也就是存储时用的Key，默认调用的是MetaNamespaceKeyFunc函数
  	id, err := f.KeyOf(obj)
  	if err != nil {
  		return KeyError{obj, err}
  	}
  
  	// Every object comes through this code path once, so this is a good
  	// place to call the transform func.  If obj is a
  	// DeletedFinalStateUnknown tombstone, then the containted inner object
  	// will already have gone through the transformer, but we document that
  	// this can happen. In cases involving Replace(), such an object can
  	// come through multiple times.
      // 这个暂时不理解
  	if f.transformer != nil {
  		var err error
  		obj, err = f.transformer(obj)
  		if err != nil {
  			return err
  		}
  	}
  
      // 获取原来的对象变更记录
  	oldDeltas := f.items[id]
      // 新的变更记录是原本的变更记录追加最新的变更方式，变更方式就是获取的对象本体+actionType
  	newDeltas := append(oldDeltas, Delta{actionType, obj})
      // 这一步操作是对更新之后的变更记录进行去重操作，例如连续的Update可能被合并，Add+Delete可能被优化掉
  	newDeltas = dedupDeltas(newDeltas)
  
      // 如果去重后仍有delta；理论上是绝对有delta的，但是为了防御性编程，写成这样
  	if len(newDeltas) > 0 {
          // 该if的解释见下面的问题1
  		if _, exists := f.items[id]; !exists {
  			f.queue = append(f.queue, id)
  		}
          // 保存最新的变更记录列表
  		f.items[id] = newDeltas
          // 唤醒所有等待该队列的消费者，比如Pop正在阻塞的goroutine
  		f.cond.Broadcast()
          // 否则newDeltas为空，理论上不会发生，因为dedupDeltas不会返回空列表（当输入非空时），这么写是为了防御性编程
  	} else {
  		// This never happens, because dedupDeltas never returns an empty list
  		// when given a non-empty list (as it is here).
  		// If somehow it happens anyway, deal with it but complain.
  		if oldDeltas == nil {
  			klog.Errorf("Impossible dedupDeltas for id=%q: oldDeltas=%#+v, obj=%#+v; ignoring", id, oldDeltas, obj)
  			return nil
  		}
  		klog.Errorf("Impossible dedupDeltas for id=%q: oldDeltas=%#+v, obj=%#+v; breaking invariant by storing empty Deltas", id, oldDeltas, obj)
  		f.items[id] = newDeltas
  		return fmt.Errorf("Impossible dedupDeltas for id=%q: oldDeltas=%#+v, obj=%#+v; broke DeltaFIFO invariant by storing empty Deltas", id, oldDeltas, obj)
  	}
  	return nil
  }
  ~~~

  1. 问题1：因为newDeltas是由oldDeltas追加新变更记录来的，而oldDeltras是通过id从items中提取的，那么`if _, exists := f.items[id]; !exists {f.queue = append(f.queue, id)}`**理论上id就不可能不存在items中，为什么还这么写？**

     **在Go中，从一个Map中读取一个不存的Key会返回该类型的零值，但是不会自动创建条目**

     `oldDeltas := f.items[id]`只是读取，如果id不存在，oldDeltas是nil，但是id仍然不在map中，后续的`f.items[id]=newDeltas`才是写入，此时id才被真正插入map

* 与Add、Update不太一样的Delete

  ~~~go
  func (f *DeltaFIFO) Delete(obj interface{}) error {
  	id, err := f.KeyOf(obj)
  	if err != nil {
  		return KeyError{obj, err}
  	}
  	f.lock.Lock()
  	defer f.lock.Unlock()
  	f.populated = true
  	if f.knownObjects == nil {
  		if _, exists := f.items[id]; !exists {
  			// Presumably, this was deleted when a relist happened.
  			// Don't provide a second report of the same deletion.
  			return nil
  		}
  	} else {
  		// We only want to skip the "deletion" action if the object doesn't
  		// exist in knownObjects and it doesn't have corresponding item in items.
  		// Note that even if there is a "deletion" action in items, we can ignore it,
  		// because it will be deduped automatically in "queueActionLocked"
  		_, exists, err := f.knownObjects.GetByKey(id)
  		_, itemsExist := f.items[id]
  		if err == nil && !exists && !itemsExist {
  			// Presumably, this was deleted when a relist happened.
  			// Don't provide a second report of the same deletion.
  			return nil
  		}
  	}
  
  	// exist in items and/or KnownObjects
  	return f.queueActionLocked(Deleted, obj)
  }
  ~~~

  



## 5.3.3 Pop方法和Replace方法的逻辑

### Pop方法的实现

> Pop会按照元素的添加或更新顺序有序的返回一个元素(Deltas)，在队列为空时会阻塞；
>
> Pop过程会先从队列中删除一个元素后返回，所以如果处理失败了，则需要通过`AddIfNotPresent`方法将这个元素加回队列中

~~~go
// 其参数是一个处理函数，Pop将队列的第一个元素出队，然后丢给Process处理，如果失败会重新入队，但是对应的Deltas和对应的错误信息会返回
func (f *DeltaFIFO) Pop(process PopProcessFunc) (interface{}, error) {
	f.lock.Lock()
	defer f.lock.Unlock()
	for {
        // 如果队列为空，让所有等待queue的程序等待
		for len(f.queue) == 0 {
			// When the queue is empty, invocation of Pop() is blocked until new item is enqueued.
			// When Close() is called, the f.closed is set and the condition is broadcasted.
			// Which causes this loop to continue and return from the Pop().
			if f.closed {
				return nil, ErrFIFOClosed
			}

			f.cond.Wait()
		}
        // 见下面的问题1
		isInInitialList := !f.hasSynced_locked()
		id := f.queue[0]	// 获取队列第一个元素的key
		f.queue = f.queue[1:]	// 将第一个元素Pop出去
		depth := len(f.queue)	// 获取Pop之后的队列长度
		if f.initialPopulationCount > 0 {
			f.initialPopulationCount--
		}
		item, ok := f.items[id]	// 从items中获取第一个key的Deltas列表，理论上不可能找不到；如果找不到就重新出队一个，因此加上了外层的for循环
		if !ok {
			// This should never happen
			klog.Errorf("Inconceivable! %q was in f.queue but not f.items; ignoring.", id)
			continue
		}
        // 从items中删除这个key的Deltas变更记录
		delete(f.items, id)
		// Only log traces if the queue depth is greater than 10 and it takes more than
		// 100 milliseconds to process one item from the queue.
		// Queue depth never goes high because processing an item is locking the queue,
		// and new items can't be added until processing finish.
		// https://github.com/kubernetes/kubernetes/issues/103789
        // 当队列长度大于10，启动性能追踪，如果处理单个item超过100ms就会记录日志，目的是检测事件处理器太慢导致队列积压；并且Pop一直持有锁导致新事件无法入队
		if depth > 10 {
			trace := utiltrace.New("DeltaFIFO Pop Process",
				utiltrace.Field{Key: "ID", Value: id},
				utiltrace.Field{Key: "Depth", Value: depth},
				utiltrace.Field{Key: "Reason", Value: "slow event handlers blocking the queue"})
			defer trace.LogIfLong(100 * time.Millisecond)
		}
        // 将这个变更记录交给process处理程序进行处理
		err := process(item, isInInitialList)
        // 如果process返回ErrRequeue错误，表示这次处理失败，重新入队重试
		if e, ok := err.(ErrRequeue); ok {
            // 调用addIfNotPresent，如果当前key不在f.items中，则重新加入队列
			f.addIfNotPresent(id, item)
			err = e.Err
		}
		// Don't need to copyDeltas here, because we're transferring
		// ownership to the caller.
		return item, err
	}
}
~~~

* 问题1：该行代码涉及一个新概念，即**全量同步**

### Replace方法的实现

> 该方法简单做了两件事
>
> 1. 给传入的对象列表添加了一个Sync/Replace DeltaType的Delta
> 2. 执行一些与删除相关的程序逻辑
>
> Replace过程简单理解为传递一个新的[]Deltas过来，如果当前DeltaFIFO中已经有这些元素，则追加一个Sync/Replace动作，反之DeltaFIFO中多出来的Deltas可能与apiserver失联导致实际被删除掉，但是删除事件并没有被监听到，所以直接追加一个类型为Deleted类型的Delta

~~~go
func (f *DeltaFIFO) Replace(list []interface{}, _ string) error {
	f.lock.Lock()
	defer f.lock.Unlock()
    // 提前创建一个用于存储最新Pod的Key的空列表
	keys := make(sets.String, len(list))

	// keep backwards compat for old clients
    // 为了和老版本保持兼容，如果是老版本使用Sync，新版本使用Replaced
	action := Sync
	if f.emitDeltaTypeReplaced {
		action = Replaced
	}

	// Add Sync/Replaced action for each new item.
    // 处理最新Pod列表中的每个对象，这里的item就是Pod对象Obj了
	for _, item := range list {
        // 获取Key名称
		key, err := f.KeyOf(item)
		if err != nil {
			return KeyError{item, err}
		}
        // 将Pod名称添加到最新名称列表中
		keys.Insert(key)
        // 如果该对象Obj本来就在items中，则追加Sync/Replaced 这样的Delta；如果没有则新添加一个items[key]=[{Sync/Rplaced, obj}]
		if err := f.queueActionLocked(action, item); err != nil {
			return fmt.Errorf("couldn't enqueue object: %v", err)
		}
	}

	// Do deletion detection against objects in the queue
    // 检测本地多余的对象，即原本在items中，但是没有出现在最新的名称列表Keys中的，也就是原本在items中，但是现在不在集群中的那些，为那些对象补发Delete事件
	queuedDeletions := 0
    // 遍历items，k为key， item为{DeltaTyep，obj}列表
	for k, oldItem := range f.items {
        // 如果key在最新名称列表中，继续
		if keys.Has(k) {
			continue
		}
		// Delete pre-existing items not in the new list.
		// This could happen if watch deletion event was missed while
		// disconnected from apiserver.
        // 如果不在，不发Deleted事件
		var deletedObj interface{}
        // 获取在失去联系之前最新版本的Obj对象，通过Delta来获取，n为{DeltaTyep，obj}
		if n := oldItem.Newest(); n != nil {
			deletedObj = n.Object

			// if the previous object is a DeletedFinalStateUnknown, we have to extract the actual Object
            // 使用DeletedFinalStateUnKnown类型来包装被删除前的最后一个版本对象
			if d, ok := deletedObj.(DeletedFinalStateUnknown); ok {
				deletedObj = d.Obj
			}
		}
		queuedDeletions++
        // 为k这个已经失去联系的对象补发Deleted事件
		if err := f.queueActionLocked(Deleted, DeletedFinalStateUnknown{k, deletedObj}); err != nil {
			return err
		}
	}

    // 同理针对KnownObjects中的对象进行残留对象检测，至于KnownObject数据从何来，请联系Indexer和ThreadSafeStore一起理解。
	if f.knownObjects != nil {
		// Detect deletions for objects not present in the queue, but present in KnownObjects
		knownKeys := f.knownObjects.ListKeys()
		for _, k := range knownKeys {
			if keys.Has(k) {
				continue
			}
			if len(f.items[k]) > 0 {
				continue
			}

			deletedObj, exists, err := f.knownObjects.GetByKey(k)
			if err != nil {
				deletedObj = nil
				klog.Errorf("Unexpected error %v during lookup of key %v, placing DeleteFinalStateUnknown marker without object", err, k)
			} else if !exists {
				deletedObj = nil
				klog.Infof("Key %v does not exist in known objects store, placing DeleteFinalStateUnknown marker without object", k)
			}
			queuedDeletions++
			if err := f.queueActionLocked(Deleted, DeletedFinalStateUnknown{k, deletedObj}); err != nil {
				return err
			}
		}
	}

	if !f.populated {
		f.populated = true
		f.initialPopulationCount = keys.Len() + queuedDeletions
	}

	return nil
}
~~~



## 5.4.1 Indexer接口和cache的实现

* Indexer介绍

  Indexer接口主要是在Store接口的基础上拓展了对象的检索功能，其接口定义如下

  ~~~go
  type Indexer interface {
  	Store
      
  	// Index returns the stored objects whose set of indexed values
  	// intersects the set of indexed values of the given object, for
  	// the named index
      // 根据索引名和给定的对象返回符合条件的所有对象
  	Index(indexName string, obj interface{}) ([]interface{}, error)
      
  	// IndexKeys returns the storage keys of the stored objects whose
  	// set of indexed values for the named index includes the given
  	// indexed value   
      // 根据索引名和索引值返回符合条件的所有对象的Key
  	IndexKeys(indexName, indexedValue string) ([]string, error)
      
  	// ListIndexFuncValues returns all the indexed values of the given index
      // 列出索引函数计算出来的所有索引值
  	ListIndexFuncValues(indexName string) []string
      
  	// ByIndex returns the stored objects whose set of indexed values
  	// for the named index includes the given indexed value
      // 根据索引名和索引值返回符合条件的所有对象
  	ByIndex(indexName, indexedValue string) ([]interface{}, error)
      
  	// GetIndexers return the indexers
      // 获取所有的Indexers，对应map[string]IndexFunc类型
  	GetIndexers() Indexers
  
  	// AddIndexers adds more indexers to this store.  If you call this after you already have data
  	// in the store, the results are undefined.
      // 该方法要在数据加入存储前调用，添加更多的索引方法，默认只通过namespace检索
  	AddIndexers(newIndexers Indexers) error
  }
  ~~~

* cache介绍

  ~~~go
  type cache struct {
  	// cacheStorage bears the burden of thread safety for the cache
  	cacheStorage ThreadSafeStore
  	// keyFunc is used to make the key for objects stored in and retrieved from items, and
  	// should be deterministic.
  	keyFunc KeyFunc
  }
  ~~~

  1. KeyFunc的定义`type KeyFunc func(obj interface{}) (string, error)`,其实就是为一个对象返回一个字符串类型的Key

     其默认实现是`MetaNamespaceKeyFunc`,其一般情况下返回值是\<namespace\>\<name\>，如果namespace为空，则直接返回name

  2. cache作为Indexer的实现，不仅需要实现Store中的函数接口，还需要实现与Indexer相关的函数接口，但是其结构体定义中除了KeyFunc，只剩下了ThreadSafeStore

     所以ThreadSafeStore的接口定义，就包含了Indexer相关函数接口和Store中的接口定义，其定义如下：

     ~~~go
     type ThreadSafeStore interface {
     	Add(key string, obj interface{})
     	Update(key string, obj interface{})
     	Delete(key string)
     	Get(key string) (item interface{}, exists bool)
     	List() []interface{}
     	ListKeys() []string
     	Replace(map[string]interface{}, string)
     	Index(indexName string, obj interface{}) ([]interface{}, error)
     	IndexKeys(indexName, indexedValue string) ([]string, error)
     	ListIndexFuncValues(name string) []string
     	ByIndex(indexName, indexedValue string) ([]interface{}, error)
     	GetIndexers() Indexers
     
     	// AddIndexers adds more indexers to this store.  If you call this after you already have data
     	// in the store, the results are undefined.
     	AddIndexers(newIndexers Indexers) error
     	// Resync is a no-op and is deprecated
     	Resync() error
     }
     ~~~

     并且，其实cache关于Indexer和Store相关函数的实现，可以发现都是直接调用cacheStorage中的相应方法的

## 5.4.2 ThreadSafeStore的实现

* ThreadSafeStore的实现：threaSafeMap，其结构体定义如下

  ~~~go
  type threadSafeMap struct {
  	lock  sync.RWMutex
  	items map[string]interface{}
  
  	// index implements the indexing functionality
  	index *storeIndex
  }
  ~~~

  1. 这里的items存储的就是obj的Key和obj本身了，不再是obj的变更记录了
  2. 在说明threadSafeMap针对Indexer和Store的函数接口实现之前，先来说明一下其中的存储结构

### threadSafeMap的存储结构

* 可以看到，threadSafeMap除了items，还有个storeIndex类型的index，其定义如下：

  ~~~go
  type storeIndex struct {
  	// indexers maps a name to an IndexFunc
  	indexers Indexers
  	// indices maps a name to an Index
  	indices Indices
  }
  
  // Indexers maps a name to an IndexFunc
  type Indexers map[string]IndexFunc
  
  // Indices maps a name to an Index
  type Indices map[string]Index
  
  // Index maps the indexed value to a set of keys in the store that match on that value
  type Index map[string]sets.String
  
  // IndexFunc knows how to compute the set of indexed values for an object.
  type IndexFunc func(obj interface{}) ([]string, error)
  ~~~

  其定义都列在代码块中了，

  1. Indexers其实就是用map来存储不同的**根据对象Obj来构造Key的函数方法**，**其实就是存储的是函数**

     比如其中默认存储的一个条目是{“namespace”, MetaNamespaceIndexFunc}，存储的是根据namespace来构建Key的函数`MetanamespaceIndexFunc`

     ==注意，这里的IndexFunc和之前我们提到的KeyFunc定义不一样，对比如下；IndexFunc返回的是一个切片，以MetaNamespaceIndexFunc为例，其只返回Pod的Namespace字符串，即使Pod理论上只属于一个Namespace，也要包装成[]string返回==

     * IndexFunc与KeyFunc的定义对比：

       ~~~go
       // IndexFunc knows how to compute the set of indexed values for an object.
       type IndexFunc func(obj interface{}) ([]string, error)
       
       // KeyFunc knows how to make a key from an object. Implementations should be deterministic.
       type KeyFunc func(obj interface{}) (string, error)
       ~~~
  
     * MetaNamespaceKeyFunc与MetaNamespaceIndexFunc对比
  
       ~~~go
       // MetaNamespaceIndexFunc is a default index function that indexes based on an object's namespace
       func MetaNamespaceIndexFunc(obj interface{}) ([]string, error) {
       	meta, err := meta.Accessor(obj)
       	if err != nil {
       		return []string{""}, fmt.Errorf("object has no meta: %v", err)
       	}
       	return []string{meta.GetNamespace()}, nil
       }
       
       // TODO maybe some day?: change Store to be keyed differently
       func MetaNamespaceKeyFunc(obj interface{}) (string, error) {
       	if key, ok := obj.(ExplicitKey); ok {
       		return string(key), nil
       	}
       	objName, err := ObjectToName(obj)
       	if err != nil {
       		return "", err
       	}
       	return objName.String(), nil
       }
       ~~~
  
  2. Indices存储的，其实就是 不同Key构建函数根据Obj对象产生的不同形式的Key的集合，**是一个String类型的集合，存储的是使用某个特定KeyFunc构建出来的所有对象的Key**
  
     比如默认存储的一个条目就是{“namespace”, Index}
  
  3. 最后，我们只需要拿着Index中某个条目中的Key，去threadSafeMap中的item中，就可以取到对应的Obj对象了
  
  其图示存储结构如下：
  ![image.png](https://typora2icture.oss-cn-beijing.aliyuncs.com/img2/202509111649474.png)

### Add、Update等方法的实现

* 通过查看Add、Update和Delete的源代码，可以发现，其中主要逻辑都在函数`updateIndices`中，接下来解析该函数的代码

  ~~~go
  // 所以这里传入的Key的含义就是对象obj的关键字，以namespace为例，key是"default/pod_1"
  func (i *storeIndex) updateIndices(oldObj interface{}, newObj interface{}, key string) {
  	var oldIndexValues, indexValues []string
  	var err error
      // 遍历所有注册的索引器Indexer
  	for name, indexFunc := range i.indexers {
          // 如果oldOjb不为空，则计算oldOjb的索引值，否则清空旧索引值列表
  		if oldObj != nil {
  			oldIndexValues, err = indexFunc(oldObj)
  		} else {
  			oldIndexValues = oldIndexValues[:0]
  		}
  		if err != nil {
  			panic(fmt.Errorf("unable to calculate an index entry for key %q on index %q: %v", key, name, err))
  		}
  
          // 如果newOjb不为空，则计算newOjb的索引值，否则清空旧索引值列表
  		if newObj != nil {
  			indexValues, err = indexFunc(newObj)
  		} else {
  			indexValues = indexValues[:0]
  		}
  		if err != nil {
  			panic(fmt.Errorf("unable to calculate an index entry for key %q on index %q: %v", key, name, err))
  		}
  
          // 获取当前索引器对应的Index Map；如果该Index为空，则初始化一个空的
  		index := i.indices[name]
  		if index == nil {
  			index = Index{}
  			i.indices[name] = index
  		}
  
          // 如果索引值未变，且只有一个，则跳过更新；例如MetaNamespaceKeyFunc，
  		if len(indexValues) == 1 && len(oldIndexValues) == 1 && indexValues[0] == oldIndexValues[0] {
  			// We optimize for the most common case where indexFunc returns a single value which has not been changed
  			continue
  		}
  
          // 删除旧索引条目：遍历所有旧索引值，从对应索引中移除当前Key
  		for _, value := range oldIndexValues {
  			i.deleteKeyFromIndex(key, value, index)
  		}
          // 添加新索引条目：遍历新对象的所有索引值，吧Key添加到这些索引值对应的键列表中
  		for _, value := range indexValues {
  			i.addKeyToIndex(key, value, index)
  		}
  	}
  }
  ~~~

  再来看看Add、Update、Delete等方法是如何使用updateIndices的

  ~~~go
  // Add是直接调用Update，由于Add的obj之前不在item中，所以传入updateIndices的oldObject是nil
  func (c *threadSafeMap) Add(key string, obj interface{}) {
  	c.Update(key, obj)
  }
  
  // 先获取oldObject，然后再用传入的obj作为newObject传入updateIndices
  func (c *threadSafeMap) Update(key string, obj interface{}) {
  	c.lock.Lock()
  	defer c.lock.Unlock()
  	oldObject := c.items[key]
  	c.items[key] = obj
  	c.index.updateIndices(oldObject, obj, key)
  }
  
  // 先检验给定的对象在不在items中，如果在，从items中获取oldObjet，传入updateIndices的newObject为nil，这样就只有删除旧indices了
  func (c *threadSafeMap) Delete(key string) {
  	c.lock.Lock()
  	defer c.lock.Unlock()
  	if obj, exists := c.items[key]; exists {
  		c.index.updateIndices(obj, nil, key)
  		delete(c.items, key)
  	}
  }
  
  // 重置整个items，并逐个item重新加入updateIndices
  func (c *threadSafeMap) Replace(items map[string]interface{}, resourceVersion string) {
  	c.lock.Lock()
  	defer c.lock.Unlock()
  	c.items = items
  
  	// rebuild any index
  	c.index.reset()
  	for key, item := range c.items {
  		c.index.updateIndices(nil, item, key)
  	}
  }
  ~~~



## 5.5.1 ListWatch对象的初始化

> ListWatch是Reflector的主要能力提供者

* 首先来看看ListWatch的定义

  ~~~go
  // ListFunc knows how to list resources
  type ListFunc func(options metav1.ListOptions) (runtime.Object, error)
  
  // WatchFunc knows how to watch resources
  type WatchFunc func(options metav1.ListOptions) (watch.Interface, error)
  
  // ListWatch knows how to list and watch a set of apiserver resources.  It satisfies the ListerWatcher interface.
  // It is a convenience function for users of NewReflector, etc.
  // ListFunc and WatchFunc must not be nil
  type ListWatch struct {
  	ListFunc  ListFunc
  	WatchFunc WatchFunc
  	// DisableChunking requests no chunking for this list watcher.
  	DisableChunking bool
  }
  ~~~

  包含了两个函数作为成员变量，`DisableChunking`的作用后面会介绍

* 来看看这个对象的初始化过程

  ~~~go
  // Getter interface knows how to access Get method from RESTClient.
  type Getter interface {
  	Get() *restclient.Request
  }
  
  // NewListWatchFromClient creates a new ListWatch from the specified client, resource, namespace and field selector.
  func NewListWatchFromClient(c Getter, resource string, namespace string, fieldSelector fields.Selector) *ListWatch {
  	optionsModifier := func(options *metav1.ListOptions) {
  		options.FieldSelector = fieldSelector.String()
  	}
  	return NewFilteredListWatchFromClient(c, resource, namespace, optionsModifier)
  }
  ~~~

  1. `Getter`接口抽象了如何而获取一个REST请求对象，用于发起GET请求

     `restclient.Request`是对HTTP请求的封装，可以链式调用添加参数、Header、Body等

     这里使用的restClient的实现在`client-go/rest/client.go`中可以看到

     这个RESTClient与我们平时使用的ClientSet的关系，其实就是ClientSet去Get一个指定名字的DaemonSet时，其调用过程如下

     ~~~go
     r.AppsV1().DaemonSets("default").Get(ctx, "test-ds", getOpt)
     ~~~

     这里使用的Get方法其实就是RESTClient提供的

     > 当使用ClientSet去获取DeamonSet的全量列表时，与Reflector通过ListFunc使用RESTClient提供的Get函数去获取DeamonSet的全量列表的过程基本一致
     >
     > 不过目的不同，ClientSet获取DeamonSet的全量列表是直接拿到结果，但是Reflector获取到DeamonSet的全量列表后，结果就进入了DeltaFIFO中，然后又通过Informer到了threadSafeStore中

  2. `NewListWatchFromClient`使用给定的客户端、资源类型、命名空间和字段选择器创建一个ListWatch对象

     * resource通常是要操作是Kubernetes资源名称，如pods、services
     * fieldsSelector是字段选择器，用于过滤资源，例如metadata.name=my-pod

  3. `optionsModifier`用于创建一个闭包函数，其接收一个ListOptions指针，将传入的fieldSelector转换为字符串，并赋值给options.FieldSelector

     该函数会在执行List和Watch之前被调用，用于设置请求参数

* 来看看主要的New逻辑函数`NewFilteredListWatchFromClient`

  ~~~go
  func NewFilteredListWatchFromClient(c Getter, resource string, namespace string, optionsModifier func(options *metav1.ListOptions)) *ListWatch {
      listFunc := func(options metav1.ListOptions) (runtime.Object, error) {
  		optionsModifier(&options)
  		return c.Get().
  			Namespace(namespace).
  			Resource(resource).
  			VersionedParams(&options, metav1.ParameterCodec).
  			Do(context.TODO()).
  			Get()
  	}
  	watchFunc := func(options metav1.ListOptions) (watch.Interface, error) {
  		options.Watch = true
  		optionsModifier(&options)
  		return c.Get().
  			Namespace(namespace).
  			Resource(resource).
  			VersionedParams(&options, metav1.ParameterCodec).
  			Watch(context.TODO())
  	}
  	return &ListWatch{ListFunc: listFunc, WatchFunc: watchFunc}
  }
  ~~~

  1. `listFunc`用于列出namespace下的某个resource

     optionsModifier：先调用传入的修饰器函数，设置过滤条件

     VersionedParams将metav1.ListOptions的options编码为URL查询参数，其实就是将过滤条件编码一下为URL形式；并使用正确的版本化编码器以确保兼容性

  2. `watchFunc`用于监听namespace下的某个resource

     options.Watch=true表示启动watch模式，使得APIServer启动流式响应

## 5.5.3 ListWatch与HTTP chunked

> Kubernetes中主要通过ListWatch机制实现组件间的异步消息通信
>
> Kubernetes中的监听watch长链接就是通过HTTP的chunked机制实现的，在响应头中加一个`Transfer-Encoding:chunked`就可以实现分段式响应

* 我们使用Go语言来模拟一下这个过程，其Demo程序的服务器端代码如下

  ~~~go
  func Server() {
  	http.HandleFunc("/name", func(writer http.ResponseWriter, request *http.Request) {
  		flusher := writer.(http.Flusher)
  		for i := 0; i < 2; i++ {
  			fmt.Fprintf(writer, "Wuyong\n")
  			flusher.Flush()
  			<-time.Tick(1 * time.Second)
  		}
  	})
  	log.Fatal(http.ListenAndServe(":8080", nil))
  }
  ~~~
  
  当访问`localhost:8080/name`时服务器端响应两行“wuyong”，通过wireShark抓包可以看到响应体
  ![image.png](https://typora2icture.oss-cn-beijing.aliyuncs.com/img2/202509191024433.png)

  chunked类型的 response由一个个的chunk组成，每个chunk的格式都是chunk Size + Chunk Data + Chunk Boundary，也就是块大小+块数据+块边界标识
  
  chunk的结尾是一个大小为0的块，就是`0\r\n`
  
  其接收代码如下
  
  ~~~go
  func Client() {
  	resp, err := http.Get("http://localhost:8080/name")
  	if err != nil {
  		log.Fatal(err)
  	}
  	defer resp.Body.Close()
  
  	fmt.Println(resp.TransferEncoding)
  
  	reader := bufio.NewReader(resp.Body)
  	for {
  		line, err := reader.ReadString('\n')
  		if len(line) > 0 {
  			fmt.Print(line)
  		}
  		if err == io.EOF {
  			break
  		}
  		if err != nil {
  			log.Fatal(err)
  		}
  	}
  }
  ~~~
  
* Kubernetes集群都是以HTTPS方式暴露API，并且开启了双向TLS，我们需要先通过Kubectl代理kube-apiserver提供HTTP的API，方便调用和抓包

  ~~~sh
  kubectl proxy --address=0.0.0.0 --port=8001 --accept-hosts='^*$'
  ~~~

  这里由于我们的抓包工具在另一台电脑上，如果使用`kubectl proxy`方式直接代理，那么只能在集群本机上使用curl命令进行访问，所以此处开启代理的命令变成上面的命令

  我们这里watch的命令如下

  ~~~sh
  curl http://192.168.3.226:8001/api/v1/watch/namespaces/k8s-learn/pods/neilats-refactor-scheduler-9b5955d75-zf55h
  ~~~

  命令行返回结果如下：

  ~~~sh
  {"type":"ADDED","object":{"kind":"Pod","apiVersion":"v1","metadata":{"name":"neilats-refactor-scheduler-9b5955d75-zf55h","generateName":"neilats-refactor-scheduler-9b5955d75-","namespace":"k8s-learn","uid":"25971d96-8051-4a3b-84ff-2be19dac3661","resourceVersion":"19381412","creationTimestamp":"2025-09-01T11:05:41Z","labels":{"component":"scheduler","pod-template-hash":"9b5955d75"},"ownerReferences":[{"apiVersion":"apps/v1","kind":"ReplicaSet","name":"neilats-refactor-scheduler-9b5955d75","uid":"e4da1506-c19f-4853-97af-69d50f2da383","controller":true,"blockOwnerDeletion":true}],"managedFields":[{"manager":"kube-controller-manager","operation":"Update","apiVersion":"v1","time":"2025-09-01T11:05:41Z","fieldsType":"FieldsV1","fieldsV1":{"f:metadata":{"f:generateName":{},"f:labels":{".":{},"f:component":{},"f:pod-template-hash":{}},"f:ownerReferences":{".":{},"k:{\"uid\":\"e4da1506-c19f-4853-97af-69d50f2da383\"}":{}}},"f:spec":{"f:containers":{"k:{\"name\":\"scheduler-plugins-scheduler\"}":{".":{},"f:command":{},"f:image":{},"f:imagePullPolicy":{},"f:livenessProbe":{".":{},"f:failureThreshold":{},"f:httpGet":{".":{},"f:path":{},"f:port":{},"f:scheme":{}},"f:initialDelaySeconds":{},"f:periodSeconds":{},"f:successThreshold":{},"f:timeoutSeconds":{}},"f:name":{},"f:readinessProbe":{".":{},"f:failureThreshold":{},"f:httpGet":{".":{},"f:path":{},"f:port":{},"f:scheme":{}},"f:periodSeconds":{},"f:successThreshold":{},"f:timeoutSeconds":{}},"f:resources":{".":{},"f:requests":{".":{},"f:cpu":{}}},"f:securityContext":{".":{},"f:privileged":{}},"f:terminationMessagePath":{},"f:terminationMessagePolicy":{},"f:volumeMounts":{".":{},"k:{\"mountPath\":\"/etc/kubernetes\"}":{".":{},"f:mountPath":{},"f:name":{},"f:readOnly":{}}}}},"f:dnsPolicy":{},"f:enableServiceLinks":{},"f:restartPolicy":{},"f:schedulerName":{},"f:securityContext":{},"f:serviceAccount":{},"f:serviceAccountName":{},"f:terminationGracePeriodSeconds":{},"f:volumes":{".":{},"k:{\"name\":\"scheduler-config\"}":{".":{},"f:configMap":{".":{},"f:defaultMode":{},"f:name":{}},"f:name":{}}}}}},{"manager":"kubelet","operation":"Update","apiVersion":"v1","time":"2025-09-01T11:06:01Z","fieldsType":"FieldsV1","fieldsV1":{"f:status":{"f:conditions":{"k:{\"type\":\"ContainersReady\"}":{".":{},"f:lastProbeTime":{},"f:lastTransitionTime":{},"f:status":{},"f:type":{}},"k:{\"type\":\"Initialized\"}":{".":{},"f:lastProbeTime":{},"f:lastTransitionTime":{},"f:status":{},"f:type":{}},"k:{\"type\":\"Ready\"}":{".":{},"f:lastProbeTime":{},"f:lastTransitionTime":{},"f:status":{},"f:type":{}}},"f:containerStatuses":{},"f:hostIP":{},"f:phase":{},"f:podIP":{},"f:podIPs":{".":{},"k:{\"ip\":\"10.244.2.208\"}":{".":{},"f:ip":{}}},"f:startTime":{}}},"subresource":"status"}]},"spec":{"volumes":[{"name":"scheduler-config","configMap":{"name":"scheduler-config","defaultMode":420}},{"name":"kube-api-access-s5dbv","projected":{"sources":[{"serviceAccountToken":{"expirationSeconds":3607,"path":"token"}},{"configMap":{"name":"kube-root-ca.crt","items":[{"key":"ca.crt","path":"ca.crt"}]}},{"downwardAPI":{"items":[{"path":"namespace","fieldRef":{"apiVersion":"v1","fieldPath":"metadata.namespace"}}]}}],"defaultMode":420}}],"containers":[{"name":"scheduler-plugins-scheduler","image":"wuyong7240/scheduler-plugins/neilats-refactor-scheduler:v2.8","command":["/bin/kube-scheduler","--config=/etc/kubernetes/scheduler-config.yaml"],"resources":{"requests":{"cpu":"100m"}},"volumeMounts":[{"name":"scheduler-config","readOnly":true,"mountPath":"/etc/kubernetes"},{"name":"kube-api-access-s5dbv","readOnly":true,"mountPath":"/var/run/secrets/kubernetes.io/serviceaccount"}],"livenessProbe":{"httpGet":{"path":"/healthz","port":10259,"scheme":"HTTPS"},"initialDelaySeconds":15,"timeoutSeconds":1,"periodSeconds":10,"successThreshold":1,"failureThreshold":3},"readinessProbe":{"httpGet":{"path":"/healthz","port":10259,"scheme":"HTTPS"},"timeoutSeconds":1,"periodSeconds":10,"successThreshold":1,"failureThreshold":3},"terminationMessagePath":"/dev/termination-log","terminationMessagePolicy":"File","imagePullPolicy":"IfNotPresent","securityContext":{"privileged":false}}],"restartPolicy":"Always","terminationGracePeriodSeconds":30,"dnsPolicy":"ClusterFirst","serviceAccountName":"neilats-refactor-scheduler","serviceAccount":"neilats-refactor-scheduler","nodeName":"node2","securityContext":{},"schedulerName":"default-scheduler","tolerations":[{"key":"node.kubernetes.io/not-ready","operator":"Exists","effect":"NoExecute","tolerationSeconds":300},{"key":"node.kubernetes.io/unreachable","operator":"Exists","effect":"NoExecute","tolerationSeconds":300}],"priority":0,"enableServiceLinks":true,"preemptionPolicy":"PreemptLowerPriority"},"status":{"phase":"Running","conditions":[{"type":"Initialized","status":"True","lastProbeTime":null,"lastTransitionTime":"2025-09-01T11:05:40Z"},{"type":"Ready","status":"True","lastProbeTime":null,"lastTransitionTime":"2025-09-01T11:06:01Z"},{"type":"ContainersReady","status":"True","lastProbeTime":null,"lastTransitionTime":"2025-09-01T11:06:01Z"},{"type":"PodScheduled","status":"True","lastProbeTime":null,"lastTransitionTime":"2025-09-01T11:05:41Z"}],"hostIP":"192.168.3.224","podIP":"10.244.2.208","podIPs":[{"ip":"10.244.2.208"}],"startTime":"2025-09-01T11:05:40Z","containerStatuses":[{"name":"scheduler-plugins-scheduler","state":{"running":{"startedAt":"2025-09-01T11:05:41Z"}},"lastState":{},"ready":true,"restartCount":0,"image":"docker.io/wuyong7240/scheduler-plugins/neilats-refactor-scheduler:v2.8","imageID":"sha256:d80464d36ddb9d27d3430a0718b76b72b0bab2faf32523dca8035bde85d2e4cb","containerID":"containerd://051c3f338fe5e3a0265b6a273c2fc16eaedad0ffb146c635ee284448cc7fff1d","started":true}],"qosClass":"Burstable"}}}
  ~~~

  通过抓包，可以看到`Transfer-Encoding`和`{"type":"ADDED"}`
  ![image.png](https://typora2icture.oss-cn-beijing.aliyuncs.com/img2/202509191105298.png)
  ![image.png](https://typora2icture.oss-cn-beijing.aliyuncs.com/img2/202509191105944.png)

  实际上说明了Kubernetes的watch长链接监听就是通过HTTP的chunked机制实现的



## 5.6.2 核心方法：Reflector.ListAndWatch()

> 首先查看Reflector的主要运行函数Run，可以发现其中主要逻辑都在函数`ListAndWatch`函数中，因此先来解析ListAndWatch

* ListAndWatch

  ~~~sh
  func (r *Reflector) ListAndWatch(stopCh <-chan struct{}) error {
  	klog.V(3).Infof("Listing and watching %v from %s", r.typeDescription, r.name)
  	var err error
  	var w watch.Interface
  	fallbackToList := !r.UseWatchList
  
  	if r.UseWatchList {
  		w, err = r.watchList(stopCh)
  		if w == nil && err == nil {
  			// stopCh was closed
  			return nil
  		}
  		if err != nil {
  			if !apierrors.IsInvalid(err) {
  				return err
  			}
  			klog.Warning("the watch-list feature is not supported by the server, falling back to the previous LIST/WATCH semantic")
  			fallbackToList = true
  			// Ensure that we won't accidentally pass some garbage down the watch.
  			w = nil
  		}
  	}
  
  	if fallbackToList {
  		err = r.list(stopCh)
  		if err != nil {
  			return err
  		}
  	}
  
  	resyncerrc := make(chan error, 1)
  	cancelCh := make(chan struct{})
  	defer close(cancelCh)
  	go r.startResync(stopCh, cancelCh, resyncerrc)
  	return r.watch(w, stopCh, resyncerrc)
  }
  ~~~




# Chapter7：Operator开发进阶

> Operator用于再一个应用部署所需的各种资源之上抽象要给Application类型，这个类型包含一些必要的字段，然后用户只需要创建一个Application，我们通过自定义控制器去完成Application相关的Deployment、Service、ConfingMap等资源的创建和管理工作
>
> ==可以说 Operator 的核心是“通过 CRD 和 Controller 来组合和管理原生资源，实现应用的自动化运维”。==

## 7.2 Application-operator-plus项目准备

### 7.2.1 创建新项目

~~~sh
cd ~/MyOperatorProjects/
mkdir application-operator/
cd application-operator/
ls
kubebuilder init --domain=danielhu.cn \
--repo=github.com/daniel-hutao/application-operator \
--owner Daniel.Hu

Writing kustomize manifests for you to edit...
Writing scaffold for you to edit...
Get controller runtime:
~~~

注意，这里项目名默认使用的是当前的目录名，也就是`application-operator`，当然也可以通过`–project-name`参数来自定义

如果中途想修改项目名，找到如下的配置项，手动调整即可

1. PROJECT文件中的`projectName`
2. `config/default/kustomization.yaml`中的`namespace`
3. `config/default/kustomization.yaml`中的`namePrefix`

### 7.2.2 项目基础结构分析

1. go.mod：该文件包含项目的基础依赖，使用`go mod tidy`进行更新

2. Makefile：该文件存放的是开发过程中构建、部署、测试等一系列相关命令

3. PROJECT：存放的是一些Kubebuilder需要用到的元数据

4. `config目录`：存放与Operator部署相关的配置文件，主要是运行Controller相关的Kustomize配置以及各种资源配置

5. main.go：

   ~~~go
   /*
   Copyright 2025 wuyong.
   
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
   	"crypto/tls"
   	"flag"
   	"os"
   
   	// Import all Kubernetes client auth plugins (e.g. Azure, GCP, OIDC, etc.)
   	// to ensure that exec-entrypoint and run can make use of them.
   	_ "k8s.io/client-go/plugin/pkg/client/auth"
   
   	"k8s.io/apimachinery/pkg/runtime"
   	utilruntime "k8s.io/apimachinery/pkg/util/runtime"
   	clientgoscheme "k8s.io/client-go/kubernetes/scheme"
   	ctrl "sigs.k8s.io/controller-runtime"
   	"sigs.k8s.io/controller-runtime/pkg/healthz"
   	"sigs.k8s.io/controller-runtime/pkg/log/zap"
   	"sigs.k8s.io/controller-runtime/pkg/metrics/filters"
   	metricsserver "sigs.k8s.io/controller-runtime/pkg/metrics/server"
   	"sigs.k8s.io/controller-runtime/pkg/webhook"
   
   	appsv1 "github.com/wuyong7240/application-operator-plus/api/v1"
   	"github.com/wuyong7240/application-operator-plus/internal/controller"
   	// +kubebuilder:scaffold:imports
   )
   
   var (
   	scheme   = runtime.NewScheme()
   	setupLog = ctrl.Log.WithName("setup")
   )
   
   func init() {
   	utilruntime.Must(clientgoscheme.AddToScheme(scheme))
   
   	utilruntime.Must(appsv1.AddToScheme(scheme))
   	// +kubebuilder:scaffold:scheme
   }
   
   // nolint:gocyclo
   func main() {
   	var metricsAddr string
   	var metricsCertPath, metricsCertName, metricsCertKey string
   	var webhookCertPath, webhookCertName, webhookCertKey string
   	var enableLeaderElection bool
   	var probeAddr string
   	var secureMetrics bool
   	var enableHTTP2 bool
   	var tlsOpts []func(*tls.Config)
   	flag.StringVar(&metricsAddr, "metrics-bind-address", "0", "The address the metrics endpoint binds to. "+
   		"Use :8443 for HTTPS or :8080 for HTTP, or leave as 0 to disable the metrics service.")
   	flag.StringVar(&probeAddr, "health-probe-bind-address", ":8081", "The address the probe endpoint binds to.")
   	flag.BoolVar(&enableLeaderElection, "leader-elect", false,
   		"Enable leader election for controller manager. "+
   			"Enabling this will ensure there is only one active controller manager.")
   	flag.BoolVar(&secureMetrics, "metrics-secure", true,
   		"If set, the metrics endpoint is served securely via HTTPS. Use --metrics-secure=false to use HTTP instead.")
   	flag.StringVar(&webhookCertPath, "webhook-cert-path", "", "The directory that contains the webhook certificate.")
   	flag.StringVar(&webhookCertName, "webhook-cert-name", "tls.crt", "The name of the webhook certificate file.")
   	flag.StringVar(&webhookCertKey, "webhook-cert-key", "tls.key", "The name of the webhook key file.")
   	flag.StringVar(&metricsCertPath, "metrics-cert-path", "",
   		"The directory that contains the metrics server certificate.")
   	flag.StringVar(&metricsCertName, "metrics-cert-name", "tls.crt", "The name of the metrics server certificate file.")
   	flag.StringVar(&metricsCertKey, "metrics-cert-key", "tls.key", "The name of the metrics server key file.")
   	flag.BoolVar(&enableHTTP2, "enable-http2", false,
   		"If set, HTTP/2 will be enabled for the metrics and webhook servers")
   	opts := zap.Options{
   		Development: true,
   	}
   	opts.BindFlags(flag.CommandLine)
   	flag.Parse()
   
   	ctrl.SetLogger(zap.New(zap.UseFlagOptions(&opts)))
   
   	// if the enable-http2 flag is false (the default), http/2 should be disabled
   	// due to its vulnerabilities. More specifically, disabling http/2 will
   	// prevent from being vulnerable to the HTTP/2 Stream Cancellation and
   	// Rapid Reset CVEs. For more information see:
   	// - https://github.com/advisories/GHSA-qppj-fm5r-hxr3
   	// - https://github.com/advisories/GHSA-4374-p667-p6c8
   	disableHTTP2 := func(c *tls.Config) {
   		setupLog.Info("disabling http/2")
   		c.NextProtos = []string{"http/1.1"}
   	}
   
   	if !enableHTTP2 {
   		tlsOpts = append(tlsOpts, disableHTTP2)
   	}
   
   	// Initial webhook TLS options
   	webhookTLSOpts := tlsOpts
   	webhookServerOptions := webhook.Options{
   		TLSOpts: webhookTLSOpts,
   	}
   
   	if len(webhookCertPath) > 0 {
   		setupLog.Info("Initializing webhook certificate watcher using provided certificates",
   			"webhook-cert-path", webhookCertPath, "webhook-cert-name", webhookCertName, "webhook-cert-key", webhookCertKey)
   
   		webhookServerOptions.CertDir = webhookCertPath
   		webhookServerOptions.CertName = webhookCertName
   		webhookServerOptions.KeyName = webhookCertKey
   	}
   
   	webhookServer := webhook.NewServer(webhookServerOptions)
   
   	// Metrics endpoint is enabled in 'config/default/kustomization.yaml'. The Metrics options configure the server.
   	// More info:
   	// - https://pkg.go.dev/sigs.k8s.io/controller-runtime@v0.21.0/pkg/metrics/server
   	// - https://book.kubebuilder.io/reference/metrics.html
   	metricsServerOptions := metricsserver.Options{
   		BindAddress:   metricsAddr,
   		SecureServing: secureMetrics,
   		TLSOpts:       tlsOpts,
   	}
   
   	if secureMetrics {
   		// FilterProvider is used to protect the metrics endpoint with authn/authz.
   		// These configurations ensure that only authorized users and service accounts
   		// can access the metrics endpoint. The RBAC are configured in 'config/rbac/kustomization.yaml'. More info:
   		// https://pkg.go.dev/sigs.k8s.io/controller-runtime@v0.21.0/pkg/metrics/filters#WithAuthenticationAndAuthorization
   		metricsServerOptions.FilterProvider = filters.WithAuthenticationAndAuthorization
   	}
   
   	// If the certificate is not specified, controller-runtime will automatically
   	// generate self-signed certificates for the metrics server. While convenient for development and testing,
   	// this setup is not recommended for production.
   	//
   	// TODO(user): If you enable certManager, uncomment the following lines:
   	// - [METRICS-WITH-CERTS] at config/default/kustomization.yaml to generate and use certificates
   	// managed by cert-manager for the metrics server.
   	// - [PROMETHEUS-WITH-CERTS] at config/prometheus/kustomization.yaml for TLS certification.
   	if len(metricsCertPath) > 0 {
   		setupLog.Info("Initializing metrics certificate watcher using provided certificates",
   			"metrics-cert-path", metricsCertPath, "metrics-cert-name", metricsCertName, "metrics-cert-key", metricsCertKey)
   
   		metricsServerOptions.CertDir = metricsCertPath
   		metricsServerOptions.CertName = metricsCertName
   		metricsServerOptions.KeyName = metricsCertKey
   	}
   
   	mgr, err := ctrl.NewManager(ctrl.GetConfigOrDie(), ctrl.Options{
   		Scheme:                 scheme,
   		Metrics:                metricsServerOptions,
   		WebhookServer:          webhookServer,
   		HealthProbeBindAddress: probeAddr,
   		LeaderElection:         enableLeaderElection,
   		LeaderElectionID:       "1b96da23.wuyong.cn",
   		// LeaderElectionReleaseOnCancel defines if the leader should step down voluntarily
   		// when the Manager ends. This requires the binary to immediately end when the
   		// Manager is stopped, otherwise, this setting is unsafe. Setting this significantly
   		// speeds up voluntary leader transitions as the new leader don't have to wait
   		// LeaseDuration time first.
   		//
   		// In the default scaffold provided, the program ends immediately after
   		// the manager stops, so would be fine to enable this option. However,
   		// if you are doing or is intended to do any operation such as perform cleanups
   		// after the manager stops then its usage might be unsafe.
   		// LeaderElectionReleaseOnCancel: true,
   	})
   	if err != nil {
   		setupLog.Error(err, "unable to start manager")
   		os.Exit(1)
   	}
   
   	if err := (&controller.ApplicationReconciler{
   		Client: mgr.GetClient(),
   		Scheme: mgr.GetScheme(),
   	}).SetupWithManager(mgr); err != nil {
   		setupLog.Error(err, "unable to create controller", "controller", "Application")
   		os.Exit(1)
   	}
   	// +kubebuilder:scaffold:builder
   
   	if err := mgr.AddHealthzCheck("healthz", healthz.Ping); err != nil {
   		setupLog.Error(err, "unable to set up health check")
   		os.Exit(1)
   	}
   	if err := mgr.AddReadyzCheck("readyz", healthz.Ping); err != nil {
   		setupLog.Error(err, "unable to set up ready check")
   		os.Exit(1)
   	}
   
   	setupLog.Info("starting manager")
   	if err := mgr.Start(ctrl.SetupSignalHandler()); err != nil {
   		setupLog.Error(err, "problem running manager")
   		os.Exit(1)
   	}
   }
   ~~~



## 7.3 定义Application资源

### 7.3.1 添加API

~~~sh
# kubebuilder create api \
--group apps --version v1 --kind Application
Create Resource [y/n]
y
Create Controller [y/n]
y
~~~

注意，这个步骤大概率需要访问外网，建议在远程主机上使用代理，具体见[[一些软件的部署#clash for Linux 安装]]

如果在这里创建API发生了网络问题，使得一些组件没有下载成功，例如controller-gen，可以使用如下命令解决：

1. 解决网络问题

   问题描述

   ~~~sh
   Downloading sigs.k8s.io/controller-tools/cmd/controller-gen@v0.18.0
   go: sigs.k8s.io/controller-tools/cmd/controller-gen@v0.18.0: sigs.k8s.io/controller-tools/cmd/controller-gen@v0.18.0: reading https://mirrors.aliyun.com/goproxy/sigs.k8s.io/controller-tools/cmd/controller-gen/@v/v0.18.0.info: 403 Forbidden
   make: *** [Makefile:204: /root/Golang/code/go/src/sigs.k8s.io/MyOperatorProjects/LSTMServerOperator/bin/controller-gen] Error 1
   ~~~

   **阿里云 Go 代理（goproxy）无法访问 `sigs.k8s.io` 下的某些模块路径**，尤其是带 `/cmd/...` 的子路径。这是 **阿里云 Go 代理的一个已知限制**：它只代理主模块（如 `sigs.k8s.io/controller-tools`），但不代理其子包（如 `sigs.k8s.io/controller-tools/cmd/controller-gen`）。

   使用支持子模块的代理

   ~~~sh
   go env -w GOPROXY=https://goproxy.cn,direct
   ~~~

   将该代理设置为全局设置

2. 强制重新创建API

   ~~~sh
   kubebuilder create api --group lstmapps --version v1 --kind LSTMPredictApp --force
   ~~~

   



### 7.3.2 自定义新API

* 首先编辑的是`application-operator-plus/api/v1/application_types.go`

  1. 第一部分：ApplicationSpec

     ~~~go
     // ApplicationSpec defines the desired state of Application
     type ApplicationSpec struct {
     	// INSERT ADDITIONAL SPEC FIELDS - desired state of cluster
     	// Important: Run "make" to regenerate code after modifying this file
     	// The following markers will use OpenAPI v3 schema to validate the value
     	// More info: https://book.kubebuilder.io/reference/markers/crd-validation.html
     
     	// foo is an example field of Application. Edit application_types.go to remove/update
     	// +optional
     	// +kubebuilder:pruning:PreserveUnknownFields
     	// +kubebuilder:validation:Schemaless
     	Deployment DeploymentTemplate `json:"deployment,omitempty"`
     	Service    ServiceTemplate    `json:"service,omitempty"`
     }
     
     type DeploymentTemplate struct {
     	appsv1.DeploymentSpec `json:",inline"`
     }
     
     type ServiceTemplate struct {
     	corev1.ServiceSpec `json:",inline"`
     }
     ~~~

     * 新引入：

       ~~~go
       appsv1 "k8s.io/api/apps/v1"
       corev1 "k8s.io/api/core/v1"
       ~~~

     * 在ApplicationSpec中的Deployment和Service成员变量上方添加注释

       ~~~go
       	// +kubebuilder:pruning:PreserveUnknownFields
       	// +kubebuilder:validation:Schemaless
       ~~~

       该注释在本文[[Kubernetes Operator开发笔记#2.5.4 CRD部署]]中有提到，用于解决Schema过分展开导致的超出code-generator大小限制的问题
       
     * 这里简单应用Kubernetes原生的DeploymentSpec对象和ServiceSpec对象来构造DeploymentTemplate和ServiceTemplate
  
  2. 第二部分：ApplicationStatus
  
     ~~~go
     // ApplicationStatus defines the observed state of Application.
     type ApplicationStatus struct {
     	// INSERT ADDITIONAL STATUS FIELD - define observed state of cluster
     	// Important: Run "make" to regenerate code after modifying this file
     
     	// For Kubernetes API conventions, see:
     	// https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md#typical-status-properties
     
     	// conditions represent the current state of the Application resource.
     	// Each condition has a unique type and reflects the status of a specific aspect of the resource.
     	//
     	// Standard condition types include:
     	// - "Available": the resource is fully functional
     	// - "Progressing": the resource is being created or updated
     	// - "Degraded": the resource failed to reach or maintain its desired state
     	//
     	Workflow appsv1.DeploymentStatus `json:"workflow"`
     	Network  corev1.ServiceStatus    `json:"network"`
     }
     ~~~
  
     * 注意，对比原本的ApplicationStatus，我们删除了如下内容
  
       ~~~go
       	// +listType=map
       	// +listMapKey=type
       	// +optional
       	Conditions []metav1.Condition `json:"conditions,omitempty"`
       ~~~
  
       注意这个注释也要删除掉，其实主要就是要删除这个注释，否则会发生如下报错：
  
       ~~~sh
       (base) root@master:~/Golang/code/go/src/sigs.k8s.io/MyOperatorProjects/application-operator-plus# make manifests
       /root/Golang/code/go/src/sigs.k8s.io/MyOperatorProjects/application-operator-plus/bin/controller-gen rbac:roleName=manager-role crd webhook paths="./..." output:crd:artifacts:config=config/crd/bases
       /root/Golang/code/go/src/sigs.k8s.io/MyOperatorProjects/application-operator-plus/api/v1/application_types.go:70:2: must apply listType to an array, found 
       /root/Golang/code/go/src/sigs.k8s.io/MyOperatorProjects/application-operator-plus/api/v1/application_types.go:70:2: must apply listMapKey to an array, found 
       Error: not all generators ran successfully
       run `controller-gen rbac:roleName=manager-role crd webhook paths=./... output:crd:artifacts:config=config/crd/bases -w` to see all available markers, or `controller-gen rbac:roleName=manager-role crd webhook paths=./... output:crd:artifacts:config=config/crd/bases -h` for usage
       make: *** [Makefile:46: manifests] Error 1
       ~~~
  
       `must apply listType to an array`、`must apply listMapKey to an array`
  
       其实就是后来我们自定义的Workflow和Network两个变量类型不再属于原本的slice切片类型了，不再适用这个注释了，这是注释是用于告诉Kubernetes如何处理数组类型字段的合并策略的
  
       > 这里两个变量的命名并没有说法，其实为了显而易见最好还是改成DeploymentStatus和ServiceStatus
  

## 7.4 实现Application Controller

### 7.4.1 实现主调谐流程

* 定义完Application资源之后，可以先使用`make generate`命令，重新生成`application-operator-plus/api/v1/zz_generated.deepcopy.go`文件，然后开始实现控制器核心调谐逻辑

  其调谐流程图如下：
  <img src="https://typora2icture.oss-cn-beijing.aliyuncs.com/img2/202510131139977.png" alt="image.png" style="zoom:80%;" />

  

  代码位于``application-operator-plus/internal/controller/application_controller.go`

  ~~~go
  var CounterReconcileApplication int64
  
  // 表示通用的重新排队时间间隔
  const GenericRequeueDuration = 1 * time.Minute
  
  // req表示需要调谐的资源，包含Namespace和Name, ctrl.Result控制是否重试，延迟重试多久，如果error非nil会触发重试
  func (r *ApplicationReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
  	_ = logf.FromContext(ctx)
  
  	// TODO(user): your logic here
      // 由于调谐过程是并发执行的，如果同时创建3个Application类型的资源示例，3个事件会被同时处理，日志会比较混乱，所以开头加了100ms的等待
  	<-time.NewTicker(100 * time.Millisecond).C
  	// 获取日志记录器，便于在不同reconcile调用中区分日志来源
  	log := log.FromContext(ctx)
  
  	// 用于统计Reconcile被调用了多少次
  	CounterReconcileApplication += 1
  	log.Info("Starting a reconcile", "number", CounterReconcileApplication)
  
  	app := &v1.Application{}
  	// 从API Server中获取Application实例
  	if err := r.Get(ctx, req.NamespacedName, app); err != nil {
  		if errors.IsNotFound(err) {
  			log.Info("Application not found.")
  			return ctrl.Result{}, nil
  		}
  		log.Error(err, "Failed to get the Application, will requeue after a short time.")
  		return ctrl.Result{RequeueAfter: GenericRequeueDuration}, err
  	}
  
  	// reconcile sub-resource, 调谐子资源
  	var result ctrl.Result
  	var err error
  
  	result, err = r.reconcileDeployment(ctx, app)
  	if err != nil {
  		log.Error(err, "Failed to reconcile Deployment.")
  		return result, err
  	}
  
  	result, err = r.reconcileService(ctx, app)
  	if err != nil {
  		log.Error(err, "Failed to reconcle Service.")
  		return result, err
  	}
  
  	log.Info("All resources have been reconciled.")
  	return ctrl.Result{}, nil
  }
  ~~~
  
  代码解释都写在注释中了，其中调谐Deployment和Service的主要逻辑都在函数`reconcileDeployment`和`reconcileService`中了
  
  * `application-operator-plus/internal/controller/deployment.go`
  
    Deployment的调谐逻辑如下：
    <img src="https://typora2icture.oss-cn-beijing.aliyuncs.com/img2/202510131144655.png" alt="image.png" style="zoom:80%;" />

    
  
    ~~~go
    package controller
    
    import (
    	"context"
    	"reflect"
    
    	v1 "github.com/wuyong7240/application-operator-plus/api/v1"
    	appsv1 "k8s.io/api/apps/v1"
    	"k8s.io/apimachinery/pkg/api/errors"
    	"k8s.io/apimachinery/pkg/types"
    	ctrl "sigs.k8s.io/controller-runtime"
    	"sigs.k8s.io/controller-runtime/pkg/log"
    )
    
    func (r *ApplicationReconciler) reconcileDeployment(ctx context.Context, app *v1.Application) (ctrl.Result, error) {
    	log := log.FromContext(ctx)
    
    	// 先根据Application中的Namespace和Name信息查询对应的Deployment是否存在
    	var dp = &appsv1.Deployment{}
    	// types.NamespaceedName用于唯一标识Kubernetes集群中的资源,dp是一个指针，如果Get方法成功执行，这个指针指向从API服务器中获取的Deployment对象
    	err := r.Get(ctx, types.NamespacedName{
    		Namespace: app.Namespace,
    		Name:      app.Name,
    	}, dp)
    
    	// 没有错误发生时，更新状态
    	if err == nil {
    		log.Info("The Deployment has already exist.")
    		// 使用reflect.DeepEqual比较Deployment的状态(dp.Status)与Application自定义资源的工作流状态(app.Status.Workflow)是否完全相同
    		// 如果相同，说明没有变化，不需要进一步处理
    		if reflect.DeepEqual(dp.Status, app.Status.Workflow) {
    			return ctrl.Result{}, nil
    		}
    
    		// 如果不同，需要更新Application的状态
    		app.Status.Workflow = dp.Status
    		// 调用r.Status().Update更新Application资源的状态
    		if err := r.Status().Update(ctx, app); err != nil {
    			log.Error(err, "Failed to update Application status")
    			// 返回一个带有重新排队时间的结果和错误，表示需要在一段时间后重试
    			return ctrl.Result{RequeueAfter: GenericRequeueDuration}, err
    		}
    		log.Info("The Application status has been updated.")
    		return ctrl.Result{}, nil
    	}
    
    	// 如果不是NotFound的错误，即发生了其他错误，结束本轮调谐，一段时间后重试
    	if !errors.IsNotFound(err) {
    		log.Error(err, "Failed to get Deployment, will requeue after a short time.")
    		return ctrl.Result{RequeueAfter: GenericRequeueDuration}, err
    	}
    
    	// 根据Application资源实例信息来构造Deployment实例
    	newDp := &appsv1.Deployment{}
    	newDp.SetName(app.Name)
    	newDp.SetNamespace(app.Namespace)
    	newDp.SetLabels(app.Labels)
    	newDp.Spec = app.Spec.Deployment.DeploymentSpec
    	// 这是Pod的模板，Pod模板的Labels是独立的，必须单独设置，如果不设置，回到是Deployment的selector无法匹配到Pod
    	newDp.Spec.Template.SetLabels(app.Labels)
    
    	// 用于建立App里擦同与Deployment之间的父子关系：Kubernetes通过owner Reference实现级联删除，当Application被删除时，Kubernetes
    	// 会自动删除它创建的Deployment; r.scheme用来识别资源类型的Scheme，确保类型正确
    	if err := ctrl.SetControllerReference(app, newDp, r.Scheme); err != nil {
    		log.Error(err, "Failed to SetControllerReference, will requeue after a short time.")
    		return ctrl.Result{RequeueAfter: GenericRequeueDuration}, err
    	}
    
    	// 在集群中创建Deployment：调用客户端的Create方法，将newDp提交到Kubernetes API Server
    	if err := r.Create(ctx, newDp); err != nil {
    		log.Error(err, "Failed to create Deployment, will requeue, after a short time.")
    		return ctrl.Result{RequeueAfter: GenericRequeueDuration}, err
    	}
    
    	log.Info("The Deployment has been created.")
    	return ctrl.Result{}, nil
    }
    ~~~
  
  * `application-operator-plus/internal/controller/service.go`
  
    Service的调谐流程与Deployment的调谐流程一致：
    <img src="https://typora2icture.oss-cn-beijing.aliyuncs.com/img2/202510131146430.png" alt="image.png" style="zoom:80%;" />

    
    
    ~~~go
    package controller
    
    import (
    	"context"
    	"reflect"
    
    	v1 "github.com/wuyong7240/application-operator-plus/api/v1"
    	corev1 "k8s.io/api/core/v1"
    	"k8s.io/apimachinery/pkg/api/errors"
    	"k8s.io/apimachinery/pkg/types"
    	ctrl "sigs.k8s.io/controller-runtime"
    	"sigs.k8s.io/controller-runtime/pkg/log"
    )
    
    func (r *ApplicationReconciler) reconcileService(ctx context.Context, app *v1.Application) (ctrl.Result, error) {
    
    	log := log.FromContext(ctx)
    
    	// 根据Application的Namespace和Name信息来查询对应的Servce资源
    	var svc = &corev1.Service{}
    	err := r.Get(ctx, types.NamespacedName{
    		Namespace: app.Namespace,
    		Name:      app.Name,
    	}, svc)
    
    	// 如果查到了对应的Service
    	if err == nil {
    		log.Info("The Service has already exist.")
    		// 利用reflect.DeepEqual判断现存的Service状态相比之前的Application中的状态是否有更新，如果有就更新Application中的Service状态
    		if reflect.DeepEqual(svc.Status, app.Status.Network) {
    			return ctrl.Result{}, err
    		}
    
    		// Service状态发生变化，将现存的Service状态赋值给Application中的Service状态，更新Application对Service的追踪
    		app.Status.Network = svc.Status
    		// 调用更新
    		if err := r.Status().Update(ctx, app); err != nil {
    			log.Error(err, "Failed to update Application Status")
    			return ctrl.Result{RequeueAfter: GenericRequeueDuration}, err
    		}
    		log.Info("The Application status has been updated.")
    		return ctrl.Result{}, nil
    	}
    
    	// 如果不是Not Found的错误，间隔一段时间后重试
    	if !errors.IsNotFound(err) {
    		log.Error(err, "Failed to get Service, will requeue after a short time.")
    		return ctrl.Result{RequeueAfter: GenericRequeueDuration}, err
    	}
    
    	// 如果是Not Found的错误，根据Application中的ServiceSpec，创建一个新的Service
    	newSvc := &corev1.Service{}
    	newSvc.SetName(app.Name)
    	newSvc.SetNamespace(app.Namespace)
    	newSvc.SetLabels(app.Labels)
    	newSvc.Spec = app.Spec.Service.ServiceSpec
    	newSvc.Spec.Selector = app.Labels
    
    	// 设置所有者引用，将Application设置为Service的所有者，
    	// 当Application被删除时，Service会被自动删除
    	if err := ctrl.SetControllerReference(app, newSvc, r.Scheme); err != nil {
    		log.Error(err, "Failed to SetControllerReference, will requeue after a short time.")
    		return ctrl.Result{RequeueAfter: GenericRequeueDuration}, err
    	}
    
    	// 调用客户端API创建Service资源
    	if err := r.Create(ctx, newSvc); err != nil {
    		log.Error(err, "Faield to create Service, will requeue after a short time.")
    		return ctrl.Result{RequeueAfter: GenericRequeueDuration}, err
    	}
    
    	log.Info("The Service has been created.")
    	return ctrl.Result{}, nil
    }
    ~~~
    
  
  **其实可能会有一个疑问，在Deployment和Service的调谐逻辑中，当从上下文中获取的Deployment或者Service的Status与Application中的Status不一致时，触发调谐逻辑后，为什么是将上下文中对象的Status赋值给Application中的Status，而不是将Applicaiton的状态赋值给现存的Deployment吗？让Deployment成为ApplicationSpec自定义资源期望的状态？**
  
  那是因为这段代码是**Status同步！**
  
  Operator中存在两种不同的同步
  
  | 同步方向                                                    | 目的                                   | 触发时机                        |
  | ----------------------------------------------------------- | -------------------------------------- | ------------------------------- |
  | **Spec 同步：`Application.Spec` → `Deployment`**            | 确保实际资源（Deployment）符合用户期望 | 当 `Application` 被创建或更新时 |
  | **Status 同步：`Deployment.Status` → `Application.Status`** | 将底层资源的实际运行状态反馈给上层 CR  | Reconcile 循环中持续观察        |
  
  代码中关于Status同步的理解是，我已经找到了这个Application对应的Deployment，现在要检查其实际运行状态`dp.status`是否已经反映到了Application的状态字段中，如果没用，就要把Deployment的状态上报到`Application.Status.Workflow`中，让Application资源实例实时反应对应子资源的状态
  
  ==这叫做状态上报==
  
* 资源调谐补充-在经历过LSTMServerOperator之后

  1. 问题：每次触发Reconcile，你觉得需不需要判定现存的Deployment和Service对应的属性与CR中定义的属性是否相同？如果不同的话将其变为一致

     是的，**一定要判定并同步差异（diff），让现存的 Deployment 与 Service 与 CR Spec 一致。**

     如果不判定同步差异，在用户修改了CR的情况下，该CR就会出现失真状态；

     **如果采用每次Reconcile就无脑用当前CR状态覆盖现存资源状态并更新，会导致不必要的重启，对有状态服务的应用来说影响很大**

     因此还是采用逐个对比，如果发生了变化再调用Kubernetes API更新现有资源

  2. 问题：在CR定义中，允许为空的变量，在Reconcile时，应该如何处理，例如Container的ResourceLimit

     在Operator的Reconcile中，永远要给这些字段一个默认值，或者安全返回值，这样可以保证即使用户没有在CR中填写这些字段，自己的Reconcile逻辑也不会panic或者创建出非法对象

     **在LSTMServerOperator的实现中，采用的方法是，在Reconciler中并没有实现默认值的注入和检查，而是在Default/Validate Webhook中实现这些允许为空的变量的默认值注入和校验**

  3. 问题：我想知道，如果Spec的Replicas类型定义为*int32，那么在yaml中应该如何写，才能被正确解析？

     设定为`*int32`是为了区分用户没有提供和用户设定为0的情况，即使类型定义为指针，在yaml中，用户写为3，解析得到依然是数字

     ~~~go
     app.Spec.Replicas = pointer.Int32(3)
     ~~~

     如果设定类型为int32，如果用户没有提供，则默认为0，而设定为`*int32`,如果用户没有提供，默认为nil

  4. 问题：引用app.Namespace和app.Name应该都是该CRD的Namespace和Name吧，在如下代码中，为什么可以确定该CRD的Deployment资源呢？

     ~~~go
     	err := r.Get(ctx, types.NamespacedName{
     		Namespace: app.Namespace,
     		Name:      app.Name,
     	}, dp)
     ~~~

     这是由控制器的约定决定的，Kubernetes并不知道CR与某个Deployment有关系，而是通过控制器代码中的相同命名

     ~~~go
     	newDp := &appsv1.Deployment{}
     	newDp.SetName(app.Name)
     	newDp.SetNamespace(app.Namespace)
     ~~~

     至于类型，即使同命名空间下也有同名的不同类型资源，这是根据`r.Get()`的第三个参数的类型，明确告知了client-go需要的资源类型

  5. 问题：**以子资源Deployment的调谐为例，如果对该CR相关属性的改动触发了其子资源Deployment的改动，这时候触发的是该CR的调谐逻辑还是DeployedController原生的调谐逻辑？如果没有改动CR的相关属性，而是直接edit了该CR子资源Deployment的相关属性，那么此时触发的是CR的调谐逻辑还是DeploymentController的调谐逻辑？该CR的对Deployment的控制和原生DeploymentController对该CR的Deployment子资源的控制的关系是？**

     * 第一种情况：**对该 CR 的相关属性改动，导致子资源 Deployment 被更新，这时触发的是谁的调谐逻辑？**

       1. 你修改了 CR 的 `Spec`；
       2. controller-runtime 的 **Informer** 检测到 CR 发生变化；
       3. 你的 **`LSTMPredictAppReconciler`** 收到事件，触发一次 `Reconcile()`；
       4. 在 Reconcile 中，你发现当前 Deployment 配置与新的 CR.Spec 不一致，于是执行 `UpdateDeployment()`；
       5. 这一步会触发：
          * 你的 Operator 对 Deployment 资源执行一次 `Update`；
          * DeploymentController 检测到该 Deployment.Spec 改变（比如 replicas 数量），于是它也会触发自身的 Reconcile。

       所以这时有两个调谐动作：

       - 你的 CR Controller：保证 Deployment 的定义与 CR 一致；
       - 原生的 Deployment Controller：保证 Deployment 与 Pod 副本状态一致。

     * 第二种情况：**没有改动 CR，而直接编辑 Deployment 的属性，这时触发的是谁的调谐逻辑？**

       会触发 **两个控制器**：

       1. **DeploymentController**（原生控制器）：

          * 它总是监听 Deployment 的变化；

          - 它会根据 Deployment 的变更，重新 reconcile 它的 ReplicaSet / Pod；
          - 例如你把 replicas 改成 10，它就会新建 Pods。

       2. **LSTMPredictAppController**（你的 Operator）：

          - 因为你在 `SetupWithManager()` 中注册了 `Owns(&Deployment{})`；
          - 所以 controller-runtime 也会监听 Deployment 的变化；
          - 一旦它发现你手动修改的 Deployment 与 CR 的 Spec 不一致，就会触发 Reconcile；
          - 在下一次 Reconcile 时，**它会把你的手动修改“纠正回去”**（因为它认为 Deployment 应该遵循 CR.Spec）。

       **所以你的 CR 控制器拥有更高层次的“定义权”，DeploymentController 只是实现底层逻辑。**

       

### 7.4.4 设置RBAC权限

> 在实现Deployment和Service的调谐逻辑后，Operator程序默认是没有操作Deployment和Service资源的权限的，但是不用给自己编写RBAC配置，只需要通过几行注释代码，工具会自动生成对应的配置文件

* 在`internal/controller/application_controller.go`文件中的`Reconcile()`方法上，有如下几行注释

  ~~~go
  // +kubebuilder:rbac:groups=apps.wuyong.cn,resources=applications,verbs=get;list;watch;create;update;patch;delete
  // +kubebuilder:rbac:groups=apps.wuyong.cn,resources=applications/status,verbs=get;update;patch
  // +kubebuilder:rbac:groups=apps.wuyong.cn,resources=applications/finalizers,verbs=update
  ~~~

  在下面添加如下注释

  ~~~go
  
  // +kubebuilder:rbac:groups=apps,resources=deployments,verbs=get;list;watch;create;update;patch;delete
  // +kubebuilder:rbac:groups=apps,resources=deployments/status,verbs=get
  // +kubebuilder:rbac:groups=core,resources=services,verbs=get;list;watch;create;update;patch;delete
  // +kubebuilder:rbac:groups=core,resources=services/status,verbs=get
  ~~~

  **但是需要注意，这段注释需要和`Reconcile`方法中间间隔空行**

  在添加完这几行注释后，执行命令`make manifests`

  执行完后，在`config/rbac/role.yaml`中可以看到对相关资源的操作权限

### 7.4.5 过滤调谐事件

> 主要用于控制调谐事件被触发的条件
>
> 1. Application创建时
> 2. Application发生变更时
> 3. Deployment和Service的一些变化事件发生时
>
> 这就需要在`internal/controller/application_controller.go`的SetUpWithManager方法中添加一些逻辑

`SetUpWithManager`是Kubernetes Operator中控制器Controller的注册逻辑，定义了ApplicationReconciler如何与Manager集成，并设置了哪些资源事件会触发Reconciler循环

结果就是启动一个控制器，监听Application资源的变化，并根据需要调谐它管理的子资源

~~~go
// SetupWithManager sets up the controller with the Manager.
func (r *ApplicationReconciler) SetupWithManager(mgr ctrl.Manager) error {
	setupLog := ctrl.Log.WithName("Setup")

	return ctrl.NewControllerManagedBy(mgr).
		// 监听Application资源，通过predicate.Funcs自定义哪些事件会触发Reconcile
		For(&v1.Application{}, builder.WithPredicates(predicate.Funcs{
			// 一旦创建Application，立即触发Reconcile
			CreateFunc: func(event event.CreateEvent) bool {
				return true
			},
			// Application被删除，但是不触发Reconcile，仅打印日志，不会执行任何清理逻辑
			DeleteFunc: func(event event.DeleteEvent) bool {
				setupLog.Info("The Application has been deleted.", "Name", event.Object.GetName())
				return false
			},
			// 只有当ResourceVersion不同，且Spec发生变化时，才触发Reconcile
			UpdateFunc: func(event event.UpdateEvent) bool {
				if event.ObjectNew.GetResourceVersion() == event.ObjectOld.GetResourceVersion() {
					return false
				}
				if reflect.DeepEqual(event.ObjectNew.(*v1.Application).Spec, event.ObjectOld.(*v1.Application).Spec) {
					return false
				}
				return true
			},
		})).
		// 监听Deployment子资源
		Owns(&appsv1.Deployment{}, builder.WithPredicates(predicate.Funcs{
			// 由于是控制器自己创建的，无需响应
			CreateFunc: func(event event.CreateEvent) bool {
				return false
			},
			// 当Deployment被删除（例如误删）,触发Reconcile，让控制器重新创建Deployment，实现自愈
			DeleteFunc: func(event event.DeleteEvent) bool {
				setupLog.Info("The Deployment has been deleted.", "Name", event.Object.GetName())
				return true
			},
			// 只有Spec变化时才触发，防止状态同步风暴，如果Deployment.Spec被外部修改，控制器会将其纠正回期望状态
			UpdateFunc: func(event event.UpdateEvent) bool {
				if event.ObjectNew.GetResourceVersion() == event.ObjectOld.GetResourceVersion() {
					return false
				}
				if reflect.DeepEqual(event.ObjectNew.(*appsv1.Deployment).Spec, event.ObjectOld.(*appsv1.Deployment).Spec) {
					return false
				}
				return true
			},
			GenericFunc: nil,
		})).
		// 监听Service资源,与Deployment资源类似
		Owns(&corev1.Service{}, builder.WithPredicates(predicate.Funcs{
			CreateFunc: func(event event.CreateEvent) bool {
				return false
			},
			DeleteFunc: func(event event.DeleteEvent) bool {
				setupLog.Info("The Service has been deleted.", "Name", event.Object.GetName())
				return true
			},
			UpdateFunc: func(event event.UpdateEvent) bool {
				if event.ObjectNew.GetResourceVersion() == event.ObjectOld.GetResourceVersion() {
					return false
				}
				if reflect.DeepEqual(event.ObjectNew.(*v1.Application).Spec, event.ObjectOld.(*v1.Application).Spec) {
					return false
				}
				return true
			},
		})).
		// 给控制器起名，日志和metrics中显示为controller "application"
		Named("application").
		// 完成注册，将Reconciler绑定到控制器上，并启动事件监听
		Complete(r)
}
~~~

1. 问题1：在Application资源的监听中，DeleteFunc仅仅打印日志，不触发Reconcile，这意味着**不执行任何清理逻辑**

   这种情况，只有在**完全依赖OwnerReference的级联删除并且没有使用Finalizer的情况下**，才能安全的返回False

   **如果使用了Finalizer来实现优雅的删除，还返回了false，资源就永远无法被删除**

2. 问题2：在Service资源的UpdateFunc中，`event.ObjectNew.(*v1.Application)`尝试将`*corev1.Service`强制转换为`*v1.Applcation*`这会导致运行时panic

### 7.4.6 资源别名

主要起到在`kubectl get `时，可以不用写全资源名称application，而使用简写app即可

只需要在`api/v1/application_types.go`的Application结构体上添加注释

~~~go
// +kubebuilder:resource:path=applications,singular=application,scope=Namespaced,shortName=app

// Application is the Schema for the applications API
type Application struct {
	metav1.TypeMeta `json:",inline"`

	// metadata is a standard object metadata
	// +optional
	metav1.ObjectMeta `json:"metadata,omitempty,omitzero"`

	// spec defines the desired state of Application
	// +required
	Spec ApplicationSpec `json:"spec"`

	// status defines the observed state of Application
	// +optional
	Status ApplicationStatus `json:"status,omitempty,omitzero"`
}
~~~



## 7.5 使用Webhook

### 7.5.1 Kubernetes API访问控制与Admission Webhook介绍

* Kubernetes API访问控制

  不管是使用Kubectl、client-go还是什么其他方式访问Kubernetes API，都要经过下图的步骤：
  <img src="https://typora2icture.oss-cn-beijing.aliyuncs.com/img2/202510141138052.png" alt="image.png" style="zoom:80%;" />

  **当一个访问请求发送到API Server时，会依次经过认证、鉴权、准入控制三个主要的过程，而Admission webhook就属于准入控制的范畴**
  
  准入控制（Admission Control）能够实现改变一个请求的内容或者决定是否拒绝一个请求的功能；准入控制主要是在一个对象发生变更时生效，变更包括创建、更新、删除等动作，不包含查询动作
  
  如果配置了多个准入模块，那么这些模块是按顺序工作的；
  
  关于拒绝请求的能力，一个请求在多个准入控制模块中，只要有一个准入控制模块拒绝，这个请求就会被拒绝；
  
  更改一个请求的内容的能力，主要用于给一些请求字段设置默认值
  
  > https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/admission-controllers/
  >
  > 上述链接中，可以看到目前存在的准入控制器及其作用，这其中的多数准入控制器只能决定他们的是否启动
  >
  > 除了在kube-apiserver内部实现的准入控制器外，还有两个特殊的准入控制器：
  >
  > 1. ValidatingAdmissionWebhook
  > 2. MutatingAdmissionWebhook
  >
  > 这是Kubernetes提供的一种拓展机制，让我们能够通过Webhook的方式独立于kube-apiserver运行自己的准入控制逻辑
  
* Admission Webhook介绍

  Admission Webhook是一个HTTP的回调钩子，可以用于接收准入请求，然后对这个请求做相应的逻辑；主要包括如下两种Admission Webhook

  1. ValidatingAdmissionWebhook
  2. MutatingAdmissionWebhook

  先执行的是`MutatingAdmissionWebhook`，这个准入控制器可以修改请求对象，主要用于注入自定义字段；当这个对象被API Server校验时，就会回调`ValidatingAdmissionWebhook`，然后相应的自定义校验策略就会被执行，以决定这个请求能否被通过

### 7.5.3 Admission Webhook的实现

* 利用Kubebuilder创建webhook脚手架

  注意，webhook脚手架不和Operator一样是单独创建的，webhook需要依赖于Operator项目进行创建，因此我们依然在上述的`application-operator-plus`项目中进行webhook脚手架的创建

  ~~~sh
  kubebuilder create webhook --group apps --version v1 --kind Application --defaulting --programmatic-validation
  ~~~

  执行完上述命令后，可以打开`internal/webhook/v1`看到很多关于webhook的代码，现在我们先实现`MutatingAdmissionWebhook`

* 实现`internal/webhool/v1/application_webhook.go`中的`MutatingAdmissionWebhook`部分

  其实就是完成上述代码的`Default()`方法；这里以Deployment的Replicas默认住注入为例，如果用户提交的Application配置中没有给出Replicas的大小，那么注入默认值3

  书本上给出的是旧版本Webhook与CRD还没有解耦时的方法，在这里给出作为参考：

  ~~~go
  func (r *Application) Default() {
  	applicationlog.Info("default", "name", r.Name)
  	
  	if r.Spec.Deployment.Replicas == nil {
  		r.Spec.Deployment.Replicas = new(int32)
  		*r.Spec.Deployment.Replicas = 2
  	}
  }
  ~~~

  这里还是实际上以高本版代码为准：

  ~~~go
  // Default implements webhook.CustomDefaulter so a webhook will be registered for the Kind Application.
  func (d *ApplicationCustomDefaulter) Default(_ context.Context, obj runtime.Object) error {
  	// 从上下文中，获取对应的Application对象
  	application, ok := obj.(*appsv1.Application)
  
  	if !ok {
  		return fmt.Errorf("expected an Application object but got %T", obj)
  	}
  	applicationlog.Info("Defaulting for Application", "name", application.GetName())
  
  	// TODO(user): fill in your defaulting logic.
  	// 查看Application对象的Replicas是否为空，如果为空，设置为3
  	if application.Spec.Deployment.Replicas == nil {
  		application.Spec.Deployment.Replicas = new(int32)
  		*application.Spec.Deployment.Replicas = 3
  	}
  
  	return nil
  }
  ~~~

* 实现`internal/webhook/v1/application_webhook.go`中的`ValidatingAdmissionWebhook`部分

  在该文件的后面，我们可以发现三个`ValidateXXX()`方法，分别是`ValidateCreate`、`ValidateUpdate`、`ValidateDelete`

  这三个Validate方法的触发条件分别是相应对象在创建、更新、删除的时候；

  在删除时，不需要做什么校验逻辑，而创建和更新时的校验逻辑几乎一致，因此我们将创建和更新时的校验逻辑封装一下，编写一个`validateApplication()`方法

  ~~~go
  func (v *ApplicationCustomValidator) validateApplication(application *appsv1.Application) error {
  	if *application.Spec.Deployment.Replicas > 10 {
  		return fmt.Errorf("replicas too many error")
  	}
  	return nil
  }
  ~~~

  这里简单校验Replicas的设置是否过大，其他业务逻辑也是类似的校验方法。如果觉得条件不满足，就返回一个error，否则返回nil

  然后在三个validate函数中调用该函数

  ~~~go
  // ValidateCreate implements webhook.CustomValidator so a webhook will be registered for the type Application.
  func (v *ApplicationCustomValidator) ValidateCreate(_ context.Context, obj runtime.Object) (admission.Warnings, error) {
  	application, ok := obj.(*appsv1.Application)
  	if !ok {
  		return nil, fmt.Errorf("expected a Application object but got %T", obj)
  	}
  	applicationlog.Info("Validation for Application upon creation", "name", application.GetName())
  
  	// TODO(user): fill in your validation logic upon object creation.
  	if err := v.validateApplication(application); err != nil {
  		return admission.Warnings{"Application Webhook Errors!"}, err
  	}
  	return nil, nil
  }
  
  // ValidateUpdate implements webhook.CustomValidator so a webhook will be registered for the type Application.
  func (v *ApplicationCustomValidator) ValidateUpdate(_ context.Context, oldObj, newObj runtime.Object) (admission.Warnings, error) {
  	application, ok := newObj.(*appsv1.Application)
  	if !ok {
  		return nil, fmt.Errorf("expected a Application object for the newObj but got %T", newObj)
  	}
  	applicationlog.Info("Validation for Application upon update", "name", application.GetName())
  
  	// TODO(user): fill in your validation logic upon object update.
  	if err := v.validateApplication(application); err != nil {
  		return admission.Warnings{"Application Webhook Errors!"}, err
  	}
  
  	return nil, nil
  }
  
  // ValidateDelete implements webhook.CustomValidator so a webhook will be registered for the type Application.
  func (v *ApplicationCustomValidator) ValidateDelete(ctx context.Context, obj runtime.Object) (admission.Warnings, error) {
  	application, ok := obj.(*appsv1.Application)
  	if !ok {
  		return nil, fmt.Errorf("expected a Application object but got %T", obj)
  	}
  	applicationlog.Info("Validation for Application upon deletion", "name", application.GetName())
  
  	// TODO(user): fill in your validation logic upon object deletion.
  	if err := v.validateApplication(application); err != nil {
  		return admission.Warnings{"Application Webhook Errors!"}, err
  	}
  
  	return nil, nil
  }
  ~~~

* 如果想在本地测试运行Webhook，默认需要准备证书，放到master主机的`/tmp/k8s-webhook-server/serving-certs/tls.{crt,key}`中，然后执行make run命令

  现在我们介绍如何准备证书

  ~~~sh
  #创建目录
  mkdir -p /tmp/k8s-webhook-server/serving-certs
  
  #使用openssl生成证书，CN必须匹配Service名
  opessl req -x509 -nodes -days 365 -newkey rsa:2048 \
  	-keyout /tmp/k8s-webhook-server/serving-certs/tls.key \
  	-out /tmp/k8s-webhook-server/serving-certs/tls.crt \
  	-subj "/CN=application-operator.svc"
  ~~~

  > application-operator.svc是你的Service名，如果没改，默认可能是controller-manager
  >
  > 如果在集群中部署，证书的CN应该是:<service-name>.<namespace>.svc

  这样，在`tmp/k8s-webhook-server/serving-certs/`下能够看到tls.key和tls.crt文件



### 7.5.4 cert-manager部署

* 在部署Webhook之前，需要先安装cert-manager，用来实现证书签发的功能；该组件在使用Admission Webhook时经常被提及

  Kubernetes的Admission Webhook是通过HTTPS调用的，API Server在创建更新资源时，会向自定义的Operator发起HTTPS请求，进行默认值设置和校验

  **为了保证安全，该通信必须是加密的，并且API Server必须验证自定义的Webhook服务器身份，这就是TLS证书的作用，因此需要一个私钥(tls.key)和一个证书(tls.crt)**

  **证书必须由API Server信任的CA签发**

  > 自签名证书虽然能工作，但是必须将CA证书注入到APIService或者MutatingWebhookConfiguration中，否则API Server不信任

  而`cert-manager`是一个Kubernetes控制器，用于自动申请、管理、续期TLS证书，可以为内部服务签发证书

* 安装方式有两个

  1. 添加仓库安装

     ~~~sh
     helm repo add jetstack https://charts.jetstack.io
     helm repo update
     helm search repo jetstack
     
     helm install \
       cert-manager jetstack/cert-manager \
       --version v1.19.0 \
       --namespace cert-manager \
       --create-namespace \
       --set crds.enabled=true
     ~~~

  2. 直接获取安装

     ~~~sh
     helm install \
       cert-manager oci://quay.io/jetstack/charts/cert-manager \
       --version v1.19.0 \
       --namespace cert-manager \
       --create-namespace \
       --set crds.enabled=true
     ~~~

* 安装完成后

  ~~~sh
  (base) root@master:~/Golang/code/go/src/sigs.k8s.io/MyOperatorProjects/application-operator-plus/test-yaml# kubectl get pod -n cert-manager -o wide
  NAME                                       READY   STATUS    RESTARTS   AGE   IP             NODE    NOMINATED NODE   READINESS GATES
  cert-manager-69c944794b-4mtc2              1/1     Running   0          21h   10.244.3.190   node3   <none>           <none>
  cert-manager-cainjector-586c585ff7-5q7j6   1/1     Running   0          21h   10.244.3.191   node3   <none>           <none>
  cert-manager-webhook-77bf6fc8f4-bfmhh      1/1     Running   0          21h   10.244.3.189   node3   <none>           <none>
  ~~~



### 7.5.5 Webhook部署运行

* 构建镜像

  这里需要注意两个点

  1. 如果本地使用的镜像构建工具不是docker，需要更改Makefile中的`CONTAINER_TOOL`

  2. 在构建go镜像时，可能会卡在go mod download,需要在Dockerfile中，在RUN go mod download之前添加

     ~~~dockerfile
     WORKDIR /workspace
     ENV GOPROXY=https://goproxy.cn,direct
     
     # Copy the Go Modules manifests
     COPY go.mod go.mod
     COPY go.sum go.sum
     # cache deps before building and copying source so that we don't need to re-download as much
     # and so that source changes don't invalidate our downloaded layer
     RUN go mod download
     ~~~

  然后使用镜像构建命令

  ~~~sh
  make docker-build IMG=application-operator-plus:v0.1
  ~~~

* 部署CRD

  ~~~sh
  make install
  ~~~

* 证书相关配置

  主要是修改operator项目中`config/default/kustomization.yaml`和`config/crd/kustomization.yaml`

  仔细阅读对应yaml文件中的注释，根据需要打开相应被注释掉的内容，如下是本次的两个文件

  1. `config/default/kustomization.yaml`

     ~~~YAML
     # Adds namespace to all resources.
     namespace: application-operator-plus-system
     
     # Value of this field is prepended to the
     # names of all resources, e.g. a deployment named
     # "wordpress" becomes "alices-wordpress".
     # Note that it should also match with the prefix (text before '-') of the namespace
     # field above.
     namePrefix: application-operator-plus-
     
     # Labels to add to all resources and selectors.
     #labels:
     #- includeSelectors: true
     #  pairs:
     #    someName: someValue
     
     resources:
     - ../crd
     - ../rbac
     - ../manager
     # [WEBHOOK] To enable webhook, uncomment all the sections with [WEBHOOK] prefix including the one in
     # crd/kustomization.yaml
     - ../webhook
     # [CERTMANAGER] To enable cert-manager, uncomment all sections with 'CERTMANAGER'. 'WEBHOOK' components are required.
     - ../certmanager
     # [PROMETHEUS] To enable prometheus monitor, uncomment all sections with 'PROMETHEUS'.
     #- ../prometheus
     # [METRICS] Expose the controller manager metrics service.
     - metrics_service.yaml
     # [NETWORK POLICY] Protect the /metrics endpoint and Webhook Server with NetworkPolicy.
     # Only Pod(s) running a namespace labeled with 'metrics: enabled' will be able to gather the metrics.
     # Only CR(s) which requires webhooks and are applied on namespaces labeled with 'webhooks: enabled' will
     # be able to communicate with the Webhook Server.
     #- ../network-policy
     
     # Uncomment the patches line if you enable Metrics
     patches:
     # [METRICS] The following patch will enable the metrics endpoint using HTTPS and the port :8443.
     # More info: https://book.kubebuilder.io/reference/metrics
     - path: manager_metrics_patch.yaml
       target:
         kind: Deployment
     
     # Uncomment the patches line if you enable Metrics and CertManager
     # [METRICS-WITH-CERTS] To enable metrics protected with certManager, uncomment the following line.
     # This patch will protect the metrics with certManager self-signed certs.
     - path: cert_metrics_manager_patch.yaml
       target:
        kind: Deployment
     
     # [WEBHOOK] To enable webhook, uncomment all the sections with [WEBHOOK] prefix including the one in
     # crd/kustomization.yaml
     - path: manager_webhook_patch.yaml
       target:
         kind: Deployment
     
     # [CERTMANAGER] To enable cert-manager, uncomment all sections with 'CERTMANAGER' prefix.
     # Uncomment the following replacements to add the cert-manager CA injection annotations
     replacements:
     - source: # Uncomment the following block to enable certificates for metrics
         kind: Service
         version: v1
         name: controller-manager-metrics-service
         fieldPath: metadata.name
       targets:
         - select:
             kind: Certificate
             group: cert-manager.io
             version: v1
             name: metrics-certs
           fieldPaths:
             - spec.dnsNames.0
             - spec.dnsNames.1
           options:
             delimiter: '.'
             index: 0
             create: true
         - select: # Uncomment the following to set the Service name for TLS config in Prometheus ServiceMonitor
             kind: ServiceMonitor
             group: monitoring.coreos.com
             version: v1
             name: controller-manager-metrics-monitor
           fieldPaths:
             - spec.endpoints.0.tlsConfig.serverName
           options:
             delimiter: '.'
             index: 0
             create: true
     
     - source:
         kind: Service
         version: v1
         name: controller-manager-metrics-service
         fieldPath: metadata.namespace
       targets:
         - select:
             kind: Certificate
             group: cert-manager.io
             version: v1
             name: metrics-certs
           fieldPaths:
             - spec.dnsNames.0
             - spec.dnsNames.1
           options:
             delimiter: '.'
             index: 1
             create: true
         - select: # Uncomment the following to set the Service namespace for TLS in Prometheus ServiceMonitor
             kind: ServiceMonitor
             group: monitoring.coreos.com
             version: v1
             name: controller-manager-metrics-monitor
           fieldPaths:
             - spec.endpoints.0.tlsConfig.serverName
           options:
             delimiter: '.'
             index: 1
             create: true
     
     - source: # Uncomment the following block if you have any webhook
          kind: Service
          version: v1
          name: webhook-service
          fieldPath: .metadata.name # Name of the service
       targets:
          - select:
              kind: Certificate
              group: cert-manager.io
              version: v1
              name: serving-cert
            fieldPaths:
              - .spec.dnsNames.0
              - .spec.dnsNames.1
            options:
              delimiter: '.'
              index: 0
              create: true
     - source:
          kind: Service
          version: v1
          name: webhook-service
          fieldPath: .metadata.namespace # Namespace of the service
       targets:
          - select:
              kind: Certificate
              group: cert-manager.io
              version: v1
              name: serving-cert
            fieldPaths:
              - .spec.dnsNames.0
              - .spec.dnsNames.1
            options:
              delimiter: '.'
              index: 1
              create: true
     
     - source: # Uncomment the following block if you have a ValidatingWebhook (--programmatic-validation)
         kind: Certificate
         group: cert-manager.io
         version: v1
         name: serving-cert # This name should match the one in certificate.yaml
         fieldPath: .metadata.namespace # Namespace of the certificate CR
       targets:
          - select:
              kind: ValidatingWebhookConfiguration
            fieldPaths:
              - .metadata.annotations.[cert-manager.io/inject-ca-from]
            options:
              delimiter: '/'
              index: 0
              create: true
     - source:
         kind: Certificate
         group: cert-manager.io
         version: v1
         name: serving-cert
         fieldPath: .metadata.name
       targets:
          - select:
              kind: ValidatingWebhookConfiguration
            fieldPaths:
              - .metadata.annotations.[cert-manager.io/inject-ca-from]
            options:
              delimiter: '/'
              index: 1
              create: true
     
     - source: # Uncomment the following block if you have a DefaultingWebhook (--defaulting )
         kind: Certificate
         group: cert-manager.io
         version: v1
         name: serving-cert
         fieldPath: .metadata.namespace # Namespace of the certificate CR
       targets:
          - select:
              kind: MutatingWebhookConfiguration
            fieldPaths:
              - .metadata.annotations.[cert-manager.io/inject-ca-from]
            options:
              delimiter: '/'
              index: 0
              create: true
     - source:
         kind: Certificate
         group: cert-manager.io
         version: v1
         name: serving-cert
         fieldPath: .metadata.name
       targets:
         - select:
             kind: MutatingWebhookConfiguration
           fieldPaths:
             - .metadata.annotations.[cert-manager.io/inject-ca-from]
           options:
             delimiter: '/'
             index: 1
             create: true
     
     # - source: # Uncomment the following block if you have a ConversionWebhook (--conversion)
     #     kind: Certificate
     #     group: cert-manager.io
     #     version: v1
     #     name: serving-cert
     #     fieldPath: .metadata.namespace # Namespace of the certificate CR
     #   targets: # Do not remove or uncomment the following scaffold marker; required to generate code for target CRD.
     # +kubebuilder:scaffold:crdkustomizecainjectionns
     # - source:
     #     kind: Certificate
     #     group: cert-manager.io
     #     version: v1
     #     name: serving-cert
     #     fieldPath: .metadata.name
     #   targets: # Do not remove or uncomment the following scaffold marker; required to generate code for target CRD.
     # +kubebuilder:scaffold:crdkustomizecainjectionname
     ~~~

  2. `config/crd/kustomization.yaml`

     ~~~YAML
     # This kustomization.yaml is not intended to be run by itself,
     # since it depends on service name and namespace that are out of this kustomize package.
     # It should be run by config/default
     resources:
     - bases/apps.wuyong.cn_applications.yaml
     # +kubebuilder:scaffold:crdkustomizeresource
     
     patches:
     # [WEBHOOK] To enable webhook, uncomment all the sections with [WEBHOOK] prefix.
     # patches here are for enabling the conversion webhook for each CRD
     # +kubebuilder:scaffold:crdkustomizewebhookpatch
     
     # [WEBHOOK] To enable webhook, uncomment the following section
     # the following config is for teaching kustomize how to do kustomization for CRDs.
     configurations:
     - kustomizeconfig.yaml
     ~~~

* 最后部署控制器

  ~~~sh
  make deploy IMG=application-operator-plus:v0.1
  ~~~

  查看Pod是否正常运行

  ~~~sh
  (base) root@master:~/Golang/code/go/src/sigs.k8s.io/MyOperatorProjects/application-operator-plus/test-yaml# kubectl get pod -n application-operator-plus-system -o wide
  NAME                                                            READY   STATUS    RESTARTS   AGE     IP             NODE    NOMINATED NODE   READINESS GATES
  application-operator-plus-controller-manager-5c878f7bc7-zwflm   1/1     Running   0          4h22m   10.244.1.167   node1   <none>           <none>
  ~~~

* 测试用yaml

  ~~~yaml
  apiVersion: apps.wuyong.cn/v1
  kind: application
  metadata:
    name: application-sample
    namespace: k8s-learn
    labels:
      app: application
  spec:
    deployment:
      selector:
        matchLabels:
          app: application
      template:
        metadata:
          labels:
            app: application
        spec:
          containers:
            - name: nginx
              image: nginx:1.14.2
              ports:
                - containerPort: 80
    service:
      type: NodePort
      ports:
        - port: 80
          targetPort: 80
          nodePort: 30080
  ~~~

  部署时，发生错误

  ~~~sh
  (base) root@master:~/Golang/code/go/src/sigs.k8s.io/MyOperatorProjects/application-operator-plus/test-yaml# kubectl apply -f apps_v1_application.yaml 
  Warning: Application Webhook Errors!
  Error from server (Forbidden): error when creating "apps_v1_application.yaml": admission webhook "vapplication-v1.kb.io" denied the request: replicas too many error
  ~~~

  然后将replicas字段删除，再次部署，可以看到成功部署，并且副本数为3

  ~~~sh
  (base) root@master:~/Golang/code/go/src/sigs.k8s.io/MyOperatorProjects/application-operator-plus/test-yaml# kubectl get pod -n k8s-learn -o wide
  NAME                                               READY   STATUS    RESTARTS   AGE    IP             NODE    NOMINATED NODE   READINESS GATES
  application-sample-7cd557df66-6zxrj                1/1     Running   0          4h3m   10.244.2.218   node2   <none>           <none>
  application-sample-7cd557df66-9knhg                1/1     Running   0          4h3m   10.244.1.168   node1   <none>           <none>
  application-sample-7cd557df66-h5sdz                1/1     Running   0          4h3m   10.244.3.193   node3   <none>           <none>
  my-custom-metrics-app-deployment-5976cd49c-5wngs   1/1     Running   0          92d    10.244.3.184   node3   <none>           <none>
  my-custom-metrics-app-deployment-5976cd49c-dr77c   1/1     Running   0          92d    10.244.3.182   node3   <none>           <none>
  neilats-refactor-controller-b7c4dc857-dh7pn        1/1     Running   0          42d    10.244.2.207   node2   <none>           <none>
  neilats-refactor-scheduler-9b5955d75-zf55h         1/1     Running   0          42d    10.244.2.208   node2   <none>           <none>
  
  (base) root@master:~/Golang/code/go/src/sigs.k8s.io/MyOperatorProjects/application-operator-plus/test-yaml# kubectl get app -n k8s-learn
  NAME                 AGE
  application-sample   4h3m
  ~~~

  

## 7.6 API多版本支持

> 一般来说，开发新项目其API是会经常变更的，这节介绍Operator是如何支持多版本API的

### 7.6.1 实现V2版本API

* 利用kubebuilder添加v2版本API

  ~~~sh
  kubebuilder create api \
  --group apps \
  --version v2 \
  --kind Application
  Create Resource [y/n]
  y
  Create Controller [y/n]
  n
  ~~~

  注意，这里不为v2版本创建controller，继续沿用v1的controller，具体后面会介绍

* 然后在api/v2中，像编辑v1的`application_types.go`一样编辑v2版本的`application_types.go`

  其余的都与v1一样，这里唯一的变化在于

  ~~~go
  type ApplicationSpec struct {
  	// INSERT ADDITIONAL SPEC FIELDS - desired state of cluster
  	// Important: Run "make" to regenerate code after modifying this file
  	// The following markers will use OpenAPI v3 schema to validate the value
  	// More info: https://book.kubebuilder.io/reference/markers/crd-validation.html
  
  	// +kubebuilder:pruning:PreserveUnknownFields
  	// +kubebuilder:validation:Schemaless
  	Workflow v1.DeploymentTemplate `json:"workflow,omitempty"`
  	Service  v1.ServiceTemplate    `json:"service,omitempty"`
  }
  ~~~

  即原本v1的Deployment，被重命名为Workflow了

* **由于有了多个版本的API，我们在API Server中只能指定持久化一个版本，即不管有多少个API版本，存储进ectd中的只有一个API版本，其余版本通过conversion Webhook转换到存储版本后存储进ectd**

  我们这里指定v2版本为存储版本，因此在V2版本的`Application`结构体上加一行注释：`// +kubebuilder:storageversion`

  ~~~go
  // +kubebuilder:object:root=true
  // +kubebuilder:storageversion
  // +kubebuilder:subresource:status
  // +kubebuilder:storageversion
  
  // Application is the Schema for the applications API
  type Application struct {
  	metav1.TypeMeta `json:",inline"`
  
  	// metadata is a standard object metadata
  	// +optional
  	metav1.ObjectMeta `json:"metadata,omitempty,omitzero"`
  
  	// spec defines the desired state of Application
  	// +required
  	Spec ApplicationSpec `json:"spec"`
  
  	// status defines the observed state of Application
  	// +optional
  	Status ApplicationStatus `json:"status,omitempty,omitzero"`
  }
  ~~~

### 7.6.2 为两个版本转换添加conversion Webhook

* 根据Kubebuilder的[官方文档](https://book.kubebuilder.io/multiversion-tutorial/conversion)，我们可能意会为，在v1版本已经创建了default/validate Webhook后，再创建一个conversion Webhook，其命令如下所示

  ~~~sh
  kubebuilder create webhook --group apps --version v1 --kind Application --conversion --spoke v2
  ~~~

  `--version`表示以哪个版本为存储版本创建conversion Webhook，`--spoke`表示那个版本为被转换的版本

  如果在v1版本已经有了default/validate Webhook的情况下，继续执行此命令的话，就会出错：

  ~~~sh
  ERROR CLI run failed error=error executing command: failed to create webhook: unable to inject the resource to "base.go.kubebuilder.io/v4": webhook resource already exists
  ~~~

  **这是因为kubebuilder在项目结构中，每个Kind的一个Api版本只允许存在一个Webhook，例如v1已经创建了default/validate Webhook，那就不能存在conversion Webhook了**

  **AI给出的解决方案是使用`--force`强制覆盖原有Webhook，但是这会导致丢失原本的Webhook；方案二是手动扩展出conversion Webhook，即手动创建`api/v1/application_conversion.go`、`api/v2/application_conversion.go`、`internal/webhook/v2/application_webhook.go`**

  **但是这样也会存在问题，就是无法生成相应的配置文件`config/crd/patches/webhool_in_apps_applications.yaml`文件和其他相应配置文件**

* 这里给出的方法是，以v2版本作为存储版本，以v2版本为核心在v2版本中创建conversion Webhook，其命令如下

  ~~~sh
  kubebuilder create webhook --group apps --version v2 --kind Application --conversion --spoke v1
  ~~~

  这样就能够成功创建conversion Webhook了，并且也能成功生成相应的配置文件；然后我们再手动补全default/validation Webhook

  > 为什么创建的时候，不能一次创建两种Webhook？
  >
  > 1. conversion Webhook必须是version pair(hub/spoke)形式，而default/validation Webhook仅仅属于一个版本
  >
  >    kubebuilder内部的Webhook scaffolding模板无法在一次命令中混合这两种逻辑
  >
  > 2. v4架构下的kubebuilder使用一个统一的Webhook registry(internal/webhook),如果一个Kind已经存在了Webhook，就拒绝写入，避免文件冲突

* 现在，我们有了`api/v1/application_conversion.go`、`api/v2/application_conversion.go`之后，根据[官方文档](https://book.kubebuilder.io/multiversion-tutorial/conversion)进行补全

  1. 但是其实还是会出错

     ~~~sh
     ERROR CLI run failed error=error executing command: failed to create webhook: unable to run post-scaffold tasks of "base.go.kubebuilder.io/v4": error running make generate: error running "make": exit status 2
     ~~~

     虽然成功生成了两个版本的`api/v*/application_conversion.go`

     **这是因为，由于我们在V2的ApplicationSpec中，引用了v1版本中的结构体DeploymentTemplate，而由于v1变为了spoke版本，所以在`api/v1/application_conversion.go`中引入了v2版本的包**

     **这样就导致了循环引用**

     这里有两种解决方法来解决v2版本依赖v1版本中的内容

     * 方法一（采用）

       抽出公共代码到shared包，即创建文件夹`api/shared/common.go`，将公共代码放到`common.go`中，避免两个版本的循环引用

       `api/shared/common.go`

       ~~~go
       appsv1.DeploymentSpec// +k8s:deepcopy-gen=package
       package shared
       
       import (
       	appsv1 "k8s.io/api/apps/v1"
       	corev1 "k8s.io/api/core/v1"
       )
       
       type DeploymentTemplate struct {
       	Spec appsv1.DeploymentSpec `json:",inline"`
       }
       
       type ServiceTemplate struct {
       	Spec corev1.ServiceSpec `json:",inline"`
       }
       ~~~

       > 抽出来后，需要在包前加注释`// +k8s:deepcopy-gen=package`
       >
       > 方便使用命令`make generate`为shared包中的内容，生成deepcopy等方法；
       >
       > 并且，由于原本的`common.go`文件如下
       >
       > ~~~go
       > // +k8s:deepcopy-gen=package
       > package shared
       > 
       > import (
       > 	appsv1 "k8s.io/api/apps/v1"
       > 	corev1 "k8s.io/api/core/v1"
       > )
       > 
       > type DeploymentTemplate struct {
       > 	appsv1.DeploymentSpec `json:",inline"`
       > }
       > 
       > type ServiceTemplate struct {
       > 	corev1.ServiceSpec `json:",inline"`
       > }
       > ~~~
       >
       > 并且v1包中定义的ApplicationSpec.Deployment字段名与官方Kubernetes类型冲突，shared.DeploymentTemplate使用了内联的appsv1.DeploymentSpec类型，使得controller-gen在生成deepcopy时，以为ApplicationSpec.Deployment是appsv1.DeploymentSpec类型，所以DeepcopyInto中就错误的把shared.DeploymentTemplate当作appsv1.DeploymentSpec处理了
       >
       > 解决方法就是避免内嵌官方类型，改为字段引用，也就是如现在的`common.go`所示
       >
       > 或者为shared类型显示定义自己的DeepCopyInto和Deepcopy方法（未采用）
       >
       > ~~~go
       > package shared
       > 
       > import (
       >     appsv1 "k8s.io/api/apps/v1"
       > )
       > 
       > type DeploymentTemplate struct {
       >     appsv1.DeploymentSpec `json:",inline"`
       > }
       > 
       > // +k8s:deepcopy-gen=true
       > func (in *DeploymentTemplate) DeepCopyInto(out *DeploymentTemplate) {
       >     *out = *in
       >     in.DeploymentSpec.DeepCopyInto(&out.DeploymentSpec)
       > }
       > 
       > func (in *DeploymentTemplate) DeepCopy() *DeploymentTemplate {
       >     if in == nil {
       >         return nil
       >     }
       >     out := new(DeploymentTemplate)
       >     in.DeepCopyInto(out)
       >     return out
       > }
       > ~~~

       采用了该方法后，两个版本的`application_types.go`中引入部分和Spec部分如下

       ~~~go
       package v1
       
       import (
       	"github.com/wuyong7240/application-operator-plus/api/shared"
       	appsv1 "k8s.io/api/apps/v1"
       	corev1 "k8s.io/api/core/v1"
       	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
       )
       type ApplicationSpec struct {
       	// +optional
       	// +kubebuilder:pruning:PreserveUnknownFields
       	// +kubebuilder:validation:Schemaless
       	Deployment shared.DeploymentTemplate `json:"deployment,omitempty"`
       	Service    shared.ServiceTemplate    `json:"service,omitempty"`
       }
       ~~~

     * 方法二

       复制定义，即将v2需要用到的v1中的内容复制一份到v2的定义中

  1. `api/v1/application_conversion.go`

     ~~~go
     package v1
     
     import (
     	"log"
     
     	"sigs.k8s.io/controller-runtime/pkg/conversion"
     
     	appsv2 "github.com/wuyong7240/application-operator-plus/api/v2"
     )
     
     // ConvertTo converts this Application (v1) to the Hub version (v2).
     func (src *Application) ConvertTo(dstRaw conversion.Hub) error {
     	dst := dstRaw.(*appsv2.Application)
     	log.Printf("ConvertTo: Converting Application from Spoke version v1 to Hub version v2;"+
     		"source: %s/%s, target: %s/%s", src.Namespace, src.Name, dst.Namespace, dst.Name)
     
     	// TODO(user): Implement conversion logic from v1 to v2
     	// Example: Copying Spec fields
     	// dst.Spec.Size = src.Spec.Replicas
     
     	// Copy ObjectMeta to preserve name, namespace, labels, etc.
     	dst.ObjectMeta = src.ObjectMeta
     
     	// Spec
     	dst.Spec.Service = src.Spec.Service
     	dst.Spec.Workflow = src.Spec.Deployment
     
     	// Status
     	dst.Status.Network = src.Status.Network
     	dst.Status.Workflow = src.Status.Workflow
     
     	return nil
     }
     
     // ConvertFrom converts the Hub version (v2) to this Application (v1).
     func (dst *Application) ConvertFrom(srcRaw conversion.Hub) error {
     	src := srcRaw.(*appsv2.Application)
     	log.Printf("ConvertFrom: Converting Application from Hub version v2 to Spoke version v1;"+
     		"source: %s/%s, target: %s/%s", src.Namespace, src.Name, dst.Namespace, dst.Name)
     
     	// TODO(user): Implement conversion logic from v2 to v1
     	// Example: Copying Spec fields
     	// dst.Spec.Replicas = src.Spec.Size
     
     	// Copy ObjectMeta to preserve name, namespace, labels, etc.
     	dst.ObjectMeta = src.ObjectMeta
     
     	// Spec
     	dst.Spec.Deployment = src.Spec.Workflow
     	dst.Spec.Service = src.Spec.Service
     
     	// Status
     	dst.Status.Network = src.Status.Network
     	dst.Status.Workflow = src.Status.Workflow
     
     	return nil
     }
     ~~~

  2. `api/v2/application_conversion.go`

     ~~~go
     package v2
     
     // EDIT THIS FILE!  THIS IS SCAFFOLDING FOR YOU TO OWN!
     
     // Hub marks this type as a conversion hub.
     func (*Application) Hub() {}
     ~~~

* 然后再对v2版本的Webhook，进行手动补全default/validation Webhook

  `internal/webhook/v2/application_webhook.go`

  ~~~go
  package v2
  
  import (
  	"context"
  	"fmt"
  
  	"k8s.io/apimachinery/pkg/runtime"
  
  	ctrl "sigs.k8s.io/controller-runtime"
  	logf "sigs.k8s.io/controller-runtime/pkg/log"
  	"sigs.k8s.io/controller-runtime/pkg/webhook"
  	"sigs.k8s.io/controller-runtime/pkg/webhook/admission"
  
  	appsv2 "github.com/wuyong7240/application-operator-plus/api/v2"
  )
  
  // nolint:unused
  // log is for logging in this package.
  var applicationlog = logf.Log.WithName("application-resource")
  
  // SetupApplicationWebhookWithManager registers the webhook for Application in the manager.
  func SetupApplicationWebhookWithManager(mgr ctrl.Manager) error {
  	return ctrl.NewWebhookManagedBy(mgr).For(&appsv2.Application{}).
  		WithValidator(&ApplicationCustomValidator{
  			DefaultDeploymentReplicasMax: 10,
  		}).
  		WithDefaulter(&ApplicationCustomDefaulter{
  			DefaultDeploymentReplicas: 3,
  		}).
  		Complete()
  }
  
  // TODO(user): EDIT THIS FILE!  THIS IS SCAFFOLDING FOR YOU TO OWN!
  type ApplicationCustomDefaulter struct {
  	DefaultDeploymentReplicas int32
  }
  
  var _ webhook.CustomDefaulter = &ApplicationCustomDefaulter{}
  
  func (d *ApplicationCustomDefaulter) Default(_ context.Context, obj runtime.Object) error {
  	application, ok := obj.(*appsv2.Application)
  	if !ok {
  		return fmt.Errorf("expected an application object but got %T", obj)
  	}
  	applicationlog.Info("Defaulting for Application", "name", application.Name)
  
  	if application.Spec.Workflow.Spec.Replicas == nil {
  		application.Spec.Workflow.Spec.Replicas = new(int32)
  		*application.Spec.Workflow.Spec.Replicas = d.DefaultDeploymentReplicas
  	}
  
  	return nil
  }
  
  // validation
  type ApplicationCustomValidator struct {
  	DefaultDeploymentReplicasMax int32
  }
  
  var _ webhook.CustomValidator = &ApplicationCustomValidator{}
  
  func (v *ApplicationCustomValidator) ValidateCreate(_ context.Context, obj runtime.Object) (admission.Warnings, error) {
  	application, ok := obj.(*appsv2.Application)
  	if !ok {
  		return nil, fmt.Errorf("expected an Application object but got %T", obj)
  	}
  	applicationlog.Info("Validation for Application upon Creation", "name", application.Name)
  
  	if err := v.validateApplication(application); err != nil {
  		return admission.Warnings{"Application Webhook v2 Errors!"}, err
  	}
  	return nil, nil
  }
  
  func (v *ApplicationCustomValidator) ValidateUpdate(_ context.Context, oldObj, newObj runtime.Object) (admission.Warnings, error) {
  	application, ok := newObj.(*appsv2.Application)
  	if !ok {
  		return nil, fmt.Errorf("expected an Application object but got %T", newObj)
  	}
  	applicationlog.Info("Validation for Application upon Update", "name", application.Name)
  
  	if err := v.validateApplication(application); err != nil {
  		return admission.Warnings{"Application Webhook v2 Errors!"}, err
  	}
  	return nil, nil
  }
  
  func (v *ApplicationCustomValidator) ValidateDelete(ctx context.Context, obj runtime.Object) (admission.Warnings, error) {
  	application, ok := obj.(*appsv2.Application)
  	if !ok {
  		return nil, fmt.Errorf("expected an Application object but got %T", obj)
  	}
  	applicationlog.Info("Validation for Application upon Update", "name", application.Name)
  
  	if err := v.validateApplication(application); err != nil {
  		return admission.Warnings{"Application Webhook v2 Errors!"}, err
  	}
  	return nil, nil
  }
  
  func (v *ApplicationCustomValidator) validateApplication(application *appsv2.Application) error {
  	if *application.Spec.Workflow.Spec.Replicas > v.DefaultDeploymentReplicasMax {
  		return fmt.Errorf("replicas too many error")
  	}
  	return nil
  }
  ~~~



### 7.6.3 部署与测试多版本API

* 现在的`config/default/kustomization.yaml`与`config/crd/kustomization.yaml`新增的改变

  1. `config/default/kustomization.yaml`

     ~~~yaml
     - source: # Uncomment the following block if you have a ConversionWebhook (--conversion)
         kind: Certificate
         group: cert-manager.io
         version: v1
         name: serving-cert
         fieldPath: .metadata.namespace # Namespace of the certificate CR
       targets: # Do not remove or uncomment the following scaffold marker; required to generate code for target CRD.
         - select:
             kind: CustomResourceDefinition
             name: applications.apps.wuyong.cn
           fieldPaths:
             - .metadata.annotations.[cert-manager.io/inject-ca-from]
           options:
             delimiter: '/'
             index: 0
             create: true
     # +kubebuilder:scaffold:crdkustomizecainjectionns
     - source:
         kind: Certificate
         group: cert-manager.io
         version: v1
         name: serving-cert
         fieldPath: .metadata.name
       targets: # Do not remove or uncomment the following scaffold marker; required to generate code for target CRD.
         - select:
             kind: CustomResourceDefinition
             name: applications.apps.wuyong.cn
           fieldPaths:
             - .metadata.annotations.[cert-manager.io/inject-ca-from]
           options:
             delimiter: '/'
             index: 1
             create: true
     # +kubebuilder:scaffold:crdkustomizecainjectionname
     ~~~

  2. `config/crd/kustomization.yaml`

     ~~~yaml
     # This kustomization.yaml is not intended to be run by itself,
     # since it depends on service name and namespace that are out of this kustomize package.
     # It should be run by config/default
     resources:
     - bases/apps.wuyong.cn_applications.yaml
     # +kubebuilder:scaffold:crdkustomizeresource
     
     patches:
     # [WEBHOOK] To enable webhook, uncomment all the sections with [WEBHOOK] prefix.
     # patches here are for enabling the conversion webhook for each CRD
     - path: patches/webhook_in_apps_applications.yaml
     # +kubebuilder:scaffold:crdkustomizewebhookpatch
     
     # [WEBHOOK] To enable webhook, uncomment the following section
     # the following config is for teaching kustomize how to do kustomization for CRDs.
     # configurations:
     # - kustomizeconfig.yaml
     ~~~

  3. `config/crd/patches/webhook_in_apps_applications.yaml`

     ~~~yaml
     # The following patch enables a conversion webhook for the CRD
     apiVersion: apiextensions.k8s.io/v1
     kind: CustomResourceDefinition
     metadata:
       name: applications.apps.wuyong.cn
     spec:
       conversion:
         strategy: Webhook
         webhook:
           clientConfig:
             service:
               namespace: system
               name: application-operator-plus-webhook-service	# 修改过，默认为Webhook-service
               path: /convert
           conversionReviewVersions:
           - v1
     ~~~

     注意，在部署之前，需要修改上述yaml中的Webhook service的名字，否则会发生如下错误：

     ~~~sh
     (base) root@master:~/Golang/code/go/src/sigs.k8s.io/MyOperatorProjects/application-operator-plus/test-yaml# kubectl apply -f apps_v2_application.yaml 
     Error from server (InternalError): error when creating "apps_v2_application.yaml": Internal error occurred: Internal error occurred: conversion webhook for apps.wuyong.cn/v2, Kind=application failed: Post "https://webhook-service.application-operator-plus-system.svc:443/convert?timeout=30s": service "webhook-service" not found
     ~~~

* 然后就是正常的部署过程，不再介绍；设定了如下两个版本的测试yaml

  1. `apps_v1_application.yaml`

     ~~~yaml
     apiVersion: apps.wuyong.cn/v1
     kind: Application
     metadata:
       name: application-sample-v1
       namespace: k8s-learn
       labels:
         app: application
     spec:
       deployment:
         replicas: 11
         selector:
           matchLabels:
             app: application
         template:
           metadata:
             labels:
               app: application
           spec:
             containers:
               - name: nginx
                 image: nginx:1.14.2
                 ports:
                   - containerPort: 80
       service:
         type: NodePort
         ports:
           - port: 80
             targetPort: 80
             nodePort: 30080
     ~~~

     

  2. `apps_v2_application.yaml`

     ~~~yaml
     apiVersion: apps.wuyong.cn/v2
     kind: Application
     metadata:
       name: application-sample-v2
       namespace: k8s-learn
       labels:
         app: application
     spec:
       workflow:
         replicas: 11
         selector:
           matchLabels:
             app: application
         template:
           metadata:
             labels:
               app: application
           spec:
             containers:
               - name: nginx
                 image: nginx:1.14.2
                 ports:
                   - containerPort: 80
       service:
         type: NodePort
         ports:
           - port: 80
             targetPort: 80
             nodePort: 30080
     ~~~

* **问题1：**部署之后仍然在对应的namespace查询不到对应的Pod和Deployment

  **当引入多个版本的API时，Controller必须以Hub存储版本为中心进行调谐；因为用户可以提交任意版本，但Kube-apiserver总是将数据转换为Hub存储版本，然后存储；**

  **Reconcile永远从Hub读取对象进行实际业务逻辑，例如创建Deployment/Service**

* **问题2：**两个版本都设置了default/validation Webhook，但是为什么即使两个版本的replicas为11，也没有报错，但是Default Webhook却生效了，可以查询到Replicas为3

  ~~~sh
  (base) root@master:~/Golang/code/go/src/sigs.k8s.io# kubectl describe app application-sample-v1 -n k8s-learn
  Name:         application-sample-v1
  Namespace:    k8s-learn
  Labels:       app=application
  Annotations:  <none>
  API Version:  apps.wuyong.cn/v2
  Kind:         Application
  Metadata:
    Creation Timestamp:  2025-10-17T10:07:57Z
    Generation:          1
    Resource Version:    30510434
    UID:                 fdf477c9-9966-4ec2-ad09-76b3df47c3d6
  Spec:
    Service:
    Workflow:
      Spec:
        Replicas:  3
        Selector:  <nil>
        Strategy:
        Template:
          Metadata:
            Creation Timestamp:  <nil>
          Spec:
            Containers:  <nil>
  Events:                <none>
  ~~~

  虽然我们在v1和v2版本下都注册了Webhook，但是这会导致两个问题：

  1. 双版本Webhook都生效，但是实际上只会触发一个版本
  2. 默认值和验证可能在conversion后被覆盖

  Webhook调用的总体流程如下表

  | 阶段               | 发生位置                          | 示例                            |
  | ------------------ | --------------------------------- | ------------------------------- |
  | 用户提交           | v1 YAML                           | `apiVersion: apps.wuyong.cn/v1` |
  | Conversion         | apiserver 调用 conversion webhook | 转换为 v2                       |
  | Mutating webhook   | 调用 v2 default webhook           | 设置 replicas=3（如为空）       |
  | Validating webhook | 调用 v2 validate webhook          | 校验 replicas ≤ 10              |
  | 存储               | etcd                              | 存储为 v2 对象                  |
  | Reconcile          | controller 使用 v2                | 创建 Deployment、Service        |

  但是，即使这样也无法解决一个问题：

  **我提交的v1版本的deployment.replicas并不为空，并且你也看到我的conversion代码，其中ConvertTo函数将v1版本转换为v2版本，其中的dst.Spec.Workflow=src.Spec.Deployment应该是保证了我提交的replicas=11并没有丢失的吧？那么在调用v2版本的default Webhook时，检查Workflow.Spec.Replicas应该不为空呀，因此也不应该被改为3呀，那么在后面调用v2版本的validate Webhook时，应该会触发检查呀？**

  原因在于，由于多版本的原因，我们使用了`shared`包，而改编之后的DeploymentTemplate和ServiceTemplate新增了Spec字段作为引用

  而yaml中定义的replicas

  ~~~yaml
  deployment:
      replicas: 11
  ---
  workflow:
      replicas: 11
  ~~~

  并没有被录入新版本的Spec中，按理来说，应该是这样，replicas才能被录入代码中

  ~~~yaml
  deployment:
  	spec:
      	replicas: 11
  ---
  workflow:
  	spec:
      	replicas: 11
  ~~~

  **因此，实际上在反序列化时，`.spec.deployment/workflow.spec`其实是空的，因此在执行v2的defaul Webhook时，设置默认值的Webhook起效了，设置为了3；导致后面的validation Webhook通过了验证**

  

* 验证我们的conversion Webhook起了作用

  注意，如果用`kubectl describe app`来apply的v1版本的application资源的话，Kubernetes默认还是读取的存储版本v2，所以看到的不会是deployment而是workflow

  正确的方法应该是：

  ~~~sh
  (base) root@master:~/Golang/code/go/src/sigs.k8s.io# kubectl get applications.v1.apps.wuyong.cn application-sample-v1 -n k8s-learn -o yaml
  apiVersion: apps.wuyong.cn/v1
  kind: Application
  metadata:
    annotations:
      kubectl.kubernetes.io/last-applied-configuration: |
        {"apiVersion":"apps.wuyong.cn/v1","kind":"Application","metadata":{"annotations":{},"labels":{"app":"application"},"name":"application-sample-v1","namespace":"k8s-learn"},"spec":{"deployment":{"replicas":11,"selector":{"matchLabels":{"app":"application"}},"template":{"metadata":{"labels":{"app":"application"}},"spec":{"containers":[{"image":"nginx:1.14.2","name":"nginx","ports":[{"containerPort":80}]}]}}},"service":{"ports":[{"nodePort":30080,"port":80,"targetPort":80}],"type":"NodePort"}}}
    creationTimestamp: "2025-10-17T10:07:57Z"
    generation: 1
    labels:
      app: application
    name: application-sample-v1
    namespace: k8s-learn
    resourceVersion: "30510434"
    uid: fdf477c9-9966-4ec2-ad09-76b3df47c3d6
  spec:
    deployment:
      Spec:
        replicas: 3
        selector: null
        strategy: {}
        template:
          metadata:
            creationTimestamp: null
          spec:
            containers: null
    service: {}
  ~~~

  可以看到，此时展现的就是v1版本的deployment了；顺便也可以看到，系统中实际存储的yaml结构是`.spec.deployment.Spec.replicas`而不是`.spec.deployment.replicas`
  
* **问题3：**在apply v2版本的资源之后，触发的却是v1版本的validation Webhook；这是为什么呢？

  在同时注册了v1和v2版本的default/validation Webhook之后，其实都会被调用；不过顺序是：

  1. 在mutating Admission Phase变更阶段：

     执行所有MutatingWebhookConfiguration中匹配到的Webhook；按Webhook名字字典序依次执行，每个Webhook修改对象完成变更之后，才会执行validating阶段

     ~~~sh
     kubectl get mutatingwebhookconfigurations
     ~~~

  2. 在Validating Admission Phase验证阶段：

     执行所有ValidatingWebhookConfiguration中匹配到的Webhook；按Webhook名字字典序依次执行，每个Webhook都会被触发

     ~~~sh
     kubectl get validatingwebhookconfigurations
     ~~~

  **由于Kubernetes的CRD多版本兼容机制，并且我们没有设置Webhook的MatchPolicy，使得其默认是Equivalent，这就使得v1的Webhook认为v2与v1是同一个资源**

  后来我们发现，我们只是针对v2版本编写了default/validation Webhook，但是并没有将其注册到集群中，使得实际上只有v1的Webhook在运行；

  我们最后使得v1和v2版本的Webhook都运行，但是v1版本Webhook只拦截v1版本资源，v2版本Webhook只拦截v2版本资源

  1. 首先在v2版本Webhook上添加注释

     ~~~go
     // TODO(user): EDIT THIS FILE!  THIS IS SCAFFOLDING FOR YOU TO OWN!
     // +kubebuilder:webhook:path=/mutate-apps-wuyong-cn-v2-application,mutating=true,failurePolicy=fail,sideEffects=None,groups=apps.wuyong.cn,resources=applications,verbs=create;update,versions=v2,name=mapplication-v2.kb.io,admissionReviewVersions=v1,matchPolicy=Exact
     
     type ApplicationCustomDefaulter struct {
     	DefaultDeploymentReplicas int32
     }
     
     // validation
     // +kubebuilder:webhook:path=/validate-apps-wuyong-cn-v2-application,mutating=false,failurePolicy=fail,sideEffects=None,groups=apps.wuyong.cn,resources=applications,verbs=create;update,versions=v2,name=vapplication-v2.kb.io,admissionReviewVersions=v1,matchPolicy=Exact
     
     type ApplicationCustomValidator struct {
     	DefaultDeploymentReplicasMax int32
     }
     ~~~

     这里给出该注释的释义：

     | 参数                           | 类型           | 示例值                                  | 说明                                                         |
     | ------------------------------ | -------------- | --------------------------------------- | ------------------------------------------------------------ |
     | **path**                       | string         | `/mutate-apps-wuyong-cn-v1-application` | Webhook 的 HTTP 路径（即 controller manager 暴露的接口路径）。K8s APIServer 调用 Webhook 时会访问这个路径。应当唯一。格式建议：`/mutate-<group>-<version>-<resource>` 或 `/validate-...`。 |
     | **mutating**                   | bool           | `true` / `false`                        | 指定 Webhook 类型：• `true` → MutatingWebhook（默认注入默认值、修正资源）• `false` → ValidatingWebhook（仅校验资源合法性） |
     | **failurePolicy**              | string         | `ignore` / `fail`                       | 当 Webhook 无法访问或返回错误时的策略：• `Fail`：阻止请求，推荐生产使用。• `Ignore`：放行请求（更容错，但有风险）。 |
     | **sideEffects**                | string         | `None` / `NoneOnDryRun`                 | 表示 Webhook 是否有副作用。• `None`：无副作用（推荐）• `NoneOnDryRun`：支持 DryRun（只读请求） |
     | **groups**                     | string         | `apps.wuyong.cn`                        | 对应 CRD 的 API Group。必须与 CRD 一致（例如：`apps.wuyong.cn/v1`）。 |
     | **resources**                  | string         | `applications`                          | 指定该 Webhook 拦截的资源（通常是 CRD 的复数形式）。         |
     | **verbs**                      | string         | `create;update`                         | 指定触发 Webhook 的操作类型：`create`、`update`、`delete`、`connect`。多个用分号分隔。 |
     | **versions**                   | string         | `v1` / `v2`                             | 指定该 Webhook 拦截的资源版本（对应 CRD 版本）。注意不同版本的 Webhook 必须区分。 |
     | **name**                       | string         | `mapplication-v1.kb.io`                 | Webhook 的唯一标识符，必须全局唯一（通常推荐格式为 `[m/v] + resource + - + version + .kb.io`）。 |
     | **admissionReviewVersions**    | string         | `v1`                                    | 指定 Kubernetes AdmissionReview API 版本（几乎总是 `v1`）。  |
     | **matchPolicy** *(可选)*       | string         | `Exact` / `Equivalent`                  | 控制匹配策略：`Exact`：只匹配指定的 group/version/resource。`Equivalent`：允许匹配等价资源版本。默认是 `Equivalent`。 |
     | **timeoutSeconds** *(可选)*    | int            | `10`                                    | 设置 webhook 请求超时时间（默认 10s）。                      |
     | **objectSelector** *(可选)*    | label selector | N/A                                     | 可选字段，用于限定 webhook 仅对特定 label 的对象生效。       |
     | **namespaceSelector** *(可选)* | label selector | N/A                                     | 限定仅作用于特定 namespace 中的对象。                        |

  2. 执行命令

     ~~~sh
     make manifests
     ~~~

     最后，查看Webhook配置文件`config/webhook/manifests.yaml`

     ~~~yaml
     ---
     apiVersion: admissionregistration.k8s.io/v1
     kind: MutatingWebhookConfiguration
     metadata:
       name: mutating-webhook-configuration
     webhooks:
     - admissionReviewVersions:
       - v1
       clientConfig:
         service:
           name: webhook-service
           namespace: system
           path: /mutate-apps-wuyong-cn-v1-application
       failurePolicy: Fail
       matchPolicy: Exact
       name: mapplication-v1.kb.io
       rules:
       - apiGroups:
         - apps.wuyong.cn
         apiVersions:
         - v1
         operations:
         - CREATE
         - UPDATE
         resources:
         - applications
       sideEffects: None
     - admissionReviewVersions:
       - v1
       clientConfig:
         service:
           name: webhook-service
           namespace: system
           path: /mutate-apps-wuyong-cn-v2-application
       failurePolicy: Fail
       matchPolicy: Exact
       name: mapplication-v2.kb.io
       rules:
       - apiGroups:
         - apps.wuyong.cn
         apiVersions:
         - v2
         operations:
         - CREATE
         - UPDATE
         resources:
         - applications
       sideEffects: None
     ---
     apiVersion: admissionregistration.k8s.io/v1
     kind: ValidatingWebhookConfiguration
     metadata:
       name: validating-webhook-configuration
     webhooks:
     - admissionReviewVersions:
       - v1
       clientConfig:
         service:
           name: webhook-service
           namespace: system
           path: /validate-apps-wuyong-cn-v1-application
       failurePolicy: Fail
       matchPolicy: Exact
       name: vapplication-v1.kb.io
       rules:
       - apiGroups:
         - apps.wuyong.cn
         apiVersions:
         - v1
         operations:
         - CREATE
         - UPDATE
         resources:
         - applications
       sideEffects: None
     - admissionReviewVersions:
       - v1
       clientConfig:
         service:
           name: webhook-service
           namespace: system
           path: /validate-apps-wuyong-cn-v2-application
       failurePolicy: Fail
       matchPolicy: Exact
       name: vapplication-v2.kb.io
       rules:
       - apiGroups:
         - apps.wuyong.cn
         apiVersions:
         - v2
         operations:
         - CREATE
         - UPDATE
         resources:
         - applications
       sideEffects: None
     ~~~

     可以看到，不管是default Webhook还是validation Webhook，两个版本都有配置，并且matchPolicy变为了Exact；接下来就是测试结果

     ~~~sh
     (base) root@master:~/Golang/code/go/src/sigs.k8s.io/MyOperatorProjects/application-operator-plus/test-yaml# kubectl apply -f apps_v1_application.yaml 
     Warning: Application Webhook v1 Errors!
     Error from server (Forbidden): error when creating "apps_v1_application.yaml": admission webhook "vapplication-v1.kb.io" denied the request: replicas too many error
     (base) root@master:~/Golang/code/go/src/sigs.k8s.io/MyOperatorProjects/application-operator-plus/test-yaml# kubectl apply -f apps_v2_application.yaml 
     Warning: Application Webhook v2 Errors!
     Error from server (Forbidden): error when creating "apps_v2_application.yaml": admission webhook "vapplication-v2.kb.io" denied the request: replicas too many error
     ~~~

     成功实现各版本Webhook拦截各自版本资源

     

## 7.7 API分组支持

> 有时候会在一个operator项目中实现多个控制器来管理不同的API资源组
>
> 比如如果要实现一个ai-operator项目，其中可能包含模型训练相关的控制器trainjob-controller和推理服务相关的控制器application-controller
>
> 需要将API分别放在apps组中和batch组中

* 使用命令变更为多组模式

  ~~~sh
  kubebuilder edit --multigroup=true
  ~~~

  但是仅有这个命令并不行，还需要我们进行很多手动操作

  这个命令仅仅将PROJECT文件中添加了`multigroup=true`

  ~~~
  layout:
  - go.kubebuilder.io/v4
  multigroup: true
  ~~~

* 变更目录：

  参考[官方文档](https://book.kubebuilder.io/migration/multi-group)中的关于v4版本的kubebuilder应该如何进行目录变更；以本项目为例

  ~~~sh
  mkdir api/apps
  mv api/* api/apps
  ~~~

  ~~~sh
  mkdir internal/controller/apps
  mv internal/controller/* internal/controller/apps
  ~~~

  ~~~sh
  mkdir internal/webhook/apps
  mv internal/webhook/* internal/webhook/apps
  ~~~

  这样就完成了目录变更

* 变更各个代码和文件中依赖的路径

  1. main.go

     ~~~go
     	appsv1 "github.com/wuyong7240/application-operator-plus/api/apps/v1"
     	appsv2 "github.com/wuyong7240/application-operator-plus/api/apps/v2"
     	controller "github.com/wuyong7240/application-operator-plus/internal/controller/apps"
     	webhookv1 "github.com/wuyong7240/application-operator-plus/internal/webhook/apps/v1"
     	webhookappsv2 "github.com/wuyong7240/application-operator-plus/internal/webhook/apps/v2"
     ~~~

  2. internal/controller/apps/application_controller.go

     ~~~go
     	v1 "github.com/wuyong7240/application-operator-plus/api/apps/v1"
     	v2 "github.com/wuyong7240/application-operator-plus/api/apps/v2"
     ~~~

  3. internal/webhook/apps/v1/application_webhook.go

     ~~~go
     	appsv1 "github.com/wuyong7240/application-operator-plus/api/apps/v1"
     ~~~

  4. internal/webhook/apps/v2/application_webhook.go

     ~~~go
     	appsv2 "github.com/wuyong7240/application-operator-plus/api/apps/v2"
     ~~~

  5. PROJECT

     ~~~PROJECT
     cliVersion: 4.8.0
     domain: wuyong.cn
     layout:
     - go.kubebuilder.io/v4
     multigroup: true
     projectName: application-operator-plus
     repo: github.com/wuyong7240/application-operator-plus
     resources:
     - api:
         crdVersion: v1
         namespaced: true
       controller: true
       domain: wuyong.cn
       group: apps
       kind: Application
       path: github.com/wuyong7240/application-operator-plus/api/apps/v1
       version: v1
       webhooks:
         defaulting: true
         validation: true
         webhookVersion: v1
     - api:
         crdVersion: v1
         namespaced: true
       domain: wuyong.cn
       group: apps
       kind: Application
       path: github.com/wuyong7240/application-operator-plus/api/apps/v2
       version: v2
       webhooks:
         conversion: true
         spoke:
         - v1
         webhookVersion: v1
     version: "3"
     ~~~






# Chapter 8：Deployment源码分析

# Chapter 9:使用Kustomize管理配置

> * Kustmoize基本概念
>
>   1. kustomization：指代一个kustomization.yaml文件，广义的是包含kustomization.yaml文件的目录及这个kustomization.yaml文件中引用的其他所有文件
>
>   2. base：指的是被其他kustomization引用的kustomization，是一个相对的概念
>
>   3. overlay：与base对应，指的是是依赖另一个kustomization的kustomization，是一个相对的概念
>
>      例如kustomization b引用了kustomization a，那么b就是a的overlay，a是b的base
>
>   4. 示例
>
>      ~~~sh
>      /myapp
>      ├── base
>      │ 	├── kustomization.yaml
>      │ 	├── nginx-deployment.yaml
>      │ 	└── nginx-service.yaml
>      └── overlays
>      	├── dev
>      	│ 	├── kustomization.yaml
>      	│ 	└── replica.yaml
>      	└── prod
>      		├── kustomization.yaml
>      		└── replica.yaml
>      ~~~
>
>      * base/kustomization.yaml
>
>        ~~~yaml
>        resource:
>        - nginx-deployment.yaml
>        - nginx-service.yaml
>        ~~~
>
>      * overlays/dev/kustomization.yaml、overlays/prod/kustomization.yaml
>
>        ~~~yaml
>        bases:
>        - ../../base
>        patchs:
>        - replica.yaml
>        ~~~
>
> * kustomize安装
>
>   ~~~sh
>   cd /usr/local/bin
>   curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh"  | bash
>   ~~~
>
>   其实kubectl也内置了kustomize，使用命令kubectl kustomize即可

## 9.3 使用kustomize生成资源

### 9.3.1 ConfigMap生成器

* 从配置文件生成ConfigMap

  事先准备好的两个文件：

  1. config.txt

     ~~~txt
     key=value
     ~~~

  2. kustomization.yaml

     ~~~yaml
     configMapGenerator:
     - name: app-config
       files:
       - config.txt
     ~~~

  ~~~sh
  (base) root@master:~/Golang/code/go/src/sigs.k8s.io/kustomize-examples/ConfigMapWithConfigFiles# kustomize build .
  apiVersion: v1
  data:
    config.txt: |
      key=value
  kind: ConfigMap
  metadata:
    name: app-config-gc6cm9fg4c
  (base) root@master:~/Golang/code/go/src/sigs.k8s.io/kustomize-examples/ConfigMapWithConfigFiles# kubectl kustomize .
  apiVersion: v1
  data:
    config.txt: |
      key=value
  kind: ConfigMap
  metadata:
    name: app-config-gc6cm9fg4c
  ~~~

  可以看到，成功生成了ConfigMap配置文件内容

* 从环境变量生成ConfigMap

  事先准备好的文件

  1. golang_env.txt

     ~~~txt
     GOVERSION=go1.24.5
     GOARCH
     ~~~

  2. kustomization.yaml

     ~~~yaml
     configMapGenerator:
     - name: app-config
       envs:
       - golang_env.txt
     ~~~

  ~~~sh
  (base) root@master:~/Golang/code/go/src/sigs.k8s.io/kustomize-examples/ConfigMapWithENV# kustomize build .
  apiVersion: v1
  data:
    GOARCH: ""
    GOVERSION: go1.24.5
  kind: ConfigMap
  metadata:
    name: app-config-cccbt2ch2c
  ~~~

  可以看到，虽然依赖文件都是.txt，但是生成的ConfigMap内容的格式是不一样的

* 从键值对字面值直接创建ConfigMap

  事先准备的文件：

  kustomization.yaml

  ~~~yaml
  configMapGenerator:
  - name: app-config
    literals:
    - Hello=world
  ~~~

  ~~~sh
  (base) root@master:~/Golang/code/go/src/sigs.k8s.io/kustomize-examples/ConfigMapWithKeyValue# kustomize build .
  apiVersion: v1
  data:
    Hello: world
  kind: ConfigMap
  metadata:
    name: app-config-m8mkk9748h
  ~~~

* 使用ConfigMap

  其实可以看到，每次生成的ConfigMap后面都带了一串后缀，如何在Deployment中引用这个ConfigMap时预知这个名字呢？

  事先准备文件

  1. config.txt

     ~~~txt
     key=value
     ~~~

  2. nginx-deployment.yaml

     ~~~yaml
     apiVersion: apps/v1
     kind: Deployment
     metadata:
       name: nginx
       labels:
         app: nginx
     spec:
       selector:
         matchLabels:
           app: nginx
       template:
         metadata:
           labels:
             app: nginx
         spec:
           containers:
           - name: nginx
             image: nginx:latest
             volumeMounts:
             - name: config
               mountPath: /config
           volumes:
           - name: config
             configMap:
               name: app-config
     ~~~

  3. kustomization.yaml

     ~~~yaml
     resources:
     - nginx-deployment.yaml
     configMapGenerator:
     - name: app-config
       files:
       - config.txt
     ~~~

  ~~~sh
  (base) root@master:~/Golang/code/go/src/sigs.k8s.io/kustomize-examples/ConfigMapUseExample# kustomize build .
  apiVersion: v1
  data:
    config.txt: |
      key=value
  kind: ConfigMap
  metadata:
    name: app-config-gc6cm9fg4c
  ---
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    labels:
      app: nginx
    name: nginx
  spec:
    selector:
      matchLabels:
        app: nginx
    template:
      metadata:
        labels:
          app: nginx
      spec:
        containers:
        - image: nginx:latest
          name: nginx
          volumeMounts:
          - mountPath: /config
            name: config
        volumes:
        - configMap:
            name: app-config-gc6cm9fg4c
          name: config
  ~~~

  可以看到，生成的ConfigMap名称在Deployment配置中同步替换了



### 9.3.2 Secret生成器

* 从配置文件生成Secret

  事先准备文件：

  1. password.txt

     ~~~txt
     username=wuyong
     password=1234
     ~~~

  2. kustomization.yaml

     ~~~yaml
     secretGenerator:
     - name: app-secret
       files:
       - password.txt
     ~~~

  ~~~sh
  (base) root@master:~/Golang/code/go/src/sigs.k8s.io/kustomize-examples/Secret/SecretWithConfigFiles# kustomize build .
  apiVersion: v1
  data:
    password.txt: dXNlcm5hbWU9d3V5b25nCnBhc3N3b3JkPTEyMzQ=
  kind: Secret
  metadata:
    name: app-secret-727k58hmgf
  type: Opaque
  ~~~

  通过解码其实可以看到那一串就是我们的用户密码

  ~~~sh
  (base) root@master:~/Golang/code/go/src/sigs.k8s.io/kustomize-examples/Secret/SecretWithConfigFiles# echo "dXNlcm5hbWU9d3V5b25nCnBhc3N3b3JkPTEyMzQ=" | base64 -d
  username=wuyong
  password=1234
  ~~~

* 通过字面值创建Secret

  事先准备文件：

  kustomization.yaml

  ~~~yaml
  secretGenerator:
  - name: app-secret
    literals:
    - username=wuyong
    - password=1234
  ~~~

  ~~~sh
  (base) root@master:~/Golang/code/go/src/sigs.k8s.io/kustomize-examples/Secret/SecretWithKeyValue# kustomize build .
  apiVersion: v1
  data:
    password: MTIzNA==
    username: d3V5b25n
  kind: Secret
  metadata:
    name: app-secret-6b55hb79mm
  type: Opaque
  ~~~

* 使用Secret

  事先准备文件

  1. password.txt

     ~~~txt
     username=wuyong
     password=1234
     ~~~

  2. nginx-deployment.yaml

     ~~~yaml
     apiVersion: apps/v1
     kind: Deployment
     metadata:
       name: nginx
       labels:
         app: nginx
     spec:
       selector:
         matchLabels:
           app: nginx
       template:
         metadata:
           labels:
             app: nginx
         spec:
           containers:
           - name: nginx
             image: nginx:latest
             volumeMounts:
             - name: password
               mountPath: /secrets
           volumes:
           - name: password
             secret:
               secretName: app-secret
     ~~~

  3. kustomization.yaml

     ~~~yaml
     resources:
     - nginx-deployment.yaml
     secretGenerator:
     - name: app-secret
       files:
       - password.txt
     ~~~

  ~~~sh
  (base) root@master:~/Golang/code/go/src/sigs.k8s.io/kustomize-examples/Secret/SecretUseExample# kustomize build .
  apiVersion: v1
  data:
    password.txt: dXNlcm5hbWU9d3V5b25nCnBhc3N3b3JkPTEyMzQ=
  kind: Secret
  metadata:
    name: app-secret-727k58hmgf
  type: Opaque
  ---
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    labels:
      app: nginx
    name: nginx
  spec:
    selector:
      matchLabels:
        app: nginx
    template:
      metadata:
        labels:
          app: nginx
      spec:
        containers:
        - image: nginx:latest
          name: nginx
          volumeMounts:
          - mountPath: /secrets
            name: password
        volumes:
        - name: password
          secret:
            secretName: app-secret-727k58hmgf
  ~~~



### 9.3.3 使用generatorOptions改变默认行为

* 如果读者不想生成的资源配置名字后有随机字符串，可以通过generatorOptions改变这种行为

  示例yaml

  ~~~yaml
  configMapGenerator:
  - name: app-config
    literals:
    - Hello=world
  generatorOptions:
    disableNameSuffixHash: true
    labels:
      type: generated
    annotations:
      note: generated
  ~~~

  ~~~sh
  (base) root@master:~/Golang/code/go/src/sigs.k8s.io/kustomize-examples/generatorOptions# kustomize build .
  apiVersion: v1
  data:
    Hello: world
  kind: ConfigMap
  metadata:
    annotations:
      note: generated
    labels:
      type: generated
    name: app-config
  ~~~



### 9.4 使用Kustomize管理公共配置项

* 由于有时经常需要在不同的资源配置文件中配置相同的字段，例如namespace、前后缀、labels、annotations

  kustomize也能管理这些

  1. nginx-deployment.yaml

     ~~~yaml
     apiVersion: apps/v1
     kind: Deployment
     metadata:
       name: nginx
       labels:
         app: nginx
     spec:
       selector:
         matchLabels:
           app: nginx
       template:
         metadata:
           labels:
             app: nginx
         spec:
           containers:
           - name: nginx
             image: nginx:latest
     ~~~

  2. kustomization.yaml

     ~~~yaml
     namespace: k8s-learn
     namePrefix: kustomize-
     nameSuffix: -v1
     commonLabels:
       version: v1
     commonAnnotations:
       owner: wuyong
     resources:
     - nginx-deployment.yaml
     ~~~

  ~~~sh
  (base) root@master:~/Golang/code/go/src/sigs.k8s.io/kustomize-examples/CommonConfig# kustomize build .
  # Warning: 'commonLabels' is deprecated. Please use 'labels' instead. Run 'kustomize edit fix' to update your Kustomization automatically.
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    annotations:
      owner: wuyong
    labels:
      app: nginx
      version: v1
    name: kustomize-nginx-v1
    namespace: k8s-learn
  spec:
    selector:
      matchLabels:
        app: nginx
        version: v1
    template:
      metadata:
        annotations:
          owner: wuyong
        labels:
          app: nginx
          version: v1
      spec:
        containers:
        - image: nginx:latest
          name: nginx
  ~~~

  

## 9.5 使用Kustomize组合资源

### 9.5.1 多个资源的组合

* 当在Kubernetes上部署一个应用时，需要用到多个资源类型配置，分别放在不同配置文件中，使用kustomize可以组合多个资源类型配置

  假设现在有nginx-deployment.yaml和nginx-service.yaml两个配置，使用如下kustomization.yaml即可组合

  ~~~yaml
  resources:
  - nginx-deployment.yaml
  - nginx-service.yaml
  ~~~

  然后再使用`kustomize build .`即可

### 9.5.2 给资源配置打补丁

> 需要给同一个资源针对不同使用场景配置不同的配置项，例如nginx应用在开发中我们给定100MB内存，但在生产环境中我们给定1GB
>
> kutomize可以通过给一个资源打不同的补丁实现多环境配置灵活管理

* patchesStrategicMerge方式自定义配置

  需要注意的是，补丁文件中描述的需要时同一个资源对象才可以；并且每个patch都实现一个明确的小功能，比如设置Qos资源是一个单独的补丁，设置亲和性策略是一个单独的补丁等等

  1. 普通的deployment配置
  
     ~~~yaml
     apiVersion: apps/v1
     kind: Deployment
     metadata:
       name: nginx
       labels:
         app: nginx
     spec:
       selector:
         matchLabels:
           app: nginx
       replicas: 3
       template:
         metadata:
           labels:
             app: nginx
         spec:
           containers:
           - name: nginx
             image: nginx:latest
             ports:
             - containerPort: 80
     ~~~
  
  2. 单独将内存配置放到一个新的文件中:nginx-memory.yaml
  
     ~~~yaml
     apiVersion: apps/v1
     kind: Deployment
     metadata:
       name: nginx
     spec:
       template:
         spec:
           containers:
           - name: nginx
             resources:
               limits:
                 memory: 100Mi
     ~~~
  
  3. kustomization.yaml
  
     ~~~yaml
     resources:
     - nginx-deployment.yaml
     patchesStrategicMerge:
     - nginx-memory.yaml
     ~~~
  
  ~~~sh
  (base) root@master:~/Golang/code/go/src/sigs.k8s.io/kustomize-examples/PatchesMerge/patchesStrategicMerge# kustomize build .
  # Warning: 'patchesStrategicMerge' is deprecated. Please use 'patches' instead. Run 'kustomize edit fix' to update your Kustomization automatically.
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    labels:
      app: nginx
    name: nginx
  spec:
    replicas: 3
    selector:
      matchLabels:
        app: nginx
    template:
      metadata:
        labels:
          app: nginx
      spec:
        containers:
        - image: nginx:latest
          name: nginx
          ports:
          - containerPort: 80
          resources:
            limits:
              memory: 100Mi
  ~~~
  
  > 其实通过输出可以发现，这里的用法已经被抛弃了，但是仍然可以使用；
  >
  > 如果要使用新的命令，可以使用`kustomize edit fix`更新新版本的kustomization.yaml
  >
  > ~~~yaml
  > resources:
  > - nginx-deployment.yaml
  > apiVersion: kustomize.config.k8s.io/v1beta1
  > kind: Kustomization
  > patches:
  > - path: nginx-memory.yaml
  > ~~~
  >
  > 然后使用`kustomize build .`也是一样的
  
* patchesJson6902方式自定义配置

  1. nginx-deployment.yaml如上

  2. patch.yaml

     ~~~yaml
     - op: replace
       path: /spec/replicas
       value: 1
     ~~~

  3. kustomization.yaml

     ~~~yaml
     resources:
     - nginx-deployment.yaml
     
     patchesJson6902:
     - target:
         group: apps
         version: v1
         kind: Deployment
         name: nginx
       path: patch.yaml
     ~~~

  ~~~sh
  (base) root@master:~/Golang/code/go/src/sigs.k8s.io/kustomize-examples/PatchesMerge/patchesJson6902# kustomize build .
  # Warning: 'patchesJson6902' is deprecated. Please use 'patches' instead. Run 'kustomize edit fix' to update your Kustomization automatically.
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    labels:
      app: nginx
    name: nginx
  spec:
    replicas: 1
    selector:
      matchLabels:
        app: nginx
    template:
      metadata:
        labels:
          app: nginx
      spec:
        containers:
        - image: nginx:latest
          name: nginx
          ports:
          - containerPort: 80
  ~~~

* 镜像自定义

  可以直接在kustomization.yaml中国使用images配置来指定镜像

  这里省略nginx-deployment.yaml

  kustomization.yaml

  ~~~yaml
  resources:
  - nginx-deployment.yaml
  images:
  - name: nginx
    newName: nginx
    newTag: 1.16.1
  ~~~

* 容器内使用其他资源对象的配置

  如果一个容器化应用需要知道某个Service的名字，而这个Service由Kustomize生成，会有一些前后缀，如何动态获取Service的名字用于配置？

  1. nginx-deployment.yaml

     ~~~yaml
     apiVersion: apps/v1
     kind: Deployment
     metadata:
       name: nginx
       labels:
         app: nginx
     spec:
       selector:
         matchLabels:
           app: nginx
       replicas: 3
       template:
         metadata:
           labels:
             app: nginx
         spec:
           containers:
           - name: nginx
             image: nginx:latest
             command: ["start", "--host", "$(SERVICE_NAME)"]
             ports:
             - containerPort: 80
     ~~~

     其实重点就在于`command`部分了

  2. nginx-service.yaml

     ~~~yaml
     apiVersion: v1
     kind: Service
     metadata:
       name: nginx-service
       labels:
         app: nginx-service
     spec:
       ports:
       - port: 80
         protocol: TCP
       selector:
         app: nginx
     ~~~

  3. kustomization.yaml

     ~~~yaml
     namePrefix: dev-
     
     resources:
     - nginx-deployment.yaml
     - nginx-service.yaml
     
     vars:
     - name: SERVICE_NAME
       objref:
         kind: Service
         name: nginx-service
         apiVersion: v1
     ~~~

     这里给两个资源都加了前缀，然后通过vars来定义SERVICE_NAME变量，该变量通过objref内的几个配置项和上面的Service关联

  ~~~sh
  (base) root@master:~/Golang/code/go/src/sigs.k8s.io/kustomize-examples/PatchesMerge/containerResourceUse# kustomize build .
  # Warning: 'vars' is deprecated. Please use 'replacements' instead. [EXPERIMENTAL] Run 'kustomize edit fix' to update your Kustomization automatically.
  apiVersion: v1
  kind: Service
  metadata:
    labels:
      app: nginx-service
    name: dev-nginx-service
  spec:
    ports:
    - port: 80
      protocol: TCP
    selector:
      app: nginx
  ---
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    labels:
      app: nginx
    name: dev-nginx
  spec:
    replicas: 3
    selector:
      matchLabels:
        app: nginx
    template:
      metadata:
        labels:
          app: nginx
      spec:
        containers:
        - command:
          - start
          - --host
          - dev-nginx-service
          image: nginx:latest
          name: nginx
          ports:
          - containerPort: 80
  ~~~





# Chapter 10：使用Helm打包应用

