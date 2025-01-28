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

### 部分核心概念

- Node：Node 是 Kubernetes 集群中的一个工作单元，表示集群中的一台物理机或虚拟机。Node 的作用是为运行容器化应用提供计算资源，一个 Node 可以运行多个 Pod。Node 分为：
  - Master Node：负责集群的控制平面，管理所有工作节点。
  - Worker Node：运行实际的应用程序工作负载，实际上就是 Pod。
- Pod：Pod 是 Kubernetes 中最小的可部署单位，表示一个或多个容器的集合。有以下特性：
  - Pod 内的容器共享一个 IP 地址，可以通过 `localhost` 相互通信。
  - Pod 内的容器可以共享挂载的存储卷（Volume）。
  - Pod 通常包含运行同一应用程序组件的容器，如主应用程序容器、日志采集容器等。
- Namespace：用于逻辑上隔离资源。可以在同一集群中运行多个项目或团队的工作负载。
- Service：用于将一组 Pod 的访问暴露给外部或集群内部的其他 Pod。
- Deployment：用于声明和管理 Pod 的期望状态，例如副本数、滚动更新等。
- Volume：提供持久化存储，允许 Pod 重启后数据仍然保留。

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



## 使用 Kubernetes 配置微服务

### 使用 Kubernetes ConfigMap

ConfigMap 是用于存储非机密配置信息的资源对象，它通常用来管理应用程序所需的配置数据。大概形式如下：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-config
data:
  config1: value1
  config2: value2
```

要想让 Pod 使用 ConfigMap 则有三种使用方式：

- 作为环境变量：将 ConfigMap 中的值注入到容器的环境变量中：

  ```yaml
  env:
  - name: CONFIG1
    valueFrom:
      configMapKeyRef:
        name: my-config
        key: config1
  ```

- 挂载为文件：

  ```yaml
  volumes:
  - name: config-volume
    configMap:
      name: my-config
  volumeMounts:
  - mountPath: /etc/config
    name: config-volume
  ```

- 作为命令行参数。

使用 ConfigMap 可以实现配置动态更新，而且使配置易于管理。

### 服务发现

K8s 内置了服务发现功能，无需做其他工作。每个服务都有一个端点资源，K8s 会使这个端点与运行该服务的容器地址保持同步。注意，每个 Pod 都有自己的 IP 地址，只有 API 服务器具有公共 IP 地址。每个服务都通过 DNS 和环境变量自动公开给集群中的其他服务：

- DNS：K8s 默认会使用 CoreDNS 提供内部的 DNS 服务，所有 Service 都会被注册到集群的 DNS 系统中，从而可以通过域名进行访问。DNS 是动态的，能够实时解析 Service 的 ClusterIP，因此即使 Service 的后端 Pod 实例发生变化（如扩缩容），DNS 解析仍然有效。
- 环境变量：当一个 Pod 被创建时，K8s 会将集群中每个 Service 的相关信息（例如 IP 和端口）注入到 Pod 的环境变量中。环境变量是静态的，仅在 Pod 创建时生成，因此如果 Service 的 IP 发生变化（例如重建），需要重启 Pod 才能更新环境变量。



## Kubernetes 与微服务安全

### 用户账户和服务账户

用户账户代表人类用户（管理员、开发者等），是外部用户与 K8s API 的交互身份。K8s 不直接创建或管理用户账户的生命周期，通常需要集成外部认证系统。

服务账户用于代表运行在集群中的应用程序或 Pod，是 Pod 与K8s API 的交互身份。默认情况下，每个命名空间都会有一个名为 `default` 的服务账户，Pod 没有指定服务账户时会自动绑定到这个默认账户。服务账户通过挂载到 Pod 的服务账户 Token 来认证自己。而且服务账户是命名空间作用域的，不能跨命名空间使用。

### 使用 Kubernetes 管理密钥

K8s 中的密钥通常有三种：

- 服务账户的 API Token 。
- 私有镜像仓库密钥。
- Opaque 密钥，这是 K8s 不知道的密钥，可以将敏感信息存储在这里。它提供了一个安全的密钥存储库，以及管理密钥的 API 。

在使用密钥时，可以将密钥打包到容器里、存储到环境变量、挂载为文件。



## API 与负载均衡器

### Kubernetes 的服务

K8s 的服务必须属于下面的某种类型：

- ClusterIP（默认值）：只能在集群内访问该服务，这是和微服务之间相互通信。
- NodePort：该服务通过所有节点上的专用端口对外开放。当请求通过专用 NodePort 进入节点时，kubelet 将负责将其转发到 Pod 所在节点。
- LoadBalancer：当 K8s 集群运行在提供负载均衡器支持的云平台上时，该类服务较常见。
- ExternalName：该类服务将对服务的请求解析为外部提供的 DNS 名称。

### 东西流量和南北流量

东西流量指服务/Pod/容器在集群内的相互通信。南北流量指对外公开服务的通信。

### ingress 和负载均衡器

关于 ingress ：

- ingress 是 K8s 中的一种资源对象，用于管理外部 HTTP/HTTPS 流量如何访问集群内部的服务。
- ingress 可以定义域名的路由规则、支持 TLS/HTTPS 加密、提供统一入口来管理多个服务。
- ingress 由 ingress 控制器和 ingress 资源组成。前者处理 ingress 规则的实现，后者为用户定义的规则和配置。

关于负载均衡器：

- 负载均衡器通常指将流量分发给多个 Pod 的机制，分为内部负载均衡和外部负载均衡。
- 内部负载均衡：
  - K8s 的 Service 对象自带负载均衡功能。
  - Service 的 `ClusterIP` 会在集群内部均衡流量到不同的 Pod。
- 外部负载均衡：
  - 在云环境中，Service 的类型设置为 `LoadBalancer` 时，会创建一个云提供商的负载均衡器（如 AWS ELB、GCP Load Balancer）。
  - 通过负载均衡器将外部流量引入到 K8s  服务中。

### 通过消息队列发送和接收事件

NATS 是一款高性能、轻量级的开源消息系统，广泛用于分布式系统和云原生应用中。这是一个 CNCF 项目，由 Go 实现。它采用发布-订阅（Pub/Sub）、请求-响应（Request-Reply）以及队列（Queue）通信模式，设计目标是简单、快速和可靠。



## 有状态服务

### 抽象存储

### 使用StatefulSet 在 Kubernetes 集群内存储数据

### 在 Kubernetes 中使用关系型和非关系型数据库





## 在 Kubernetes 上运行 Serverless 任务

### 云中的 Serverless

### Delinkcious 链接检查

### 其他 Kubernetes Serverless 框架





## 微服务的部署



## 监控、日志和指标



## 服务网格与 Istio



## 微服务和 Kubernetes 的未来

