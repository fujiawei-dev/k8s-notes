---
date: 2021-03-29T08:56:56+08:00  # 创建日期
author: "Rustle Karl"

title: "Kubernetes 简介"
url:  "posts/k8s/quickstart/README"  # 永久链接
tags: [ "K8S", "README" ]  # 标签
categories: [ "K8S 学习笔记" ]  # 分类

toc: true  # 目录
draft: true  # 草稿
---

## 什么是 Kubernetes

Kubernetes 这个单词来自于希腊语，含义是 舵手 或 领航员。

K8s 是生产环境级别的容器编排，编排是什么意思？

1. 按照一定的目的依次排列；
2. 调配、安排；

Kubernetes，也称为 K8S，其中 8 是代表中间“ubernete”的 8 个字符，是 Google 在 2014 年开源的一个容器编排引擎，用于自动化容器化应用程序的部署、规划、扩展和管理，它将组成应用程序的容器分组为逻辑单元，以便于管理和发现，用于管理云平台中多个主机上的容器化的应用，Kubernetes 的目标是让部署容器化的应用简单并且高效，很多细节都不需要运维人员去进行复杂的手工配置和处理；

## Kubernetes 整体架构

![image](https://i.loli.net/2021/03/29/WcqXn7KPbNL2klz.png)

### Master

k8s集群控制节点，对集群进行调度管理，接受集群外用户去集群操作请求； 

Master Node 由 API Server、Scheduler、ClusterState Store（ETCD 数据库）和 Controller MangerServer 所组成；

### Nodes

集群工作节点，运行用户业务应用容器；

Nodes 节点也叫 Worker Node，包含 kubelet、kube proxy 和 Pod（Container Runtime）；
