---
date: 2021-03-20T14:41:38+08:00  # 创建日期
author: "Rustle Karl"  # 作者

title: "K8s 初次手动安装体验"
url:  "posts/quickstart/install/quickstart"
tags: [ "Docker", "K8s"]
categories: [ "K8s 学习笔记"]

index: true  # 是否可以被索引
toc: true  # 是否自动生成目录
draft: false  # 草稿
---

手动安装体验，熟悉基本安装流程。实际现在有很多自动化安装方式，但不利于理解。

## 1. 机器准备

准备相互连通的三台 linux ，虽然 kubeadm 支持在单节点部署一个 k8s 集群，但是为了能做到真正演示 k8s 集群工作原理，建议使用多个节点。

这里三台机器分别是 ubuntu@node1、ubuntu@node2、ubuntu@node3，并且关闭三个节点的 swap。

在此之前认为已经进行了基本设置，比如 SSH。

### 关闭防火墙

```shell
systemctl stop firewalld && systemctl disable firewalld
```

### 关闭 Swap

```shell
cat /etc/fstab
```

```shell
sed -i '/swap/s/^/#/' /etc/fstab
```

或者：

```shell
sed -i '/swap/s/^\(.*\)$/#\1/g' /etc/fstab
```

### 临时关闭

```shell
swapoff -a
```

## 2. 准备 docker

k8s 本质上一个容器编排工具，所以容器是 k8s 运作的前提，k8s 支持各种容器运行时，这里我们使用 docker。

见 Docker 相关笔记。

```shell
cat /etc/docker/daemon.json
```

```shell
tee /etc/docker/daemon.json <<-'EOF'
{
  "insecure-registries": [
    "192.168.0.16:5000"
  ],
  "registry-mirrors": [
    "http://192.168.0.16:5000",
    "https://ba0d43c4ea0f4de5a5528e288a251804.mirror.swr.myhuaweicloud.com"
  ]
}
EOF
```

### 测试

```shell
docker run --rm hello-world
```

## 3. 安装 k8s 的基础组件

https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/

任选一台机器作 k8s master 节点，后面所有的安装如果没有特殊标明都在 master 上执行，使用 sudo 命令或 root 用户依次执行下面的命令完成以下各个组件的安装:

```shell
apt-get update && apt-get install -y apt-transport-https ca-certificates curl
```

```shell
curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
```

```shell
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

```shell
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

kubelet 现在每隔几秒就会重启，因为它陷入了一个等待 kubeadm 指令的死循环。

这里简单介绍一下这三个组件的作用：

- kubelet 是 work node 节点负责 Pod 生命周期状态管理以及和 master 节点交互的组件
- kubectl k8s 的命令行工具，负责和 master 节点交互
- kubeadm 搭建 k8s 的官方工具

## 4. 安装 k8s 依赖的各种镜像

使用 root 执行下面命令

```shell
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
```

--pod-network-cidr=192.168.0.0/16 是 calica 网络组件必须的参数。

安装完成之后会有如下的输出：

```
kubeadm join 192.168.75.134:6443 --token 74mit6.k2u4py7r0xw41kxa \
        --discovery-token-ca-cert-hash sha256:b72a32ff16e9f6239d00b339df1b12960dca8e495a29b4468eb4961d29316755
```

**撤回上面的初始化**

```bash
kubeadm reset
```

```bash
rm -rf $HOME/.kube
```

按照提示你应该再继续执行以下操作：

- 让 k8s 的命令可以在非 root 用户下执行
- 安装 k8s 的网络组件

所以你要记住上面输出的提示操作，找个地方把他们存下来。

## 5. 让非 root 用户可以使用 k8s

```shell
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

为了能够在 worker 节点也可以使用 kubectl 命令，建议再 worker node 加入集群之后，把 `$HOME/.kube/config` 文件也复制一份到 worker node 节点。

或者，如果你是 root 用户，则可以运行：

```shell
export KUBECONFIG=/etc/kubernetes/admin.conf
```

## 6. 安装 Calico 网络组件

k8s 的网络组件非常多，为了演示方便我们使用 Calico:

```shell
kubectl create -f https://projectcalico.docs.tigera.io/manifests/tigera-operator.yaml
```

```bash
kubectl create -f https://projectcalico.docs.tigera.io/manifests/custom-resources.yaml
```

```bash
watch kubectl get pods -n calico-system
```

安装完成之后检查 corndns :

```shell
kubectl get pods --all-namespaces
```

确保所有的 STATUS 都是 running 说明安装成功。

不成功就放弃吧。因为没有解决方法。

## 7. 加入 worker node

在上面 kubeadm init 命令的输出中有加入节点的命令:

```shell
kubeadm join 10.10.76.89:6443 --token t51nk2.4rtw6twkmca53nbu \
    --discovery-token-ca-cert-hash sha256:8356397a4ccd20b2d6506130e8bb4ad13ccc3fe33302296d3c48651601be5bfd
```

这个 token 是必须的，而且 token 的有效期是 24 小时，分别在其它被选为 work node 的节点执行这个加入命令，token 的获取和使用参考 https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-token/

执行完成之后，在 master 节点执行 `kubectl get nodes`，会看到如下输出，说明其他节点加入成功:

```
NAME           STATUS   ROLES    AGE    VERSION
10-10-158-97   Ready    <none>   2d2h   v1.14.1
10-10-23-227   Ready    <none>   2d2h   v1.14.1
10-10-76-89    Ready    master   2d2h   v1.14.1
```

查看组成 k8s 的组件:

```shell
kubectl get pods --namespace=kube-system 
```

```shell
NAME                                  READY   STATUS    RESTARTS   AGE
calico-node-9vrt4                     2/2     Running   0          2d22h
calico-node-rklrf                     2/2     Running   0          2d22h
calico-node-wb7ln                     2/2     Running   0          2d22h
coredns-9846cf96-dblrm                1/1     Running   2          2d22h
coredns-9846cf96-x7z7l                1/1     Running   2          2d22h
etcd-10-10-76-89                      1/1     Running   2          2d22h
kube-apiserver-10-10-76-89            1/1     Running   0          2d22h
kube-controller-manager-10-10-76-89   1/1     Running   0          2d22h
kube-proxy-2nbm6                      1/1     Running   0          2d22h
kube-proxy-vvdpk                      1/1     Running   0          2d22h
kube-proxy-whjqt                      1/1     Running   0          2d22h
kube-scheduler-10-10-76-89            1/1     Running   0          2d22h
```

可以看到 k8s 主要有以下核心组组件组成:

- etcd 保存了整个集群的状态；
- apiserver 提供了资源操作的唯一入口，并提供认证、授权、访问控制、API 注册和发现等机制；
- controller manager 负责维护集群的状态，比如故障检测、自动扩展、滚动更新等；
- scheduler 负责资源的调度，按照预定的调度策略将 Pod 调度到相应的机器上；
- kubelet 负责维护容器的生命周期，同时也负责 Volume（CVI）和网络（CNI）的管理；
- Container runtime 负责镜像管理以及 Pod 和容器的真正运行（CRI）；
- kube-proxy 负责为 Service 提供 cluster 内部的服务发现和负载均衡
