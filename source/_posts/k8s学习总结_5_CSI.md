---
title: kubelabs——k8s学习与实验【5】容器存储接口CSI（下）：NFS CSI 插件实验
date: 2024/2/10 21:10:15
tags:
- k8s
- csi
- nfs
categories:
- kubelabs
---

# kubelabs——k8s学习与实验【5】容器存储接口CSI（下）：NFS CSI 插件实验

## 1 相关概念简介

上一篇文章中介绍了 Kubernetes 存储相关的概念，并进行了实验。本篇文章更详细地介绍其中的 PV 和 PVC 的持久化方式，并引入 CSI（Container Storage Interface）的概念。

### 1.1 StorageClass

上一篇文章提到，PVC 用来描述 pod 希望使用的 PV 的属性，比如大小，读写权限等等。用户只需要关注 PVC ，VolumeController 的 PersistVolumeController 会根据 PVC 去寻找匹配的 PV 进行绑定。PV 资源的创建是我们手动创建的本地卷。

在实际生产中，手动创建并提供 PV 限制了部署的灵活性。StorageClass是为了解决 PV 的供应（Provisioning）问题，例如，当集群管理员创建的 PV 中，任何一个都无法满足 PVC 的需求，Kubernetes 系统就会动态创建 PV 。这一动态供应（Dynamic Provisioning） 过程由 Storage Class负责，如下图所示。

![storage-class](https://92697-imgs.oss-cn-hangzhou.aliyuncs.com/blogs/20240209143746.png)

### 1.2 CSI

CSI 是一个用于为 Kubernetes 集群提供持久化存储的标准接口。在 CSI 之前，扩展存储插件功能的方式相对复杂且不统一。CSI 推出之后，类似 CNI、CRI，该接口提供规范，第三方存储供应商（如云提供商和存储厂商）按照此规范设计插件，Kubernetes 可以使用该插件创建和管理持久化存储，从而支持更多的存储解决方案，如块存储、文件存储或对象存储。CSI 插件通过 grpc 与 Kubernetes 的系统组件进行通信。

> kubelet不直接调用CSI接口来管理PV的整个生命周期。Kubernetes中的控制器组件（如CSI Provisioner(external-provisioner)和CSI Attacher）才是直接与CSI驱动程序交互，负责动态地根据PVC的需求进行PV的供应（Provisioning）、绑定（Binding）及挂载到节点上。

CSI 主要由两个部分组成：在控制节点的 CSI Provisioner 和 工作节点的 CSI Plugin。

CSI Provisioner 运行在 Kubernetes 集群的控制平面节点上，通常是在集群的管理服务器上。其主要功能是处理存储卷的高级生命周期管理和自动化操作。如创建、删除 （PV）或动态供应 PVC 时所需的存储资源。CSI 控制器插件通过监听 Kubernetes API 对象的变化并与之交互来实现上述功能。它们使用 gRPC 接口与 Kubernetes 控制器管理器进行通信。

CSI Plugin 部署在每个 Kubernetes 工作节点上，它们负责执行与存储卷挂载到节点本地以及卸载存储卷相关的低级操作。
包括：

- 将远程或网络存储资源在节点上实际挂载为文件系统或者作为原始块设备。
- 在 Pod 创建或调度到该节点时，根据 PVC 请求将合适的 PV 挂载到Pod容器内。
- 当 Pod 销毁或者从节点上移除时，执行对应的 detach 操作，从节点上卸载存储卷。

同样地，CSI 节点插件也是通过 gRPC 接口与 Kubernetes 节点上的 kubelet 进程进行通信，kubelet 根据 Pod 的定义信息请求 CSI 插件完成具体的存储操作。

常见的 CSI 插件（CSI Driver）包括：

- **AWS EBS CSI Driver**  
该 CSI 驱动允许 Kubernetes 用户在 AWS 上动态创建和管理基于块存储的 PersistentVolumes。
- **Google Cloud Persistent Disk (GCP PD) CSI Driver**  
Google Cloud Platform 提供的 CSI 驱动。
- **Azure Disk CSI Driver**  
Microsoft Azure 的 CSI 驱动程序，让 Kubernetes 能够与 Azure 磁盘服务进行交互，提供块存储给集群内的 Pod 使用。
- [**Alibaba Cloud Kubernetes CSI Plugin（ACK）**](https://github.com/kubernetes-sigs/alibaba-cloud-csi-driver)  
阿里云提供了自家云平台上的多种存储产品（如块存储EBS、文件存储NAS等）对应的 CSI 驱动，便于 Kubernetes 用户无缝地使用阿里云存储资源。
- **NFS CSI Driver**  
针对网络文件系统 (NFS) 的 CSI 驱动插件，可以自动配置 NFS 卷并将其挂载到 Kubernetes Pods 中

## 2 实验

在本地的 kubernetes 集群上使用阿里云的 CSI 插件使用有一系列的前置工作，如：

- 接入VNode
- 集群各主机通过VPN接入阿里云ECI
- ...

> 官方文档：[``https://help.aliyun.com/zh/eci/user-guide/overview-5?spm=a2c4g.11186623.0.0.3e6c72a5IjaDwp``](https://help.aliyun.com/zh/eci/user-guide/overview-5?spm=a2c4g.11186623.0.0.3e6c72a5IjaDwp)

我们本地是由 kind 搭建的集群，集群节点是 docker 容器，在第一篇博客中就曾因为宿主机配置了代理产生了较复杂的问题；如果将 kind 集群接入ECI，相当于本地网络还要经过一层docker网络，比较复杂。我们的学习博客目标在于通过实验初步地了解相关组件原理和效果，暂不需要使用复杂度较高的适用于生产环境的组件，因此这里暂不使用阿里云 CSI，转而使用普通的NFS CSI Driver进行试验。

> 后续博客中会尝试阿里云 CSI 插件。

我们继续沿用上一篇博客创建的 kind 集群 ``storage-cluster``。

### 2.1 安装 NFS Server

新建一个docker容器，并将其加入到集群的docker容器的网络中，以便直接通信。

查看集群的网络：

```bash
$ docker inspect --format='{{json .NetworkSettings.Networks}}' storage-cluster-worker
{"kind":{"IPAMConfig":null,"Links":null,"Aliases":["1cfe2d76e6f7","storage-cluster-worker"],"NetworkID":"53a809b86d8fe0fef419b8a1a05feaf3be95a8681f6d2787ecd536a0b4d575ae","EndpointID":"4414964e344455622be8137404c0e78d64f4685129303101277091ec38e431a3","Gateway":"172.18.0.1","IPAddress":"172.18.0.4","IPPrefixLen":16,"IPv6Gateway":"fc00:f853:ccd:e793::1","GlobalIPv6Address":"fc00:f853:ccd:e793::4","GlobalIPv6PrefixLen":64,"MacAddress":"02:42:ac:12:00:04","DriverOpts":null}}
```

我们直接部署容器化的 NFS 服务器，使用 10M+ 下载量的镜像 [``itsthenetwork/nfs-server-alpine``](https://hub.docker.com/r/itsthenetwork/nfs-server-alpine) 。

```bash
docker pull itsthenetwork/nfs-server-alpine
```

创建容器 ``nfs-server`` ，将宿主机上的 ``/home/nfsshare`` 挂载到 NFS 服务器中的 ``/nfsshare``目录。

> 宿主机对应的是20049端口，因为宿主机使通过VMWARE虚拟的Ubuntu，本身2049已经被占用了。

```bash
docker volume create --name nfs-volume
```

```bash
docker run \
--name nfs-server \
--privileged \
--restart=unless-stopped \
-p 2049:2049 \
-v nfs-volume:/nfsshare \
-e SHARED_DIRECTORY=/nfsshare \
--network kind \
itsthenetwork/nfs-server-alpine
```

查看现在的容器，有一个nfs-server容器，另外三个容器是kind集群创建的三个节点，这4个容器在同一个网络中。

```bash
$ docker ps
CONTAINER ID   IMAGE                             COMMAND                  CREATED          STATUS         PORTS                                         NAMES
9b95c2f7a0eb   itsthenetwork/nfs-server-alpine   "/usr/bin/nfsd.sh"       13 minutes ago   Up 3 minutes   0.0.0.0:20049->2049/tcp, :::20049->2049/tcp   nfs-server
1cfe2d76e6f7   kindest/node:v1.29.0              "/usr/local/bin/entr…"   21 hours ago     Up 3 minutes                                                 storage-cluster-worker
34309581dffe   kindest/node:v1.29.0              "/usr/local/bin/entr…"   21 hours ago     Up 3 minutes   127.0.0.1:42707->6443/tcp                     storage-cluster-control-plane
f998b1033432   kindest/node:v1.29.0              "/usr/local/bin/entr…"   21 hours ago     Up 3 minutes                                                 storage-cluster-worker2

```

查看 ``kind`` 网络下的容器。

```bash
$ docker network inspect kind | jq '.[0].Containers | to_entries[] | .value'
{
  "Name": "storage-cluster-worker2",
  "EndpointID": "407df86e981930be8277f91f855b9a30e8db9dd26d230dbd00f68f13ff2ae2c2",
  "MacAddress": "02:42:ac:12:00:05",
  "IPv4Address": "172.18.0.5/16",
  "IPv6Address": "fc00:f853:ccd:e793::5/64"
}
{
  "Name": "nfs-server",
  "EndpointID": "c64dbe8c9d36c2f75784ccfe51aa854a75536402cf89cd58042959403623f95c",
  "MacAddress": "02:42:ac:12:00:02",
  "IPv4Address": "172.18.0.2/16",
  "IPv6Address": "fc00:f853:ccd:e793::2/64"
}
{
  "Name": "storage-cluster-control-plane",
  "EndpointID": "3dd90fe415d89a31ee480970884695768d2bde757d60ddb8b397e220b301f36b",
  "MacAddress": "02:42:ac:12:00:04",
  "IPv4Address": "172.18.0.4/16",
  "IPv6Address": "fc00:f853:ccd:e793::4/64"
}
{
  "Name": "storage-cluster-worker",
  "EndpointID": "28d4a47609b70b051e003ef3791c97e09641bae0270b28602a3df49cd3ba725a",
  "MacAddress": "02:42:ac:12:00:03",
  "IPv4Address": "172.18.0.3/16",
  "IPv6Address": "fc00:f853:ccd:e793::3/64"
}

```

进入 NFS 服务器 ``nfs-server``，在 ``/nfsshare`` 目录下写入文件 ``NFS.txt`` 备用。

```bash
docker exec -it nfs-server bash -c 'echo "This is NFS" > /nfsshare/NFS.txt'
```

### 2.2 检查 NFS 服务

拉取镜像 [``d3fk/nfs-client``](https://hub.docker.com/r/d3fk/nfs-client) 。

```bash
docker pull d3fk/nfs-client
```

运行容器 ``nfs-client`` ，将容器加入 ``kind`` 网络

```bash
docker run -itd --name nfs-client --privileged=true --net=kind d3fk/nfs-client
```

进入 ``nfs-client`` ，挂载 ``nfs-server`` 上的 ``/nfsshare`` 到 ``nfs-client`` 上的 ``~/temp``。

> 注意：确保 ``nfs-client`` 上存在 ``~/temp`` 目录。

```bash
$ docker exec -it nfs-client sh
mount -t nfs nfs-server:/ ~/temp
```

> 注意：<a id="anchor1">[[1]](#bug1)<a> 如果使用命令 ``mount -t nfs nfs-server:/nfsshare ~/temp`` 则会提示**目录不存在**。因为这个 ``nfs-server`` 镜像设定了 ``fsid=0``，默认挂载的是 ``nfs-server`` 中的共享根目录。如果在 ``nfs-server`` 上存在 ``/nfsshare/a-dir`` 目录，则可以通过命令 ``mount -t nfs nfs-server:/a-dir ~/temp`` 将该目录挂载到 ``nfs-client`` 的 ``~/temp`` 上。

检查挂载 nfs 是否成功。

```bash
$ docker exec -it nfs-client sh
~ # cat ~/temp/NFS.txt 
This is NFS
```

NFS 服务已经可用。

### 2.3 在 k8s 集群安装 NFS CSI 插件

> kind 节点容器 ``kindest/node`` 默认已安装 ``nfs-common`` 客户端。

参照文档 [``https://github.com/kubernetes-csi/csi-driver-nfs/blob/master/docs/install-csi-driver-v4.6.0.md``](https://github.com/kubernetes-csi/csi-driver-nfs/blob/master/docs/install-csi-driver-v4.6.0.md)。

进入 Master 节点 ``storage-cluster-control-plane``

```bash
docker exec -it storage-cluster-control-plane /bin/bash
```

安装CSI插件

> 如果有网络障碍，先在宿主机 ``git clone https://github.com/kubernetes-csi/csi-driver-nfs.git`` 下载文件，``docker cp csi-driver-nfs/ storage-cluster-control-plane:/root/csi-driver-nfs`` 复制到容器内，``cd csi-driver-nfs``、
``./deploy/install-driver.sh master local``本地安装。

```bash
$ curl -skSL https://raw.githubusercontent.com/kubernetes-csi/csi-driver-nfs/v4.6.0/deploy/install-driver.sh | bash -s v4.6.0 --
use local deploy
Installing NFS CSI driver, version: master ...
serviceaccount/csi-nfs-controller-sa created
serviceaccount/csi-nfs-node-sa created
clusterrole.rbac.authorization.k8s.io/nfs-external-provisioner-role created
clusterrolebinding.rbac.authorization.k8s.io/nfs-csi-provisioner-binding created
csidriver.storage.k8s.io/nfs.csi.k8s.io created
deployment.apps/csi-nfs-controller created
daemonset.apps/csi-nfs-node created
NFS CSI driver installed successfully.
```

检查 csi 插件状态，卡在了镜像拉取环节。

```bash
root@storage-cluster-control-plane:~/csi-driver-nfs# kubectl -n kube-system get pod -o wide -l app=csi-nfs-node
NAME                 READY   STATUS         RESTARTS   AGE    IP           NODE                            NOMINATED NODE   READINESS GATES
csi-nfs-node-4hj66   0/3     ErrImagePull   0          3m5s   172.18.0.4   storage-cluster-worker          <none>           <none>
csi-nfs-node-5qj48   0/3     ErrImagePull   0          3m5s   172.18.0.3   storage-cluster-control-plane   <none>           <none>
csi-nfs-node-sz24f   0/3     ErrImagePull   0          3m5s   172.18.0.2   storage-cluster-worker2         <none>           <none>
```

检查日志

```bash
root@storage-cluster-control-plane:~/csi-driver-nfs# kubectl describe pod csi-nfs-node-4hj66 -n kube-system
Name:                 csi-nfs-node-4hj66
...
  Warning  Failed     85s (x2 over 3m34s)   kubelet            Error: ErrImagePull
  Normal   Pulling    85s (x2 over 3m34s)   kubelet            Pulling image "registry.k8s.io/sig-storage/csi-node-driver-registrar:v2.10.0"
  Warning  Failed     42s (x2 over 2m51s)   kubelet            Failed to pull image "registry.k8s.io/sig-storage/csi-node-driver-registrar:v2.10.0": failed to pull and unpack image "registry.k8s.io/sig-storage/csi-node-driver-registrar:v2.10.0": failed to resolve reference "registry.k8s.io/sig-storage/csi-node-driver-registrar:v2.10.0": failed to do request: Head "https://us-west2-docker.pkg.dev/v2/k8s-artifacts-prod/images/sig-storage/csi-node-driver-registrar/manifests/v2.10.0": dial tcp 108.177.125.82:443: connect: connection refused
  Warning  Failed     42s (x2 over 2m51s)   kubelet            Error: ErrImagePull
  Normal   Pulling    42s (x2 over 2m51s)   kubelet            Pulling image "gcr.io/k8s-staging-sig-storage/nfsplugin:canary"
```

该安装脚本的核心是使用 ``kubectl`` 部署了四个 ``yaml`` 文件网络连通性存在障碍，按顺序依次是：

- ``deploy/rbac-csi-nfs.yaml``
- ``deploy/csi-nfs-driverinfo.yaml``
- ``deploy/csi-nfs-controller.yaml``
- ``deploy/csi-nfs-node.yaml``

这些文件中指定的镜像包括

- ``registry.k8s.io/sig-storage/csi-provisioner:v4.0.0``
- ``registry.k8s.io/sig-storage/csi-snapshotter:v6.3.3``
- ``registry.k8s.io/sig-storage/livenessprobe:v2.12.0``
- ``gcr.io/k8s-staging-sig-storage/nfsplugin:canary``
- ``registry.k8s.io/sig-storage/csi-node-driver-registrar:v2.10.0``

在 kind 集群中拉去这些镜像时，网络存在连通障碍，因此我们拉取在宿主机中从阿里云镜像中心拉取镜像，并用 ``docker tag`` 修改对应的tag，再通过 ``kind load`` 加载到集群中。

> 也可以通过代理进行 docker pull，再使用 ``kind load``，**此处使用代理**。

```bash
docker pull registry.k8s.io/sig-storage/csi-provisioner:v4.0.0
docker pull registry.k8s.io/sig-storage/csi-snapshotter:v6.3.3
docker pull registry.k8s.io/sig-storage/livenessprobe:v2.12.0
docker pull gcr.io/k8s-staging-sig-storage/nfsplugin:canary
docker pull registry.k8s.io/sig-storage/csi-node-driver-registrar:v2.10.0

kind load docker-image registry.k8s.io/sig-storage/csi-provisioner:v4.0.0            --name storage-cluster       
kind load docker-image registry.k8s.io/sig-storage/csi-snapshotter:v6.3.3            --name storage-cluster       
kind load docker-image registry.k8s.io/sig-storage/livenessprobe:v2.12.0             --name storage-cluster       
kind load docker-image gcr.io/k8s-staging-sig-storage/nfsplugin:canary               --name storage-cluster   
kind load docker-image registry.k8s.io/sig-storage/csi-node-driver-registrar:v2.10.0 --name storage-cluster                   
```

加载完成后，重新部署 CSI 插件

```bash
yzq@yzq-virtual-machine:~$ docker exec -it storage-cluster-control-plane /bin/bash
root@storage-cluster-control-plane:/# cd ~/csi-driver-nfs/
root@storage-cluster-control-plane:~/csi-driver-nfs# ./deploy/install-driver.sh master local
```

再次检查 CSI 插件状态，已经部署成功

```bash
root@storage-cluster-control-plane:~/csi-driver-nfs# kubectl -n kube-system get pod -o wide -l app=csi-nfs-controller
NAME                                 READY   STATUS    RESTARTS   AGE   IP           NODE                            NOMINATED NODE   READINESS GATES
csi-nfs-controller-65df4b8c9-5d7lx   4/4     Running   0          44m   172.18.0.2   storage-cluster-control-plane   <none>           <none>
root@storage-cluster-control-plane:~/csi-driver-nfs# kubectl -n kube-system get pod -o wide -l app=csi-nfs-node
NAME                 READY   STATUS    RESTARTS   AGE   IP           NODE                            NOMINATED NODE   READINESS GATES
csi-nfs-node-4hj66   3/3     Running   0          8h    172.18.0.4   storage-cluster-worker          <none>           <none>
csi-nfs-node-5qj48   3/3     Running   0          8h    172.18.0.2   storage-cluster-control-plane   <none>           <none>
csi-nfs-node-sz24f   3/3     Running   0          8h    172.18.0.3   storage-cluster-worker2         <none>           <none>
```

### 2.4 创建 NFS 类型的 PV

先创建 Storage Class，它会利用NFS CSI插件动态供应PV。

```yaml
# nfs-sc.yaml
allowVolumeExpansion: true # 允许扩张
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-sc
provisioner: nfs.csi.k8s.io # 指定 CSI 驱动程序
parameters:
  server: nfs-server # NFS 服务器 IP 地址 （Docker 网络）
  share: / # 在NFS服务器上共享的路径
reclaimPolicy: Retain # 可根据需要设置为Delete或其他策略
```

保存为 ``nfs-csi-sc.yaml`` 并应用：

```bash
kubectl apply -f nfs-sc.yaml
```

再创建 PVC，指定 Storage Class，请求 10MB 空间。

```bash
# nfs-pvc
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-pvc
spec:
  accessModes:
    - ReadWriteMany 
  storageClassName: nfs-sc # 使用刚才创建的 Storage Class 名
  resources:
    requests:
      storage: 10Mi # 
```

保存为 ``nfs-pvc.yaml`` 并应用：

```bash
kubectl apply -f nfs-pvc.yaml
```

检查 PVC 状态：

```bash
root@storage-cluster-control-plane:~/csi-exp# kubectl get pvc
NAME      STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
nfs-pvc   Pending                                      nfs-sc         <unset>                 2m14s
root@storage-cluster-control-plane:~/csi-exp# kubectl describe pvc nfs-pvc
Name:          nfs-pvc
Namespace:     default
StorageClass:  nfs-sc
Status:        Pending
Volume:        
Labels:        <none>
Annotations:   volume.beta.kubernetes.io/storage-provisioner: nfs.csi.k8s.io
               volume.kubernetes.io/storage-provisioner: nfs.csi.k8s.io
Finalizers:    [kubernetes.io/pvc-protection]
Capacity:      
Access Modes:  
VolumeMode:    Filesystem
Used By:       <none>
Events:
  Type     Reason                Age                    From                                                                               Message
  ----     ------                ----                   ----                                                                               -------
  Warning  ProvisioningFailed    2m37s (x2 over 2m40s)  nfs.csi.k8s.io_storage-cluster-control-plane_4472bfa5-01fc-4ccd-a3a6-ddf147075886  failed to provision volume with StorageClass "nfs-sc": rpc error: code = InvalidArgument desc = invalid parameter "path" in storage class
  Normal   Provisioning          97s (x7 over 2m40s)    nfs.csi.k8s.io_storage-cluster-control-plane_4472bfa5-01fc-4ccd-a3a6-ddf147075886  External provisioner is provisioning volume for claim "default/nfs-pvc"
  Warning  ProvisioningFailed    97s (x5 over 2m39s)    nfs.csi.k8s.io_storage-cluster-control-plane_4472bfa5-01fc-4ccd-a3a6-ddf147075886  failed to provision volume with StorageClass "nfs-sc": rpc error: code = InvalidArgument desc = invalid parameter "fsType" in storage class
  Warning  ProvisioningFailed    33s                    persistentvolume-controller                                                        storageclass.storage.k8s.io "nfs-sc" not found
  Normal   ExternalProvisioning  3s (x11 over 2m40s)    persistentvolume-controller                                                        Waiting for a volume to be created either by the external provisioner 'nfs.csi.k8s.io' or manually by the system administrator. If volume creation is delayed, please verify that the provisioner is running and correctly registered.

```

再创建一个 Pod ，该 Pod 使用创建的 PVC ``nfs-pvc``。该 Pod 持续写入日志

```bash
apiVersion: v1
kind: Pod
metadata:
  name: nfs-pod
spec:
  containers:
  - name: go-container
    image: go-hello-world-image:v0.0.1
    volumeMounts:
    - mountPath: /mnt/nfs # 挂载点
      name: nfs-volume
    command: ["/bin/sh", "-c"]
    args:
    - while true; do echo "$(date +'%H:%M:%S') Hello World" >> /mnt/nfs/output.txt; sleep 5; done # 每5秒向//mnt/nfs/output.txt中追加当前时间和消息
  volumes:
  - name: nfs-volume
    persistentVolumeClaim:
      claimName: nfs-pvc # 绑定 PVC
```

命名为 ``nfs-pod.yaml`` 并应用：

```bash
kubectl apply -f nfs-pod.yaml
```

检查 pod 状态：

```bash
kubectl get pod
NAME      READY   STATUS    RESTARTS   AGE
nfs-pod   1/1     Running   0          26s
```

### 2.5 检查 NFS PV 的挂载

进入Pod ``nfs-pod``，检查目录 ``/mnt/nfs/`` 目录下的 ``output.txt`` 文件，文件无误。

```bash
root@storage-cluster-control-plane:~/csi-exp# kubectl exec -it nfs-pod -- sh -c "tail /mnt/nfs/output.txt"
11:57:12 Hello World
11:57:17 Hello World
11:57:22 Hello World
11:57:27 Hello World
11:57:32 Hello World
11:57:37 Hello World
11:57:42 Hello World
11:57:47 Hello World
11:57:52 Hello World
11:57:57 Hello World
```

进入 NFS 服务器 ``nfs-server``：

```bash
docker exec -it nfs-server bash
```

检查共享文件夹 ``/nfsshare``:

```bash
root@1538c90c8c07:/nfsshare # cd /nfsshare && ls
root@1538c90c8c07:/nfsshare # cd /nfsshare && ls -l
total 12
-rw-r--r--    1 root     root            12 Feb  9 15:21 NFS.txt
drwxr-xr-x    2 root     root          4096 Feb 10 11:53 pvc-b320c46c-582e-4580-94b7-7eedb1003140
```

存在之前与之的NFS.txt，以及 kubernetes 集群创建的文件夹 ``
pvc-b320c46c-582e-4580-94b7-7eedb1003140`` 。

```bash
root@1538c90c8c07:/nfsshare # cat pvc-b320c46c-582e-4580-94b7-7eedb1003140/output.txt | grep 11:57:57
11:57:57 Hello World
```

进入此目录，检查其内容，是 ``nfs-pod`` Pod 中服务写入的文件。

NFS CSI 插件安装完成。

## 参考

[1] [``https://baijiahao.baidu.com/s?id=1716423711735232537&wfr=spider&for=pc``](https://baijiahao.baidu.com/s?id=1716423711735232537&wfr=spider&for=pc)  
[2] [``https://developer.aliyun.com/article/727327``](https://developer.aliyun.com/article/727327)  
[3] [``https://github.com/kubernetes-csi/csi-driver-nfs/blob/master/docs/install-csi-driver-v4.6.0.md``](https://github.com/kubernetes-csi/csi-driver-nfs/blob/master/docs/install-csi-driver-v4.6.0.md)  
[4] [``https://blog.csdn.net/fengcai_ke/article/details/129151551``](https://blog.csdn.net/fengcai_ke/article/details/129151551)

## BUG 记录

<a id="bug1">[[1]](#anchor1)</a> 挂载 nfs 时，导出目录 ``nfsshare`` 出现错误。
