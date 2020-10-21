---
blogpost: true
date: Oct 21, 2020
author: JeffreyGuan
location: BJ
category: Kubernetes
tags: kubernetes, client-go, informer, k8s源码分析
language: English
---

# client-go系列之5---Informer

摘要：介绍client-go informer及controller.

![](https://gitee.com/double12gzh/wiki-pictures/raw/master/2020-10-13-client-go-kubeconfig/0.png)

## 1. 写在前面

> 个人主页: https://gzh.readthedocs.io
> 
> 关注容器技术、关注`Kubernetes`。
> 
> 问题或建议，请公众号（`double12gzh`）留言。

依然秉承本系列的传统，在文章开始都会再次上一下下面这经经典的图(足见其重要性，哈哈哈)。

![](https://raw.githubusercontent.com/kubernetes/sample-controller/master/docs/images/client-go-controller-interaction.jpeg)

在[client-go系列之1---client-go代码结构讲解](https://double12gzh.github.io/2020/10/11/client-go%E7%B3%BB%E5%88%97%E4%B9%8B1-client-go%E4%BB%A3%E7%A0%81%E7%BB%93%E6%9E%84%E8%AE%B2%E8%A7%A3/)中简单介绍一个client-go中的相关的模块(即图中上半部分)，其实，在client-go中不只是有前面提到的模块，还包括上图中下半部分的内容，即自定义控制器部分，如：

* informer reference： 是一个`Informer`的实例，主要用于处理与CRD（自定义资源）对象相关的。当我们开发自定义控制器（custom controller）时，需要这个控制器开创建相匹配的`Informer`。
* indexer reference：是一个`Indexer`的实例，主要用于处理与CRD（自定义资源）对象相关的。当我们开发自定义控制器时，需要创建`Indexer`的实例，这个实例主要作用是实现`存储+索引`。
* WorkQueue：工作者队列。前面我们提到过`Informer`，它除了更新本地缓存之外，还要将数据同步给相应控制器，`WorkQueue`就是为了数据同步的问题而产生的。当有资源被添加、修改或删除，`Informer`/`SharedInformer`就会将相应的事件加入到`WorkQueue`中。其它所有的控制器需要排队对这个queue进行读取，如果某个控制器发现这个事件与自己相关，就执行相应的操作。如果操作失败，就会把刚才取出的事件再放回到`WorkQueue`中，等再轮到自己执行时会再去重试这次失败的操作。如果操作成功，就将该事件从队列中删除。
* Resource Event Handler：这是一个回调函数，当一个`Informer`/`SharedInformer`要分发一个对象到控制器时，会调用此函数。例如：将对象的`Key`放在`WorkQueue`中并等待后续的处理。
* Process Item：用户自定义的处理`WorkQueue`中的相应`Item`的函数。 如，我们可以在这里面使用`Indexer`或`Listing wrapper`来根据相应的`Key`检索对象。

> Item的内容如下：

![](https://gitee.com/double12gzh/wiki-pictures/raw/master/2020-10-16-custom-controller/20201021-items.png)

> queue中的内容如下：

![](https://gitee.com/double12gzh/wiki-pictures/raw/master/2020-10-16-custom-controller/20201021-queue.png)

> 在client-go的controller中给出了如何定义`Indexer`和`Informer`的方法，代码位置：client-go/tools/cache/controller.go，代码如下：

```go
  344 func NewIndexerInformer(
  345     lw ListerWatcher,                 // 用于获取/监控需要Informer处理的资源
  346     objType runtime.Object,           // 订阅的对象类型
  347     resyncPeriod time.Duration,       // 非0时，将会一直list我们所关心的对象； 0时，‘重新list’将会被推迟
  348     h ResourceEventHandler,           // 用于处理与resources相关的事件
  349     indexers Indexers,
  350 ) (Indexer, Controller) {
  351     // This will hold the client state, as we know it.
  352     clientState := NewIndexer(DeletionHandlingMetaNamespaceKeyFunc, indexers)
  353
  354     return clientState, newInformer(lw, objType, resyncPeriod, h, clientState)
  355 }
```

> ResourceEventHandler的定义如下。其代码位置: client-go/tools/cache/controller.go。这里的`Evnet`只是具有`通知`的作用，因为，我们不应该对这里收到的对象进行任何修改。

```go
212 type ResourceEventHandler interface {
213     OnAdd(obj interface{})                  // 当有新的对象被创建时，将会调用这个函数
214     OnUpdate(oldObj, newObj interface{})    // 当对象被修改时，将会调用这个函数。除此之外，当有`re-list`操作时，这个函数也会被再次调用
215     OnDelete(obj interface{})               // 当对象被删除时，将会调用这个函数。
216 }
```

## 2. Informer简介

一句话背景介绍：为了减少当多个控制器对k8s-api-server进行大量访问时对api-server造成压力。

### 2.1 产生的背景

随着Controller越来越多，如果Controller直接访问k8s-apiserver，那么将会导致其压力过大，于是在这样的背景下就有了`Informer`的概念。其发展到今天这个架构，大概可以总结出以下迭代思路：

第一阶段，`Controller`直接访问k8s-api-server。存在的问题：多个控制器大量访问k8s-apiserver时会对其造成巨大的压力。

第二阶段，`Informer`代替`Controller`去访问k8s-apiserver。而`Controller`的所有操作操作(如：查状态、对资源进行伸缩等）都和`Informer`进行交互。但`Informer`没有必要每次都去访问k8s-apiserver，它只要在需要的时候通过`ListAndWatch`(即通过k8s List API获取所有资源的最新状态；通过Wath API去监听这些资源状态的变化)与k8s-apiserver交互即可。

> ListAndWatch的代码位置: client-go/tools/cache/reflector.go 
> 
> func (r *Reflector) ListAndWatch(stopCh <-chan struct{}) error{
>    ...
> }

第三阶段， `Informer`并没有直接访问k8s-api-server，而是通过一个叫`Reflector`的对象进行api-server的访问。上面所说的 `ListAndWatch` 事实上是由Reflector`实现的。

第四阶段, 通过指定资源类型来`Watch`特定资源。

```go
// 代码位置: client-go/tools/cache/listwatch.go
36 // Watcher is any object that knows how to start a watch on a resource.
37 type Watcher interface {
38     // Watch should begin a watch at the specified version.
39     Watch(options metav1.ListOptions) (watch.Interface, error)
40 }
```

第五阶段，定义`SharedInformer`。如果`Controller`与`Informer`是一一对应的关系，那么k8s-api-server的压力也还是挺大的。但是类似于Pod这样的资源来说，`Deployment`和`StatefulSet`都能对它进行管理，当多个控制器同时想查Pod的状态时，实现上，只需要有一个`Informer`就能满足需求了，即: `SharedInformered`。

第六阶段，解决多个不同的控制器排除与重试问题，引入`DeltaFIFOQueue`。每当资源被修改时，`Reflector`就会收到事件通知，并将对应的事件放入`DeltaFIFOQueue`中。另外，`SharedInformer`会不断从`DeltaFIFOQueue`中读取事件并更新本地缓存(`ThreadSafeStore`)的状态。

### 2.2 主要功能

* 通过`List()`/`Get()`（代码位置：client-go/tools/cache/listwatch.go）获取资源对象
* 监听事件(`OnAdd`, `OnUpdate`, `OnDelete`)，并触发回调(`ResourceEventHandler`)
* 支持二级缓存(`DeltaFIFOQueue`和`ThreadSafeStore`，前者用于存储`Watch`返回的事件，后者是一个LocalStore，能够被`Lister`的`List()`和`Getter`的`Get()`方法访问)

> 需要注意的是，`Informer`与`k8s-apiserver`之间没有同步机制，但是`Informer`内部的二级缓存之间是有同步机制的。

上面提到的`List()`/`Get()`的定义位于：client-go/tools/cache/listwatch.go

```go
29 // Lister is any object that knows how to perform an initial list.
30 type Lister interface {
31     // List should return a list type object; the Items field will be extracted, and the
32     // ResourceVersion field will be used to start the watch in the right place.
33     List(options metav1.ListOptions) (runtime.Object, error)
34 }
... 
64 // Getter interface knows how to access Get method from RESTClient.
65 type Getter interface {
66     Get() *restclient.Request
67 }
```

### 2.3 主要模块

* Reflector
* DeltaFIFO
* ThreadSafeStore（LocalStore）
* Controller
* Lister
* Processor

> 需要注意的是：
> * 这里提到的`Controller`不是`Kubernetes Controller`（前几篇文章中我们实际上已经提到过，再次重审一下，这两个Controller并没有任何联系）
> * `Reflector`主要用于监听与指定资源类型相关的事件
> * `DeltaFIFO`和`ThreadSafeStore（LocalStore）`是`Informer`的二级缓存
> * `Lister`主要是被调用`List/Get`方法
> * `Processor`中记录了所有的回调函数（即 `ResourceEventHandler`）的实例，并负责触发之

### 2.4 类图

![](https://gitee.com/double12gzh/wiki-pictures/raw/master/2020-10-11-client-go/2020-shared-informer.png)

[出处](https://gitee.com/double12gzh/wiki-pictures/raw/master/2020-10-11-client-go/2020-shared-informer.png)

### 2.5 SharedInformer实现机制

> 当同一个资源(如：pdod)的`Informer`被实例化多次后，将会产生多个`Reflector`，如果这些`Reflector`都去调用`ListAndWatch`来获取资源时，将会对k8s-apiserver造成巨大的压力。

`SharedInformer`的意思是指：对于属于同一类型的资源来说，他们将会共享同一个`Informer`和`Reflector`，其实现代码如下：

```go
// 代码位置: client-go/informers/factory.go
55 type sharedInformerFactory struct {
56     client           kubernetes.Interface
57     namespace        string
58     tweakListOptions internalinterfaces.TweakListOptionsFunc
59     lock             sync.Mutex
60     defaultResync    time.Duration
61     customResync     map[reflect.Type]time.Duration
62
63     informers map[reflect.Type]cache.SharedIndexInformer     // sharedInformer的数据都存放在这个map中。
64     // startedInformers is used for tracking which informers have been started.
65     // This allows Start() to be called multiple times safely.
66     startedInformers map[reflect.Type]bool
67 }
```

### 2.6 不同资源的Informer定义

对于不同的资源(如：pods, deployments, ...)都有一个与之相对应的`xxxInformer`存在，他们的位置为(以pod为例):

```go
// 代码位置：clent-go/informers/core/v1/pod.go
   35 // PodInformer 提供了与pod相关的shared informer及lister进行交互的途径 
   36 // 每个k8s resource都会有这样一个Informer，且Informer中都有以下两个方法：Informer(), Lister()
   37 type PodInformer interface {
   38     Informer() cache.SharedIndexInformer
   39     Lister() v1.PodLister
   40 }
   41
   42 type podInformer struct {
   43     factory          internalinterfaces.SharedInformerFactory
   44     tweakListOptions internalinterfaces.TweakListOptionsFunc
   45     namespace        string
   46 }
```

调用不同资源的Informer的使用示例如下：

```go
deployInformer = sharedInformer.
               apps().              // client-go/informers/apps
               V1().                // client-go/informers/apps/v1
               Deployments().       // client-go/informers/apps/v1/deployment.go
               Informer()           // client-go/informers/apps/v1/deployment.go: type DeploymentInformer interface{}
```

## 3. Controller

### 3.1 Controller定义

代码位置: `client-go/tools/cache/controller.go`

```go
96 // 这是一个Base Controller, 是其它所有控制器的`基类`。在`SharedInformer`中将会被用到。
97 // 
98 type Controller interface {
99      // Run 只做两件事情：
100     // 1. 构建并启动一个`Reflector`。把从`Config.ListerWatcher`中抛出的对象或通知发送到`Config.Queue`中，另外也可能会触发队列同步`Resync`
102     // 2. 不断的从queue中弹出item，并使用Config.ProcessFunc进行处理。
103     // 
104     // 当channel `stopCh`被关闭时，上面两个操作会停止。
105     Run(stopCh <-chan struct{})
106
107     // HasSynced Config的队列是否已经同步过了
108     HasSynced() bool
109
110     // LastSyncResourceVersion delegates to the Reflector when there
111     // is one, otherwise returns the empty string
112     LastSyncResourceVersion() string
113 }
```

`Controller`的具体实现如下：

```go
   89 type controller struct {
   90     config         Config
   91     reflector      *Reflector
   92     reflectorMutex sync.RWMutex
   93     clock          clock.Clock
   94 }
```

从上面的实现中， 我们可以看到， 一个`Controller`实际上是一个以`Config`为参数，并将会被`Informer`使用到的一个low-level的控制器。

### 3.2 Controller关键方法

```go
// 代码位置: client-go/tools/cache/controller.go
// Run 开始处理items，当stopCh被关闭或stopCh收到某个值是将会停止执行。
127 func (c *controller) Run(stopCh <-chan struct{}) {
128     defer utilruntime.HandleCrash()
129     go func() {
130         <-stopCh
131         c.config.Queue.Close()
132     }()
133     r := NewReflector(             // 创建一个Reflector， 用于ListAndWatch， 从而获取指定资源的当前状态
134         c.config.ListerWatcher,    // List/Watch指定的资源对象
135         c.config.ObjectType,
136         c.config.Queue,
137         c.config.FullResyncPeriod,
138     )
139     r.ShouldResync = c.config.ShouldResync
140     r.WatchListPageSize = c.config.WatchListPageSize
141     r.clock = c.clock
142     if c.config.WatchErrorHandler != nil {
143         r.watchErrorHandler = c.config.WatchErrorHandler
144     }
145
146     c.reflectorMutex.Lock()
147     c.reflector = r
148     c.reflectorMutex.Unlock()
149
150     var wg wait.Group
151
152     wg.StartWithChannel(stopCh, r.Run) // r.Run中的最主要的方法就是执行Reflector的ListAndWatch()获取资源对象的值
153     // c.ProcessLoop(client-go/tools/cache/controller.go)会从queue中取出资源Delta并进行处理
154     wait.Until(c.processLoop, time.Second, stopCh)
155     wg.Wait()
156 }
```

> c.processLoop的实现如下：

```go
181 func (c *controller) processLoop() {
182     for {
183         obj, err := c.config.Queue.Pop(PopProcessFunc(c.config.Process))
184         if err != nil {
185             if err == ErrFIFOClosed {
186                 return
187             }
188             if c.config.RetryOnError {
189                 // This is the safe way to re-enqueue.
190                 c.config.Queue.AddIfNotPresent(obj)  // Queue是一个DeltaFIFO
191             }
192         }
193     }
194 }
```

### 3.3 Controller小结

* 有一个FIFO Queue的索引
* 有一个ListWatcher的索引 
* 通过Pop从FIFO queue中消费资源对象(即: items)
* 创建Reflector
* 提供一个proccessLoop处理资源对象以达到期望的状态

> 另外需要注意的是，不要把tools/cache/controller.go中的`Controller`与`Cumstom Controller`混为一谈，两者是完全不同的两个东西。

### 3.4 Controller示例

前面从“理论”方面对Controller/Informer做了简单的介绍，俗话说的好“纸上得来终觉浅，绝知此事要耕行”，建议学完上面的理论后，还是结合下面例子在实践中再体会一下。

Controller的工作流程：
请参考: client-go/tools/cache/controller_test.go: Example()

---------------

欢迎关注我的微信公众号：

![](https://gitee.com/double12gzh/wiki-pictures/raw/master/wechat_public.jpg)