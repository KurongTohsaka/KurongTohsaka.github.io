---
title: "apollo简单入门"
date: 2025-09-14
aliases: ["/Daily Dev"]
tags: ["Apollo", "Golang"]
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

在现代微服务架构中，服务数量的激增使得传统的配置文件管理方式（如本地文件、Git 仓库）变得越来越难以维护。配置的修改、发布、回滚、权限控制以及实时生效等需求，催生了分布式配置中心这一关键中间件。

[Apollo (阿波罗)](https://github.com/apolloconfig/apollo) 是由携程框架部门研发的开源配置管理中心，它功能强大、生态丰富，能够集中化管理应用在不同环境、不同集群的配置。本文将带你深入了解 Apollo 的世界，从核心概念到架构原理，再到 Go 语言的实战集成。

## 一、Apollo 核心概念

要用好 Apollo，首先需要理解它的几个核心概念，这些概念构成了其灵活、强大的配置管理模型。

#### 1. `Application` (应用)

这是配置管理的基本单元，代表一个独立的应用或服务。每个应用都有一个全局唯一的 `AppId`，这是客户端与服务端交互时的身份标识。

#### 2. `Environment` (环境)

用于隔离不同部署环境的配置，如 `DEV` (开发环境)、`FAT` (测试环境)、`UAT` (预发布环境)、`PRO` (生产环境)。Apollo 的权限管理也是基于环境的，可以精细控制不同环境的配置修改和发布权限。

#### 3. `Cluster` (集群)

一个应用在同一个环境下，可以部署在不同的集群中。这个概念主要用于支持多数据中心（IDC）或者隔离部署。例如，你可以为上海机房和北京机房创建两个不同的 Cluster，为它们配置不同的数据库地址。默认情况下，会有一个 `default` 集群。

#### 4. `Namespace` (命名空间)

这是配置项的集合，是配置隔离的最小单元。通过 Namespace，可以把不同类型的配置分门别类地管理。

- **私有 Namespace**：其作用域仅限于当前应用。每个应用创建时都会有一个默认的 `application` 命名空间。
- **公共 Namespace**：可以被多个应用关联和共享。常用于存储一些公共的配置，如数据库连接信息、中间件地址等。
- **关联 Namespace (继承)**：应用可以关联（继承）公共 Namespace 的配置，实现配置的复用。如果私有配置与公共配置有相同的 Key，私有配置会覆盖公共配置。
- **格式支持**：Namespace 支持多种格式，如 `properties` (默认)、`XML`、`JSON`、`YAML`、`TXT`。

#### 5. `Release` (发布)

当一个 Namespace 的配置修改完成后，需要通过“发布”操作才能被客户端感知到。每一次发布都会生成一个唯一的版本号，且发布后的配置是 **不可修改的**。如果需要变更，只能创建一个新的发布。这个机制保证了配置的可追溯和快速回滚。

#### 6. `Gray Release` (灰度发布)

这是 Apollo 的一大亮点。它允许你将新发布的配置先生效于指定的 **部分** 客户端实例上。你可以根据客户端的 IP 地址或预设的标签来圈定灰度范围。待验证无误后，再进行全量发布。这极大地提高了配置发布的安全性。



## 二、Apollo 核心架构与工作流程

Apollo 的高可用和实时推送能力得益于其精巧的架构设计。

#### 主要组件

- **Portal**: 用户界面，提供配置的可视化管理、发布、审计等功能。
- **Admin Service**: Portal 的后端服务，负责配置的创建、修改、发布等操作，直接与数据库交互。
- **Config Service**: 客户端的配置拉取服务。它提供了高性能的接口供客户端获取配置。为了高可用，Config Service 可以水平扩展部署多台。
- **Client**: 集成在应用中的 SDK。它负责从 Config Service 拉取配置、在本地缓存配置，并监听配置变化。
- **Eureka**: 一个服务注册与发现组件。Admin Service 和 Config Service 都会注册到 Eureka 中，使得 Portal 和 Client 能够动态发现可用的服务实例。
- **Database**: Apollo 的核心数据存储，包含 `PortalDB` 和 `ConfigDB` 两个库，分别存储用户、权限、审计信息和核心的配置信息。

#### 工作流程：实时更新（核心）

Apollo 如何做到配置修改后，客户端能秒级感知的？答案是 **长轮询（Long Polling）**。

1. **客户端发起请求**：Client 启动后，会向 Config Service 发起一个 HTTP 请求，询问自己关心的 Namespace 是否有更新。
2. **服务端挂起请求**：Config Service 收到请求后，如果发现配置没有变化，它 **不会** 立即返回，而是将这个请求挂起（hold 住），最长可达 60 秒。
3. **用户发布配置**：用户在 Portal 上修改并发布了配置。
4. **服务端通知**：Config Service 有一个独立的线程每秒会扫描一次数据库，发现配置有更新。它会立即通知（release）之前被挂起的那些请求，并告诉它们哪个 Namespace 更新了。
5. **客户端拉取新配置**：Client 收到响应后，得知配置已更新，会立刻发起另一个请求去拉取最新的配置内容。
6. **循环往复**：拉取完新配置后，Client 再次发起一个新的长轮询请求，回到第 1 步，如此循环，保证了实时性。

这种方式相比于传统的定时轮询，既保证了实时性，又大大减少了无效的轮询请求，降低了对客户端和服务端的压力。



## 三、Apollo 面试高频考点

#### 1. Apollo 的实时推送原理是什么？

这道题直接命中核心。回答要点就是 **长轮询（Long Polling）** 机制。可以详细描述上面提到的工作流程，并强调它与普通轮询（客户端定时拉取）和 WebSocket（全双工通信）的区别与优势。

#### 2. 如果 Apollo Server 全部宕机，客户端会怎么样？

Apollo 客户端有完善的 **容灾设计**。

- **本地缓存**：客户端在成功拉取配置后，会在本地文件系统（例如 `/opt/data/{appId}/config-cache`）中缓存一份配置。
- **启动保障**：如果应用启动时无法连接到 Apollo Server，它会读取本地缓存的配置来启动，保证了即使配置中心挂了，应用也能正常启动和运行。
- **运行时影响**：正在运行的应用将继续使用内存中最后一次成功的配置，不会受到影响。只是无法再获取到配置的更新。

#### 3. Apollo 如何保证高可用？

- **服务无状态与水平扩展**：`Config Service` 和 `Admin Service` 都是无状态的，可以部署多个实例组成集群，并通过 Eureka 进行服务发现和负载均衡。单个实例宕机不影响整体服务。
- **多数据中心部署**：通过 `Cluster` 概念，可以将配置服务部署在不同的数据中心，应用就近访问，实现异地容灾。
- **客户端容灾**：如上一点所述，客户端的本地缓存机制是最后一道防线。

#### 4. Apollo 与 Nacos、Spring Cloud Config 有什么区别？

- **Apollo**: 功能最全面、最完善。拥有强大的可视化界面、精细的权限控制、完整的发布审核、回滚、灰度发布流程。生态支持多语言，架构成熟稳定，适用于对配置管理有高要求的复杂场景。
- **Nacos**: "多面手"。除了配置管理，它还集成了服务发现和动态 DNS 功能。配置管理功能比 Apollo 稍弱（例如灰度发布、审计等流程相对简单），但其多合一的特性在阿里巴巴生态和 Spring Cloud Alibaba 中非常受欢迎，部署和使用更简单。
- **Spring Cloud Config**: 设计简单，通常与 Git/SVN/Vault 等后端存储结合。配置的修改和版本控制依赖于后端的存储系统。它没有可视化的管理界面，配置更新通常需要手动触发（例如通过 `/actuator/refresh` 端点），实时性较差。



## 四、Golang 实战：集成 Apollo 客户端

我们将使用社区维护的 Go 语言客户端 `agollo` 来演示如何集成。

#### 准备工作

1. 确保你有一个正在运行的 Apollo 环境。

2. 在 Apollo Portal 上创建一个新的应用，获取其 `AppId`。

3. 在该应用默认的 `application` Namespace 下创建一些配置，例如 `timeout=300`，并 **发布** 它们。

4. 安装 `agollo` 库：

   ```bash
   go get github.com/apolloconfig/agollo/v4
   ```

#### 示例代码

创建一个 `main.go` 文件。

```go
package main

import (
	"fmt"
	"github.com/apolloconfig/agollo/v4"
	"github.com/apolloconfig/agollo/v4/env/config"
	"github.com/apolloconfig/agollo/v4/storage"
	"log"
	"time"
)

type CustomChangeListener struct{}

func (c *CustomChangeListener) OnChange(changeEvent *storage.ChangeEvent) {
	fmt.Println("--- Configuration Change Detected ---")
	fmt.Printf("Namespace: %s\n", changeEvent.Namespace)
	for key, change := range changeEvent.Changes {
		fmt.Printf("Key: %s, ChangeType: %s, OldValue: %s, NewValue: %s\n",
			key, change.ChangeType, change.OldValue, change.NewValue)
	}
	fmt.Println("--- End of Change ---")
}

func (c *CustomChangeListener) OnNewestChange(event *storage.FullChangeEvent) {
	log.Printf("Newest configurations for namespace %s: %v\n", event.Namespace, event.Changes)
}

func main() {
	clientConfig := &config.AppConfig{
		AppID:          "1145141919810",
		Cluster:        "default",
		IP:             "http://113.44.146.232:8080/",
		NamespaceName:  "application",
		IsBackupConfig: true,
		Secret:         "",
	}

	client, err := agollo.StartWithConfig(func() (*config.AppConfig, error) {
		return clientConfig, nil
	})
	if err != nil {
		log.Fatalf("Failed to start Apollo client: %v", err)
	}
	log.Println("Apollo client started successfully.")

	listener := &CustomChangeListener{}
	client.AddChangeListener(listener)
	log.Println("Registered a change listener.")

	const namespace = "application"
	const defaultTimeout = "100"

	timeout := client.GetConfig(namespace).GetStringValue("timeout", defaultTimeout)

	fmt.Printf("Initial value for 'timeout': %s\n", timeout)
	fmt.Println("Application is running... Try changing the 'timeout' value in the Apollo portal.")
	fmt.Println("Press Ctrl+C to exit.")
	for {
		time.Sleep(5 * time.Second)
		currentTimeout := client.GetConfig(namespace).GetStringValue("timeout", defaultTimeout)
		fmt.Printf("Heartbeat: Current 'timeout' is %s\n", currentTimeout)
	}
}
```

#### 如何运行 Demo

1. 将代码中的 `YOUR_APP_ID` 替换成你在 Apollo Portal 中创建的 AppId。

2. 将 `IP` 字段的值 `http://localhost:8080` 替换成你的 Apollo Meta Server (即 Config Service) 的地址。

3. 运行程序：

   ```bash
   go run main.go
   ```

4. 程序启动后，会打印出初始的 `timeout` 值。

5. 现在，去 Apollo Portal 界面，修改 `timeout` 的值并 **发布**。

6. 你会立刻在运行程序的终端上看到 `CustomChangeListener` 打印出的配置变更通知。



## 总结

Apollo 凭借其成熟的架构、强大的功能和对多语言的良好支持，已经成为构建大规模分布式系统中不可或缺的一环。它不仅仅是一个配置存储，更是一套完整的配置管理、发布和治理解决方案。通过本文的学习，相信你已经对 Apollo 的核心思想有了深入的理解，并能熟练地将其应用在你的 Go 语言项目中，从而极大地提升系统的灵活性和可维护性。
