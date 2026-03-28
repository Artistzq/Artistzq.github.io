---
title: kubelabs——k8s学习与实验【2】容器网络接口（CNI）以及切换CNI插件
date: 2024/1/14 11:23:23
tags:
- k8s
- cni
categories:
- kubelabs
---

# kubelabs——k8s学习与实验【2】容器网络接口（CNI）以及切换CNI插件

## 概念

``Container Network Interface（CNI）``，即容器网络接口，是一种标准化的API规范，旨在为容器提供一致且灵活的网络配置方案。在Kubernetes环境中，Kubelet通过调用遵循CNI标准的API与各类网络插件进行交互，以实现容器间多样化的网络连接与管理。接口定义的基本操作包括 ``ADD`` 和 ``DEL`` 等，负责将Pod添加到网络和从网络中删除。

k8s通过配置文件来设置使用的CNI插件。在创建pod之前，<a id="ref1"> 管理员可以放置好CNI配置文件 </a> ，当kubelet创建pod时，通过读取配置文件，根据其中指定的CNI插件，执行相应的二进制文件，从而配置Pod网络。当kubelet对Pod执行 ADD 操作时，对应的CNI插件将为Pod分配IP地址、设置路由规则以及其他必要的网络配置；当Pod终止或被删除时，kubelet会执行 DEL 操作，这时CNI插件负责清理相关的网络资源，如释放IP地址等。

不同的CNI插件有不同的实现方式，例如Overlay和Underlay。常见的CNI插件有Calico和Flannel等，它们各自具备不同的功能特点与适用场景，从而满足不同规模、性能及安全需求的容器网络部署要求。

- ``Overlay Network``

  - 是一种虚拟化技术。通过在底层数据包的基础上，添加额外的封装头部信息，借助隧道打通网络连接，可以屏蔽底层网络通信方式的差异，兼容多种异构底层网络。部署方便；扩展性好。
  
  - 例如：Flannel-vxlan 插件

- ``Underlay Network``

  - 提供了真正的物理层和数据链路层的连接，网络性能直接取决于硬件设备。直接基于底层传输数据，没有额外数据封装，性能高，吞吐量大，延迟低。但在异构网络环境下扩展困难，网络配置复杂。某些基于Underlay的插件无法使用复杂均衡和服务发现。

  - 例如：clico-bgp 插件

实际选择时，要从多个需求出发考虑，例如安全需求、负载均衡需求、性能需求等。

## 实验

### 1 创建集群，不配置CNI插件

``kind`` 默认使用了 ``kindnetd`` 作为网络插件。

```bash
yzq@ubuntu:~$ docker exec -it hello-cluster-worker ls /etc/cni/net.d
10-kindnet.conflist
```

为了实验，设置创建集群时不配置CNI插件。

```bash
$ kind create cluster --name cni-test-cluster --config kind-config.yaml
# 加载本地镜像（如果不加载会出现ImagePullBackOff、ErrImagePull等错误）
$ kind load docker-image go-hello-world-image:v0.0.1 --name cni-test-cluster
```

```yaml
# kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  # the default CNI will not be installed
  disableDefaultCNI: true
nodes:
- role: control-plane
  image: kindest/node:v1.29.0
- role: worker
  image: kindest/node:v1.29.0
- role: worker
  image: kindest/node:v1.29.0
```

查看 ``/etc/cni/net.d`` 下目录为空。

查看 ``nodes``，状态为 not ready。

```bash
$ kubectl get nodes
NAME                             STATUS     ROLES           AGE    VERSION
cni-test-cluster-control-plane   NotReady   control-plane   2d2h   v1.29.0
cni-test-cluster-worker          NotReady   <none>          2d2h   v1.29.0
cni-test-cluster-worker2         NotReady   <none>          2d2h   v1.29.0
```

查看 ``pods`` 。``core-dns`` 和 ``local-path-provisioner`` 不在运行。

```bash
$ kubectl get pods --all-namespaces
NAMESPACE            NAME                                                     READY   STATUS    RESTARTS         AGE
kube-system          coredns-76f75df574-9swqg                                 0/1     Pending   0                47h
kube-system          coredns-76f75df574-j5mgx                                 0/1     Pending   0                47h
kube-system          etcd-cni-test-cluster-control-plane                      1/1     Running   0                3m7s
kube-system          kube-apiserver-cni-test-cluster-control-plane            1/1     Running   0                3m7s
kube-system          kube-controller-manager-cni-test-cluster-control-plane   1/1     Running   7 (3m25s ago)    47h
kube-system          kube-proxy-l8cpb                                         1/1     Running   71 (3m25s ago)   47h
kube-system          kube-proxy-nwpzn                                         1/1     Running   4 (3m25s ago)    47h
kube-system          kube-proxy-zhhfn                                         1/1     Running   71 (3m25s ago)   47h
kube-system          kube-scheduler-cni-test-cluster-control-plane            1/1     Running   7 (3m25s ago)    47h
local-path-storage   local-path-provisioner-6f8956fb48-f59n6                  0/1     Pending   0                47h
```

``kubectl apply -f cni-deploy.yaml`` 部署服务后，Pod处于Pending状态。

```bash
kubectl get pods
NAME                                   READY   STATUS    RESTARTS   AGE
cni-test-deployment-58569cd74c-qrxz5   0/1     Pending   0          142m
cni-test-deployment-58569cd74c-v86tf   0/1     Pending   0          142m
cni-test-deployment-58569cd74c-wg9rn   0/1     Pending   0          142m
```

### 2 手动添加flannel插件

Flannel插件旨在为不同节点上的容器重新规划IP地址的使用规则，分配同属一个内网且不重复的IP地址。

#### 2.1 准备好配置文件

如[上文](#ref1)提到，添加CNI插件需要将配置文件放入节点的/etc/cni/net.d。

可以自动化地进行。

```sh
# 下载 kube-flannel.yaml 文件
wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
# 部署 flannel
kubectl apply -f kube-flannel.yml
```

节点已经处于 ``Ready`` 状态。

```bash
$ kubectl get nodes --all-namespaces
NAME                             STATUS   ROLES           AGE    VERSION
cni-test-cluster-control-plane   Ready    control-plane   2d2h   v1.29.0
cni-test-cluster-worker          Ready    <none>          2d2h   v1.29.0
cni-test-cluster-worker2         Ready    <none>          2d2h   v1.29.0
```

配置文件已自动写入工作节点 ``/etc/cni/net.d`` 目录下。

```bash
$ docker exec -it cni-test-cluster-worker ls /etc/cni/net.d
10-flannel.conflist
```

但pods依然未就绪：

```bash
$ kubectl get pods --all-namespaces
NAMESPACE            NAME                                                     READY   STATUS              RESTARTS         AGE
default              cni-test-deployment-58569cd74c-qrxz5                     0/1     ContainerCreating   0                165m
default              cni-test-deployment-58569cd74c-v86tf                     0/1     ContainerCreating   0                165m
default              cni-test-deployment-58569cd74c-wg9rn                     0/1     ContainerCreating   0                165m
kube-flannel         kube-flannel-ds-bkl5b                                    1/1     Running             0                4m1s
kube-flannel         kube-flannel-ds-qfhwm                                    1/1     Running             0                4m1s
kube-flannel         kube-flannel-ds-xwn6m                                    1/1     Running             0                4m1s
kube-system          coredns-76f75df574-9swqg                                 0/1     ContainerCreating   0                2d2h
kube-system          coredns-76f75df574-j5mgx                                 0/1     ContainerCreating   0                2d2h
kube-system          etcd-cni-test-cluster-control-plane                      1/1     Running             0                5m31s
kube-system          kube-apiserver-cni-test-cluster-control-plane            1/1     Running             0                5m31s
kube-system          kube-controller-manager-cni-test-cluster-control-plane   1/1     Running             8 (6m26s ago)    2d2h
kube-system          kube-proxy-l8cpb                                         1/1     Running             72 (6m26s ago)   2d2h
kube-system          kube-proxy-nwpzn                                         1/1     Running             5 (6m26s ago)    2d2h
kube-system          kube-proxy-zhhfn                                         1/1     Running             72 (6m26s ago)   2d2h
kube-system          kube-scheduler-cni-test-cluster-control-plane            1/1     Running             8 (6m26s ago)    2d2h
local-path-storage   local-path-provisioner-6f8956fb48-f59n6                  0/1     ContainerCreating   0                2d2h

```

通过 ``kubectl describe pod cni-test-deployment-58569cd74c-qrxz5`` 查看信息，提示在委派 ``ADD`` 操作时，无法找到 ``/opt/cni/bin`` 目录下的``bridge``二进制文件插件。

```bash
$ kubectl describe pod cni-test-deployment-58569cd74c-qrxz5
...
Conditions:
  Type                        Status
  PodReadyToStartContainers   False 
  Initialized                 True 
  Ready                       False 
  ContainersReady             False 
  PodScheduled                True 
Events:
  Type     Reason                  Age                     From               Message
  ----     ------                  ----                    ----               -------
  Warning  FailedCreatePodSandBox  4m16s                   kubelet            Failed to create pod sandbox: rpc error: code = Unknown desc = failed to setup network for sandbox "0ca3e187ace57dd4af46bb8e4c60b5b5973510f5f4a02e8efe1954d34f1b7101": plugin type="flannel" failed (add): failed to delegate add: failed to find plugin "bridge" in path [/opt/cni/bin]
```

#### 2.2 手动安装插件的二进制文件

在运行 kind 的宿主机上下载文件

```bash
# 下载文件
$ wget https://github.com/containernetworking/plugins/releases/download/v1.4.0/cni-plugins-linux-amd64-v1.4.0.tgz
```

复制到多个工作节点（docker容器），并解压。创建脚本自动处理这一过程：

```bash
# ./unzip_to_docker.sh

# 获取所有名称以 'cni-test-cluster' 开头的容器
container_names=$(docker ps --format "{{.Names}}" | grep '^cni-test-cluster')

# 定义本地压缩文件路径和目标解压目录（请替换为实际值）
local_archive="cni-plugins-linux-amd64-v1.4.0.tgz"
target_directory="/opt/cni/bin"

# 遍历每个匹配到的容器
for name in $container_names; do
    # 将本地压缩文件复制到容器内
    docker cp "$local_archive" "$name:$target_directory"
    # 在容器内部执行解压命令
    docker exec -it "$name" bash -c "cd '$target_directory' && tar -zxvf '$local_archive'"
    # 展示文件夹下内容
    docker exec -it "$name" bash -c "cd '$target_directory' && ls"
done
echo "Finished deploying files to containers."
```

在运行 kind 的宿主机上运行此脚本。

```bash
$ bash unzip_to_docker.sh 
Successfully copied 46.9MB to cni-test-cluster-control-plane:/opt/cni/bin
./
./loopback
./bandwidth
./ptp
./vlan
./host-device
./tuning
./vrf
./sbr
./tap
./dhcp
./static
./firewall
./macvlan
./dummy
./bridge
./ipvlan
./portmap
./host-local
bandwidth  cni-plugins-linux-amd64-v1.4.0.tgz  dummy	 flannel      host-local  loopback  portmap  sbr     tap     vlan
bridge	   dhcp				       firewall  host-device  ipvlan	  macvlan   ptp      static  tuning  vrf
Successfully copied 46.9MB to cni-test-cluster-worker2:/opt/cni/bin
./
./loopback
./bandwidth
./ptp
./vlan
./host-device
./tuning
./vrf
./sbr
./tap
./dhcp
./static
./firewall
./macvlan
./dummy
./bridge
./ipvlan
./portmap
./host-local
bandwidth  cni-plugins-linux-amd64-v1.4.0.tgz  dummy	 flannel      host-local  loopback  portmap  sbr     tap     vlan
bridge	   dhcp				       firewall  host-device  ipvlan	  macvlan   ptp      static  tuning  vrf
Successfully copied 46.9MB to cni-test-cluster-worker:/opt/cni/bin
./
./loopback
./bandwidth
./ptp
./vlan
./host-device
./tuning
./vrf
./sbr
./tap
./dhcp
./static
./firewall
./macvlan
./dummy
./bridge
./ipvlan
./portmap
./host-local
bandwidth  cni-plugins-linux-amd64-v1.4.0.tgz  dummy	 flannel      host-local  loopback  portmap  sbr     tap     vlan
bridge	   dhcp				       firewall  host-device  ipvlan	  macvlan   ptp      static  tuning  vrf
Finished deploying files to containers.
```

进入workNode查看网络，可以看到网卡flannel.1

``` bash
$ docker exec -it cni-test-cluster-worker /bin/bash
root@cni-test-cluster-worker:/# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: flannel.1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN group default 
    link/ether a6:b3:45:67:33:b2 brd ff:ff:ff:ff:ff:ff
    inet 10.244.1.0/32 scope global flannel.1
       valid_lft forever preferred_lft forever
10: eth0@if11: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 02:42:ac:12:00:04 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.18.0.4/16 brd 172.18.255.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fc00:f853:ccd:e793::4/64 scope global nodad 
       valid_lft forever preferred_lft forever
    inet6 fe80::42:acff:fe12:4/64 scope link 
       valid_lft forever preferred_lft forever
```

进入masterNode后，检查子网环境变量，已写入。

```bash
root@cni-test-cluster-control-plane:/var/run/flannel# cat subnet.env 
FLANNEL_NETWORK=10.244.0.0/16
FLANNEL_SUBNET=10.244.0.1/24
FLANNEL_MTU=1450
FLANNEL_IPMASQ=true
```

检查pods，``coredns`` 和自定义的pod ``cni-test-deployment`` 已正常运行
```bash
$ kubectl get pods --all-namespaces
NAMESPACE            NAME                                                     READY   STATUS    RESTARTS       AGE
default              cni-test-deployment-58569cd74c-7hdmp                     1/1     Running   0              9m52s
default              cni-test-deployment-58569cd74c-9wcqw                     1/1     Running   0              9m52s
default              cni-test-deployment-58569cd74c-bbwhk                     1/1     Running   0              9m52s
kube-flannel         kube-flannel-ds-7dfkb                                    1/1     Running   1 (35m ago)    56m
kube-flannel         kube-flannel-ds-vfvbz                                    1/1     Running   1 (35m ago)    56m
kube-flannel         kube-flannel-ds-zc88f                                    1/1     Running   2 (34m ago)    56m
kube-system          coredns-76f75df574-9swqg                                 1/1     Running   0              2d3h
kube-system          coredns-76f75df574-j5mgx                                 1/1     Running   0              2d3h
kube-system          etcd-cni-test-cluster-control-plane                      1/1     Running   1 (35m ago)    95m
kube-system          kube-apiserver-cni-test-cluster-control-plane            1/1     Running   1 (35m ago)    95m
kube-system          kube-controller-manager-cni-test-cluster-control-plane   1/1     Running   9 (35m ago)    2d3h
kube-system          kube-proxy-l8cpb                                         1/1     Running   73 (35m ago)   2d3h
kube-system          kube-proxy-nwpzn                                         1/1     Running   6 (35m ago)    2d3h
kube-system          kube-proxy-zhhfn                                         1/1     Running   73 (35m ago)   2d3h
kube-system          kube-scheduler-cni-test-cluster-control-plane            1/1     Running   9 (35m ago)    2d3h
local-path-storage   local-path-provisioner-6f8956fb48-f59n6                  1/1     Running   0              2d3h
```

进入在 ``worker2`` 节点上运行的 ``IP = 10.244.2.7`` 的pod ，访问在 ``worker`` 节点上运行的``IP = 10.244.1.8`` 的pod，可以访问。

``` bash
$ kubectl get pods -owide
NAME                                   READY   STATUS    RESTARTS   AGE   IP           NODE                       NOMINATED NODE   READINESS GATES
cni-test-deployment-58569cd74c-7hdmp   1/1     Running   0          11m   10.244.2.7   cni-test-cluster-worker2   <none>           <none>
cni-test-deployment-58569cd74c-9wcqw   1/1     Running   0          11m   10.244.1.8   cni-test-cluster-worker    <none>           <none>
cni-test-deployment-58569cd74c-bbwhk   1/1     Running   0          11m   10.244.1.7   cni-test-cluster-worker    <none>           <none>

$ kubectl exec cni-test-deployment-58569cd74c-7hdmp -it sh
/app # curl 10.244.1.8:8080/test
hello world
/app # exit
```
