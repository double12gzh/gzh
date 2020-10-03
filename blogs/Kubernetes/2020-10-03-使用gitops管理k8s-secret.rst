.. post:: Sep 27, 2020
   :author: 大海星
   :category: Kubernetes
   :location: BJ
   :tags: kubernetes, gitops
.. :excerpt: 1

使用GitOps管理K8S Secret
============================

|image0|

1. 写在前面
-----------

GitOps是伴随着云原生产生的一个新的概念，它的核心是以一种\ ``声明式的``\ 方式管理资源，表示当前的状态，让你在任何时候都能知道git中的情况，并将这种声明式的状态解析 为集群。

我们在GitOps上犯的最大错误是\ ``结构``\ 。仓库的结构至关重要，你选择如何在公司里组织GitOps，将决定它的成败。

当你解决了结构问题之后，我们面临的下一个最大的问题就是\ ``如何保证Secret的安全``\ 。一般来说，最后的结果是人们有不需要的访问权限，或者有一个共享的\ ``Secret``\ ，通过Slack或1Password传来传去。

`Sealed
Secrets <https://github.com/bitnami-labs/sealed-secrets>`__\ 是Kubernetes上解决\ ``Secret``\ 的一种流行方式，它是一个不错的解决方案，依靠\ ``PKI``\ 和\ ``共享公钥``\ 来加密，并在集群上安装私钥。

本文将尝试解释如何使用GitOps来管理k8s的\ ``Secret``\ 资源。

2. 管理Secret比较好的方式
-------------------------

`Mozilla的SOPS <https://github.com/mozilla/sops>`__\ 。

SOPS支持多种来源的密钥输入，包括AWS KMS、GCE、Vault、PGP等。它提供了使用AWS KMS的外部密钥作为输入来加密和解密秘密的能力，但它也支持\ `Shamir的\ ``Secret`` <https://en.wikipedia.org/wiki/Shamir's_Secret_Sharing>`__\ 共享和密钥组，本质上允 许对解密所需的密钥以及这些密钥的数量进行策略。

由于SOPS让我们有能力使用AWS KMS这样的服务，我们可以控制谁有加密和解密的权限，但这也意味着我们可以利用AWS的实例配置文件，让机器做它们设计时最擅长的事情，通过自动化让我们的生活更轻松。

最后一个我想提请大家注意的功能是，一旦文件被加密，所有关于如何加密的元数据都是文件本身的一部分，不需要第三方服务、数据库、隐藏目录或二级文件来知道如何解密文件。

   Flux对SOPS有一流的支持。

3. 结构
-------

一开始我就说过，正确架构你的仓库是至关重要的。这一点来源于个人在工程实践中总结的一些经验之谈，对于不同的场景，您可能需要权衡一下我这里的“最佳实践”。

对于\ ``结构化``\ 化来说，通常可以简单的分成两个部分：

-  仓库中内容的布局
-  使用仓库的数量

在我的项目中，我通常会使用三个库： - 定义集群运行所需的每一个基础资源 -
处理工程团队负责的产品的所有发布 - 给\ ``Secret``\ 使用

项目中我只使用了SOPS与专用的\ ``Secret``\ 库，使用Flux和SOPS使我们获得了这种能力，具有前所未有的灵活性。

3.1 目录结构
~~~~~~~~~~~~

.. code:: bash

   |-- .sops.yaml
   |-- secrets
       |-- dev (environment/cluster)
           |-- database.yaml
       |-- stg (environment/cluster)
           |-- database.yaml
       |-- prd (environment/cluster)
           |-- database.yaml

这个文件布局为我们提供了使用SOPS创建规则来控制文件如何加密的基础。我们可以使用不同的密钥、密钥组和阈值。它还允许我们告诉Flux在克隆和解析集群上需要的东西时只关心一个特定的路径。下面是一个例子(注意：这不是一个完整可运行的例子)。

.. code:: yaml

   creation_rules:
   - path_regex: secrets/dev/.*
     encrypted_regex: "^(data|stringData)$"
     kms:
       arn: kms-arn
       role: ksm-role
   - path_regex: secrets/stg/.*
     encrypted_regex: "^(data|stringData)$"
     kms:
       arn: kms-arn
       role: ksm-role
   - path_regex: secrets/prd/.*
     encrypted_regex: "^(data|stringData)$"
     kms:
       arn: kms-arn
       role: ksm-role

3.2 Scaling
~~~~~~~~~~~

使用这个设置，我们几乎可以无限地扩展这个设置。它只受存储在版本库中的数据量和服务于它的git服务器的限制。秘密一般不会经常变化，但它们会被轮换，这就是像\ `Reloader <https://github.com/stakater/Reloader>`__\ 这样的额外工具的作用(这里不会作详细介绍)。

   Reloader:

   它可以用于监视ConfigMap和Secret是否发生变化，如果发生了变化，它将触发与之相关的以下资源实现滚动更新：

    -  DeploymentConfig
    -  Deployment
    -  Daemonset
    -  Statefulset

4. 设置SOPS
-----------

前面我们已经讲解了SOPS的目录结构和设计原则，下面我们通过一个小例子实践一下。

有多种方法可以利用SOPS，但我只介绍其中一种：使用AWS KMS与多个键和AWS Instance Profile(Flux可以使用)。

4.1 设置AWS
~~~~~~~~~~~

在真正的使用场景中，我会建议使用多个AWS账户，并假设角色权限，但对于这样的文章来说，这可能会变得不必要的复杂，所以我们要假设一个单一的AWS账户，但仍然利用使用3个密 钥。

.. code:: bash

   aws kms create-key

..

   一定要把ARN收集起来，以便以后使用。

接下来我们需要创建我们的\ ``.sops.yaml``\ 文件与创建规则。在真正的使用场景中，你会希望你的阈值是2或更多，并且让你的密钥至少比阈值高N+1。
让我们从创建一个基础目录开始，然后创建\ ``.sops.yaml``\ 文件，内容如下，确保用你的有效ARNs交换ARNs。

.. code:: yaml

   creation_rules:
   - path_regex: secrets/dev/.*
     encrypted_regex: "^(data|stringData)$"
     shamir_threshold: 2
     key_groups:
       - kms:
           - arn: arn:aws:kms:us-west-2:000000000000:key/b5d44bf0-7ec5-49a9-b404-bc4d8b4036fb
       - kms:
           - arn: arn:aws:kms:us-west-2:000000000000:key/16d44186-2393-40d9-90e1-9a2f92fd5863
       - kms:
           - arn: arn:aws:kms:us-west-2:000000000000:key/2120d2c1-a89e-4aeb-844f-f17ae2abd210

这个创建规则规定，secrets/dev目录下的所有文件都要用3个不同的密钥进行加密，解密阈值为2。

接下来创建你的secrets/dev目录，并创建一个example.yaml文件，内容如下：

.. code:: yaml

   apiVersion: v1
   kind: Secret
   metadata:
     name: example
   type: Opaque
   data:
     username: gzhGZHSBWUDESIN=
     password: SBALIBSBWUDESIN=

使用如下命令结上面的yaml文件进行加密：

.. code:: bash

   sops -e -i secrets/dev/example.yaml

..

   说明:

    -  -e: 加密
    -  -i: 支持原地修改(如果您不想修改原文件并且想看加密后的输入内容，请忽略此参数)

上面的命令执行完成后您就可以提交到仓库中了。

4.2 AWS实例配置文件
~~~~~~~~~~~~~~~~~~~

这个系统的好处是，\ ``Flux``\ 可以简单地利用AWS实例配置文件，通过STS承担角色，并获得临时的密钥，以便在集群本身运行时能够解密。
要实现这一点，您需要设置一个角色，该角色具有\ ``Encrypt/Decrypt``\ 和可能的\ ``Generate``\ 权限，具体取决于您使用的密钥类型，并设置您的AWS实例以使用配置文件。

一旦您完成了这些设置，只需使用正确的 CLI 选项将 Flux 安装到群集上，Flux就会让您的群集保持最新状态。

4.3 设置Flux
~~~~~~~~~~~~

你只需要做几件事情来确保\ ``Flux``\ 的正常运行，你需要启用\ ``SOPS``\ ，这取决于你的部署方法，可能是\ ``配置选项``\ 或CLI标志\ ``--sops``\ 。你需要启用
``SOPS``\ ，根据你的部署方法，可能是一个\ ``配置选项``\ 或 CLI 标志``--sops``\ 。你还需要指示你的 ``Flux``实例只关心默认分支中的一个特定路径。\ ``--git-path`` 需要设置为
``Secrets`` 下的一个目录，比如 ``--git-path=secrets/dev``\ 。

5. 小结
-------

使用这种方法可以让你更好地控制\ ``Secret``\ 及谁有访问权，另外，它也允许您让AWS处理提供\ ``Secret``\ 中比较难搞的部分，以通过自动化和短暂的\ ``Secret``\ 来访问\ ``Secret``\ 解密，您可以将所有的\ ``Secret``\ 保存在一个地方，以便于管理和轮换。

.. |image0| image:: https://gitee.com/double12gzh/wiki-pictures/raw/master/2020-10-03-gitops-manage-k8s-secret.jpg