
---
blogpost: true
date: Oct 14, 2020
author: JeffreyGuan
location: BJ
category: Kubernetes
tags: k8s, client-go, restclient, k8s源码分析
language: English
---

# client-go系列之3---restclient的使用

摘要：介绍如何使用client-go中的restclient，可以结合“系列之1”来看。

![](https://gitee.com/double12gzh/wiki-pictures/raw/master/2020-10-13-client-go-kubeconfig/0.png)

## 0. 背景

> 个人主页: https://gzh.readthedocs.io
> 
> 关注容器技术、关注`Kubernetes`。问题或建议，请公众号留言。

首先我通过`kind`创建了一个6节点的集群，本文章中所有的操作都是在这个集群中进行的。

通过本文的讲解，希望您能了解如何使用client-go中的RESTClient来对资源进行操作，这里我只是举了最简单的例子---pod资源获取。

文中用到的软件的版本如下：

* kind

```bash
[root@xxx-wsl ~/client-go-example] kind version
kind v0.9.0 go1.15.2 linux/amd64
```

## 1. 环境准备

通过kind创建多节点的k8s集群: 3个master节点 + 3个worker节点

```bash
[root@xxx-wsl ~/init_kind_clusters] kind create cluster --config=init_cluster.yaml
Creating cluster "kind" ...
 ✓ Ensuring node image (kindest/node:v1.19.1) 🖼
 ✓ Preparing nodes 📦 📦 📦 📦 📦 📦
 ✓ Configuring the external load balancer ⚖️
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing CNI 🔌
 ✓ Installing StorageClass 💾
 ✓ Joining more control-plane nodes 🎮
 ✓ Joining worker nodes 🚜
Set kubectl context to "kind-kind"
You can now use your cluster with:

kubectl cluster-info --context kind-kind

Not sure what to do next? 😅  Check out https://kind.sigs.k8s.io/docs/user/quick-start/
```

其中`init_cluster.yaml`内容如下：

```yaml
[root@xxx-wsl ~/init_kind_clusters] cat init_cluster.yaml
# a cluster with 3 control-plane nodes and 3 workers
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: control-plane
- role: control-plane
- role: worker
- role: worker
- role: worker
```

## 2. RestClient使用示例

这段代码的作用：从namespace为`kube-system`中获取所有的`pod`并输出到屏幕

`main.go`

```go

package main

import (
        "context"
        "fmt"

        corev1 "k8s.io/api/core/v1"
        metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"

        "k8s.io/client-go/kubernetes/scheme"
        "k8s.io/client-go/rest"
        "k8s.io/client-go/tools/clientcmd"
)

func main() {
        fmt.Println("Prepare config object.")

        // 加载k8s配置文件，生成Config对象
        config, err := clientcmd.BuildConfigFromFlags("", "/root/.kube/config")
        if err != nil {
                panic(err)
        }

        config.APIPath = "api"
        config.GroupVersion = &corev1.SchemeGroupVersion
        config.NegotiatedSerializer = scheme.Codecs

        fmt.Println("Init RESTClient.")

        // 定义RestClient，用于与k8s API server进行交互
        restClient, err := rest.RESTClientFor(config)
        if err != nil {
                panic(err)
        }

        fmt.Println("Get Pods in cluster.")

        // 获取pod列表。这里只会从namespace为"kube-system"中获取指定的资源(pods)
        result := &corev1.PodList{}
        if err := restClient.
                Get().
                Namespace("kube-system").
                Resource("pods").
                VersionedParams(&metav1.ListOptions{Limit: 500}, scheme.ParameterCodec).
                Do(context.TODO()).
                Into(result); err != nil {
                panic(err)
        }

        fmt.Println("Print all listed pods.")

        // 打印所有获取到的pods资源，输出到标准输出
        for _, d := range result.Items {
                fmt.Printf("NAMESPACE: %v NAME: %v \t STATUS: %v \n", d.Namespace, d.Name, d.Status.Phase)
        }
}
```

`go.mod`

```bash
module main

go 1.15

require (
        github.com/go-logr/logr v0.2.1 // indirect
        github.com/google/gofuzz v1.2.0 // indirect
        github.com/imdario/mergo v0.3.11 // indirect
        golang.org/x/crypto v0.0.0-20201002170205-7f63de1d35b0 // indirect
        golang.org/x/net v0.0.0-20201010224723-4f7140c49acb // indirect
        golang.org/x/oauth2 v0.0.0-20200902213428-5d25da1a8d43 // indirect
        golang.org/x/sys v0.0.0-20201009025420-dfb3f7c4e634 // indirect
        golang.org/x/time v0.0.0-20200630173020-3af7569d3a1e // indirect
        k8s.io/api v0.19.2
        k8s.io/apimachinery v0.19.2
        //k8s.io/client-go v11.0.0+incompatible
        //      k8s.io/client-go v11.0.0+incompatible
        k8s.io/klog v1.0.0 // indirect
        k8s.io/klog/v2 v2.3.0 // indirect
        k8s.io/utils v0.0.0-20201005171033-6301aaf42dc7 // indirect
)

require k8s.io/client-go v0.19.2
```

## 3. 输出结果

```bash
[root@xxx-wsl ~/client-go-example] go run main.go
Prepare config object.
Init RESTClient.
Get Pods in cluster.
Print all listed pods.
NAMESPACE: kube-system NAME: coredns-f9fd979d6-rhzfd     STATUS: Running
NAMESPACE: kube-system NAME: coredns-f9fd979d6-whrj2     STATUS: Running
NAMESPACE: kube-system NAME: etcd-kind-control-plane     STATUS: Running
NAMESPACE: kube-system NAME: etcd-kind-control-plane2    STATUS: Running
NAMESPACE: kube-system NAME: etcd-kind-control-plane3    STATUS: Running
NAMESPACE: kube-system NAME: kindnet-bpsfl       STATUS: Running
NAMESPACE: kube-system NAME: kindnet-ks6zv       STATUS: Running
NAMESPACE: kube-system NAME: kindnet-pm6zl       STATUS: Running
NAMESPACE: kube-system NAME: kindnet-qfhqt       STATUS: Running
NAMESPACE: kube-system NAME: kindnet-s7qqn       STATUS: Running
NAMESPACE: kube-system NAME: kindnet-trk5l       STATUS: Running
NAMESPACE: kube-system NAME: kube-apiserver-kind-control-plane   STATUS: Running
NAMESPACE: kube-system NAME: kube-apiserver-kind-control-plane2          STATUS: Running
NAMESPACE: kube-system NAME: kube-apiserver-kind-control-plane3          STATUS: Running
NAMESPACE: kube-system NAME: kube-controller-manager-kind-control-plane          STATUS: Running
NAMESPACE: kube-system NAME: kube-controller-manager-kind-control-plane2         STATUS: Running
NAMESPACE: kube-system NAME: kube-controller-manager-kind-control-plane3         STATUS: Running
NAMESPACE: kube-system NAME: kube-proxy-7gz67    STATUS: Running
NAMESPACE: kube-system NAME: kube-proxy-bvbkk    STATUS: Running
NAMESPACE: kube-system NAME: kube-proxy-clf72    STATUS: Running
NAMESPACE: kube-system NAME: kube-proxy-d8zpb    STATUS: Running
NAMESPACE: kube-system NAME: kube-proxy-dsmsj    STATUS: Running
NAMESPACE: kube-system NAME: kube-proxy-fplkk    STATUS: Running
NAMESPACE: kube-system NAME: kube-scheduler-kind-control-plane   STATUS: Running
NAMESPACE: kube-system NAME: kube-scheduler-kind-control-plane2          STATUS: Running
NAMESPACE: kube-system NAME: kube-scheduler-kind-control-plane3          STATUS: Running
```