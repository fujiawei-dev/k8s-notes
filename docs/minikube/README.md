---
date: 2022-07-09T23:18:09+08:00
author: "Rustle Karl"

title: "minikube 基础教程"
url:  "posts/k8s/docs/minikube/README"  # 永久链接
tags: [ "K8S", "README" ]  # 标签
categories: [ "K8S 学习笔记" ]  # 分类

toc: true  # 目录
draft: true  # 草稿
---

https://minikube.sigs.k8s.io/docs/start/

## 预备

Ubuntu 官方提供了镜像，可直接用：

```bash
multipass launch -n minikube -c 2 -m 8G -d 80G minikube
```

```bash
multipass shell minikube
```

```bash
multipass exec minikube -- sudo bash
```

我不理解了，根本没有预装相关软件。。。

### 安装 Docker

## 安装

### Windows

```shell
choco install minikube
```

### ARM64

```shell
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube_latest_arm64.deb
sudo dpkg -i minikube_latest_arm64.deb
```

### AMD64

```shell
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube_latest_amd64.deb
```

```shell
sudo dpkg -i minikube_latest_amd64.deb
```

## 启动

```shell
minikube start --force --subnet=192.168.10.0/24 --nodes=4

minikube start --force --nodes=4
```

```shell
minikube stop
minikube delete
```

## 交互

```shell
minikube kubectl -- get po -A
```

```shell
minikube dashboard
```

```bash
kubectl proxy --address='0.0.0.0' --accept-hosts='^localhost$,^127\.0\.0\.1$,^\[::1\]$,^192.168.0.\d+'
```

## 部署应用

```shell
kubectl create deployment hello-minikube --image=k8s.gcr.io/echoserver:1.4
```

```shell
kubectl expose deployment hello-minikube --type=NodePort --port=8080
```

```shell
kubectl get services hello-minikube
```

```shell
minikube service hello-minikube
```

```shell
kubectl port-forward service/hello-minikube 7080:8080
```

功能残缺。

```shell

```

```shell

```
