---
title: kubelabs——k8s学习与实验【4】容器存储接口CSI（上）：Kubernetes存储
date: 2024/2/4 20:34:20
tags:
- k8s
- csi
categories:
- kubelabs
---

# kubelabs——k8s学习与实验【4】容器存储接口CSI（上）：Kubernetes存储

## 1 相关概念简介

k8s集群存储的概念建立在容器存储的基础之上，因此这里讨论两个关键的领域：Docker存储和k8s存储。

### 1.1 Docker存储

> 容器存储：镜像层（只读）+ 可写层（可写）+ 外部存储（挂载或卷）

#### 1.1.1 镜像

Docker镜像是一个静态的、只读的模板，它包含了运行某个应用所需要的所有内容，包括代码、运行时环境、依赖库、配置文件等。镜像按照层次结构存储，每一层都是对上一层的修改，有利于资源复用。镜像是不可变的，一旦创建后就不能更改。如果需要更改，则基于原有的镜像创建新的镜像层。

#### 1.1.2 非持久化数据（Storage Driven）

通过分层，多个容器可以共享一个镜像资源（一个镜像创建多个容器），甚至更细粒度地共享镜像层（多个镜像在同一个镜像的基础上构建）。这些层是只读的，以保证镜像不会在创建容器后就被改变。

容器继承了镜像的内容并在其基础上添加了一个可写层，允许容器在运行过程中进行数据读写操作，当容器停止运行时，这些改变不会影响到原始的镜像。对于读，遵循上层数据覆盖下载数据的原则；对于写，遵循写时复制的规则。

当一个容器启动时，它不会直接复制整个底层镜像的所有数据到新的存储空间中。相反，它创建了一个新的可写层，并将所有读操作指向父镜像层。只有当容器内进程试图修改已存在于只读镜像层中的文件或目录时，Docker 才会执行写时复制操作，即将该文件从只读层复制到当前容器自己的可写层中，然后在这个可写的副本上进行修改。在写时复制策略下，多个容器可以共享同一份基础镜像的只读部分，直到它们各自需要修改不同的文件为止。

容器可以被创建、启动、停止、删除等，具有短暂和轻量的特点，可以快速地创建和销毁。当容器被销毁后，镜像的存储不会受到影响，但是可写层的数据会丢失，无法持久化。Docker 通过 Storage Driven 来管理以上非持久化存储。

> 另外，docker 提供了tmpfs mount，将文件存储在主机内存中，当宿主机关闭时，数据丢失。如果容器生成的是非持久状态数据，建议考虑使用tmpfs挂载，这样既避免了永久存储数据，又能通过不向容器可写层写入数据的方式提升容器性能。

#### 1.1.3 持久化数据

> 文档见 [``Docker Docs / Docker Engine / storage``](https://docs.docker.com/storage/)

默认情况下，在容器内创建的所有文件都存储在可写层。这意味着：

- 当该容器不再存在时，数据不会保留；且如果另一个进程需要数据，则很难将数据从容器中取出。
- 容器的可写层与主机上运行容器的位置紧密耦合，无法轻松地将数据移动到其他位置。
- 写入容器的可写层需要存储驱动程序来管理文件系统，降低了性能。

为了使写入的内容不会丢失，Docker使用了多种方法，包括bind-mount和volume，将数据存储在宿主机上。允许数据持久化并跨越容器生命周期 。

- **bind mount**

    绑定挂载是 Docker 早期引入的功能，其功能相对数据卷而言较为有限。在使用绑定挂载时，会将宿主机上的**某个具体文件或目录**通过**绝对路径**映射到容器内部。
    对于要挂载的文件或目录，无需提前确保它存在于Docker宿主机上；若宿主机不存在此目录，会在实际需要时动态创建。

    尽管绑定挂载在性能方面表现出色，但它的正常运行依赖于宿主机文件系统中特定目录结构的存在，且无法直接利用 ``Docker CLI`` 命令对绑定挂载进行管理操作。

- **volume**

    与 bind mount 不同的是，数据卷的运作方式是在宿主机的Docker存储目录下新建一个子目录，并由Docker系统来管理该目录内的内容。相比绑定挂载具有多项优势：

    1. 更易于备份或迁移。
    2. 可以通过Docker CLI命令或Docker API来管理数据卷。
    3. 数据卷不仅适用于Linux容器，也支持Windows容器。
    4. 数据卷能够更安全地在多个容器之间共享。

![docker-storage](https://92697-imgs.oss-cn-hangzhou.aliyuncs.com/blogs/20240204161318.png)

### 1.2 Kubernetes存储

> 文档见 [``https://kubernetes.io/zh-cn/docs/concepts/storage/``](https://kubernetes.io/zh-cn/docs/concepts/storage/)


#### 1.2.1 临时卷（Ephemeral Volume）

临时卷是一个 Pod 级别的概念。

有些应用需要额外的存储空间，但不关心数据在重启后是否保存。为此场景，k8s设计了临时卷的概念，其生命周期与Pod相同，与 Pod 一起创建和删除。临时卷包括：

- emptyDir

    该卷是在Pod创建的空目录，用于在容器之间共享文件。

- configMap、downloadAPI、secret

    configMap 卷提供了向 Pod 注入配置数据的方法。 ConfigMap 对象中存储的数据可以被 configMap 类型的卷引用，然后被 Pod 中运行的容器化应用使用。
    downwardAPI 卷用于为应用提供 downward API 数据。 在这类卷中，所公开的数据以纯文本格式的只读文件形式存在。
    secret 卷用来给 Pod 传递敏感信息，例如密码。

- 通用临时卷
    类似于 emptyDir 卷，为每个 Pod 提供临时数据存放目录，在最初制备完毕时一般为空。

> 一个Pod内的容器都可以共用，它们可以指定各自的 mount 路径。

#### 1.2.2 持久卷（Persistent Volumes）

持久卷是一个集群级别的概念，常见的包括 hostPath、PV等。PV是类似Pod的抽象对象。

- **hostPath**

    hostPath 卷是讲节点主机上的目录直接挂载到Pod中，从而使Pod中的数据在Pod销毁后依然能保存。

    > 极不推荐使用 hostPath ，会带来安全风险。如果要使用，也应该定义一个 ``local`` PV，用来代替 hostPath。

- **PV（PersistentVolume）**

    PV可以抽象地理解为一个存储资源，描述的是集群中可供使用的持久化存储资源。

    Kubernetes 内置或通过 CSI 支持多种类型的 Volume 插件，例如对于本地磁盘、网络存储（NFS存储, iSCSI, Ceph RBD）、云服务商提供的块存储（如 AWS EBS, GCP Persistent Disk 或 Azure Disk）等。

    当创建一个 PV 并指定其 storageClassName 和相关属性时，实际上是告诉 Kubernetes 使用特定的 Volume 插件来连接到实际的存储资源。

- **PVC（PersistentVolumeClaim）**

    开发人员或运维人员定义并创建PVC，它代表了对存储资源的需求，包括所需容量、访问模式（如 ReadWriteOnce, ReadOnlyMany 或 ReadWriteMany）和 StorageClass 等属性

    PV控制器会自动查找满足PVC的PV，将PV和PVC绑定。当Pod停止并重新部署时，数据持久化在独立于Pod的PV中，并通过PVC再次挂载PV。

    > 一般来说，PV由运维工程师（存储工程师）来进行维护，而PVC则是由开发人员自己申请使用即可。当一个容器需要进行数据卷挂载，只需要写一个PVC来绑定PV就可以，k8s自身会查找符合条件的PV。

- **CSI（Container Storage Interface）**

    CSI是由Kubernetes、Docker等社区联合制定的行业标准接口规范，旨在将任何存储系统暴露给容器化应用程序。存储服务提供方需要实现该接口规范的一些方法，包括：创建存储卷、删除存储卷、挂载卷、卸载卷、创建卷快照、删除卷快照等方法。任何实现了CSI的存储插件，都可以在k8s中使用。**通过CSI可以创建可以被k8s使用的PV**。

## 2 实验

### 2.1 Docker存储

docker的非持久化存储不做赘述，在此处针对docker的两种持久化存储进行实验。

#### bind mount

在主机创建目录，如：

```bash
mkdir ~/tempdir/dockerdir
```

启动容器，并将目录挂载到容器内部的 ``~/downloads`` 目录下，这里依然使用之前博客的镜像。

```bash
docker run -itd --name storage-test -v ~/tempdir/dockerdir:/root/downloads go-hello-world-image:v0.0.1
```

进入容器，检查 ``~/downloads`` 目录，并写入文件 ``hello.txt`` 。

```bash
yzq@ubuntu:~$ docker exec -it storage-test sh
/app # echo "Hello" > ~/downloads/hello.txt
/app # ls ~/downloads/
hello.txt
```

检查宿主机，文件已存在。

```bash
yzq@ubuntu:~$ ls tempdir/dockerdir/
hello.txt
```

删除容器，并检查宿主机 ``~/tempdir/dockerdir/`` 目录下文件是否存在。

```bash
yzq@ubuntu:~$ docker stop storage-test && docker rm storage-test && ls ~/tempdir/dockerdir/ && cat ~/tempdir/dockerdir/hello.txt 
storage-test
storage-test
hello.txt
Hello
```

文件和文件内容均无误。

#### volume

创建名为 ``temp-volume`` 的volume，检查volume。除了刚才创建的volume外，还有之前任务留下的volume。

```bash
yzq@ubuntu:~$ docker volume create temp-volume
temp-volume
yzq@ubuntu:~$ docker volume ls
DRIVER    VOLUME NAME
local     36ea4e358c355f1a1154767e98bd1e5bdb8448d2c0077b82ed37908aea586e96
local     2683dee230e5c309a797b89d092c0072ef62244ad313ba91b0d28b4d5496dcb4
local     636005babd42afa31a1e8dd3e04dd5060b2eb330906db2da7b2d46c9aed47fcf
local     bc4ff633f0f6ba98f704229dc5ab70f5a31488163d188dcc8228fd3d4fbbf59c
local     c28dbe3855e69424a686db9da8a0c23912ece48500e0d488d1bf2311cfa00db1
local     d5a5bb54399aad85e8eddf512e737921d3fcae4d1299e55535cfd970998fec57
local     dbe071cbcad47a9eec711b2387e3927b1ed209c9f7ff03ac8ae26dc2fb760d8a
local     e4b0d76afe344d2e0cd4c015527aa276a1002bd753e5a9587e19a851f420f840
local     fe295be57089470edea5a63adf2b6fe2c06745b168e4285f9d7625e3a6c0d1e6
local     minikube
local     temp-volume
```

启动容器时挂载该volume至容器内的 ``~/downloads`` 路径：

```bash
yzq@ubuntu:~$ docker run -itd --name storage-test2 -v temp-volume:/root/downloads go-hello-world-image:v0.0.1
yzq@ubuntu:~$ docker exec -it storage-test2 sh
/app # cd /root/downloads/
~/downloads # echo "Hello" > ~/downloads/volume.txt
~/downloads # cat volume.txt 
Hello
```

删除容器，并重新创建容器，但将volume挂载在 ``~/files`` 目录下。随后检查该目录下的文件 ``volume.txt`` 。

```bash
yzq@ubuntu:~$ docker exec -it storage-test3 sh
/app # cat ~/files/volume.txt 
Hello
```

### 2.2 Kubernetes存储

#### 2.2.1 emptyDir

利用本系列第一篇博客的配置文件 [``https://github.com/Artistzq/kubelabs/tree/main/hello``](https://github.com/Artistzq/kubelabs/tree/main/hello) ，创建集群 ``storage-cluster``。

```bash
bash create.sh -n storage-cluster
```

并进入 ``control plane`` ，创建一个使用emptyDir的Pod，该Pod中的服务每5秒写入当前时间。

```bash
docker exec -it storage-cluster-control-plane
```

该 pod 的 yaml 配置文件如下：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-emptydir
  namespace: default
  labels:
    app: experiment-emptydir
spec:
  containers:
  - name: myapp-container
    image: go-hello-world-image:v0.0.1
    volumeMounts:
    - mountPath: /data     # 容器内部将emptyDir挂载到/data路径
      name: temporary-storage
    command: ["/bin/sh", "-c"]
    args:
    - while true; do echo "$(date) Hello World" >> /data/output.txt; sleep 5; done # 这条命令会让容器每5秒向/data/output.txt中追加当前时间和消息
  volumes:
  - name: temporary-storage
    emptyDir: {} # 使用emptyDir卷类型
```

手动部署该Pod。（未通过Deployment）

```bash
kubectl apply -f experiment-emptydir.yaml
```

进入pod，检查文件 ``/data/output.txt`` ，持续写入日志。

```bash
yzq@ubuntu:~/Documents/k8s-proj/store-test$ kubectl exec -it pod-with-emptydir -- tail /data/output.txt
04:09:22 UTC 2024 Hello World
04:09:27 UTC 2024 Hello World
04:09:32 UTC 2024 Hello World
04:09:37 UTC 2024 Hello World
04:09:42 UTC 2024 Hello World
04:09:47 UTC 2024 Hello World
04:09:52 UTC 2024 Hello World
04:09:57 UTC 2024 Hello World
04:10:02 UTC 2024 Hello World
04:10:07 UTC 2024 Hello World
```

删除此pod，并用配置文件 ``experiment-emptydir.yaml`` 重新部署 pod

```bash
kubectl delete pod pod-with-emptydir && kubectl apply -f experiment-emptydir.yaml
```

检查日志是否存在之前的记录。之前的记录已丢失，验证了 emptyDir 的临时性。

```bash
kubectl exec -it pod-with-emptydir -- tail /data/output.txt | grep 04:10:07
# 输出空，没有时间为 04:10:07的记录
```

> 注：hostpath 卷的方式与docker bind mount类似，直接将宿主机上的绝对路径挂载到pod中，以达到持久化目的。此处不进行实验。

#### 2.2.2 PV和PVC

首先需要搭建一个PV资源（存储资源，包括本地、NFS等）。在初步实验中，我们搭建一个基于本地存储的PV存储。

> 这里只进行最基本的 host path PV，**后续实验将会在CSI的讨论中展开**。

进入工作节点 ``storage-cluster-worker`` ，创建一个基于hostpath的PV，该方法是使用宿主机上的一块资源创建PV。

> 生产中不要使用这种方法，其一，数据应该使用外挂盘，否则宿主机空间占满会影响服务的运行；其二，该方法创建的PV与宿主机Node绑定了，需要为Pod制定节点亲和性（nodeAffinity），不够方便。对于生产环境，请考虑使用网络存储解决方案来提供跨节点的数据持久化和高可用性。

```bash
yzq@ubuntu:~$ docker exec -it storage-cluster-worker /bin/bash
```

在 ``storage-cluster-worker`` 内创建目录 ``/data``，基于此创建存储容量为10MB的PV，并设定延迟绑定。

```bash
mkdir /data/hostpath-pv
```

> 通过设定storageClassName实现PV和PVC的延迟绑定，提高灵活性。

```yaml
# PV: pv-hostpath.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: hostpath-pv
spec:
  capacity:
    storage: 10Mi # 设置存储容量为10MB
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain # 根据实际情况选择合适的策略
  storageClassName: manual 
  hostPath:
    path: /data/hostpath-pv
```

进入 ``control plane (MasterNode)`` 并应用 ``pv-hostpath.yaml``。

```bash
yzq@ubuntu:~$ docker exec -it torage-cluster-control-plane /bin/bash 
root@storage-cluster-control-plane:~/storage-test# kubectl apply -f pv-hostpath.yaml
root@storage-cluster-control-plane:~/storage-test# kubectl get pv
NAME          CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
hostpath-pv   10Mi       RWO            Retain           Available           manual         <unset> 
```

```yaml
# PVC: pvc-hostpath.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: hostpath-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: manual # 与上面PV的storageClassName一致
  resources:
    requests:
      storage: 10Mi # 与PV中定义的存储容量一致
```

```yaml
# Pod: pod-with-affinity.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-hostpath-affinity
spec:
  affinity:
    nodeAffinity: # 节点亲和性，
      requiredDuringSchedulingIgnoredDuringExecution: # 强制性亲和性规则，在调度时必须满足，但不影响运行时
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/hostname
            operator: In
            values:
            - storage-cluster-worker # 节点名称
  containers:
  - name: myapp-container
    image: go-hello-world-image:v0.0.1
    volumeMounts:
    - mountPath: /go-logs # 把 PV 挂载 到 Pod 内的目录 /go-logs
      name: hostpath-volume
    command: ["/bin/sh", "-c"]
    args:
    - while true; do echo "$(date +'%H:%M:%S') Hello World" >> /go-logs/output.txt; sleep 5; done # 每5秒向/go-logs/output.txt中追加当前时间和消息
  volumes:
  - name: hostpath-volume
    persistentVolumeClaim:
      claimName: hostpath-pvc # 使用PVC申请存储资源
```

再依次应用 ``pvc-hostpath.yaml`` 和 ``pod-with-affinity.yaml``。如果先应用 pod ，会导致 pod 处于 pending 状态，因为请求不到存储资源。

```bash
root@storage-cluster-control-plane:~/storage-test# kubectl apply -f pvc-hostpath.yaml 
persistentvolumeclaim/hostpath-pvc created
root@storage-cluster-control-plane:~/storage-test# kubectl apply -f pod-with-affinity.yaml 
pod/pod-with-hostpath-affinity created
```

查看各资源和服务情况，pod运行在 ``storage-cluster-worker`` 工作节点上。

```bash
root@storage-cluster-control-plane:/# kubectl get pv
NAME          CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                  STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
hostpath-pv   10Mi       RWO            Retain           Bound    default/hostpath-pvc   manual         <unset>                          45m
root@storage-cluster-control-plane:/# kubectl get pvc
NAME           STATUS   VOLUME        CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
hostpath-pvc   Bound    hostpath-pv   10Mi       RWO            manual         <unset>                 3m38s
root@storage-cluster-control-plane:/# kubectl get pods -owide
NAME                         READY   STATUS    RESTARTS   AGE     IP           NODE                     NOMINATED NODE   READINESS GATES
pod-with-hostpath-affinity   1/1     Running   0          3m48s   10.244.2.2   storage-cluster-worker   <none>           <none>

```

接下来**检查卷的情况**。首先进入 pod ``pod-with-hostpath-affinity``中，该 pod 中服务每5秒写入一条日志到 ``/go-logs/output.txt`` 。

```bash
root@storage-cluster-control-plane:/# kubectl exec -it pod-with-hostpath-affinity /bin/sh
/app # ls
/app tail /go-logs/output.txt
15:10:18 Hello World
...
15:11:03 Hello World

```

同时，进入工作节点 ``storage-cluster-worker`` ，这是指定节点亲和性后调度器为pod ``pod-with-hostpath-affinity`` 选择的工作节点。检查目录 ``/data/hostpath-pv/`` 下的 ``output.txt`` 文件，是否与上述 pod 内检查的相同。

```bash
# 退回到运行 kind 的宿主机后
$ docker exec -it storage-cluster-worker /bin/bash
$ cd /data/hostpath-pv
$ tail output.txt 
15:10:18 Hello World
...
15:11:03 Hello World
```

![result-pv](https://92697-imgs.oss-cn-hangzhou.aliyuncs.com/blogs/20240208231426.png)

接下来检查持久性，回到 ``control-plane`` ，删除 pod ``pod-with-hostpath-affinity``。

```bash
# 退回到运行 kind 的宿主机后
docker exec -it storage-cluster-control-plane /bin/bash
root@storage-cluster-control-plane:~# kubectl delete pod pod-with-hostpath-affinity
```

检查工作节点 ``storage-cluster-worker`` 下的目录 ``/data/hostpath-pv`` ，依然存在，文件 ``output.txt`` 内容也无误。

```bash
# 退回到运行 kind 的宿主机后
$ docker exec -it storage-cluster-worker /bin/bash
$ cd /data/hostpath-pv
$ cat output.txt | grep 15:11:03
15:11:03 Hello World # 之前的记录依然存在
```

## 参考

[1] [``https://blog.csdn.net/weixin_43970890/article/details/102566854``](https://blog.csdn.net/weixin_43970890/article/details/102566854)  
[2] [``https://cloud.tencent.com/developer/article/2234476``](https://cloud.tencent.com/developer/article/2234476)
