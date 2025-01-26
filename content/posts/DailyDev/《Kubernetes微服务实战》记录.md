---
title: "《Kubernetes微服务实战》记录"
date: 2025-01-25
aliases: ["/Daily Dev"]
tags: ["微服务", "Kubernetes"]
categories: ["Daily Dev"]
author: "Kurong"
showToc: true
TocOpen: true
draft: false
hidemeta: false
comments: false
description: ""
disableHLJS: false # to disable highlightjs
disableShare: true
disableHLJS: false
hideSummary: false
searchHidden: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowRssButtonInSectionTermList: true
UseHugoToc: true
---

# 《Kubernetes微服务实战》记录

## 面向开发人员的 Kubernetes 简介

### Kubernetes 架构

每个集群都有一个控制平面和数据平面，数据平面由多个节点组成，控制平面将在这些节点上部署并运行 Pod（容器组），监控变更并做出响应。

控制平面包含以下组件：

- API 服务器。
- etcd 存储。
- 调度器：kube 调度器负责将 Pod 调度到工作节点。
- 控制器管理器：kube 控制器管理器是包含多个控制器的单个进程，这些控制器监控着集群事件和对集群的更改做出响应。
  - 节点控制器：负责在节点出现故障时进行通知和响应。
  - 副本控制器：确保每个副本集（replica set）或副本控制器对象中有正确数量的 Pod 。
  - 端点控制器：为每个服务分配一个列出该服务的 Pod 的端点对象。
  - 服务账户和令牌控制器：使用默认服务账户和响应的 API 访问令牌对新的命名空间进行初始化。

数据平面是集群中将容器化工作负载转为 Pod 运行的节点的集合。

K8s 会在每个节点上安装一些组件以进行 Pod 的通信、监控和调度，包括：

- kublet ：是 K8s 的代理，负责与 API 服务器进行通信，并运行和管理在节点上的 Pod 。
- kube proxy ：负责节点的网络连接，充当服务的本地前端，并且可以转发 TCP 和 UDP 数据包。它通过 DNS 或环境变量来发现服务的 IP 地址。
- 容器运行时：Docker、gRPC 。
- kubectl ：命令行工具 CLI 。

### 微服务的完美搭档

ReplicaSet 是具有一定数量副本的 Pod 集合，创建它时 K8s 会确保始终有指定数量的 Pod 在集群中运行。以下是一个 K8s 部署清单：

```yaml
apiVersion: apps/v1  # 用来标记 K8s 的资源版本
kind: Deployment  # 指定处理的资源或 API 对象
metadata: 
  name: nginx  # 用来引用此特定资源
  labels:  # 标签允许 K8s 对共享相同标签的一组资源进行操作
    app: nginx
spec:  # Deployment 的规范
  replicas: 3  # 副本数
  selector:  # 选择器，管理具有与 matchLabels 匹配的标签的 Pod
    matchLabels: 
      app: nginx
  template:  # Pod 模板
    metadata:
      labels:
        app: nginx
    spec:  # Pod 的规范
      containers:
      - name: nginx
        image: nginx:1.15.4
        ports:
        - containerPort: 80
```

K8s 提供了 Service 服务资源，用于微服务的公开和发现：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service  # Service 名称
  labels:
    app: nginx
spec:
  selector:  # 选择与 Deployment 中标签匹配的 Pod
    app: nginx
  ports:
  - protocol: TCP
    port: 80  # Service 的端口
    targetPort: 80  # 映射到 Pod 的 containerPort
  type: ClusterIP  # Service 类型，默认为 ClusterIP
```

关于微服务安全，K8s 有以下措施：

- 通过命名空间将集群的不同部分相互隔离。
- 服务账户和权限。
- 密钥管理。
- 利用客户端证书对任何外部通信进行双重认证，如 HTTPS。
- 通过网络策略，定义和调整集群中的网络流量和访问。

使用 K8s 扩展微服务有两种方式，这两个方式都可以通过更新部署的副本数显式扩展微服务：

- 扩展 Pod 的数量。
- 扩展集群的总容量。

同时，K8s 可以对 Pod、资源等进行监控，日志也实现了集中管理。
