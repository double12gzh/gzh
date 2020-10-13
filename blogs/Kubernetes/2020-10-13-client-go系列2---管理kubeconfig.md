---
blogpost: true
date: Oct 13, 2020
author: JeffreyGuan
location: BJ
category: Kubernetes
tags: client-go, kubeconfig
language: English
---

# client-go系列之2---管理kubeconfig

![](https://gitee.com/double12gzh/wiki-pictures/raw/master/2020-10-13-client-go-kubeconfig/0.png)

## 1. 写在前面

摘要：分析如何通过client-go来管理kubeconfig。

> 个人主页: https://gzh.readthedocs.io
> 
> 关注容器技术、关注`Kubernetes`。问题或建议，请公众号留言。

本系列内容都是基于这个版本的[client-go](https://github.com/kubernetes/client-go/tree/becbabb360023e1825a48b4db85f454e452ae249)进行讲解，不同版本的略有差异。

在使用client-go开发时，通常会遇到两种情况：

* 在集群内部访问kubernetes资源
* 在集群外部访问kubernetes资源

关于这两种方式的区别，可以分别看一下下面这段代码直观感受一下：

* [in-cluster-client-configuration](https://github.com/kubernetes/client-go/tree/becbabb360023e1825a48b4db85f454e452ae249/examples/in-cluster-client-configuration)

* [out-of-cluster-client-configuration](https://github.com/kubernetes/client-go/tree/becbabb360023e1825a48b4db85f454e452ae249/examples/out-of-cluster-client-configuration)


## 2. 集群配置管理

k8s的配置文件默认会存放在~/.kube/config中，其内容如下：

```bash
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0xxxxxxxxCg==
    server: https://127.0.0.1:33629
  name: kind-kind
contexts:
- context:
    cluster: kind-kind
    user: kind-kind
  name: kind-kind
current-context: kind-kind
kind: Config
preferences: {}
users:
- name: kind-kind
  user:
    client-certificate-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1Jxxxxxxx==
    client-key-data: LS0tLS1yyyyyyyCg==
```

上面这是单个配置文件情况，client-go提供了合并多个配置文件的能力，合并完成后如下：

```bash
apiVersion: v1

clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQtCg==
    server: https://127.0.0.1:51107
  name: kind-guan
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJtCg==
    server: https://127.0.0.1:33629
  name: kind-kind

contexts:
- context:
    cluster: kind-guan
    user: kind-guan
  name: kind-guan
- context:
    cluster: kind-kind
    user: kind-kind
  name: kind-kind


current-context: kind-kind

kind: Config

preferences: {}

users:
- name: kind-guan
  user:
    client-certificate-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUUtLS0tLQo=
    client-key-data: LS0tLS1CRUdJTiBxxxxxS0tCg==
- name: kind-kind
  user:
    client-certificate-data: LS0tLS1CRUdJTiBDRVJxxxxxxEUtLS0tLQo=
    client-key-data: LS0tLS1CRUdJTiBSU0EgUFJJVkFURSxxxxg==
```

client-go中对于配置文件的管理可以简单概括为以下两点：

* 加载配置文件
* 合并配置文件

接下来分别从代码层面看一下。

### 2.1 加载配置文件

加载配置文件的代码如下：

```go
func main() {
        var kubeconfig *string

        // 默认会从~/.kube/config路径下获取配置文件
        if home := homeDir(); home != "" {
                kubeconfig = flag.String("kubeconfig", filepath.Join(home, ".kube", "config"), "(optional)absolute path to the kubeconfig file")
        } else {
                kubeconfig = flag.String("kubeconfig", "", "absolute path to the kubeconfig file")
        }

        flag.Parse()

        // 使用k8s.io/client-go/tools/clientcmd加载配置文件并生成config的对象
        if config, err := clientcmd.BuildConfigFromFlags("", *kubeconfig); err != nil {
                panic(err.Error())
        }
}
```

`clientcmd.BuildConfigFromFlags`加载配置文件后, 会生成一个[Config](https://github.com/kubernetes/client-go/blob/becbabb360023e1825a48b4db85f454e452ae249/rest/config.go#L53)的对象，这个对象中会包含如：apiserver地址、用户名、密码、token等信息。

进入到`clientcmd.BuildConfigFromFlags`查看其进行配置加载相关的代码：

代码位置：`tools/clientcmd/client_config.go`
![tools/clientcmd/client_config.go](https://gitee.com/double12gzh/wiki-pictures/raw/master/2020-10-13-client-go-kubeconfig/1.png)

生成一个结构体：`ClientConfigLoadingRules`，这里面记录了当前传入的kubeconfig文件的位置。另外，在这个结构体中有一个变量`load ClientConfigLoader`，对于kubeconfig文件的管理，就是通过其中定义的[ClientConfigLoadingRules.Load()](https://github.com/kubernetes/client-go/blob/becbabb360023e1825a48b4db85f454e452ae249/tools/clientcmd/loader.go#L76)实现的。

![](https://gitee.com/double12gzh/wiki-pictures/raw/master/2020-10-13-client-go-kubeconfig/2.png)

![](https://gitee.com/double12gzh/wiki-pictures/raw/master/2020-10-13-client-go-kubeconfig/3.png)


### 2.2 合并配置文件

上面对kubeconfig完成加载后，下一步就需要对kubeconfig做合并处理。

代码位置: `tools/clientcmd/loader.go`

![](https://gitee.com/double12gzh/wiki-pictures/raw/master/2020-10-13-client-go-kubeconfig/4.png)

---
欢迎关注我的微信公众号：

![](https://gitee.com/double12gzh/wiki-pictures/raw/master/wechat_public.jpg)