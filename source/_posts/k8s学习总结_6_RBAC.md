---
title: kubelabs——k8s学习与实验【6】k8s中的RBAC：基于角色的访问控制实验
date: 2024/2/19 14:28:47
tags:
- k8s
- rbac
categories:
- kubelabs
---

# RBAC

## 1 概念

完整的认证与鉴权包含两个部分：

- 认证。某用户是否能通过认证，从而访问系统。
- 鉴权。通过了认证的用户是否有权限做某些操作。

本篇文章重点讨论的是第二点，即鉴权部分。

### 1.1 基于角色的访问控制（Role-Based Access Control）

基于角色的访问控制是一种访问控制的策略，通过为用户分配角色，并将权限与这些角色相关联来管理对资源的访问。该模型的主要组成部分包括角色、权限和用户。角色是一组权限的集合，用户通过被分配到不同的角色来获得相应的权限。权限定义了对资源的操作，例如读取、写入、删除等。用户是系统中的实体，可以被分配到一个或多个角色。

在 RBAC 中，用户通过被分配到的角色来获得对资源的访问权限，而不是直接将权限赋予用户。这种方法有助于简化权限管理，特别是在大型组织中，其中可能存在大量用户和资源。通过RBAC，管理员可以更轻松地管理用户的权限，只需维护角色和与之相关的权限即可，而无需为每个用户单独分配权限。

假设有三个用户和三个权限。其中，User1 需要拥有所有三个权限，而 User2 和 User3 则需要共享权限2和权限3。

在不使用 RBAC 模型的情况下，权限管理可能会变得复杂且缺乏复用性。例如，每个用户可能需要单独分配其所需的权限，这将导致权限的重复分配和管理。

![访问控制](https://92697-imgs.oss-cn-hangzhou.aliyuncs.com/blogs/20240217195809.png)

通过 RBAC 模型，我们可以更有效地管理这些权限。我们可以创建两个角色，然后将权限与这两个角色相关联：角色1拥有所有三个权限，而角色2只拥有权限2和权限3。

接下来，我们将用户分配到相应的角色中：用户1被分配到角色1，而用户2和用户3则被分配到角色2。这样一来，即使有新用户加入或权限需求发生变化，也只需调整角色与权限的关联关系，而无需单独为每个用户管理权限。

![rbac](https://92697-imgs.oss-cn-hangzhou.aliyuncs.com/blogs/20240217195934.png)

通过在用户和权限之间加入角色，RBAC 将用户和权限之间复杂的关系解耦，提高了系统的扩展性。当用户权限发生变动，只需要更改其拥有的角色属性；当需要新的权限组合，只需要添加新的角色。

### 1.2 Kubernetes 中的 RBAC

Kubernetes 通过运行在 Master 节点上的 API Server 向外提供服务。API 访问大致包含认证和授权先后两个步骤，其中认证可以有 Service Account 服务实现；在授权步骤，Kubernetes 提供了 ABAC（基于属性的访问控制）、RBAC、Webhook、Node、AlwaysDeny（一直拒绝）和AlwaysAllow（一直允许）6种模式。我们深入学习讨论 RBAC 模式。

Kubernetes 中的 RBAC 包含多个基本概念和元素：

- Role（角色）：一个 Role 定义了对于资源的访问控制的一组规则，资源的范围限定在某个特定的 namespace
- RoleBinding：将角色和对象进行绑定，范围限定在 namespace；
- ClusterRole：集群级别的角色，定义了集群范围内对于某资源的访问规则
- ClusterRoleBinding：集群级别的角色绑定，范围不限定于 namespace 之内

![k8s-rbac](https://92697-imgs.oss-cn-hangzhou.aliyuncs.com/blogs/20240217205148.png)

## 2 实验

> 前置工作：利用第一篇博客的步骤创建一个集群 ``rbac-cluster`` 备用。

### 2.1 创建账号

创建两个通过证书认证的账号，分别是 ``user1`` 和 ``user2`` 。首先进入集群 Master 节点 ``rbac-cluster-control-plane``，

```bash
docker exec -it rbac-cluster-control-plane /bin/bash
```

分别为 ``user1`` 和 ``user2`` 生成证书签名请求（CSR）文件，以及相应的私钥：

```bash
openssl req -new -newkey rsa:4096 -nodes -out user1.csr -keyout user1.key -subj "/CN=user1/O=Kubernetes User"
openssl req -new -newkey rsa:4096 -nodes -out user2.csr -keyout user2.key -subj "/CN=user2/O=Kubernetes User"
```

使用集群的 CA 私钥签署上述 CSR 文件，生成客户端证书：

```bash
openssl x509 -req -in user1.csr -CA /etc/kubernetes/pki/ca.crt -CAkey /etc/kubernetes/pki/ca.key -CAcreateserial -out user1.crt -days 365
openssl x509 -req -in user2.csr -CA /etc/kubernetes/pki/ca.crt -CAkey /etc/kubernetes/pki/ca.key -CAcreateserial -out user2.crt -days 365
```

改写 kubeconfig，添加用户和上下文信息

```bash
bash create_config.sh -n user1 -c "/user1.crt" -k "/user1.key"
bash create_config.sh -n user2 -c "/user2.crt" -k "/user2.key"
```

``create_config.sh`` 脚本内容如下：

```bash
# create_config.sh
# 传入参数 n
while getopts ":n:c:k:" opt; do
  case $opt in
    n) user_name="$OPTARG";;
    c) crt_path="$OPTARG";;
    k) key_path="$OPTARG";;
    \?) echo "Invalid option -$OPTARG" >&2 ;;
  esac
done

shift $((OPTIND-1))

USER_CERT_PATH="$crt_path" # 刚才生成的.crt文件位置
USER_KEY_PATH="$key_path" # 刚才生成的.key文件位置

kubectl config set-credentials $user_name --client-certificate=$USER_CERT_PATH --client-key=$USER_KEY_PATH

kubectl config set-cluster rbac-cluster --server=https://rbac-cluster-control-plane:6443

kubectl config set-context $user_name-context --user=$user_name --cluster=rbac-cluster
```

查看 kubectl config，包含了user1 和 user2 。

```bash
root@rbac-cluster-control-plane:/# kubectl config view
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: DATA+OMITTED
    server: https://rbac-cluster-control-plane:6443
  name: rbac-cluster
contexts:
- context:
    cluster: rbac-cluster
    user: kubernetes-admin
  name: kubernetes-admin@rbac-cluster
- context:
    cluster: rbac-cluster
    user: user1
  name: user1-context
- context:
    cluster: rbac-cluster
    user: user2
  name: user2-context
current-context: kubernetes-admin@rbac-cluster
kind: Config
preferences: {}
users:
- name: kubernetes-admin
  user:
    client-certificate-data: DATA+OMITTED
    client-key-data: DATA+OMITTED
- name: user1
  user:
    client-certificate: /user1.crt
    client-key: /user1.key
- name: user2
  user:
    client-certificate: /user2.crt
    client-key: /user2.key
```

切换到 user1 ，尝试访问 pod ，提示无权限

```bash
root@rbac-cluster-control-plane:/# kubectl config use-context user1-context
Switched to context "user1-context".
root@rbac-cluster-control-plane:/# kubectl get pods
Error from server (Forbidden): pods is forbidden: User "user1" cannot list resource "pods" in API group "" in the namespace "default"
```

### 2.2 创建角色

先切换成 ``kubernetes-admin`` 用户，以为 ``user1`` 和 ``user2`` 创建角色。

```bash
kubectl config use-context kubernetes-admin@rbac-cluster
```

#### 创建 Role

创建包括查看、创建、删除 Pod 的角色 ``pod-read-write`` 和 只能查看 Pod 的角色 ``pod-read-only`` 。``role.yaml`` 配置文件如下：

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-read-write
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list", "create", "update", "patch", "delete"]

---

apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-read-only
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
```

#### 创建 RoleBinding

> 没有绑定 ServiceAccount，而是直接绑定 User

绑定 ``user1`` 到 ``pod-read-write`` 角色，绑定 ``user2`` 到 ``pod-read-only`` 角色。``role-bingding.yaml`` 配置文件如下：

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: user1-pod-access
  namespace: default
subjects:
- kind: User
  name: user1
  namespace: default
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: pod-read-write

---

apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: user2-pod-access
  namespace: default
subjects:
- kind: User
  name: user2
  namespace: default
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: pod-read-only
```

使用 kubectl apply 这些配置：

```bash
kubectl apply -f role.yaml
kubectl apply -f role-bingding.yaml
```

### 2.3 验证 RBAC

使用 ``user1`` 创建一个 pod，镜像使用第一篇博客的 ``go-hello-world-image``。

切换到 ``user1`` 

```bash
kubectl config use-context user1-context
```

创建一个 pod ，其配置文件如下。

```bash
# ~/Documents/kubelabs/rbac-test/pod-hello.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-hello
spec:
  containers:
  - name: myapp-container
    image: go-hello-world-image:v0.0.1
```

部署该 pod

```bash
kubectl apply -f pod-hello.yaml 
```

查看 pod ，有权限进行查看操作。

```bash
root@rbac-cluster-control-plane:/# kubectl get pods
NAME        READY   STATUS    RESTARTS   AGE
pod-hello   1/1     Running   0          4s
```

切换到 ``user2``，并尝试 ``get pods`` 操作，有权限；尝试删除操作，**无权限**

```bash
root@rbac-cluster-control-plane:/# kubectl config use-context user2-context
Switched to context "user2-context".
root@rbac-cluster-control-plane:/# kubectl get pods
NAME        READY   STATUS    RESTARTS   AGE
pod-hello   1/1     Running   0          119s
root@rbac-cluster-control-plane:/# kubectl delete pod pod-hello
Error from server (Forbidden): pods "pod-hello" is forbidden: User "user2" cannot delete resource "pods" in API group "" in the namespace "default"
```

切换回 ``user1``，并删除。有权限进行此操作，Pod被删除。

```bash
root@rbac-cluster-control-plane:/# kubectl config use-context user1-context
Switched to context "user1-context".
root@rbac-cluster-control-plane:/# kubectl delete pod pod-hello
pod "pod-hello" deleted
root@rbac-cluster-control-plane:/# kubectl get pods
No resources found in default namespace.
```
