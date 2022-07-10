---
date: 2022-06-27T12:43:36+08:00
author: "Rustle Karl"

title: "AutoK3s 介绍"
url:  "posts/docker/docs/k3s/autok3s"  # 永久链接
tags: [ "Docker", "README" ]  # 标签
categories: [ "Docker 学习笔记" ]  # 分类

toc: true  # 目录
draft: true  # 草稿
---

> https://docs.rancher.cn/docs/k3s/autok3s/_index

AutoK3s 是用于简化 K3s 集群管理的轻量级工具。

## 关键特性

- 通过 API、CLI 和 UI 等方式快速创建 K3s。
- 云提供商集成（简化 CCM 设置）。
- 灵活安装选项，例如 K3s 集群 HA 和数据存储（内置 etcd、RDS、SQLite 等）。
- 低成本（尝试云中的竞价实例）。
- 通过 UI 简化操作。
- 多云之间弹性迁移，借助诸如 backup-restore-operator 这样的工具进行弹性迁移。

## 快速体验

### Docker

您可以通过以下 Docker 命令，一键启动 AutoK3s 本地 UI，快速体验相关功能。

```shell
docker run --rm -p 8080:8080 cnrancher/autok3s
```

```shell
docker run -d --restart always -v /root/.ssh:/root/.ssh -p 8080:8080 cnrancher/autok3s
```

如果您想要在 docker 中使用 K3d provider，那么您需要使用宿主机网络启动 AutoK3s 镜像。

```shell
docker run --rm --net host -v /var/run/docker.sock:/var/run/docker.sock cnrancher/autok3s
```

> 正式部署不要用 Docker。

### 原生安装

```shell
curl -sS https://rancher-mirror.rancher.cn/autok3s/install.sh  | INSTALL_AUTOK3S_MIRROR=cn sh
```

您可以通过以下 CLI 命令启动本地 UI。

```shell
autok3s serve
```

```shell
autok3s serve --bind-address 0.0.0.0 --bind-port 18480
```

## 卸载

```shell
/usr/local/bin/autok3s-uninstall.sh
```

## 注意点

不能即是主节点，又是从节点。否则报错：

```
master node internal ip address can not be empty
```

## 查询节点

```shell
autok3s kubectl get nodes
```

```shell

```
