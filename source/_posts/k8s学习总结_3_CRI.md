---
title: kubelabs——k8s学习与实验【3】容器运行时接口（CRI）以及切换容器运行时
date: 2024/1/26 23:17:15
tags:
- k8s
- cri
categories:
- kubelabs
---

# kubelabs——k8s学习与实验【3】容器运行时接口（CRI）以及切换容器运行时

## 1 概念

容器运行时（Container Runtime）是 Kubernetes（k8s）集群中每个节点的核心组件，负责管理容器的整个生命周期，包括从拉取和运行容器镜像等关键任务。

<div align=center>
<img src="https://92697-imgs.oss-cn-hangzhou.aliyuncs.com/blogs/20240124152148.png">
</div>

在k8s 1.23版本之前，kubelet依赖于dockershim来操作Docker容器运行时。Docker内部集成了containerd并在此基础上构建了更加用户友好的功能层，但这也意味着dockershim需要随着Docker的更新持续维护跟进。

为了实现Kubernetes与具体容器运行时技术的解耦，自k8s 1.5版本起，社区引入了容器运行时接口（Container Runtime Interface，CRI）标准，它提供了一组标准化接口以供 Kubernetes 平台与各种容器运行时进行交互，并推荐使用原生支持CRI的容器运行时如containerd和CRI-O。这一转变标志着Kubernetes未来将不再直接绑定Docker，而是通过兼容CRI的标准运行时统一管控容器，从而提高了容器运行时的可插拔性以及Kubernetes本身的稳定性和灵活性。kubelet通过gRPC协议结合protobuf序列化方式来实现对CRI接口的远程调用过程。

尽管如此，这并不影响继续使用通过Docker打包的镜像。实际上，任何遵循OCI（开放容器倡议，Open Container Initiative）规范创建的镜像都可以被兼容。由于Docker打包的镜像同样遵循OCI规范，因此它们仍能在k8s环境中得到有效的管理和运行。

CRI相关的概念如下图所示。

![cri-arch](https://92697-imgs.oss-cn-hangzhou.aliyuncs.com/blogs/20240124144045.png)

### 1.1 Docker 和 dockershim

Docker是一个方便的工具，用于处理镜像打包、发布、容器运行等操作。Docker本身包含了多个组件，如：

- docker-cli
- containerd
- runc
- ...

k8s 中包含了一个组件 dockershim，使其能够支持Docker。Docker 没有实现CRI接口（Docker出现时间更早），因此 k8s 不得不自己维护一个工具，通过此工具操作Docker，这就是dockershim。

随着容器化成为行业标准，k8s 项目为了增加了对额外运行时的支持，提出了 CRI，更多的运行时实现了此接口。k8s 项目对 dockershim 和docker的依赖使整个项目变得脆弱，且对 dockershim 的单独维护增加了成本，在1.23版本后，k8s 开始弃用 dockershim，取消对Docker的直接支持。

另外，Docker 的多层封装和调用，导致其在可维护性上略逊一筹，增加了线上问题的定位难度。

### 1.2 CRI

类似 JAVA 的 SPI 机制，k8s 定义了 CRI 后，不需要再像维护 dockershim 那样，为某个容器运行时维护一套适配工具。只需要各个容器运行时的提供方实现了这个接口，k8s 就可以直接调用对应的操作，例如containerd和CRI-O。

containerd 是一个来自 Docker 的高级容器运行时，并实现了 CRI 规范。它是从 Docker 项目中分离出来的，因此 Docker 自己在内部使用 containerd。containerd 通过其 CRI 插件实现了 k8s 容器运行时接口（CRI），它可以管理容器的整个生命周期，包括从镜像的传输、存储到容器的执行、监控再到网络。

CRI-O 是另一个实现了容器运行时接口（CRI）的高级容器运行时，是 containerd 的一个替代品。

使用 containerd 和 CRI-O 的方案比起 docker 简洁很多，因为省去了冗余的多层封装。

### 1.3 OCI

OCI 是指开放容器倡议（Open Container Initiative），旨在制定容器镜像和运行时的行业标准。只要是符合规范的不同运行时，这些运行时可以有不同的底层实现。例如，符合 OCI 的在 linux 上的容器运行时和在 Windows 上的容器运行时。遵循该倡议的运行时通常更底层，和操作系统进行交互，从而创建容器。runC和kata-runtime就是这样的容器运行时。

runc 是轻量级的通用运行时容器，它遵守 OCI 规范，是实现 OCI 接口的最低级别的组件，它与内核交互创建并运行容器。runc 为容器提供了所有的低级功能，与现有的低级 Linux 功能交互，如命名空间和控制组，它使用这些功能来创建和运行容器进程。

### 1.4 dockershim的弃用

dockershim 的弃用并不意味着 k8s 将不能运行 Docker 格式的容器。containerd 和 CRI-O 都可以运行 Docker 格式的镜像，因为 Docker 格式的镜像也遵循了 OCI 格式，只是它们不再需要使用 docker 命令或 Docker 守护程序。

从功能性来讲，containerd 和 CRI-O 都符合 CRI 和 OCI 的标准。从稳定性来说，containerd 一直在 docker 里使用，生产环境经验比较充足，因此更推荐选择 containerd 作为 k8s 的容器运行时。

## 2 实验

实验目的：使用 containerd 或 CRI-O 作为容器运行时，并在工作节点中使用 ``crictl`` 命令代替 docker 命令。

### 2.1 kind 默认使用 containerd

查阅文档得知，Kind 内部使用 Kubeadm 创建和启动集群节点，并使用 containerd 作为容器运行时，所以弃用 dockershim对 Kind 没有什么影响。另一个部署单机集群的工具 ``minikube`` 使用 cri-o 作为默认容器运行时。

进入Master Node，查看使用的容器运行时：

```bash
yzq@ubuntu:~/Documents/k8s-proj/cni-test$ docker exec -it cni-test-cluster-control-plane /bin/bash
root@cni-test-cluster-control-plane:/# kubectl get node cni-test-cluster-worker -o json | jq '.status.nodeInfo.containerRuntimeVersion'
"containerd://1.7.1"
```

使用 ``crictl`` 查看镜像

```bash
root@cni-test-cluster-control-plane:/# crictl images ps
IMAGE                                      TAG                  IMAGE ID            SIZE
docker.io/flannel/flannel-cni-plugin       v1.2.0               a55d1bad692b7       3.88MB
docker.io/flannel/flannel                  v0.24.0              0dc86fe0f22e6       28MB
docker.io/kindest/kindnetd                 v20230511-dc714da8   b0b1fa0f58c6e       27.7MB
docker.io/kindest/local-path-helper        v20230510-486859a6   be300acfc8622       3.05MB
docker.io/kindest/local-path-provisioner   v20230511-dc714da8   ce18e076e9d4b       19.4MB
docker.io/library/go-hello-world-image     v0.0.1               586004f581b85       21.3MB
registry.k8s.io/coredns/coredns            v1.11.1              cbb01a7bd410d       18.2MB
registry.k8s.io/etcd                       3.5.10-0             a0eed15eed449       56.6MB
registry.k8s.io/kube-apiserver             v1.29.0              4a9136aaad730       86.1MB
registry.k8s.io/kube-controller-manager    v1.29.0              23d55452a141c       80.3MB
registry.k8s.io/kube-proxy                 v1.29.0              fa4dee78049db       83.5MB
registry.k8s.io/kube-scheduler             v1.29.0              ba16e783eea7b       60.6MB
registry.k8s.io/pause                      3.7                  221177c6082a8       311kB
```

### 2.2 将 kind 使用的运行时替换成 CRI-O

在利用 kind 工具构建集群时，默认会采用 ``kindest/node`` 镜像作为节点镜像，该镜像内部预设的容器运行时是 containerd。若要将 containerd 更改为 crio 运行时，有两种可行方案：

1. 通过自定义节点镜像的方式进行切换：首先，在 ``kindest/node`` 镜像的基础之上重新构建一个新的节点镜像，并结合使用 ``kind create cluster`` 命令以及适配 crio 的定制版 ``kind-config.yaml`` 文件来创建新的集群。可以参考： [``https://gist.github.com/aojea/bd1fb766302779b77b8f68fa0a81c0f2``](https://gist.github.com/aojea/bd1fb766302779b77b8f68fa0a81c0f2) 和 [``https://github.com/warm-metal/kindest-base-crio/tree/main``](https://github.com/warm-metal/kindest-base-crio/tree/main) 。

    >在实际操作过程中，参考网站提供的镜像版本较为陈旧，和现有的 kind 以及 k8s 有兼容性问题。

2. 在已用 kind 创建好的 k8s 集群上直接操作：进入 Worker 节点或 Master 节点（它们实质上是由 kind 创建的 Docker 容器），手动安装并配置 crio 运行时，然后对节点进行重新初始化或重新加入集群。

    > 为了深入理解 k8s 集群创建过程中的各个环节，并尽可能保留关键操作步骤，我们选择手动在集群内安装 crio 运行时。后续的文章中，我们将尝试探索如何打包适配 crio 的节点镜像。

本篇文章重点关注第2种方法。

#### 2.2.1 创建新集群

利用本系列第一篇博客的配置文件 [``https://github.com/Artistzq/kubelabs/tree/main/hello``](https://github.com/Artistzq/kubelabs/tree/main/hello) ，重新创建集群 ``crio-hello-cluster``。

```bash
bash create.sh -n crio-hello-cluster
```

进入 Master 节点 ``crio-hello-cluster-control-plane``

```bash
docker exec -it crio-hello-cluster-control-plane /bin/bash
```

查看信息

- 查看 k8s 版本

    ```bash
    root@hello-crio-cluster-control-plane:/# kubectl version
    Client Version: v1.29.0
    Kustomize Version: v5.0.4-0.20230601165947-6ce0bf390ce3
    Server Version: v1.29.0
    ```

- 查看 linux 发行版本

    ```bash
    root@crio-hello-cluster-control-plane:/# cat /etc/issue
    Debian GNU/Linux 11 \n \l
    ```

- 查看节点详细信息

    ```bash
    root@crio-hello-cluster-control-plane:/# kubectl get nodes -owide
    NAME                               STATUS   ROLES           AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE                         KERNEL-VERSION      CONTAINER-RUNTIME
    crio-hello-cluster-control-plane   Ready    control-plane   34m   v1.29.0   172.18.0.6    <none>        Debian GNU/Linux 11 (bullseye)   5.15.0-92-generic   containerd://1.7.1
    crio-hello-cluster-worker          Ready    <none>          34m   v1.29.0   172.18.0.7    <none>        Debian GNU/Linux 11 (bullseye)   5.15.0-92-generic   containerd://1.7.1
    crio-hello-cluster-worker2         Ready    <none>          34m   v1.29.0   172.18.0.5    <none>        Debian GNU/Linux 11 (bullseye)   5.15.0-92-generic   containerd://1.7.1
    ```

- 查看容器运行时，都是 ``containerd://1.7.1``。

    ```bash
    root@crio-hello-cluster-control-plane:/# kubectl get nodes -o json | jq -r '.items[] | {Name: .metadata.name, ContainerRuntime: .status.nodeInfo.containerRuntimeVersion}'
    {
    "Name": "crio-hello-cluster-control-plane",
    "ContainerRuntime": "containerd://1.7.1"
    }
    {
    "Name": "crio-hello-cluster-worker",
    "ContainerRuntime": "containerd://1.7.1"
    }
    {
    "Name": "crio-hello-cluster-worker2",
    "ContainerRuntime": "containerd://1.7.1"
    }
    ```

#### 2.2.2 为 WorkerNode 安装CRIO

``kind-config.yaml`` 配置文件指定了1个 MasterNode 和2个 WorkerNode，我们进入其中一个WorkerNode，按照官网 [``https://cri-o.io/``](https://cri-o.io/) 和 Github 官方仓库 [``https://github.com/cri-o/cri-o/blob/main/install.md``](https://github.com/cri-o/cri-o/blob/main/install.md) 上的指导，为其安装 CRIO-O。

```bash
docker exec -it crio-hello-cluster-worker /bin/bash
```

> 注：以下的安装 CRI-O 的操作都是在此节点容器内进行的！



1. 指定 ``VERSION`` 和 ``OS`` 环境变量。  

    OS指定当前系统的版本，官网给出了对应关系，例如Ubuntu 20.04的主机，需要 ``export OS=xUbuntu_20.04`` 。 kind 创建的节点为Debian 11，对应 ``export OS=Debian_11``。

    VERSION指定crio的版本，支持主版本和具体版本的指定，例如 ``1.24:1.24.1`` 。这里有坑：**``opensuse`` 上支持的最新的版本是 ``1.24.6``** ，更高的版本如 github 上的最新版 ``1.29.1`` ，是无法在 [``https://download.opensuse.org/repositories``](https://download.opensuse.org/repositories) 中下载的，而文档没有说明这一点，只说明了支持主版本号和具体版本号。因此安装时，如果指定版本为 ``1.29.1`` ，会报错，提示无法找到文件。

    ```bash
    export OS=Debian_11
    export VERSION=1.24.6
    export SUBVERSION=$(echo $VERSION | awk -F'.' '{print $1"."$2}') # 1.24
    ```

2. 安装相关工具

    ```bash
    apt-get install -y gnupg tzdata
    ```

3. 添加 apt 源，并下载安装 crio

    ```bash
    CONTAINERS_URL="https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/${OS}/"
    # crio 镜像地址
    CRIO_URL="http://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable:/cri-o:/${SUBVERSION}:/${VERSION}/${OS}/"

    # 把上述 url 添加到 apt 的源
    echo "deb ${CONTAINERS_URL} /" > /etc/apt/sources.list.d/devel:kubic:libcontainers:stable.list
    echo "deb ${CRIO_URL} /" > /etc/apt/sources.list.d/devel:kubic:libcontainers:stable:cri-o:${VERSION}.list

    # 下载密钥，并将其添加到 apt 信任密钥列表
    curl -L ${CONTAINERS_URL}Release.key | apt-key add - || true
    curl -L ${CRIO_URL}Release.key | apt-key add - || true

    # 通过 apt 安装 cri-o 和 cri-o-runc
    apt-get update
    apt-get install -y cri-o cri-o-runc
    ```

    如果下载太慢可以临时添加代理环境变量，例如：

    ```bash
    export http_proxy="http:192.168.115.1:7890"
    export https_proxy="http:192.168.115.1:7890"
    env | grep proxy # 检查
    ```

    > 注意：在后续要取消 ``proxy`` 相关环境变量，或加入 ``no_proxy`` 的变量，否则导致此工作节点无法加入集群。

    安装结束时会提示 ``crictl.yaml`` 已存在，因为该节点之前使用的是 ``containerd`` ，已经配置了 ``crictl`` 。这里选择覆盖 ``Y`` 。

    ```bash
    Configuration file '/etc/crictl.yaml'
    ==> File on system created by you or by a script.
    ==> File also in package provided by package maintainer.
    What would you like to do about it ?  Your options are:
        Y or I  : install the package maintainer's version
        N or O  : keep your currently-installed version
        D     : show the differences between the versions
        Z     : start a shell to examine the situation
    The default action is to keep your current version.
    *** crictl.yaml (Y/I/N/O/D/Z) [default=N] ? Y
    ```

4. 配置 cri-o

    安装完后配置 ``cri-o``

    ```bash
    echo "[crio.runtime]
    cgroup_manager=\"cgroupfs\"
    conmon_cgroup=\"pod\"
    pause_image = \"k8s.gcr.io/pause:3.2\"
    storage_driver = \"vfs\"" > /etc/crio/crio.conf

    ```

5. 设置 crictl

    ```bash
    # 将 crictl.yaml 中所有出现的 containerd 字符串替换为 crio
    sed -i 's/containerd/crio/g' /etc/crictl.yaml
    ```

6. 检查 containerd 和 crio

    检查当前状态

    ```bash
    # containerd 状态
    $ systemctl status containerd | grep Active # Runing
     Active: active (running) since Fri 2024-01-26 08:23:49 UTC; 1h 8min ago
    # crio 状态
    $ systemctl status crio | grep Active # Dead
     Active: inactive (dead)
    ```

    关闭 ``containerd`` 服务，启动 ``crio`` 服务

    ```bash
    $ systemctl disable containerd && systemctl stop containerd
    Removed /etc/systemd/system/multi-user.target.wants/containerd.service.
    $ systemctl enable crio && systemctl start crio
    ```

    再次检查状态

    ```bash
    $ systemctl status containerd | grep Active # Dead
     Active: inactive (dead) since Fri 2024-01-26 09:36:03 UTC; 50s ago
    $ systemctl status crio | grep Active # Runing
     Active: active (running) since Fri 2024-01-26 09:35:08 UTC; 1min 50s ago
    ```

#### 2.2.3 使用 ``Kubeadm`` 重新加入集群

1. 检查节点状态

    重新开一个终端，进入 MasterNode，检查集群状态

    > 此命令在宿主机上执行

    ```bash
    $ docker exec -it crio-hello-cluster-control-plane kubectl get nodes -owide
    NAME                               STATUS     ROLES           AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE                         KERNEL-VERSION      CONTAINER-RUNTIME
    crio-hello-cluster-control-plane   Ready      control-plane   77m   v1.29.0   172.18.0.6    <none>        Debian GNU/Linux 11 (bullseye)   5.15.0-92-generic   containerd://1.7.1
    crio-hello-cluster-worker          NotReady   <none>          76m   v1.29.0   172.18.0.7    <none>        Debian GNU/Linux 11 (bullseye)   5.15.0-92-generic   containerd://Unknown
    crio-hello-cluster-worker2         Ready      <none>          76m   v1.29.0   172.18.0.5    <none>        Debian GNU/Linux 11 (bullseye)   5.15.0-92-generic   containerd://1.7.1
    ```

    观察到前面操作的节点 ``crio-hello-cluster-worker`` 已经进入 ``NotReady`` 状态，CONTAINER-RUNTIME 为 ``containerd://Unknown`` 。

2. 使用 ``kubeadm join`` 将节点加入集群

    此时，为了将工作节点重新加入此集群，需要先在MasterNode上生成token：

    > 此命令在MasterNode中执行。记得关闭代理，否则无法加入。可以 ``unset http_proxy && unset https_proxy``。如果添加环境变量 ``export no_proxy="localhost,127.0.0.1,10.0.0.0/8,192.168.0.0/16,172.16.0.0/12"`` ，测试即使重启也无效。

    ```bash
    $ kubeadm token create --print-join-command
    kubeadm join crio-hello-cluster-control-plane:6443 --token xeogi6.plnsolw9cd8457mc --discovery-token-ca-cert-hash sha256:0f84c44d447641f23345cd8269a27645fe149d19028a6afe6a07f0b7aac191c5 
    ```

    但在 WorkerNode上执行返回的命令，会报错，提示预检错误，表明kubelet在执行系统验证时尝试加载名为“configs”的内核模块以解析内核配置，但未能在 ``/lib/modules/5.15.0-92-generic`` 目录下找到该模块。

    > 以下命令在WorkerNode ``crio-hello-cluster-worker``上执行

    ```bash
    root@hello-crio-worker:/home# kubeadm join crio-hello-cluster-control-plane:6443 --token xeogi6.plnsolw9cd8457mc --discovery-token-ca-cert-hash sha256:0f84c44d447641f23345cd8269a27645fe149d19028a6afe6a07f0b7aac191c5 
    [preflight] Running pre-flight checks
        [WARNING Swap]: swap is supported for cgroup v2 only; the NodeSwap feature gate of the kubelet is beta but disabled by default
        [WARNING FileExisting-socat]: socat not found in system path
    [preflight] The system verification failed. Printing the output from the verification:
    ...
    error execution phase preflight: [preflight] Some fatal errors occurred:
        [ERROR SystemVerification]: failed to parse kernel config: unable to load kernel module: "configs", output: "modprobe: FATAL: Module configs not found in directory /lib/modules/5.15.0-92-generic\n", err: exit status 1
    [preflight] If you know what you are doing, you can make a check non-fatal with `--ignore-preflight-errors=...`
    To see the stack trace of this error execute with --v=5 or higher
    ```

    暂时有两个解决办法：

    - 忽略问题。[``https://github.com/kubernetes/kubernetes/issues/41025``](https://github.com/kubernetes/kubernetes/issues/41025)
    - 升级内核镜像、配置容器。[``https://stackoverflow.com/questions/54128045/errors-while-creating-master-in-cluster-of-kubernetes-in-lxc-container``](https://stackoverflow.com/questions/54128045/errors-while-creating-master-in-cluster-of-kubernetes-in-lxc-container)

    目前采用的是忽略问题，暂时无法判断使用此选项的后续问题。加上忽略错误，节点添加成功！

    ```bash
    $ kubeadm join crio-hello-cluster-control-plane:6443 --token xeogi6.plnsolw9cd8457mc --discovery-token-ca-cert-hash sha256:0f84c44d447641f23345cd8269a27645fe149d19028a6afe6a07f0b7aac191c5 --ignore-preflight-errors=SystemVerification

    [preflight] Running pre-flight checks
        [WARNING Swap]: swap is supported for cgroup v2 only; the NodeSwap feature gate of the kubelet is beta but disabled by default
        [WARNING FileExisting-socat]: socat not found in system path
    [preflight] The system verification failed. Printing the output from the verification:
    KERNEL_VERSION: 5.15.0-92-generic
    OS: Linux
    ...
        [WARNING SystemVerification]: failed to parse kernel config: unable to load kernel module: "configs", output: "modprobe: FATAL: Module configs not found in directory /lib/modules/5.15.0-92-generic\n", err: exit status 1
    [preflight] Reading configuration from the cluster...
    [preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
    [kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
    [kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
    [kubelet-start] Starting the kubelet
    [kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...

    This node has joined the cluster:
    * Certificate signing request was sent to apiserver and a response was received.
    * The Kubelet was informed of the new secure connection details.

    Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
    ```

3. 检查集群状态

    从宿主机进入MasterNode

    ```bash
    yzq@ubuntu$ docker exec -it crio-hello-cluster-control-plane /bin/bash
    ```

    > 以下命令在 ``crio-hello-cluster-control-plane`` 中执行

    检查节点状态，节点 ``crio-hello-cluster-worker`` 已处于 ``Ready`` 状态，容器运行时已切换为 ``cri-o://1.24.6`` 。

    ```bash
    $ kubectl get nodes -owide
    NAME                               STATUS   ROLES           AGE    VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE                         KERNEL-VERSION      CONTAINER-RUNTIME
    crio-hello-cluster-control-plane   Ready    control-plane   106m   v1.29.0   172.18.0.6    <none>        Debian GNU/Linux 11 (bullseye)   5.15.0-92-generic   containerd://1.7.1
    crio-hello-cluster-worker          Ready    <none>          11s    v1.29.0   172.18.0.7    <none>        Debian GNU/Linux 11 (bullseye)   5.15.0-92-generic   cri-o://1.24.6
    crio-hello-cluster-worker2         Ready    <none>          106m   v1.29.0   172.18.0.5    <none>        Debian GNU/Linux 11 (bullseye)   5.15.0-92-generic   containerd://1.7.1
    ```

    ```bash
    $ kubectl get nodes -o json | jq -r '.items[] | {Name: .metadata.name, ContainerRuntime: .status.nodeInfo.containerRuntimeVersion}'
    {
    "Name": "crio-hello-cluster-control-plane",
    "ContainerRuntime": "containerd://1.7.1"
    }
    {
    "Name": "crio-hello-cluster-worker",
    "ContainerRuntime": "cri-o://1.24.6"
    }
    {
    "Name": "crio-hello-cluster-worker2",
    "ContainerRuntime": "containerd://1.7.1"
    }
    ```

### 2.3 自己构建 ``kind node`` 镜像

后续文章尝试

---

## BUG

### 以前的配置无法创建集群

中途突然无法启动kubelet，任何多节点的集群都无法创建，即使使用前面博客文章的配置文件也无法创建集群，提示kubelet无法启动。

```bash
$ bash create.sh -name test
cluster-name: ame
Creating cluster "ame" ...
 ✓ Ensuring node image (kindest/node:v1.29.0) 🖼
 ✓ Preparing nodes 📦 📦 📦  
 ✓ Writing configuration 📜 
 ✗ Starting control-plane 🕹️ 
Deleted nodes: ["ame-worker" "ame-worker2" "ame-control-plane"]
ERROR: failed to create cluster: failed to init node with kubeadm: command "docker exec --privileged ame-control-plane kubeadm init --skip-phases=preflight --config=/kind/kubeadm.conf --skip-token-print --v=6" failed with error: exit status 1
Command Output: I0125 11:40:19.477864     172 initconfiguration.go:260] loading configuration from "/kind/kubeadm.conf"
···
[control-plane] Creating static Pod manifest for "kube-apiserver"
I0125 11:40:24.816401     172 local.go:65] [etcd] wrote Static Pod manifest for a local etcd member to "/etc/kubernetes/manifests/etcd.yaml"
···
[control-plane] Creating static Pod manifest for "kube-controller-manager"
I0125 11:40:24.820970     172 manifests.go:157] [control-plane] wrote static Pod manifest for component "kube-controller-manager" to "/etc/kubernetes/manifests/kube-controller-manager.yaml"
I0125 11:40:24.821181     172 manifests.go:102] [control-plane] getting StaticPodSpecs
[control-plane] Creating static Pod manifest for "kube-scheduler"
I0125 11:40:24.821653     172 manifests.go:128] [control-plane] adding volume "kubeconfig" for component "kube-scheduler"
I0125 11:40:24.822478     172 manifests.go:157] [control-plane] wrote static Pod manifest for component "kube-scheduler" to "/etc/kubernetes/manifests/kube-scheduler.yaml"
I0125 11:40:24.822522     172 kubelet.go:68] Stopping the kubelet
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Starting the kubelet
I0125 11:40:24.982554     172 waitcontrolplane.go:83] [wait-control-plane] Waiting for the API server to be healthy
I0125 11:40:24.983290     172 loader.go:395] Config loaded from file:  /etc/kubernetes/admin.conf
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
I0125 11:40:24.985303     172 round_trippers.go:553] GET https://ame-control-plane:6443/healthz?timeout=10s  in 0 milliseconds
···
[kubelet-check] Initial timeout of 40s passed.
I0125 11:41:04.986148     172 round_trippers.go:553] GET https://ame-control-plane:6443/healthz?timeout=10s  in 0 milliseconds
[kubelet-check] It seems like the kubelet isn't running or healthy.
[kubelet-check] The HTTP call equal to 'curl -sSL http://localhost:10248/healthz' failed with error: Get "http://localhost:10248/healthz": dial tcp [::1]:10248: connect: connection refused.
I0125 11:41:05.487445     172 round_trippers.go:553] GET https://ame-control-plane:6443/healthz?timeout=10s  in 0 milliseconds
···
[kubelet-check] It seems like the kubelet isn't running or healthy.
[kubelet-check] The HTTP call equal to 'curl -sSL http://localhost:10248/healthz' failed with error: Get "http://localhost:10248/healthz": dial tcp [::1]:10248: connect: connection refused.
I0125 11:41:10.486636     172 round_trippers.go:553] GET https://ame-control-plane:6443/healthz?timeout=10s  in 0 milliseconds
···
[kubelet-check] It seems like the kubelet isn't running or healthy.
[kubelet-check] The HTTP call equal to 'curl -sSL http://localhost:10248/healthz' failed with error: Get "http://localhost:10248/healthz": dial tcp [::1]:10248: connect: connection refused.
I0125 11:41:20.486427     172 round_trippers.go:553] GET https://ame-control-plane:6443/healthz?timeout=10s  in 0 milliseconds
···
couldn't initialize a Kubernetes cluster
[kubelet-check] It seems like the kubelet isn't running or healthy.
[kubelet-check] The HTTP call equal to 'curl -sSL http://localhost:10248/healthz' failed with error: Get "http://localhost:10248/healthz": dial tcp [::1]:10248: connect: connection refused.

Unfortunately, an error has occurred:
	timed out waiting for the condition

This error is likely caused by:
	- The kubelet is not running
	- The kubelet is unhealthy due to a misconfiguration of the node in some way (required cgroups disabled)
···
```

先后检查了节点的 ```kubelet status```、 ```api-server```、 ```container runtime``` 等，无问题，最后参考 [``https://github.com/kubernetes-sigs/kind/issues/2702``](https://github.com/kubernetes-sigs/kind/issues/2702) ，可能是资源占用太大，因此删除了之前的集群，重新建立，错误解决。

### 使用 kind-crio 镜像无法创建集群

使用 [``https://github.com/warm-metal/kindest-base-crio/tree/main``](https://github.com/warm-metal/kindest-base-crio/tree/main) 提供的镜像，依然提示 kubelet 无法连接，错误同上。

查阅[``https://hub.docker.com/r/warmmetal/kindest-node-crio``](https://hub.docker.com/r/warmmetal/kindest-node-crio)，该镜像最近一个月更新过，但即使使用最新的也无法创建集群。 [``https://gist.github.com/aojea/bd1fb766302779b77b8f68fa0a81c0f2``](https://gist.github.com/aojea/bd1fb766302779b77b8f68fa0a81c0f2) 提到可能是 kind 和 k8s 更新太快，兼容性被打破，因此本实验抛弃了使用该仓库的镜像 ``kindest-base-crio`` 的方案。
