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

K8s 的存储模型包含以下几个概念：

- 卷：是 K8s 中最基本的存储单元，用于为 Pod 提供持久化存储。
- 持久卷：是集群级别的存储资源，由管理员进行配置。
- 持久卷声明：是用户对存储资源的请求，类似于 Pod 对计算资源的请求。持久卷声明与持久卷绑定，Pod 通过持久卷声明使用持久卷。

- 存储类：用于动态创建持久卷，允许按需分配存储资源。每个存储类对应一种存储后端（如SSD、HDD），并定义了持久卷的属性（如卷类型、回收策略）。

- 动态供给：允许根据持久卷声明自动创建持久卷，无需管理员手动干预。
- CSI（容器存储接口）：是 K8s 与存储插件之间的标准接口，允许第三方存储提供商集成。

### 使用 StatefulSet 在 Kubernetes 集群内存储数据

StatefulSet 是K8s 中用于管理有状态应用的工作负载 API 对象。与 Deployment 不同，StatefulSet 为每个 Pod 提供唯一的标识和稳定的网络标识、持久化存储，确保 Pod 在重启、迁移或扩展时保持一致性。StatefulSet 适用于需要稳定网络标识、有序部署和持久化存储的应用，如数据库（MySQL、PostgreSQL）、分布式系统（Zookeeper、Kafka）等。

StatefulSet 有一些核心特性：

- 稳定的网络标识：每个 Pod 都有一个唯一的、稳定的网络标识（如 `pod-name-0`, `pod-name-1`），即使 Pod 被重新调度，其名称和网络标识也不会改变。
- 有序部署和扩展：Pod 的创建和扩展是按顺序进行的。
- 有序终止和缩容：Pod 的删除是按逆序进行的。
- 持久化存储：每个 Pod 可以绑定一个独立的持久卷，确保数据在 Pod 重启或迁移时不会丢失。
- 稳定存储：通过配置，StatefulSet 可以为每个 Pod 动态的创建持久卷声明。

StatefulSet 通常与 Headless Service 使用。Headless Service 不会分配 ClusterIP，而是直接返回 Pod 的 IP 和 DNS 记录。

下面是一个 StatefulSet 配置：

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx"  # 关联的 Headless Service 名称
  replicas: 3           # Pod 副本数
  selector:
    matchLabels:
      app: nginx        # 匹配 Pod 的标签
  template:
    metadata:
      labels:
        app: nginx      # Pod 的标签
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
        volumeMounts:
        - name: www      # 挂载的卷名称
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:  # 动态创建 PVC
  - metadata:
      name: www          # PVC 名称
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1Gi   # 存储大小
```

### 在 Kubernetes 中使用关系型和非关系型数据库

使用 StatefulSet 部署 MySQL：

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: "mysql"
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:5.7
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: "password"
        ports:
        - containerPort: 3306
        volumeMounts:
        - name: mysql-data
          mountPath: /var/lib/mysql
  volumeClaimTemplates:
  - metadata:
      name: mysql-data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 10Gi
```

使用 StatefulSet 部署 Redis：

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
spec:
  serviceName: "redis"  # 关联的 Headless Service 名称
  replicas: 3           # Redis 实例数量
  selector:
    matchLabels:
      app: redis         # 匹配 Pod 的标签
  template:
    metadata:
      labels:
        app: redis       # Pod 的标签
    spec:
      containers:
      - name: redis
        image: redis:6.2  # Redis 镜像
        ports:
        - containerPort: 6379  # Redis 默认端口
        volumeMounts:
        - name: redis-data     # 挂载的卷名称
          mountPath: /data     # Redis 数据目录
  volumeClaimTemplates:        # 动态创建 PVC
  - metadata:
      name: redis-data         # PVC 名称
    spec:
      accessModes: [ "ReadWriteOnce" ]  # 访问模式
      resources:
        requests:
          storage: 1Gi         # 存储大小
```

这是 Redis 的 Headless Service 配置：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis
  labels:
    app: redis
spec:
  clusterIP: None  # Headless Service
  ports:
  - port: 6379     # Redis 默认端口
    name: redis
  selector:
    app: redis     # 匹配 StatefulSet 的 Pod 标签
```



## 在 Kubernetes 上运行 Serverless 任务

### 云中的 Serverless

Serverless（无服务器）是一种云计算模式，在这种模式下，开发者不需要管理服务器或底层基础设施，而是专注于业务代码的编写。所有的基础资源（计算、存储、网络等）由云服务提供商自动管理和调度。

在 Serverless 模式中，应用被拆分为一个个独立的函数（即“函数即服务”，FaaS），每个函数只在特定事件触发时运行。云平台会根据请求的数量动态分配资源，并根据实际运行时间收费。例如，当一个 HTTP 请求触发一个函数时，该函数就会被加载、执行，并在完成后自动销毁，用户只需为这段实际运行的时间付费。

Serverless的优缺点：

- 优势：
  - 降低运维复杂度，开发者可专注于业务逻辑。
  - 经济高效，按需计费模式避免资源浪费。
  - 弹性扩展，能够快速应对流量变化。
- 局限：
  - 冷启动问题：长时间不调用的函数可能存在首次调用延迟。
  - 依赖云服务商：可能会受到厂商锁定（vendor lock-in）的影响。
  - 调试和监控较为复杂：由于执行环境由云平台管理，本地调试和日志追踪可能不如传统服务直观。

### 常见的 Serverless 框架

- Nuclio：一个知名的 Serverless 平台，支持高性能、低延迟的函数执行。
- Knative：由 Google 等公司主导开发，Knative 提供了构建、部署和管理 Serverless 应用的完整解决方案。与 Kubernetes 深度集成，支持蓝绿部署、滚动更新和无缝扩展，适用于构建大规模、云原生的 Serverless 应用。
- Kubeless：Kubeless 是一个原生 K8s 的 Serverless 框架，允许用户以 YAML 文件的形式直接部署函数。适合中小型应用或实验性项目，快速上手部署简单的函数服务。
- OpenFaaS：OpenFaaS（Functions as a Service）通过 Docker 容器化的方式来封装函数，提供了简洁的 UI 和 CLI 工具。适用于混合云和多云环境中快速构建和部署 Serverless 应用。
- Fission：Fission 是一个专为 K8s 设计的 Serverless 框架，以低延迟和高性能著称。适合对响应速度有较高要求的业务场景，能够高效处理大量短时间任务。



## 微服务的部署

### 部署对象

- Depolyment：Deployment 是最常用的部署对象，主要用于管理无状态（Stateless）的应用。它定义了 Pod 模板、期望副本数、更新策略等，并通过管理底层的 ReplicaSet 实现 Pod 的创建、扩缩容和滚动升级。
- StatefulSet：用于管理有状态（Stateful）的应用，如数据库或分布式存储系统。它提供了稳定的网络标识、持久化存储绑定以及有序的启动和终止顺序。
- DaemonSet：保证集群中每个（或指定一部分）节点上都运行一个 Pod。常用于部署日志收集、监控代理、安全扫描等守护进程。
- ReplicaSet：用于保证一组 Pod 始终保持在指定的副本数量。通常由 Deployment 控制，不需要直接创建，但在一些场景下也可以单独使用。

### 多环境部署

在实际开发和运维过程中，通常需要在不同的环境（如开发、测试、预发布、生产）中部署应用。多环境部署主要关注环境隔离、配置管理和自动化流程等问题。

- NameSpace 隔离：K8s 中的 Namespace 用于逻辑隔离资源。不同环境可以分配不同的 Namespace，如 `dev`、`staging`、`prod`。这不仅便于资源管理，还能设置不同的资源配额和权限控制策略。
- 配置管理：利用 ConfigMap 管理非敏感配置信息，利用 Secret 管理敏感信息。不同环境下的配置可以通过不同的 ConfigMap/Secret 来区分，再通过环境变量或挂载文件的方式传递给 Pod。或者部署时可利用 YAML 文件中 `env` 字段注入环境变量，或使用 `args`/`command` 自定义启动参数，实现同一镜像在不同环境下的不同行为。
- Helm：Helm 是 K8s 的包管理工具，通过 Chart 定义应用的各种资源。借助 Helm，可以使用同一个 Chart 针对不同环境传入不同的 values 参数，实现配置复用与环境定制化部署。
- CI/CD：通过 CI/CD 工具（如 Jenkins、GitLab CI、Argo CD 等）实现代码构建、镜像打包、自动测试以及自动部署。流水线中可设置不同的分支对应不同环境，并在每次提交后自动触发相应环境的部署更新。利用 CI/CD 流水线自动化部署的同时，可以记录每次部署的版本信息，结合 K8s 自身的回滚能力，实现快速恢复和版本切换。

### 高级部署策略

- 滚动更新：滚动更新是一种渐进式更新策略，通过逐步替换旧版 Pod 为新版 Pod 来实现应用升级，确保在整个过程中服务始终可用。
  - 无需中断服务，更新过程透明。
  - 可以根据流量情况和节点容量动态控制更新速率。
- 蓝绿部署：蓝绿部署将旧版本（蓝）与新版本（绿）完全隔离部署在不同的环境中，通过切换流量路由实现版本切换。通常利用外部负载均衡器（或 Ingress 控制器）进行流量切换。
  - 可快速回滚，降低风险。
  - 测试与验证环境独立，不影响现有业务。
- 金丝雀发布：金丝雀发布是指将新版本部署到少量 Pod 上，先让少部分流量进入新版本以验证稳定性，再逐步扩大新版本的流量比例，最终完全替换旧版本。
  - 降低新版本升级风险。
  - 在实际流量中验证新版本的性能和稳定性，及时发现潜在问题。
- A/B 测试和灰度发布：A/B 测试指在相同的生产环境中，同时部署两个或多个版本，通过不同用户组的反馈和数据对比，选择最优方案。类似于金丝雀发布，但灰度发布往往针对特定用户或特定条件进行版本切换，既可以逐步扩大流量，也可以根据用户特征精细控制版本分布。



## 监控、日志和指标

### 自愈机制

自愈是 K8s 集群中一个核心特性，指的是集群能够在发生故障或异常时，自动将实际状态恢复到期望状态，保证系统持续可用。主要体现在以下几个方面：

- 控制器自愈
  - Deployment、ReplicaSet 和 StatefulSet 控制器：
    - 声明式管理：通过定义期望状态，K8s 控制器会不断对比实际状态，如发现 Pod 崩溃、失联或数量不足时，自动重启或创建新的 Pod。
    - 滚动升级和回滚：在更新过程中，如果发现新版本出现问题，可以自动回滚到稳定版本，保证服务不中断。

- 健康检查（Liveness & Readiness Probes）

  - Liveness Probe：
    - 作用：检测应用是否处于健康状态，如发现故障则重启容器。
    - 实现：通过 HTTP 请求、TCP 检查或执行命令来判断容器状态。

  - Readiness Probe：
    - 作用：判断容器是否已经准备好接收流量，避免在启动或更新过程中将流量导入未就绪的容器。

- 自动重调度与节点自愈

  - 节点状态监控：K8s 的 Node Controller 会定期检测节点的健康状态，一旦发现节点失联或状态异常，会将该节点标记为不可用，并将其上运行的 Pod 调度到其他健康节点上。

  - Pod 重启策略：若容器因内部错误退出，Pod 会按照重启策略（如 Always、OnFailure、Never）自动进行重启，确保服务恢复。

### 集群自动伸缩

集群自动伸缩（Autoscaling）是提高资源利用率和降低成本的重要手段，主要分为 Pod 级别的自动伸缩和节点级别的自动伸缩。

- Pod 级别自动伸缩

  - Horizontal Pod Autoscaler (HPA)

    - 作用：根据应用实际的 CPU、内存或者自定义指标，对 Deployment、ReplicaSet、StatefulSet 中的 Pod 数量进行动态调整。

    - 工作原理：
      - HPA 定期查询指定指标数据（通常由 Prometheus Adapter 或 Metrics Server 提供），与设定的目标值比较。
      - 若指标超出目标范围，则按比例增加 Pod 副本；若指标低于目标范围，则减少 Pod 数量。

  - Vertical Pod Autoscaler (VPA)

    - 作用：根据 Pod 的资源使用情况，自动调整容器的 CPU 和内存请求与限制，帮助在固定 Pod 数量下优化资源分配。

    - 特点：
      - 适用于难以通过增加副本解决资源瓶颈的场景。
        - 配置和调整相对复杂，可能需要重启 Pod 生效。

- 节点级别自动伸缩

  - Cluster Autoscaler

    - 作用：针对托管在云平台上的 K8s 集群（如 AWS EKS、GKE、AKS 等），当集群内资源不足（无法调度新 Pod）时，自动增加节点；当节点长期处于低负载、空闲状态时，自动缩减节点数，降低成本。

    - 工作原理：
      - 持续监控调度器的排队情况和节点资源利用率。
      - 在 Pod 无法调度时，向云平台 API 请求增加节点；在节点空闲且 Pod 可迁移时，安全驱逐并缩减节点数。

### 监控

主要组件和工具：

- Prometheus：
  - 作用：开源监控系统和时序数据库，专门用于抓取、存储和查询各种指标数据。
  - 采集方式：通过 HTTP Pull 模式定时抓取目标暴露的指标（通常为 `/metrics` 接口）。
- Grafana：
  - 作用：数据可视化工具，可以与 Prometheus 等时序数据库配合，通过仪表板展示实时和历史指标数据。
  - 功能：支持自定义告警规则、报表制作和数据关联分析，帮助运维人员直观地监控集群运行情况。
- Alertmanager：
  - 作用：与 Prometheus 配合使用，实现告警信息的分组、抑制和路由。
  - 告警渠道：支持邮件、Slack、PagerDuty 等多种告警方式，便于及时通知相关人员。

### 日志

日志采集方式：

- 容器标准输出：K8s 中容器的日志一般输出到标准输出（stdout）和标准错误（stderr），由 kubelet 写入宿主机上的日志文件（通常在 `/var/log/containers`）。
- DaemonSet 日志采集器：在每个节点上部署日志采集代理（如 Fluentd、Filebeat 或 Logstash），以 DaemonSet 的方式运行，保证所有节点的日志能够被采集并转发到集中式日志存储系统中。

常见日志系统：

- ELK/EFK Stack：
  - Elasticsearch：日志存储和检索引擎，可以存储海量日志数据，并支持高效查询。
  - Logstash/Fluentd：日志采集和处理工具，负责解析、过滤并将日志数据传输到 Elasticsearch。
  - Kibana：数据可视化和分析工具，通过图形化界面帮助运维人员快速定位问题。
- Loki + Promtail + Grafana：
  - Loki：由 Grafana Labs 开发的日志聚合系统，专门设计用于与 Prometheus 数据模型兼容，并与 Grafana 紧密集成。
  - Promtail：日志采集器，类似于 Fluentd，但与 Loki 集成更紧密。

### 指标

- 自定义指标：利用 Prometheus Client 库（如 Prometheus Python Client、Java Client 等）在应用中嵌入指标采集功能，暴露 HTTP 接口供 Prometheus 抓取。
- Kubernetes 内置指标：如 cAdvisor 提供容器级别的资源使用情况，kube-state-metrics 提供集群状态信息。
- 展示与告警：利用 Grafana 等工具构建仪表板，并设置阈值告警，确保在关键指标异常时能迅速通知运维人员。



## 服务网格与 Istio



