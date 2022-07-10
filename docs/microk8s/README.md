---
date: 2022-07-10T11:10:48+08:00
author: "Rustle Karl"

title: "MicroK8s 部署教程"
url:  "posts/k8s/docs/microk8s/README"  # 永久链接
tags: [ "K8S", "README" ]  # 标签
categories: [ "K8S 学习笔记" ]  # 分类

toc: true  # 目录
draft: true  # 草稿
---

MicroK8s 是由 Canonical 开发的 Kubernetes 发行版，其突出特点是部署快速简单，对于本地运行 Kubernetes 来说，十分方便。

与 Minikube 不同，IT 管理员或开发人员可以使用 MicroK8s 创建多节点集群。如果 MicroK8s 在 Linux 上运行，甚至不需要 VM。在 Windows 和 macOS 上，MicroK8s 使用名为 Multipass 的 VM 框架为 Kubernetes 集群创建 VM。

## 通过虚拟机安装

```shell
multipass launch -n microk8s -c 4 -m 8G -d 80G 22.04
```

```shell
multipass shell microk8s
```

```shell
sudo snap install microk8s --classic
```

## 基本配置

https://microk8s.io/docs/getting-started


### 三级

```shell

```

```shell

```


## 二级

### 三级

```shell

```

```shell

```


## 二级

### 三级

```shell

```

```shell

```


## 二级

### 三级

```shell

```

```shell

```


