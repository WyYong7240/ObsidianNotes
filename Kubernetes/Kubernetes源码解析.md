---
tags:
  - Kubernetes
  - client-go
  - Informer
---

## Informer机制

### Informer机制架构设计

在Informer架构设计中，有多个核心组件，分别介绍如下

1. Reflector

   Reflector用于监控（Watch）指定的Kubernetes资源，当监控的资源发生变化时，触发相应的变更事件，例如Added（资源添加）事件、Updated（资源更新）事件、Deleted（资源删除）事件，并将其资源对象存放到本地缓存DeltaFIFO中。

2. DeltaFIFO

   DeltaFIFO可以分开理解，FIFO是一个先进先出的队列，它拥有队列操作的基本方法，例如Add、Update、Delete、List、Pop、Close等，而Delta是一个资源对象存储，它可以保存资源对象的操作类型，例如Added（添加）操作类型、Updated（更新）操作类型、Deleted（删除）操作类型、Sync（同步）操作类型等。

3. Indexer

   Indexer是client-go用来存储资源对象并自带索引功能的本地存储，Reflector从DeltaFIFO中将消费出来的资源对象存储至Indexer。Indexer与Etcd集群中的数据完全保持一致。client-go可以很方便地从本地存储中读取相应的资源对象数据，而无须每次从远程Etcd集群中读取，以减轻Kubernetes API Server和Etcd集群的压力。

直接阅读Informer机制代码会比较晦涩，通过Informers Example代码示例来理解Informer，印象会更深刻。Informers Example代码示例如下：

~~~go
~~~

1. 首先通过`kubernetes.NewForConfig`创建clientset对象，**Informer需要通过ClientSet与Kubernetes API Server进行交互**。另外，创建`stopCh`对象，该对象用于**在程序进程退出之前通知Informer提前退出，因为Informer是一个持久运行的goroutine。**

2. `informers.NewSharedInformerFactory`函数实例化了SharedInformer对象

   它接收两个参数：**第1个参数clientset是用于与Kubernetes API Server交互的客户端**，

   **第2个参数time.Minute用于设置多久进行一次resync（重新同步），resync会周期性地执行List操作，将所有的资源存放在Informer Store中，如果该参数为0，则禁用resync功能。**

3. 在Informers Example代码示例中，通过`sharedInformers.Core().V1().Pods().Informer`可以得到具体Pod资源的informer对象。通过informer.AddEventHandler函数可以为Pod资源添加资源事件回调方法，支持3种资源事件回调方法，分别介绍如下。

   * AddFunc:当创建Pod资源对象时触发的事件回调方法。
   * UpdateFunc:当更新Pod资源对象时触发的事件回调方法。
   * DeleteFunc:当删除Pod资源对象时触发的事件回调方法。

> 通过Informer机制可以很容易地监控我们所关心的资源事件，例如，当监控Kubernetes Pod资源时，如果Pod资源发生了Added（资源添加）事件、Updated（资源更新）事件、Deleted（资源删除）事件，就通知client-go，告知Kubernetes资源事件变更了并且需要进行相应的处理。

#### 资源Informer

每一个Kubernetes资源上都实现了Informer机制。每一个Informer上都会实现Informer和Lister方法，例如PodInformer，代码示例如下：

调用不同资源的Informer，代码示例如下：

定义不同资源的Informer，允许监控不同资源的资源事件，例如，监听Node资源对象，当Kubernetes集群中有新的节点（Node）加入时，client-go能够及时收到资源对象的变更信息。

#### Shared Informer共享机制

Informer也被称为Shared Informer，它是可以共享使用的。在用client-go编写代码程序时，若同一资源的Informer被实例化了多次，每个Informer使用一个Reflector，那么会运行过多相同的ListAndWatch，太多重复的序列化和反序列化操作会导致Kubernetes API Server负载过重。

Shared Informer可以使同一类资源Informer共享一个Reflector，这样可以节约很多资源。通过map数据结构实现共享的Informer机制。Shared Informer定义了一个map数据结构，用于存放所有Informer的字段，代码示例如下：

### Reflector

Informer可以对Kubernetes API Server的资源执行监控（Watch）操作，资源类型可以是Kubernetes内置资源，也可以是CRD自定义资源，其中最核心的功能是Reflector。**Reflector用于监控指定资源的Kubernetes资源，当监控的资源发生变化时，触发相应的变更事件，例如Added（资源添加）事件、Updated（资源更新）事件、Deleted（资源删除）事件，并将其资源对象存放到本地缓存DeltaFIFO中。**

通过NewReflector实例化Reflector对象，**实例化过程中须传入ListerWatcher数据接口对象，它拥有List和Watch方法**，用于获取及监控资源列表。只要实现了List和Watch方法的对象都可以称为ListerWatcher。Reflector对象通过Run函数启动监控并处理监控事件。而在Reflector源码实现中，其中最主要的是ListAndWatch函数，**它负责获取资源列表（List）和监控（Watch）指定的Kubernetes API Server资源。**

**ListAndWatch函数实现可分为两部分：第1部分获取资源列表数据，第2部分监控资源对象。**

#### 1. 获取资源列表数据

ListAndWatch List在程序第一次运行时获取该资源下所有的对象数据并将其存储至DeltaFIFO中。以Informers Example代码示例为例，在其中，我们获取的是所有Pod的资源数据。ListAndWatch List流程图如图5-6所示。

