---
date: 2022-06-26T23:57:06+08:00
author: "Rustle Karl"

title: "K3s 基础教程"
url:  "posts/k8s/docs/k3s/README"  # 永久链接
tags: [ "Docker", "README" ]  # 标签
categories: [ "Docker 学习笔记" ]  # 分类

toc: true  # 目录
draft: true  # 草稿
---

> https://github.com/k3s-io/k3s.git

https://docs.rancher.cn/docs/k3s/_index

K3s 是一个轻量级的 Kubernetes 发行版，它针对边缘计算、物联网等场景进行了高度优化。

- K3S Server 端最低内存要求：512 MB
- K3S Agent 端内存最低要求：75MB
- 磁盘空间最低要求：200 MB
- 支持的硬件架构：x86_64, ARMv7, ARM64

## 私有镜像配置

## 安装

```shell
apt install -y curl
```

```shell
curl -sfL https://get.k3s.io | sh -
```

安装完成之后，服务会自动启动。另外：

- K3s 服务将被配置为在节点重启后或进程崩溃或被杀死时自动重启。
- 将安装其他实用程序，包括 kubectl、crictl、ctr、k3s-killall.sh 和 k3s-uninstall.sh。
- 将 kubeconfig 文件写入到 /etc/rancher/k3s/k3s.yaml，由 K3s 安装的 kubectl 将自动使用该文件

## 获取运行状态

```shell
systemctl status k3s.service
```

```shell
journalctl -xeu k3s.service
```

## 确认是否就绪：

```shell
sudo kubectl get nodes
```

## 获取节点 Token

```shell
cat /var/lib/rancher/k3s/server/node-token
```

```
K10fbc1f2145b06ef930a18b2be5867ff161c14652362292b4903dfbfd76107d6e9::server:c5a741686bc1dc85e436777be83a9c00
```

## 加入工作节点

```shell
export K3S_URL=https://10.110.155.246:6443
```

```shell
export K3S_TOKEN=K10fbc1f2145b06ef930a18b2be5867ff161c14652362292b4903dfbfd76107d6e9::server:c5a741686bc1dc85e436777be83a9c00
```

```shell
curl -sfL https://get.k3s.io | sh -
```

每台计算机必须具有唯一的主机名。如果您的计算机没有唯一的主机名，请传递 K3S_NODE_NAME 环境变量，并为每个节点提供一个有效且唯一的主机名。

```shell
watch sudo kubectl get nodes
```

## 卸载

如果您使用安装脚本安装了 K3s，那么在安装过程中会生成一个卸载 K3s 的脚本。

卸载 K3s 会删除集群数据和所有脚本。要使用不同的安装选项重新启动集群，请使用不同的标志重新运行安装脚本。

```shell
/usr/local/bin/k3s-uninstall.sh
```

```shell
/usr/local/bin/k3s-agent-uninstall.sh
```
