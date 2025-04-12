---
title: "《Kubernetes编程》记录"
date: 2025-04-09
aliases: ["/Daily Dev"]
tags: ["Kubernetes", "Golang"]
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
searchHidden: false
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowRssButtonInSectionTermList: true
UseHugoToc: true
---

# 《Kubernetes编程》记录

## 概论

### 控制器和 Operator

控制器实现了一个控制循环，通过 API 服务器监测集群中的共享状态，进行必要的变更。Operator 也是一种控制器，但是包含了一些运维逻辑，如应用程序的生命周期管理。

控制循环的基本流程如下：

1. 读取资源的状态，通常采用事件驱动模式，使用 Watch 来实现。
2. 改变集群中的对象状态或集群外部系统的状态。
3. 通过 API 服务器更新第 1 步中所提到的资源的状态，状态放在 etcd 中。
4. 循环执行这些逻辑。

控制器通常会使用以下的数据结构：

- Informer ：Informer 负责观察资源的目标状态，它还负责实现重新同步机制从而保证定时地解决冲突。
- 工作队列：事件处理器把状态变化情况放入工作队列。

多个相互独立的控制循环通过 API 服务器上的对象状态变化相互通信，对象状态的变化由 Informer 以事件的方式通知到控制器。这些通过 Watch 机制发送的事件对用户来说几乎不可见，在 API 服务器的审计机制中也看不到这些事件，只能看到对象更新的动作。控制器在处理事件时，以输出日志的方式记录它做的动作。

> Watch 事件：在 API 服务器与控制器之间通过 HTTP 流的方式发送，用于驱动 Informer 。
>
> Event 对象：是一种用户可见的日志机制，如 kubelet 通过 Event 来汇报 Pod 的生存周期事件。

### 乐观并发

Kubernetes 采用乐观并发机制，假设多用户操作同一资源的冲突概率较低，允许并发修改，仅在提交时检测冲突。

每个 Kubernetes 资源对象（如 Pod、Deployment）的元数据中都有一个 `resourceVersion` 字段，用于标识资源的当前版本号（由 etcd 生成）。这是乐观并发的核心：

- 读取资源时：客户端获取对象的 `resourceVersion`（例如 `123`）。
- 更新资源时：客户端必须在请求中携带该版本号（通过 `metadata.resourceVersion` 字段）。
- 服务器端校验：API Server 比较请求中的 `resourceVersion` 与当前存储中的版本：
  - 匹配：允许更新，`resourceVersion` 递增。
  - 不匹配：返回 `409 Conflict` 错误，拒绝更新（说明资源已被其他客户端修改过）。

当发生冲突时，客户端需重新获取资源的最新状态，然后基于最新数据重新提交修改。



## Kubernetes API 基础

### API 服务器

主控节点上的控制平面由 API 服务器、控制器管理器和调度器组成，API服务器是完成中心管理的实体，也是唯一直接与 etcd 交互的组件。

API 服务器提供了 RESTful API ，提供 Json 或 Protobuf 格式的数据内容。

在 API 服务器的上下文中，会使用到以下术语：

- 类别（Kind）：一个实体的类型。如一个 Pod 有三种类型，Object 用于系统中的持久化的实体对象、List 是多种实体的列表集、其他特殊类别用于某些特定动作或非持久化实体。
- API 组：一组逻辑上相关的 Kind 。

大部分 API 对象都区分资源的期望状态和当前状态，规格（Specification, Spec）是对于某种资源的期望状态的完整描述，被持久化到 etcd 中。



## 使用自定义资源

### 核心概念

自定义资源（Custom Resource Definition，CRD）是 Kubernetes 的扩展机制，允许用户定义自己的资源类型（如`MyApp`、`Database`等），这些资源会像原生资源（如Pod、Deployment）一样通过`kubectl`或API操作。

一旦创建 CRD，Kubernetes API 会为其生成对应的 RESTful 端点，并享受与内置资源相同的特性。

CRD 的关键内容有以下：

- Group/Version：定义 API 分组和版本。
- Kind：资源类型名称。
- Schema：描述资源的数据结构。

### CRD 应用

CRD 可以用来封装复杂应用，例如定义`RedisCluster`资源，用户只需声明副本数和版本，控制器自动处理配置生成、节点发现等细节。

### CRD 的高级特性

- 支持 API 版本演进。
- 子资源：
  - `/status`：分离用户输入（`spec`）和系统状态（`status`）。
  - `/scale`：支持HPA自动扩缩容。
- 验证：支持OpenAPI v3 Schema校验，例如限制字段格式、必填项等。
- 默认值：自动填充未指定的字段值，简化用户配置。



## 编写 Operator 的方案

### client-go 的两种使用场景

`client-go` 是 Kubernetes 官方提供的基础编程库，封装了 API Server 通信、资源操作等底层能力。Operator 是基于 `client-go` 构建的高阶设计模式，它利用`client-go` 的 `Informer`、 `Workqueue` 等机制实现自动化控制循环。

如果 API 只涉及到简单的一次性操作、仅需原生 API 、无状态工具那只需要简单实用 `client-go` 即可，无需过度封装。

但是如果有以下需求：

- 需要管理有状态应用。
- 需要自定义领域模型。
- 需要事件驱动自动化。

那就是 Operator 的使用场景了。

### Operator 框架

- Kubebuilder：Kubernetes 官方维护的 Operator 开发框架，集成 controller-runtime 库，提供完整的脚手架工具链。适合纯 Go 开发。
- Operator SDK：Red Hat 主导的 Operator 框架（底层基于 Kubebuilder），内置与 Operator Lifecycle Manager(OLM) 的集成。适合快速支持非 Go 开发。

两者之间的差距很小，一般情况下用 Kubebuilder 更多些。
