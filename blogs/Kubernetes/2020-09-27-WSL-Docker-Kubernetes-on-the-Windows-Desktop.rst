.. post:: Sep 27, 2020
   :author: 大海星
   :category: Kubernetes
   :location: BJ
   :tags: kubernetes, wsl, docker, win10, desktop
.. :excerpt: 1

在windows上通过wsl来部署k8s
==============================

摘要：是Windows10和WSL2的新手，还是Docker和Kubernetes的新手？欢迎来到这篇博文，我们将在Docker KinD和Minikube中从头开始安装Kubernetes。

1. 写在前面
-----------

为什么要在windows上使用kubernetes?

在过去的几年里，Kubernetes成为了在分布式环境中运行容器化服务和应用的事实标准平台。虽然存在各种各样的发行版和安装程序来部署Kubernetes在云环境（公共的、私有的或混合的），或在裸机环境中，但仍然需要在本地部署和运行Kubernetes，例如，在开发者的工作站上。

Kubernetes最初被设计为在Linux环境中部署和使用。然而，有不少用户（不仅是应用开发者）使用Windows操作系统作为日常工具。当微软揭示了WSL--Windows Subsystem for Linux时，Windows和Linux环境之间的界限变得更加不明显。

同时，WSL还带来了在Windows上几乎无缝运行Kubernetes的能力!

下面，我们将简要介绍如何安装和使用各种解决方案在本地运行Kubernetes。

2. 前提条件
-----------

尽管我们将会安装\ `Kind <https://kind.sigs.k8s.io/>`__\ ，但是，讲解太多关于如何安装的细节，如果您对这个比较关注，请关注我后续的文章。

首先，我们先看一下我们继续进行下面的操作前，我们的最小的软件版本的需求:

-  OS：Win10 version 2004, build 19041。
-  WSL2开启。安装完成后，可以使用\ ``wsl.exe --set-default-version 2``\ 设置使用WSL2。
-  WSL2可以通过win应用商店来安装，这里使用的是Ubuntu 18.04。
-  `Docker-desktop-for-windows <https://hub.docker.com/editions/community/docker-ce-desktop-windows>`__\ 。
-  (可选)Microsoft Terminal。可以从windows应用商店进行安装。

.. figure:: https://gitee.com/double12gzh/wiki-pictures/raw/master/2020-09-27-wsl_docker_kubernetes_win10/0.png
   :alt: 

.. tip:: 
   如何查看win10的版本信息？
    -  win+R
    -  输入winver

3. WSL2：首次接触
-----------------

上面的软件都安装完成后，我们就可以从\ ``开始->Ubuntu``\ 来打开wsl2了。

.. figure:: https://gitee.com/double12gzh/wiki-pictures/raw/master/2020-09-27-wsl_docker_kubernetes_win10/1.png
   :alt: 

打开后，你完全可以把它看成是一个正常的Linux的发行，同样的，你也可以在里面设置用户等。

.. figure:: https://gitee.com/double12gzh/wiki-pictures/raw/master/2020-09-27-wsl_docker_kubernetes_win10/2.png
   :alt: 

3.1 (可选)设置``sudoers``
~~~~~~~~~~~~~~~~~~~~~~~~~

由于我们是在本地计算机上工作，通常情况下，更新sudoers，并将%sudo组设置为无密码组，可能会更好。

.. code:: bash

    # 用visudo打开sudoer文件
    sudo visudo

    # 设置sudo group免密登陆
    %sudo   ALL=(ALL:ALL) NOPASSWD: ALL

    # 按 CTRL+X 退出
    # 按 Y 保存
    # 按 Enter 确认

.. figure:: https://gitee.com/double12gzh/wiki-pictures/raw/master/2020-09-27-wsl_docker_kubernetes_win10/3.png
   :alt: 

3.2 更新Ubuntu
~~~~~~~~~~~~~~

在进行后面的操作前，我们先升级一下我们的系统。

.. code:: bash

    # 更新apt-get源
    sudo apt update
    # 基于最新的源更新系统。automatically
    sudo apt upgrade -y

.. figure:: https://gitee.com/double12gzh/wiki-pictures/raw/master/2020-09-27-wsl_docker_kubernetes_win10/4.png
   :alt: 

4. Docker Desktop
-----------------

Docker-Windows-Desktop与WSL2更配哦。

在我们进入设置之前，让我们做一个小测试，它将显示出与Docker Desktop的新集成是多么的酷。

.. code:: bash

    # 看一下docker的版本及docker是否被安装
    docker version
    # 看一下kubectl的版本及kubectl是否被安装
    kubectl version

.. figure:: https://gitee.com/double12gzh/wiki-pictures/raw/master/2020-09-27-wsl_docker_kubernetes_win10/5.png
   :alt: 

你有遇到错误吗？有错误就对了，所以现在让我们继续进行设置。

4.1 设置docker desktop：开启对WSL2的支持
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

如果还没有打开Docker-For-Windows的话，首先我们来启动它。打开Windows的开始菜单，输入"docker"，点击名称启动应用程序。

.. figure:: https://gitee.com/double12gzh/wiki-pictures/raw/master/2020-09-27-wsl_docker_kubernetes_win10/6.png
   :alt: 

现在在你的任务栏中就能看到它的图标了。

.. figure:: https://gitee.com/double12gzh/wiki-pictures/raw/master/2020-09-27-wsl_docker_kubernetes_win10/7.png
   :alt: 

下面我们在docker-for-windows的中设置一下，让他使用wsl2:

.. figure:: https://gitee.com/double12gzh/wiki-pictures/raw/master/2020-09-27-wsl_docker_kubernetes_win10/8.png
   :alt: 

这个选项默认情况下并没有开启:

.. figure:: https://gitee.com/double12gzh/wiki-pictures/raw/master/2020-09-27-wsl_docker_kubernetes_win10/9.png
   :alt: 

这个功能在幕后所做的就是在WSL2中创建了两个新的发行版，包含并运行所有需要的后端socket、守护进程以及CLI工具（如：docker和kubectl命令）。

但是，第一种设置仍然不足以在我们的发行版中运行命令。如果我们尝试，我们会遇到和之前一样的错误。

为了解决这个问题，并最终能够使用这些命令，我们需要告诉Docker桌面也要"附加 "到我们的发行版上。

.. figure:: https://gitee.com/double12gzh/wiki-pictures/raw/master/2020-09-27-wsl_docker_kubernetes_win10/10.png
   :alt: 

接下来，我们再看一下命令是否正常了呢：

.. code:: bash

    # 看一下docker的版本及docker是否被安装
    docker version
    # 看一下kubectl的版本及kubectl是否被安装
    kubectl version

.. figure:: https://gitee.com/double12gzh/wiki-pictures/raw/master/2020-09-27-wsl_docker_kubernetes_win10/11.png
   :alt: 

    说明:

    -  如果没有发生任何事情，重启Docker
       Desktop并在Powershell中重启WSL进程。重启服务LxssManager并启动一个新的Ubuntu会话。
    -  如果您的docker-desktop-for-win中没有\ ``Use the WSL2 based engine``\ 的，请重新安装它的\ ``edge``\ 版本。

5. Kind: 让k8s方便的运行在容器中
--------------------------------

现在，我们的Docker已经安装好了，配置好了，上次的测试也很正常。

但是，如果我们仔细观察kubectl命令，它找到了"客户端版本"（1.15.5），但它没有找到任何服务器。

这很正常，因为我们没有启用Docker Kubernetes集群。所以让我们安装KinD并创建我们的第一个集群。

我们将按照KinD官方网站上的\ `教程（部分） <https://kind.sigs.k8s.io/docs/user/quick-start/>`__\ 进行操作。

.. code:: bash

    # 下载 KinD
    curl -Lo ./kind https://github.com/kubernetes-sigs/kind/releases/download/v0.7.0/kind-$(uname)-amd64
    # 增加可执行权限
    chmod +x ./kind
    # 移动到系统PATH中
    sudo mv ./kind /usr/local/bin/

.. figure:: https://gitee.com/double12gzh/wiki-pictures/raw/master/2020-09-27-wsl_docker_kubernetes_win10/12.png
   :alt: 

5.1 创建单节点的集群
~~~~~~~~~~~~~~~~~~~~

下面，我们将会创建我们的第一个集群：

.. code:: bash

    # 检查 KUBECONFIG 是否存在
    echo $KUBECONFIG
    # Check if the .kube directory is created > if not, no need to create it
    ls $HOME/.kube
    # Create the cluster and give it a name (optional)
    kind create cluster --name wslkind
    # Check if the .kube has been created and populated with files
    ls $HOME/.kube

.. figure:: https://gitee.com/double12gzh/wiki-pictures/raw/master/2020-09-27-wsl_docker_kubernetes_win10/13.png
   :alt: 

    说明：对于执行的每一步，都会有一个小图标与之显示。

运行完上面的命令后，k8s集群就已经创建好了，因为我们使用是docker desktop，所以，里面的网络默认使用的是\ ``as is``\ 。

我们可以在浏览器中打开\ ``kubernetes master``\ 的页面：

.. figure:: https://gitee.com/double12gzh/wiki-pictures/raw/master/2020-09-27-wsl_docker_kubernetes_win10/14.png
   :alt: 

而这才是Docker Desktop for Windows与WSL2后台的真正优势。Docker真的做了一个惊人的整合。

5.2 创建多节点的集群
~~~~~~~~~~~~~~~~~~~~

上一节中，我们刚创建的那是一个单节点的集群。

.. code:: bash

    # Check how many nodes it created
    kubectl get nodes
    # Check the services for the whole cluster
    kubectl get all --all-namespaces

.. figure:: https://gitee.com/double12gzh/wiki-pictures/raw/master/2020-09-27-wsl_docker_kubernetes_win10/15.png
   :alt: 

虽然那也已经能满足大多数人的需求了，但是，下面我们还是来继续看一下它另外一个更加酷炫的能力：创建多节点的集群。

.. code:: bash


    # Delete the existing cluster
    kind delete cluster --name wslkind
    # Create a config file for a 3 nodes cluster
    cat << EOF > kind-3nodes.yaml
    kind: Cluster
    apiVersion: kind.x-k8s.io/v1alpha4
    nodes:
      - role: control-plane
      - role: worker
      - role: worker
    EOF
    # Create a new cluster with the config file
    kind create cluster --name wslkindmultinodes --config ./kind-3nodes.yaml
    # Check how many nodes it created
    kubectl get nodes

.. figure:: https://gitee.com/double12gzh/wiki-pictures/raw/master/2020-09-27-wsl_docker_kubernetes_win10/16.png
   :alt: 

    提示：根据我们运行 "get nodes"命令的速度，可能不是所有的节点都准备好了，等几秒钟再运行，一切都应该准备好了。

到此，我们已经创建完了三节点的一个集群，我们通过下面的命令再来看一下，如果你仔细看你会发现，每一个服务都是三副本的了。

.. code:: bash

    # Check the services for the whole cluster
    kubectl get all --all-namespaces

.. figure:: https://gitee.com/double12gzh/wiki-pictures/raw/master/2020-09-27-wsl_docker_kubernetes_win10/17.png
   :alt: 

5.3 查看k8s Dashboard
~~~~~~~~~~~~~~~~~~~~~

对于开发人员来说，在命令行上执行操作是可以接受，并且这样效率也比较高。然而，在处理Kubernetes时，我们可能希望在某些时候有一个可视化的概述。为此，我们创建了\ `Kubernetes
Dashboard <https://github.com/kubernetes/dashboard>`__\ 项目。安装和第一次连接测试是相当快的，所以我们来做吧。

.. code:: bash

    # Install the Dashboard application into our cluster
    kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-rc6/aio/deploy/recommended.yaml
    # Check the resources it created based on the new namespace created
    kubectl get all -n kubernetes-dashboard

.. figure:: https://gitee.com/double12gzh/wiki-pictures/raw/master/2020-09-27-wsl_docker_kubernetes_win10/18.png
   :alt: 

虽然kind使用\ ``ClusterIP``\ 为集群创建了一个\ ``service``\ ，但是由于这个IP是一个内网的IP，所以我们现在还无法从页面上直接访问\ ``dashboard``\ 服务。

.. figure:: https://gitee.com/double12gzh/wiki-pictures/raw/master/2020-09-27-wsl_docker_kubernetes_win10/19.png
   :alt: 

这个问题可以通过创建一个\ ``proxy``\ 来解决：

.. code:: bash

    # Start a kubectl proxy
    kubectl proxy
    # Enter the URL on your browser: http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/

.. figure:: https://gitee.com/double12gzh/wiki-pictures/raw/master/2020-09-27-wsl_docker_kubernetes_win10/20.png
   :alt: 

最后要登录，我们可以输入一个我们没有创建的Token，或者从我们的Cluster中输入kubeconfig文件。

如果我们尝试用kubeconfig登录，我们会得到错误 "Internal error (500): Not enough data to create auth info structure"。没有足够的数据来创建Auth信息结构"。这是由于kubeconfig文件中缺乏凭证。

因此，为了避免你以同样的错误结束，让我们按照推荐的RBAC方法。

让我们打开一个新的WSL2会话。

.. code:: bash

    # Create a new ServiceAccount
    kubectl apply -f - <<EOF
    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: admin-user
      namespace: kubernetes-dashboard
    EOF
    # Create a ClusterRoleBinding for the ServiceAccount
    kubectl apply -f - <<EOF
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRoleBinding
    metadata:
      name: admin-user
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: ClusterRole
      name: cluster-admin
    subjects:
    - kind: ServiceAccount
      name: admin-user
      namespace: kubernetes-dashboard
    EOF

.. figure:: https://gitee.com/double12gzh/wiki-pictures/raw/master/2020-09-27-wsl_docker_kubernetes_win10/21.png
   :alt: 

.. code:: bash

    # Get the Token for the ServiceAccount
    kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep admin-user | awk '{print $1}')
    # Copy the token and copy it into the Dashboard login and press "Sign in"

.. figure:: https://gitee.com/double12gzh/wiki-pictures/raw/master/2020-09-27-wsl_docker_kubernetes_win10/22.png
   :alt: 

.. figure:: https://gitee.com/double12gzh/wiki-pictures/raw/master/2020-09-27-wsl_docker_kubernetes_win10/23.png
   :alt: 

6. minikube: 让k8s可任意部署
----------------------------

现在，我们的Docker已经安装好了，配置好了，上次的测试也很正常。

但是，如果我们仔细观察kubectl命令，它找到了 "Client Version"（1.15.5），但它没有找到任何服务器。

这很正常，因为我们没有启用Docker Kubernetes集群。所以让我们安装Minikube并创建我们的第一个集群。

我们将按照\ `Kubernetes.io网站上的方法（部分） <https://kubernetes.io/docs/tasks/tools/install-minikube/>`__\ 进行操作。

.. code:: bash

    # Download the latest version of Minikube
    curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
    # Make the binary executable
    chmod +x ./minikube
    # Move the binary to your executable path
    sudo mv ./minikube /usr/local/bin/

.. figure:: https://gitee.com/double12gzh/wiki-pictures/raw/master/2020-09-27-wsl_docker_kubernetes_win10/24.png
   :alt: 

6.1 更新Host
~~~~~~~~~~~~

如果我们按照how-to，它提示我们应该使用--driver=none标志，以便直接在主机和Docker上运行Minikube。

不幸的是，我们会得到一个关于运行Kubernetes v 1.18需要 "conntrack"的错误信息

.. code:: bash

    # Create a minikube one node cluster
    minikube start --driver=none

.. figure:: https://gitee.com/double12gzh/wiki-pictures/raw/master/2020-09-27-wsl_docker_kubernetes_win10/25.png
   :alt: 

根据提示，我们是缺少了一个包，下面我们就把它安装上：

.. code:: bash

    # Install the conntrack package
    sudo apt install -y conntrack

安装完成后，再次启动：

.. code:: bash

    # Create a minikube one node cluster
    minikube start --driver=none
    # We got a permissions error > try again with sudo
    sudo minikube start --driver=none

.. figure:: https://gitee.com/double12gzh/wiki-pictures/raw/master/2020-09-27-wsl_docker_kubernetes_win10/26.png
   :alt: 

这次是不是没有出错呢。

6.2 启用\ ``systemd``
~~~~~~~~~~~~~~~~~~~~~

为了能够使用\ ``systemd``\ ，我们需要按照这个\ `脚本 <https://github.com/DamionGans/ubuntu-wsl2-systemd-script>`__\ 来配置一下。

    说明：如果您想了解更多关于这个脚本背后的考虑，请\ `参考 <https://forum.snapcraft.io/t/running-snaps-on-wsl2-insiders-only-for-now/13033>`__

在终端中输入以下命令：

.. code:: bash


    # Install the needed packages
    sudo apt install -yqq daemonize dbus-user-session fontconfig

.. figure:: https://gitee.com/double12gzh/wiki-pictures/raw/master/2020-09-27-wsl_docker_kubernetes_win10/27.png
   :alt: 

.. code:: bash

    # Create the start-systemd-namespace script
    sudo vi /usr/sbin/start-systemd-namespace
    #!/bin/bash

    SYSTEMD_PID=$(ps -ef | grep '/lib/systemd/systemd --system-unit=basic.target$' | grep -v unshare | awk '{print $2}')
    if [ -z "$SYSTEMD_PID" ] || [ "$SYSTEMD_PID" != "1" ]; then
        export PRE_NAMESPACE_PATH="$PATH"
        (set -o posix; set) | \
            grep -v "^BASH" | \
            grep -v "^DIRSTACK=" | \
            grep -v "^EUID=" | \
            grep -v "^GROUPS=" | \
            grep -v "^HOME=" | \
            grep -v "^HOSTNAME=" | \
            grep -v "^HOSTTYPE=" | \
            grep -v "^IFS='.*"$'\n'"'" | \
            grep -v "^LANG=" | \
            grep -v "^LOGNAME=" | \
            grep -v "^MACHTYPE=" | \
            grep -v "^NAME=" | \
            grep -v "^OPTERR=" | \
            grep -v "^OPTIND=" | \
            grep -v "^OSTYPE=" | \
            grep -v "^PIPESTATUS=" | \
            grep -v "^POSIXLY_CORRECT=" | \
            grep -v "^PPID=" | \
            grep -v "^PS1=" | \
            grep -v "^PS4=" | \
            grep -v "^SHELL=" | \
            grep -v "^SHELLOPTS=" | \
            grep -v "^SHLVL=" | \
            grep -v "^SYSTEMD_PID=" | \
            grep -v "^UID=" | \
            grep -v "^USER=" | \
            grep -v "^_=" | \
            cat - > "$HOME/.systemd-env"
        echo "PATH='$PATH'" >> "$HOME/.systemd-env"
        exec sudo /usr/sbin/enter-systemd-namespace "$BASH_EXECUTION_STRING"
    fi
    if [ -n "$PRE_NAMESPACE_PATH" ]; then
        export PATH="$PRE_NAMESPACE_PATH"
    fi

.. code:: bash


    # Create the enter-systemd-namespace
    sudo vi /usr/sbin/enter-systemd-namespace
    #!/bin/bash

    if [ "$UID" != 0 ]; then
        echo "You need to run $0 through sudo"
        exit 1
    fi

    SYSTEMD_PID="$(ps -ef | grep '/lib/systemd/systemd --system-unit=basic.target$' | grep -v unshare | awk '{print $2}')"
    if [ -z "$SYSTEMD_PID" ]; then
        /usr/sbin/daemonize /usr/bin/unshare --fork --pid --mount-proc /lib/systemd/systemd --system-unit=basic.target
        while [ -z "$SYSTEMD_PID" ]; do
            SYSTEMD_PID="$(ps -ef | grep '/lib/systemd/systemd --system-unit=basic.target$' | grep -v unshare | awk '{print $2}')"
        done
    fi

    if [ -n "$SYSTEMD_PID" ] && [ "$SYSTEMD_PID" != "1" ]; then
        if [ -n "$1" ] && [ "$1" != "bash --login" ] && [ "$1" != "/bin/bash --login" ]; then
            exec /usr/bin/nsenter -t "$SYSTEMD_PID" -a \
                /usr/bin/sudo -H -u "$SUDO_USER" \
                /bin/bash -c 'set -a; source "$HOME/.systemd-env"; set +a; exec bash -c '"$(printf "%q" "$@")"
        else
            exec /usr/bin/nsenter -t "$SYSTEMD_PID" -a \
                /bin/login -p -f "$SUDO_USER" \
                $(/bin/cat "$HOME/.systemd-env" | grep -v "^PATH=")
        fi
        echo "Existential crisis"
    fi

.. code:: bash

    # Edit the permissions of the enter-systemd-namespace script
    sudo chmod +x /usr/sbin/enter-systemd-namespace
    # Edit the bash.bashrc file
    sudo sed -i 2a"# Start or enter a PID namespace in WSL2\nsource /usr/sbin/start-systemd-namespace\n" /etc/bash.bashrc

.. figure:: https://gitee.com/double12gzh/wiki-pictures/raw/master/2020-09-27-wsl_docker_kubernetes_win10/28.png
   :alt: 

现在再重新打开一个wsl2的窗口，\ **没有必要**\ 把前一个窗口关闭。

.. figure:: https://gitee.com/double12gzh/wiki-pictures/raw/master/2020-09-27-wsl_docker_kubernetes_win10/29.png
   :alt: 

6.3 创建集群
~~~~~~~~~~~~

下面我们创建我们的第一个集群:

.. code:: bash

    # Check if the KUBECONFIG is not set
    echo $KUBECONFIG
    # Check if the .kube directory is created > if not, no need to create it
    ls $HOME/.kube
    # Check if the .minikube directory is created > if yes, delete it
    ls $HOME/.minikube
    # Create the cluster with sudo
    sudo minikube start --driver=none

为了不用\ ``sudo``\ 就能执行\ ``kubelet``\ ，\ ``Minikube``\ 建议执行一下\ ``chown``\ 命令:

.. code:: bash

    # Change the owner of the .kube and .minikube directories
    sudo chown -R $USER $HOME/.kube $HOME/.minikube
    # Check the access and if the cluster is running
    kubectl cluster-info
    # Check the resources created
    kubectl get all --all-namespaces

.. figure:: https://gitee.com/double12gzh/wiki-pictures/raw/master/2020-09-27-wsl_docker_kubernetes_win10/30.png
   :alt: 

集群成功创建了，并且\ ``Minikube``\ 使用的是wsl2的IP，这样以来，我们不需要新配置\ ``proxy``\ 就可以直接访问dashboard了。

.. figure:: https://gitee.com/double12gzh/wiki-pictures/raw/master/2020-09-27-wsl_docker_kubernetes_win10/31.png
   :alt: 

而WSL2集成的真正优势，\ ``8443``\ 端口一旦在WSL2发行版上打开，它其实是转发到Windows上的，所以我们不需要代理这个IP地址，也可以通过localhost访问到Kubernetes的dashboard
URL。

.. figure:: https://gitee.com/double12gzh/wiki-pictures/raw/master/2020-09-27-wsl_docker_kubernetes_win10/32.png
   :alt: 

6.4 访问dashboard
~~~~~~~~~~~~~~~~~

在命令行上工作总是好的，而且非常有见地。然而，当处理Kubernetes时，我们可能会在某些时候希望有一个可视化的概述。

为此，Minikube嵌入了Kubernetes Dashboard。得益于它，运行和访问Dashboard非常简单。

.. code:: bash

    # Enable the Dashboard service
    sudo minikube dashboard
    # Access the Dashboard from a browser on Windows side

.. figure:: https://gitee.com/double12gzh/wiki-pictures/raw/master/2020-09-27-wsl_docker_kubernetes_win10/33.png
   :alt: 

该命令还创建了一个代理，这意味着一旦我们按CTRL+C键结束该命令，Dashboard将不再被访问。

不过，如果我们查看命名空间kubernetes-dashboard，我们会发现服务仍然存在。

.. code:: bash

    # Get all the services from the dashboard namespace
    kubectl get all --namespace kubernetes-dashboard

.. figure:: https://gitee.com/double12gzh/wiki-pictures/raw/master/2020-09-27-wsl_docker_kubernetes_win10/34.png
   :alt: 

接下来，我们更改一下\ ``service``\ 的类型，让他以\ ``LoadBalancer``\ 作为后端：

.. code:: bash

    # Edit the Dashoard service
    kubectl edit service/kubernetes-dashboard --namespace kubernetes-dashboard
    # Go to the very end and remove the last 2 lines
    status:
      loadBalancer: {}
    # Change the type from ClusterIO to LoadBalancer
      type: LoadBalancer
    # Save the file

.. figure:: https://gitee.com/double12gzh/wiki-pictures/raw/master/2020-09-27-wsl_docker_kubernetes_win10/35.png
   :alt: 

再次查看dashboard相关的\ ``service``\ ，这次我们通过\ ``LoadBalancer``\ 来访问：

.. code:: bash

    # Get all the services from the dashboard namespace
    kubectl get all --namespace kubernetes-dashboard
    # Access the Dashboard from a browser on Windows side with the URL: localhost:<port exposed>

.. figure:: https://gitee.com/double12gzh/wiki-pictures/raw/master/2020-09-27-wsl_docker_kubernetes_win10/36.png
   :alt: 

7. 结论
-------

很明显，我们还远远没有完成，因为我们可以实现一些LoadBalancing和/或其他服务（存储、入口、注册表等）。

关于WSL2上的Minikube，由于它需要启用SystemD，我们可以把它看作是一个中间层来实现。

那么，在两种解决方案中，什么才是 "最适合你"的呢？两者都有各自的优势和不便，所以这里仅从我们的角度来概述一下。

+--------------------+--------------------------------+------------+
| 对比项             | Kind                           | Minikube   |
+====================+================================+============+
| 与WSL2共同使用     | 非常简单                       | 一般简单   |
+--------------------+--------------------------------+------------+
| 多节点             | 支持                           | 不支持     |
+--------------------+--------------------------------+------------+
| 插件               | 手动安装                       | 自动安装   |
+--------------------+--------------------------------+------------+
| 持久化             | 支持(但这并不是它的本初设计)   | 支持       |
+--------------------+--------------------------------+------------+
| 同类型的其它产品   | k3s                            | MicroK8s   |
+--------------------+--------------------------------+------------+

我们希望你能真正体验到不同组件之间的整合。WSL2 - Docker Desktop -KinD/Minikube。这给了你一些想法，甚至更好的是，给了你在Windows和WSL2上使用KinD和/或Minikube的Kubernetes工作流的一些答案。

8. 出处
-------

`WSL+Docker: Kubernetes on the Windows
Desktop <https://kubernetes.io/blog/2020/05/21/wsl-docker-kubernetes-on-the-windows-desktop/>`__