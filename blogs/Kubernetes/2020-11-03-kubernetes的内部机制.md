---
blogpost: true
date: Nov 08, 2020
author: JeffreyGuan
location: BJ
category: Kubernetes
tags: kubernetes, k8s源码分析
language: English
---

# kubernetes内部机制

摘要：kubernetes内部机制简介。

![](https://gitee.com/double12gzh/wiki-pictures/raw/master/2020-10-13-client-go-kubeconfig/0.png)

## 1. 写在前面

> 个人主页: https://gzh.readthedocs.io
> 
> 关注容器技术、关注`Kubernetes`。
> 
> 问题或建议，请公众号（`double12gzh`）留言。

Kubernetes是一个容器编排引擎，设计用于在一组节点（通常称为集群）上托管容器化应用。使用系统建模方法，本系列旨在推进对Kubernetes及其基础概念的理解。
对于本篇博文，建议您需要提前对Kubernetes、Kubernetes对象和Kubernetes控制器有所理解。

Kubernetes的特点是声明式容器编排引擎：在声明式系统中，用户向系统提供系统的期望状态的表示。然后，系统考虑当前状态和期望状态，确定从当前状态过渡到期望状态的需要执行哪些命令序列。
因此，"声明式系统 "这个术语激发了一个经过计算的、具有明确目的的协调工作的概念，其最终目的是以便从当前状态过渡到期望状态。

然而，这不是Kubernetes的实际工作方式!

Kubernetes并没有根据当前状态和期望状态确定一个经过计算、协调的命令执行序列。相反，Kubernetes仅根据当前状态迭代确定下一个要执行的命令。如果以及无法确定下一条命令时，Kubernetes就达到了稳定状态。

## 2. 状态转换机制

本段概述了Kubernetes的状态转换语义的抽象模型。接下来的段落概述了一个基于部署对象和部署控制器的具体示例。

![图1](https://gitee.com/double12gzh/wiki-pictures/raw/master/2020-11-03-k8s-mechanics/1.png)

```als
fact {
    all k8s : K8s - last | let k8s' = k8s.next {
        some c : NextCommand[k8s] {
            command.source = k8s and command.target = k8s'
        }
    }
    NextCommand[k8s.last] = none
}
```

上述代码展示了Kubernetes的状态转换语义。给定下一个命令函数，系统将根据当前状态k8s确定下一个命令，将系统从当前状态k8s过渡到下一个状态k8s'。

上述代码中使用到的`NextCommand()`的实现逻辑如下：

```als
fun NextCommand(k8s : K8s) : set Command {
  DeploymentController.NextCommand[k8s] +
  ReplicaSetController.NextCommand[k8s] +
  ...
}
```

从概念上讲，`NextCommand`函数是由每个Kubernetes控制器的`NextCommand`函数的组成。

```als
pred Steady(k8s : K8s) { NextCommand[k8s] = none }

```

状态序列以一个状态k8s.last结束，对于这个状态，`NextCommand`函数没有产生下一条命令，这个状态通常称为稳态。

## 3. k8s对象

Kubernetes对象存储是一组Kubernetes对象。Kubernetes对象是一些有不同规格的数据记录，称为`kind`。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gzh-deployment
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: mydemo
        image: busybox:latest
```

如上所示即为一种k8s对象。

## 4. k8s控制器

每个Kubernetes Controller都会为下一条命令产生输入，一个Controller被实现为一个连续的过程，它根据Kubernetes的当前状态产生后续的命令。

```tla
process Controller = "Deployment Controller"
begin
    ControlLoop:
      while TRUE do
        \* The Deployment Controller monitors Deployment Objects
        with d ∈ {d ∈ k8s: d.kind = "Deployment"} do
          \* 1. Enabling Condition
          if Cardinality({r \in k8s: r.kind = "ReplicaSet" ∧ match(d.spec.labelSelector, r.meta.labels)}) < 1 then
            \* Reconciling Command
            CREATE([kind |-> "ReplicaSet", spec |-> [replicas |-> d.spec.replicas, template |-> d.spec.template]]);
          end if;
          \* 2. Enabling Condition
          if Cardinality({r \in k8s: r.kind = "ReplicaSet" ∧ match(d.spec.labelSelector, r.meta.labels)}) > 1 then
            \* Reconciling Command
            with r ∈ {r \in k8s: r.kind = "ReplicaSet" ∧ match(d.spec.labelSelector, r.meta.labels)} do
               DELETE(r);
            end with;
          end if;
        end with;
      end while;
end process;
```

上述代码5说明了deployment控制器。控制器监控deployment对象，并对每个对象执行一组条件语句。

* 条件

如果匹配的ReplicaSet对象少于1个

* 命令
然后，deployment控制器将产生一个`创建 ReplicaSet `命令。

* 条件

如果有超过1个匹配的ReplicaSet对象。

* 命令
然后，deployment控制器将发出`删除 ReplicaSet 命令`。

从Controller的角度来看，如果Controller的条件都没有启用，Kubernetes就处于稳定状态，也就是说，Controller不会产生任何命令。

## 5. 级联指令

控制器（可以）级联地相互启用:

* 给定k8s的状态，如果Kubernetes控制器C被启用，C将执行一个过渡到k8s'的命令。
* 给定k8s'的状态，如果启用了Kubernetes控制器C'，C'将执行一个过渡到k8s''的命令。

![图2](https://gitee.com/double12gzh/wiki-pictures/raw/master/2020-11-03-k8s-mechanics/2.png)

图2显示了用户向API服务器提交deployment对象后产生的命令级联。

## 6. k8s是一个声明式系统吗？

![图3](https://gitee.com/double12gzh/wiki-pictures/raw/master/2020-11-03-k8s-mechanics/3.png)

```als
fact {
    all sys : Sys - last | let sys' = sys.next {
        some c : Command {
            command.source = sys and command.target = sys'
        }
    }
    Desired[sys.last]
}
```

上述代码定义了声明式系统的状态转换机制。给定一个期望状态谓词，系统将确定一个命令序列，使系统从当前状态k8s.first过渡到期望状态k8s.last。

![图4](https://gitee.com/double12gzh/wiki-pictures/raw/master/2020-11-03-k8s-mechanics/4.png)

如果我们不把Kubernetes对象解释为事实的记录，而是解释为意图的记录，那么我们就可以认同Kubernetes是一个声明式系统的概念。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gzh-deployment
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: mydemo
      image: busybox:latest
```

例如，我们可以将代码中的deployment对象解释为存在一组3个副本对象的意图，因此对象的

* `.spec.container[0].image`等于`busyBox:latest`

然而，这一概念并非没有微妙之处：如果将一个对象解释为意图记录，则会出现多个选项。例如，Deployment Object 可以解释为：

* 应有一个 ReplicaSet 或
* 有一些pods

根据解释，当前状态可能与所需状态相匹配，也可能不相匹配:

* 如果有ReplicaSet和可选的Pods或
* 如果有一个ReplicaSet，并且必须有一组Pods

与我们的解释无关

* 如果有ReplicaSet对象，K8s与部署对象的关系处于稳定状态（deployment控制器不会产生Commands）。
* 如果有一组pod对象，K8s与ReplicaSet对象的关系处于稳定状态（ReplicaSet控制器不会产生命令）。

## 7. 结论

Kubernetes可能被描述为一个声明式系统，而Kubernetes对象可能被描述为意向记录。
然而，当你推理Kubernetes和Kubernetes的行为时，你应该记住，Kubernetes不会做出协调的努力来过渡到所需的状态。
相反，Kubernetes会做出持续的、不协调的努力来过渡到稳定状态。


---------------

欢迎关注我的微信公众号：

![](https://gitee.com/double12gzh/wiki-pictures/raw/master/wechat_public.jpg)