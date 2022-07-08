---
date: 2022-07-08T19:33:55+08:00
author: "Rustle Karl"

title: "容器运行时错误"
url:  "posts/k8s/docs/question/containerd"  # 永久链接
tags: [ "K8S", "README" ]  # 标签
categories: [ "K8S 学习笔记" ]  # 分类

toc: true  # 目录
draft: true  # 草稿
---

## container runtime is not running

### 查看 Docker 进程

```shell
ps aux | grep docker | grep -v grep
```

```
root        2720  0.1  1.9 1309596 78104 ?       Ssl  19:31   0:00 /usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock
```

### 查看 Containerd 配置

```shell
cat /etc/containerd/config.toml
```

```shell
disabled_plugins = ["cri"]
```

containerd 进程禁用了 cri 模块插件。

### 解决方案


```shell
rm -fr /etc/containerd/config.toml
```

```shell
systemctl restart containerd
```

```shell
systemctl status containerd.service
```

```shell
cat /usr/lib/systemd/system/containerd.service
```

## 二级

### 三级

```shell

```

```shell

```
