.. post:: Oct 11, 2020
   :author: 大海星
   :category: Kubernetes
   :location: BJ
   :tags: kubernetes, client-go
.. :excerpt: 1

client-go系列1---client-go代码详解
============================

1. 写在前面
-----------

   个人主页: https://gzh.readthedocs.io

   关注容器技术、关注\ ``Kubernetes``\ 。问题或建议，请公众号留言。

本系列内容都是基于这个版本的\ `client-go <https://github.com/kubernetes/client-go/tree/becbabb360023e1825a48b4db85f454e452ae249>`__\ 进行讲解，不同版本的略有差异。

.. code:: bash

   [root@77DDE94FF07FCC1-wsl /ACode/client-go] git rev-parse HEAD
   becbabb360023e1825a48b4db85f454e452ae249

下面简单罗列一下本文中可能会用提到的一些常用缩写，详细的介绍请参考后面的文章(TODO)\ `谈起k8s中的GVR我们实际在讲什么 <https://double12gzh.github.io/2020/10/11/client-go%E7%B3%BB%E5%88%97%E4%B9%8B1-client-go%E4%BB%A3%E7%A0%81%E7%BB%93%E6%9E%84%E8%AE%B2%E8%A7%A3-copy/>`__\ 。

|image0|

   说明

   看一下k8s的资源模型：

   apiVersion: extensions/v1beta1

   kind: ReplicaSet

   对于Resource我们可以将之类比于编程语言中的“包”的概念，主要用于区分不同的API,这样可以有效避免Kinds重名的问题。通常会使用公司的域名等具有独特明显区分的字符串作为Resource的值，如：alibaba-inc.com

   Version用于区分不同API的稳定程度及兼容性，如：v1beta1, v1

   Kind即为API所对应的名字，如：Deployment, Service

|image1|

2. 代码结构
-----------

从这个包的名字可以很明显的知道，\ ``client-go``\ 其实主要是提供了用户与k8s交互时使用客户端，方便大家编程。

   个别目录已过滤掉。

.. code:: bash

   [root@77DDE94FF07FCC1-wsl /ACode/client-go] tree -d -L 1 -I "testing|examples|*_test*|Godeps|third_party|metadata|deprecated|restmapper"
   .
   ├── discovery                   # 定义DsicoveryClient客户端。作用是用于发现k8s所支持GVR(Group, Version, Resources)。
   ├── dynamic                     # 定义DynamicClient客户端。可以用于访问k8s Resources(如: Pod, Deploy...)，也可以访问用户自定义资源(即: CRD)。
   ├── informers                   # k8s中各种Resources的Informer机制的实现。
   ├── kubernetes                  # 定义ClientSet客户端。它只能用于访问k8s Resources。每一种资源(如: Pod等)都可以看成是一个客端，而ClientSet是多个客户端的集合，它对RestClient进行了封装，引入了对Resources和Version的管 
   |                               # 理。通常来说ClientSet是client-gen来自动生成的。
   ├── listers                     # 提供对Resources的获取功能。对于Get()和List()而言，listers提供给二者的数据都是从缓存中读取的。
   ├── pkg
   ├── plugin                      # 提供第三方插件。如：GCP, OpenStack等。
   ├── rest                        # 定义RestClient，实现了Restful的API。同时会支持Protobuf和Json格式数据。
   ├── scale                       # 定义ScalClient。用于Deploy, RS, RC等的扩/缩容。
   ├── tools                       # 定义诸如SharedInformer、Reflector、DealtFIFO和Indexer等常用工具。实现client查询和缓存机制，减少client与api-server请求次数，减少api-server的压力。
   ├── transport
   └── util                        # 提供诸如WorkQueue、Certificate等常用方法。

   12 directories

3. 代码使用简单示例
-------------------

在对每一部分进行讲解前，先用一个图来讲解各部分之间的关系：

|image2|

对于图中的每一个带有标号的部分，下面给出简单的代码使用展示，
如果暂时不明白下面的代码可以先进行下一章节的学习。

3.1 获取kubeconfig及context
~~~~~~~~~~~~~~~~~~~~~~~~~~~

这一部分对应于序号1—tools/clientcmd。

.. code:: go

   func main() {
           var kubeconfig *string

           // 默认会从~/.kube/config路径下获取配置文件
           if home := homeDir(); home != "" {
                   kubeconfig = flag.String("kubeconfig", filepath.Join(home, ".kube", "config"), "(optional)absolute path to the kubeconfig file")
           } else {
                   kubeconfig = flag.String("kubeconfig", "", "absolute path to the kubeconfig file")
           }

           flag.Parse()

           // 使用k8s.io/client-go/tools/clientcmd生成config的对象
           if config, err := clientcmd.BuildConfigFromFlags("", *kubeconfig); err != nil {
                   panic(err.Error())
           }
   }

3.2 创建ClientSet
~~~~~~~~~~~~~~~~~

这一部分对应于序号2—ClientSet。

.. code:: go

   // 使用k8s.io/client-go/kubernetes生成一个ClientSet的客户端，客户端生成后，就可以使用这个客户端与k8s API server进行交互了，如获取资源列表、Create/Update/Delete资源等
   clientset, err := kubenetes.NewForConfig(config)
   if err != nil {
       panic(err.Error())
   }

3.3 使用ClientSet获取集群中的pods
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

这一部分对应于序号2/3/4—RestClient。

.. code:: go

   for {
       // 使用ClientSet客户端获取集群中所有的Pods。其中：ListOptions的结构如下：
       // type ListOptions struct {
       //      TypeMeta `json:",inline"`
       //      LabelSelector string `json:"labelSelector,omitempty"`
       //      FieldSelector string `json:"fieldSelector,omitempty"`    
       //}
       pods, err := clientset.CoreV1().Pods("").List(metav1.ListOptions{})
       if err != nil {
           panic(err.Error())
       }

       fmt.Printf("Number of pods are: %d\n", len(pods.Items))
   }

3.4 使用ClientSet获取指定的pod
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

这一部分对应于序号2/3/4—tools/clientcmd。

.. code:: go

   for {
       // 在这里我们从default这个namespace中获取了名为my-pod的Pod对象
       pod, err := clientset.CoreV1().Pods("default").Get("my-pod", metav1.GetOptions{})
       if err != nil {
           painc(err.Error())
       }

       fmt.Printf("%v\n\n\n\n", pod.spec)
   }

4. 各种Clients详解
------------------

client-go中定义的比较重要的client有：

-  RestClient
-  ClientSet
   (`用法示例 <https://github.com/kubernetes/client-go/tree/becbabb360023e1825a48b4db85f454e452ae249/examples/create-update-delete-deployment>`__)
-  DiscoveryClient
-  DynamicClient
   (`用法示例 <https://github.com/kubernetes/client-go/tree/becbabb360023e1825a48b4db85f454e452ae249/examples/dynamic-create-update-delete-deployment>`__)

其中，RestClient是所有客户端的基础，后三者都是对RestClient的封装。RestClient它通过kubeconfig与k8s-api-server进行交互。详细结构如下图：

|image3|

ClientSets使用\ ``预生成的API对象``,
这样的好处是当本地的API对象与k8s-api-server进行交互时会变得比较方便，方便的同时，随之也带来了版本与类型强耦合的问题。

DynamicClient则使用\ ``unstructured.Unstructured``\ 表示来自API
Server的所有对象值。\ ``Unstructured``\ 类型是一个嵌套的\ ``map[string]inferface{}``\ 值的集合来创建一个内部结构，这一点类似于RESTful
API中的Json数据，这样可以解决ClientSet中出现的强耦合的问题，换句话说，当客户端的API发生变化时，DynamicClient无需重新编译。DynamicClient使所有数据实现延时绑定，即只有到运行时才会实现绑定，这意味着程序运行之前，使用\ ``DynamicClient``\ 的程序将不会对对象进行Validation，这也是本client的一个缺点。

5. 其它组件
-----------

client-go中除了上面提到比较重要的客户端外,
本库还包含了各种机制(\ ``tools/cache``)。

下图比较直观的展示了client-go与customer
controller及client-go各组件之间的交互关系，是我们在开发自定义控制器时经常需要使用的机制，了解这个图有助于我们更好的理解client-go及controller背后的实现逻辑。

|image4|

   如果您对client-go之前就比较了解，建议您移步\ `sample-controller <https://github.com/kubernetes/sample-controller>`__\ 看一下控制器实现的具体代码。

5.1 Reflector
~~~~~~~~~~~~~

refelector是定义在包缓存里面的\ `Reflector <https://github.com/kubernetes/client-go/blob/becbabb360023e1825a48b4db85f454e452ae249/tools/cache/reflector.go#L49>`__\ 结构体，可以用于监视指定资源类型（kind）的Kubernetes  
API。

实现这个功能的函数是\ ``ListAndWatch``\ 。监视的对象可以是一个内置的资源，也可以是一个自定义的资源(CRD)。

当reflector通过watch
API接收到关于新资源实例存在的通知时，它会使用相应的listing
API获取新创建的对象，并将其放在\ ``watchHandler``\ 函数里面的\ ``DeltaFIFO``\ 队列中。

5.2 Informer
~~~~~~~~~~~~

它是定义在包缓存中的一个基础控制器，它可以w使用函数\ ``processLoop``\ 从\ ``DeltaFIFO``\ 队列中取出对象。

这个基础控制器的工作是保存对象以便以后检索，并调用我们的控制器将对象传递给它。

5.3 Indexer
~~~~~~~~~~~

提供对对象的索引功能。它被定义在\ ``tools/cache``\ 包中的\ ``Indexer``\ 类型中。

一个典型的索引用例是基于对象标签创建一个索引。Indexer可以基于几个索引函数来维护索引。Indexer使用一个线程安全的数据存储来存储对象和它们的键值。

在\ ``tools/cache``\ 内的\ ``Store``\ 类型中定义了一个名为\ ``MetaNamespaceKeyFunc``\ 的默认函数，该函数为该对象生成一个对象的键，作为\ ``<namespace>/<name>``\ 组合。

5.4 WorkQueue
~~~~~~~~~~~~~

这是在控制器代码中创建的队列，用于将对象的分发与处理解耦。编写
``Resource Event Handler``
函数来提取所分发对象的键值并将其添加到工作队列中。

--------------

欢迎关注我的微信公众号：

|image5|

.. |image0| image:: https://gitee.com/double12gzh/wiki-pictures/raw/master/2020-10-11-client-go/3-abbr.png
.. |image1| image:: https://gitee.com/double12gzh/wiki-pictures/raw/master/2020-10-11-client-go/1-group-version-resource.png
.. |image2| image:: https://gitee.com/double12gzh/wiki-pictures/raw/master/2020-10-11-client-go/2-details.png
.. |image3| image:: https://gitee.com/double12gzh/wiki-pictures/raw/master/2020-10-11-client-go/0-client-go-arch.png
.. |image4| image:: https://raw.githubusercontent.com/kubernetes/sample-controller/master/docs/images/client-go-controller-interaction.jpeg
.. |image5| image:: https://gitee.com/double12gzh/wiki-pictures/raw/master/wechat_public.jpg