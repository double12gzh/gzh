.. post:: Sep 21, 2020
   :author: 大海星
   :category: GoLang
   :location: BJ
   :tags: golang, waitgroup, error
.. :excerpt: 1

.. figure:: https://gitee.com/double12gzh/wiki-pictures/raw/master/2020-09-26-k8s-logo.png
   :alt:

1. 写在前面
-----------

    微信公众号：\ **[double12gzh]**

    个人主页: https://gzh.readthedocs.io

    关注容器技术、关注\ ``Kubernetes``\ 。问题或建议，请公众号留言。

Kubernetes架构的设计模式，我们可以很方便的使用\ `CRD(Custom Resource
Definitions) <https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/>`__\ 对k8s
API进行扩展。但是问题，通过\ `client-go <https://github.com/kubernetes/client-go>`__\ 来获取这些CRD或开发用户自定义控制器，那是比较麻烦的一件事情，除此之外，市面上对于\ ``client-go``\ 的介绍并不是很多。

本文将会通过一个示例，简单介绍一下如何通过client-go获取CRD。

2. 写作动机
-----------

我在PaaS平台的日常开发工作中，想要将第三方存储厂商集成到Kubernetes集群中时，遇到了这个挑战。计划是使用自定义资源定义来定义诸如文件系统池和文件系统。然后，一个自定义的Operator可以监听这些资源的创建和删除，并负责这些资源的生命周期的管理
。

3. 定义CR(Custom Resource)
--------------------------

在本文中，我们将以一个简单的例子来进行演示。使用kubectl可以很容易地创建自定义资源定义，对于这个例子，我们将从一个简单的资源定义开始做起：

.. code-block:: yaml
   :caption: test.yaml
   :name: test.yaml
   :linenos:

    apiVersion: "apiextensions.k8s.io/v1beta1"
    kind: "CustomResourceDefinition"
    metadata:
        name: "projects.examples-gzh.com"
    spec:
        group: "examples-gzh.com"
        version: "v1alpha1"
        scope: "Namespaced"
        names:
        plural: "projects"
        singular: "project"
        kind: "Project"
        validation:
        openAPIV3Schema:
            required: ["spec"]
            properties:
            replicas:
                type: "integer"
                minimum: 1

-  确定\ ``Group``\ 的名字。

   在定义CRD时，我们首先需要定义它所在的\ ``Group``\ (在上面的代码中，其``Group``\ 为：\ ``examples-gzh.com``)。对于Group的定义，为了避免命名冲突，通常会使用一些比较特别的字符串(如：你的个人主页的地址、你公司的域名等)，\ ``Group``\ 名
字确定了之后，由于CRD的名字是按\ ``<plural-resource-name>.<api-group-name>``\ 这个格式进行命名的，所以这里我们的CRD的名字为\ ``projects.example-gzh.com``\ 。

-  确定\ ``version``\ 。

   这里的\ ``version``\ ，即\ ``spec.version``\ 。如果你的代码没有开发完成，或者还在快速迭代中，那么，建议你使用\ ``alpha``\ 这样的命名规则。这样的好处是，如果别人想使用你的代码去使用，那么，他单从版本号上就可以很方便的快速知道，你这 
是一个不稳定的版本。

-  schema校验

   在上面我们的CRD中，我们引入了\ ``spec.validation.openAPIV3Schema``\ ，它的作用是对其中的字段进行校验，如果用户在使用我们的CRD时，提供了一个不符合要求的字段后，validation可以很方便的对其进行校验。除了在这里引用validation之外，我们还
可以选择在admintion
   controller中通过\ ``Validate``\ 阶段进行验证，不过样是需要开启admission
   webhook的。

将上面的代码保存到一个文件中之后，我们就可以通过\ ``kubectl apply -f demo.yaml``\ 进行部署了。我在本机通过minikue启动了一个K8S集群：

.. code:: bash

    PS C:\Users\guanzenghui> kubectl get po -A
    NAMESPACE              NAME                                        READY   STATUS    RESTARTS   AGE
    kube-system            coredns-f9fd979d6-7h2b7                     1/1     Running   1          9h
    kube-system            etcd-minikube                               0/1     Running   2          9h
    kube-system            kube-apiserver-minikube                     1/1     Running   2          9h
    kube-system            kube-controller-manager-minikube            0/1     Running   2          9h
    kube-system            kube-proxy-p8zb7                            1/1     Running   1          9h
    kube-system            kube-scheduler-minikube                     0/1     Running   2          9h
    kube-system            storage-provisioner                         1/1     Running   1          9h
    kubernetes-dashboard   dashboard-metrics-scraper-c95fcf479-gvhpd   1/1     Running   1          9h
    kubernetes-dashboard   kubernetes-dashboard-5c448bc4bf-lpwqh       1/1     Running   1          9h

    PS C:\Users\guanzenghui> kubectl version
    Client Version: version.Info{Major:"1", Minor:"16+", GitVersion:"v1.16.6-beta.0", GitCommit:"e7f962ba86f4ce7033828210ca3556393c377bcc", GitTreeState:"clean", BuildDate:"2020-01-15T08:26:26Z", GoVersion:"go1.13.5", Compiler:"gc", Platform:"windows/amd64"}
    Server Version: version.Info{Major:"1", Minor:"19", GitVersion:"v1.19.2", GitCommit:"f5743093fd1c663cb0cbc89748f730662345d44d", GitTreeState:"clean", BuildDate:"2020-09-16T13:32:58Z", GoVersion:"go1.15", Compiler:"gc", Platform:"linux/amd64"}

部署我们的CRD:

.. code:: bash

    PS C:\Users\guanzenghui\Documents> kubectl apply -f .\Untitled-2.yaml
    customresourcedefinition.apiextensions.k8s.io/projects.examples-gzh.com created

    PS C:\Users\guanzenghui\Documents> kubectl get crd
    NAME                        CREATED AT
    projects.examples-gzh.com   2020-09-25T10:40:01Z

如果需要查看其详情，可以使用命令:
``kubectl describe crd projects.examples-gzh.com``

既然CRD已经创建完成了，接下来我们看一下如何使用这个CRD来创建与之相对应的CR。CR相关的文件内容如下：

.. code:: yaml

    apiVersion: "examples-gzh.com/v1alpha1"
    kind: Project
    metadata:
      name: gzh-cr
      namespace: default
    spec:
      replica: 2

创建CR

.. code:: bash

    PS C:\Users\guanzenghui\Documents> kubectl apply -f cr.yaml
    project.examples-gzh.com/gzh-cr created

    PS C:\Users\guanzenghui\Documents> kubectl get Project
    NAME     AGE
    gzh-cr   39s

接下来，我们将使用client-go来获取这个CR。

4. 创建golang client
--------------------

在进行本节前，我假设您已经对client-go、k8s控制器机制有所理解，并且有一定的GoLang的开发经验。

另外，与其它一些讲解Operator的文章不同的是，这些使用CRD的文档会假设你正在使用某种代码生成器来自动生成客户端库。然而，对于这个过程的文档很少，而且从阅读Github上的一些激烈的讨论中，我们可以看出，它仍然是一个正在进行中的工作。

本文中，我将坚持使用（大部分）手动实现的客户端的方式给大家展示。

首先，您可以创建一个自己的项目路径，并安装依赖:

.. code:: bash

    mkdir github.com/double12gzh/k8s-crd-demo
    go get k8s.io/client-go@v0.17.0
    go get k8s.io/apimachinery@v0.17.0

4.1 定义类型
~~~~~~~~~~~~

.. code-block:: go
   :caption: example.go
   :linenos:

    package v1alpha1

    import metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"

    type ProjectSpec struct {
      Replicas int `json:"replicas"`
    }

    type Project struct {
      metav1.TypeMeta   `json:",inline"`
      metav1.ObjectMeta `json:"metadata,omitempty"`
      Spec ProjectSpec `json:"spec"`
    }

    type ProjectList struct {
        metav1.TypeMeta `json:",inline"`
        metav1.ListMeta `json:"metadata,omitempty"`
        Items []Project `json:"items"`
    }

``metav1.ObjectMeta``\ 中包含了一个比较重要的类型\ ``metadata``\ ，k8s中所有的资源有都这个属性，这里面可以定义诸如：\ ``name``\ ，\ ``namespace``\ ，\ ``label``\ 等的属性。

4.2 定义DeepCopy方法
~~~~~~~~~~~~~~~~~~~~

Kubernetes API 所服务的每个类型（在本例中，Project 和
ProjectList）都需要实现 k8s.io/apimachinery/pkg/runtime.Object
接口。这个接口定义了两个方法GetObjectKind()和DeepCopyObject()。第一个方法已经由内嵌的metav1.TypeMeta结构提供了；第二个方法你必须自己实现。

DeepCopyObject方法的目的是生成一个对象的深度拷贝。由于这涉及到大量的模板代码，所以这些方法通常是自动生成的。在本文中，我们将手动进行。继续在同一个包中添加第二个文件
deepcopy.go。

.. code-block:: go
   :caption: deepcopy.go
   :linenos:

    package v1alpha1

    import "k8s.io/apimachinery/pkg/runtime"

    // DeepCopyInto 把一个对象的所有属性复制给此对象类型的指针
    func (in *Project) DeepCopyInto(out *Project) {
        out.TypeMeta = in.TypeMeta
        out.ObjectMeta = in.ObjectMeta
        out.Spec = ProjectSpec{
            Replicas: in.Spec.Replicas,
        }
    }

    // DeepCopyObject 返回一个对象类型
    func (in *Project) DeepCopyObject() runtime.Object {
        out := Project{}
        in.DeepCopyInto(&out)

        return &out
    }

    // DeepCopyObject 返回一个对像类型
    func (in *ProjectList) DeepCopyObject() runtime.Object {
        out := ProjectList{}
        out.TypeMeta = in.TypeMeta
        out.ListMeta = in.ListMeta

        if in.Items != nil {
            out.Items = make([]Project, len(in.Items))
            for i := range in.Items {
                in.Items[i].DeepCopyInto(&out.Items[i])
            }
        }

        return &out
    }

上面这个DeepCopy是我们手动来生成的，你可能已经注意到，定义所有这些不同的
DeepCopy
方法并不是一件很有趣的事情。有很多不同的工具和框架可以自动生成这些方法（所有的文档和整体成熟度都有很大的不同）。我发现效果最好的是控制器生成工具，它是\ `Kubebuilder <https://github.com/kubernetes-sigs/kubebuilder>`__\ 框架的一部分。  

下面我们就来看一下：

``go get -u github.com/kubernetes-sigs/controller-tools/cmd/controller-gen``

为了能够使用\ ``controller-gen``\ ，我们需要在CRD类型上面的添加一个annotation，如下：

.. code-block:: go
   :caption: test.go
   :linenos:

    // +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object
    type Project struct {
        // ...
    }

    // +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object
    type ProjectList struct {
        // ...
    }


.. tip:: 
   说明：对于这些annotation我们没有必要去全部记住，只有当使用到的时候再去查阅一下就行，根据二八原则，只需要记住一些常用的就可以了，其它那些不常用的只需要了解一下。

写好了上述代码，我们运行一下命令\ ``controller-gen object paths=./api/types/v1alpha1/project.go``\ 即可生成需要代码。

为了更加的简化，你甚至可以在代码文件的前面加一个声明\ ``go:generate``\ ，具体请\ `参考 <https://blog.golang.org/generate>`__\ 。如：

.. code-block:: go
   :caption: test.go
   :linenos:

    package v1alpha1

    import metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"

    //go:generate controller-gen object paths=$GOFILE

    // ...

然后只需要在代码的根路径中执行\ ``go generate ./...``\ 即可。

4.3 注册类型
~~~~~~~~~~~~

接下来，你需要让客户端库知道你的新类型。这将允许客户端在与API服务器通信时（或多或少）自动处理你的新类型。

为此，在你的包中添加一个新文件 register.go。

.. code-block:: go
   :caption: register.go
   :linenos:

    package v1alpha1

    import (
        metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
        "k8s.io/apimachinery/pkg/runtime"
        "k8s.io/apimachinery/pkg/runtime/schema"
    )

    const GroupName = "example-gzh.com"
    const GroupVersion = "v1alpha1"

    var SchemeGroupVersion = schema.GroupVersion{Group: GroupName, Version: GroupVersion}

    var (
        SchemeBuilder = runtime.NewSchemeBuilder(addKnownTypes)
        AddToScheme   = SchemeBuilder.AddToScheme
    )

    func addKnownTypes(scheme *runtime.Scheme) error {
        scheme.AddKnownTypes(SchemeGroupVersion,
            &Project{},
            &ProjectList{},
        )

        metav1.AddToGroupVersion(scheme, SchemeGroupVersion)
        return nil
    }

正如你所注意到的，这段代码还没有真正做任何事情（除了创建一个新的runtime.SchemeBuilder实例）。重要的部分是AddToScheme函数（第16行），它是第15行创建的runtime.SchemeBuilder类型的导出结构成员。只要Kubernetes客户端被初始化以注册你的类型定 
义，你就可以在以后从客户端代码的任何部分调用这个函数。

4.4 创建HTTP Client
~~~~~~~~~~~~~~~~~~~

在定义了类型并添加了一个方法来在全局方案构建器上注册它们之后，你现在可以创建一个能够加载你的自定义资源的HTTP客户端。

为此，将以下代码添加到你的包的main.go文件中：

.. code-block:: go
   :caption: main.go
   :linenos:

    package main

    import (
        "flag"
        "log"

        "ks.io/apimachinery/pkg/runtime/schema"
        "ks.io/apimachinery/pkg/runtime/serializer"

        "github.com/double12gzh/k8s-demo/api/types/valpha"
        "ks.io/client-go/kubernetes/scheme"
        "ks.io/client-go/rest"
        "ks.io/client-go/tools/clientcmd"
    )

    var kubeconfig string

    func init() {
        flag.StringVar(&kubeconfig, "kubeconfig", "", "path to Kubernetes config file")
        flag.Parse()
    }

    func main() {
        var config *rest.Config
        var err error

        if kubeconfig == "" {
            log.Printf("using in-cluster configuration")
            config, err = rest.InClusterConfig()
        } else {
            log.Printf("using configuration from '%s'", kubeconfig)
            config, err = clientcmd.BuildConfigFromFlags("", kubeconfig)
        }

        if err != nil {
            panic(err)
        }

        valpha.AddToScheme(scheme.Scheme)

        crdConfig := *config
        crdConfig.ContentConfig.GroupVersion = &schema.GroupVersion{Group: valpha.GroupName, Version: valpha.GroupVersion}
        crdConfig.APIPath = "/apis"
        crdConfig.NegotiatedSerializer = serializer.NewCodecFactory(scheme.Scheme)
        crdConfig.UserAgent = rest.DefaultKubernetesUserAgent()

        exampleRestClient, err := rest.UnversionedRESTClientFor(&crdConfig)
        if err != nil {
            panic(err)
        }
    }

现在你可以使用第48行创建的exampleRestClient来查询example.martin-helmich.de/v1alpha1
API组中的所有自定义资源。例如：

.. code:: go

    result := v1alpha1.ProjectList{}
    err := exampleRestClient.Get().Resource("projects").Do().Into(&result)

为了以一种更安全的方式使用你的API，通常情况下，我们最好在自己的clientet中封装这些操作。为此，创建一个新的子包clientet/v1alpha1。

首先，实现一个定义你的API组类型的接口，并将配置设置从你的主方法移到该clientet的构造函数中（下面例子中的NewForConfig）。

.. code:: go

    package valpha

    import (
        "github.com/double12gzh/k8s-demo/api/types/valpha"
        "ks.io/apimachinery/pkg/runtime/schema"
        "ks.io/client-go/kubernetes/scheme"
        "ks.io/client-go/rest"
    )

    type ExampleVAlphaInterface interface {
        Projects(namespace string) ProjectInterface
    }

    type ExampleVAlphaClient struct {
        restClient rest.Interface
    }

    func NewForConfig(c *rest.Config) (*ExampleVAlphaClient, error) {
        config := *c
        config.ContentConfig.GroupVersion = &schema.GroupVersion{Group: valpha.GroupName, Version: valpha.GroupVersion}
        config.APIPath = "/apis"
        config.NegotiatedSerializer = scheme.Codecs.WithoutConversion()
        config.UserAgent = rest.DefaultKubernetesUserAgent()

        client, err := rest.RESTClientFor(&config)
        if err != nil {
            return nil, err
        }

        return &ExampleVAlphaClient{restClient: client}, nil
    }

    func (c *ExampleVAlphaClient) Projects(namespace string) ProjectInterface {
        return &projectClient{
            restClient: c.restClient,
            ns:         namespace,
        }
    }

下面的代码还不能编译，因为它仍然缺少\ ``ProjectInterface``\ 和\ ``projectClient``\ 类型。我们稍后将讨论这些类型。

``ExampleV1Alpha1Interface``\ 和它的实现\ ``--ExampleV1Alpha1Client``\ 结构现在是访问自定义资源的中心点。现在，你可以在\ ``main.go``\ 中简单地调用\ ``clientet, err := v1alpha1.NewForConfig(config)``\ 来创建一个新的客户集。

接下来，你需要实现一个特定的\ ``clientset``\ 来访问Project自定义资源（注意，上面的例子已经使用了\ ``ProjectInterface``\ 和\ ``projectClient``\ 类型，我们仍然需要提供）。在同一个包中创建第二个文件\ ``projects.go``\ 。

.. code:: go

    package valpha

    import (
        "github.com/double12gzh/k8s-demo/api/types/valpha"
        metav "ks.io/apimachinery/pkg/apis/meta/v"
        "ks.io/apimachinery/pkg/watch"
        "ks.io/client-go/kubernetes/scheme"
        "ks.io/client-go/rest"
    )

    type ProjectInterface interface {
        List(opts metav.ListOptions) (*valpha.ProjectList, error)
        Get(name string, options metav.GetOptions) (*valpha.Project, error)
        Create(*valpha.Project) (*valpha.Project, error)
        Watch(opts metav.ListOptions) (watch.Interface, error)
        // ...
    }

    type projectClient struct {
        restClient rest.Interface
        ns         string
    }

    func (c *projectClient) List(opts metav.ListOptions) (*valpha.ProjectList, error) {
        result := valpha.ProjectList{}
        err := c.restClient.
            Get().
            Namespace(c.ns).
            Resource("projects").
            VersionedParams(&opts, scheme.ParameterCodec).
            Do().
            Into(&result)

        return &result, err
    }

    func (c *projectClient) Get(name string, opts metav.GetOptions) (*valpha.Project, error) {
        result := valpha.Project{}
        err := c.restClient.
            Get().
            Namespace(c.ns).
            Resource("projects").
            Name(name).
            VersionedParams(&opts, scheme.ParameterCodec).
            Do().
            Into(&result)

        return &result, err
    }

    func (c *projectClient) Create(project *valpha.Project) (*valpha.Project, error) {
        result := valpha.Project{}
        err := c.restClient.
            Post().
            Namespace(c.ns).
            Resource("projects").
            Body(project).
            Do().
            Into(&result)

        return &result, err
    }

    func (c *projectClient) Watch(opts metav.ListOptions) (watch.Interface, error) {
        opts.Watch = true
        return c.restClient.
            Get().
            Namespace(c.ns).
            Resource("projects").
            VersionedParams(&opts, scheme.ParameterCodec).
            Watch()
    }

这个client显然还不完善，还缺失了删除、更新等方法。不过，这些方法可以和已有的方法类似实现。看看现有的clientset（例如，Pod
clientset）以获得灵感。

在创建了clientset之后，用它来列出你现有的资源就变得非常容易了。

.. code:: go

    package main

    import (
        "fmt"

        clientValpha "github.com/double12gzh/k8s-demo/clientset/valpha"
    )

    // ...

    func main() {
        // ...

        clientSet, err := clientValpha.NewForConfig(config)
        if err != nil {
            panic(err)
        }

        projects, err := clientSet.Projects("default").List(metav.ListOptions{})
        if err != nil {
            panic(err)
        }

        fmt.Printf("projects found: %+v\n", projects)
    }

4.5 生成Informer
~~~~~~~~~~~~~~~~

在构建Kubernetes
Operator时，您通常希望能够对新创建或更新的资源做出反应。理论上，您可以定期调用List()方法，检查是否有新资源被添加。在实践中，这是一个次优的解决方案，尤其是当您有很多这样的资源时。

大多数Operator的工作方式是通过使用初始List()调用来初始加载资源的所有相关实例，然后使用Watch()调用来订阅更新。然后，初始对象列表和从Watch接收到的更新被用来构建一个本地缓存，允许快速访问任何自定义资源，而不必每次都打到API服务器。       

这种模式非常常见，以至于client-go库为此提供了一个助手：k8s.io/client-go/tools/cache包中的Informer。您可以为您的自定义资源构建一个新的
Informer，如下所示：

.. code:: go

    package main

    import (
        "time"

        "github.com/double12gzh/k8s-demo/api/types/valpha"
        client_valpha "github.com/double12gzh/k8s-demo/clientset/valpha"
        metav "ks.io/apimachinery/pkg/apis/meta/v"
        "ks.io/apimachinery/pkg/runtime"
        "ks.io/apimachinery/pkg/util/wait"
        "ks.io/apimachinery/pkg/watch"
        "ks.io/client-go/tools/cache"
    )

    func WatchResources(clientSet client_valpha.ExampleVAlphaInterface) cache.Store {
        projectStore, projectController := cache.NewInformer(
            &cache.ListWatch{
                ListFunc: func(lo metav.ListOptions) (result runtime.Object, err error) {
                    return clientSet.Projects("some-namespace").List(lo)
                },
                WatchFunc: func(lo metav.ListOptions) (watch.Interface, error) {
                    return clientSet.Projects("some-namespace").Watch(lo)
                },
            },
            &valpha.Project{},
            *time.Minute,
            cache.ResourceEventHandlerFuncs{},
        )

        go projectController.Run(wait.NeverStop)
        return projectStore
    }

``NewInformer``\ 方法返回两个对象。第二个返回值，控制器控制\ ``List()``\ 和\ ``Watch()``\ 调用，并在第一个返回值，即存储中填充一个（或多或少）最近在API服务器上被监视的资源状态的缓存（在本例中，项目CRD）。

现在，你可以使用 ``store`` 来轻松访问你的 ``CRD``\ ，要么列出所有的
CRD，要么通过名称来访问它们。请记住，存储函数返回的是通用\ ``interface{}``\ 类型，所以您必须将它们类型化回您的CRD类型。

.. code:: go

    store := WatchResource(clientSet)
    project := store.GetByKey("some-namespace/some-project").(*v1alpha1.Project)

5. 总结
-------

为Custom Resources构建客户端是（至少，目前）只有很少的文档，有时可能会有点棘手。

如本文所示，为你的Custom
Resource建立一个客户端库，以及相应的Informer是一个很好的起点，可以构建你自己的Kubernetes
Operator，对Custom Resource的变化做出反应。

    您可以到我的\ `github <https://github.com/double12gzh/k8s-demo.git>`__\ 上查看完整代码