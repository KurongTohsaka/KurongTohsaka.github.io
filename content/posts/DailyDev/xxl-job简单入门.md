---
title: "xxl-job简单入门"
date: 2025-09-13
aliases: ["/Daily Dev"]
tags: ["xxl-job", "Golang"]
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

在任何复杂的业务系统中，定时任务、延时任务、周期性报表生成等调度需求都不可或令。虽然 Linux 的 `cron` 和编程语言内置的定时器可以解决简单问题，但在微服务架构下，它们面临着单点故障、无法统一管理、不支持集群和分片等诸多挑战。

[XXL-JOB](https://www.xuxueli.com/xxl-job/) 是一个由大众点评开源的轻量级分布式任务调度平台，其核心设计目标是开发迅速、学习简单、轻量级、易扩展。凭借其清晰的架构、丰富的功能和极低的接入成本，XXL-JOB 已成为国内最流行的任务调度解决方案之一。

本文将带你全面解析 XXL-JOB，从核心概念、架构原理，到面试中的高频问题，并最终使用 Go 语言构建一个可以实际运行的执行器。

## 一、XXL-JOB 核心概念

理解 XXL-JOB 的工作模式，需要先掌握它的几个基本组件和术语。

#### 1. `调度中心` (Admin)

调度中心是 XXL-JOB 的“大脑”，是一个独立部署的 Web 应用。它负责：

- **任务管理**：提供可视化的 UI，用于创建、编辑、启动/停止任务。
- **调度触发**：内置调度器，根据任务配置的 `Cron` 表达式或固定延时，准时地向执行器发起调度请求。
- **注册管理**：维护在线的执行器列表，感知执行器的上下线。
- **日志监控**：汇集所有任务的执行日志，方便开发者查看任务执行情况和排查问题。

#### 2. `执行器` (Executor)

执行器是集成在业务应用中的一个组件（通常是一个 SDK 或库）。它负责：

- **自我注册**：启动后，执行器会周期性地向调度中心发送心跳，报告自己的地址，完成“服务注册”。
- **接收调度**：内置一个嵌入式的 HTTP 服务器，用于接收来自调度中心的任务触发请求。
- **执行任务**：根据请求中的参数，调用应用内具体的业务代码（JobHandler）。
- **上报结果**：将任务的执行结果（成功/失败）和执行日志回调给调度中心。

#### 3. `任务` (Job) 与 `JobHandler`

- **任务**：在调度中心 UI 上创建的一个调度单元，包含了 Cron 表达式、路由策略、负责人、超时时间等元数据。
- **JobHandler**：在执行器项目中，具体承载任务业务逻辑的代码块。在 Java 中通常是一个方法，在 Go 中则是一个注册的函数。一个任务必须绑定一个唯一的 JobHandler 名称。

#### 4. `路由策略` (Route Strategy)

当一个执行器以集群模式部署（即多个应用实例）时，路由策略决定了调度中心如何从集群中选择一个或多个实例来执行任务。常用策略包括：

- **第一个/最后一个**：选择集群中第一个或最后一个在线的实例。
- **轮询 (Round Robin)**：按顺序轮流选择。
- **一致性哈希 (Consistent HASH)**：根据任务 ID 计算哈希值，选择固定的一个实例，可用于实现粘性调度。
- **分片广播 (Sharding)**：**核心功能**。将任务“分片”给集群中的所有实例并行执行。例如，处理 100 万条数据，可以分给 10 个实例，每个实例处理 10 万条。调度中心会向每个实例传递分片参数（分片总数、当前分片索引）。

#### 5. `阻塞处理策略` (Block Strategy)

当一个任务的执行时间过长，超过了其调度周期时，此策略决定如何处理下一次的触发。

- **单机串行 (Serial)**：排队等待，上一个执行完，下一个才开始。
- **丢弃后续调度 (Discard Later)**：如果上一个还在运行，则默默丢弃本次调度。
- **覆盖之前调度 (Cover Early)**：立即终止上一个任务的执行，并启动本次调度。



## 二、XXL-JOB 核心架构与工作流程

XXL-JOB 的架构非常简洁，其核心是“去中心化”的设计思想。调度中心和执行器之间没有强依赖，通信协议为简单的 HTTP。

1. **执行器注册**：执行器应用启动，其内置的客户端会周期性地（如每 30 秒）向调度中心发起心跳请求，上报自己的 `AppName` 和地址（IP:Port）。调度中心据此维护一个在线执行器地址列表。
2. **调度线程触发**：调度中心内部有一个时间轮或类似的调度器，每秒扫描一次数据库，找出未来 5 秒内将要触发的所有任务。
3. **发起远程调用**：当一个任务到达触发时间点，调度中心会根据其配置的 **路由策略** 从注册列表中选择一个或多个执行器实例。然后，它向目标执行器发起一个 HTTP POST 请求（通常是 `/run` 接口）。
4. **执行器执行任务**：执行器接收到请求后，开启一个独立的线程（或 Goroutine）来执行绑定的 `JobHandler` 逻辑。这样做可以避免业务代码的阻塞影响到执行器与调度中心的通信。
5. **结果与日志回调**：`JobHandler` 执行过程中，产生的日志会通过一个异步队列，批量回调给调度中心的 `/callback` 接口。任务执行结束后，最终的成功或失败状态也会被回调。



## 三、XXL-JOB 面试高频考点

#### 1. XXL-JOB 的“中心化”体现在哪里？“去中心化”又体现在哪里？

- **中心化**：调度行为是中心化的。所有的任务触发都由调度中心统一发起，便于集中管理和监控。
- **去中心化**：任务的执行是去中心化的。任务逻辑（JobHandler）分散在各个业务系统的执行器中，随业务应用一同部署和伸缩。调度中心不关心任务的具体实现，只负责触发。

#### 2. XXL-JOB 是如何实现服务发现的？为什么不用 Zookeeper/Eureka？

XXL-JOB 采用了一种非常轻量级的 **客户端自动注册** 模式。执行器主动向调度中心上报心跳来完成注册。这种方式的优点是：

- **无额外依赖**：不需要引入 Zookeeper、Nacos、Eureka 等额外的服务注册组件，简化了整体架构和部署。
- **简单高效**：对于任务调度场景，这种心跳式的健康检查和地址管理已经足够满足需求。

#### 3. XXL-JOB 如何实现高可用（HA）？

- **调度中心 HA**：调度中心本身是无状态的（所有状态信息都存储在数据库中）。因此，你可以部署多个调度中心实例，并通过 NGINX 等负载均衡器进行代理。为了防止多个调度器实例重复触发同一个任务，XXL-JOB 在调度时使用了 **数据库行锁**（`SELECT ... FOR UPDATE`），确保在同一时刻只有一个调度器实例能成功锁定并触发任务。
- **执行器 HA**：执行器天生就是集群模式。你只需要部署多个业务应用实例，它们会自动注册到调度中心。调度中心会通过路由策略（如轮询）将任务分发到健康的实例上。如果某个实例宕机，心跳会中断，调度中心会将其从可用列表中移除，实现了自动的故障转移。

#### 4. 请解释一下分片广播（Sharding）的应用场景。

分片广播是处理海量数据的核心利器。典型场景包括：

- **数据同步**：将一个大表的数据同步到另一个系统，可以按 ID 范围或取模进行分片，每个执行器实例负责一个数据分片。

- **文件处理**：扫描一个包含海量文件的目录，每个执行器负责处理一部分文件。

- 消息队列消费：如果某个 Topic 的消息积压严重，可以启动一个分片任务，让多个执行器实例并行地从 MQ 中拉取和处理消息。

  在 JobHandler 中，执行器可以获取到两个关键参数：ShardingTotal（分片总数）和 ShardingIndex（当前分片序号，从 0 开始）。业务代码可以根据这两个参数来决定自己需要处理的数据范围。



## 四、Golang 实战：构建一个 XXL-JOB 执行器

官方虽然主推 Java，但其简单的 HTTP 协议使得其他语言接入非常容易。我们使用社区优秀的 Go 客户端库 `xxl-job-executor-go` 来创建一个执行器。

#### 准备工作

1. 部署并运行 XXL-JOB Admin。

2. 在 Admin -> 执行器管理中，新增一个执行器，`AppName` 记为 `xxl-job-executor-golang`，`名称` 自定义。

3. 在 Admin -> 任务管理中，新增一个任务，选择刚才创建的执行器，配置 Cron 表达式（如 `*/5 * * * * ?` 表示每 5 秒），并指定 `JobHandler` 名称。

4. 安装 Go client 库：

   ```Bash
   go get github.com/xxl-job/xxl-job-executor-go
   ```

#### 示例代码

创建一个 `main.go` 文件。

```go
package main

import (
	"context"
	"fmt"
	"github.com/xxl-job/xxl-job-executor-go"
	"log"
	"time"
)

func main() {
	exec := xxl.NewExecutor(
		xxl.ServerAddr("http://113.44.146.232:8080/xxl-job-admin"),
		xxl.AccessToken(""),
		xxl.ExecutorIp("127.0.0.1"),
		xxl.ExecutorPort("9999"),
		xxl.RegistryKey("xxl-job-executor-golang"),
	)
	exec.Init()

	exec.RegTask("task.simple", simpleTask)
	exec.RegTask("task.sharding", shardingTask)

	log.Println("XXL-JOB Go executor is starting...")

	err := exec.Run()
	if err != nil {
		log.Fatalf("Failed to start the executor: %v", err)
	}
}

func simpleTask(cxt context.Context, param *xxl.RunReq) (msg string) {
	log.Printf("Executing simple task with param: %s", param.ExecutorParams)
	log.Printf("Log ID: %d, Job ID: %d", param.LogID, param.JobID)

	time.Sleep(2 * time.Second)

	log.Println("Simple task executed successfully.")
	return "Simple task ran successfully"
}

func shardingTask(cxt context.Context, param *xxl.RunReq) (msg string) {
	log.Println("--- Executing Sharding Task ---")
	log.Printf("Sharding Index: %d", param.BroadcastIndex)
	log.Printf("Sharding Total: %d", param.BroadcastTotal)

	for i := int64(0); i < 5; i++ {
		if i%param.BroadcastTotal == param.BroadcastIndex {
			log.Printf("Instance %d is processing data slice %d", param.BroadcastIndex, i)
		}
	}

	log.Println("--- Sharding Task Finished ---")
	return fmt.Sprintf("Shard %d/%d finished.", param.BroadcastIndex, param.BroadcastTotal)
}
```

#### 如何运行 Demo

1. 确保你的 XXL-JOB Admin 正在运行，并且地址是 `http://localhost:8080/xxl-job-admin`。

2. 在 Admin 中，创建一个 `AppName` 为 `xxl-job-executor-golang` 的执行器。

3. 创建两个任务：

   - **任务一**：`JobHandler` 设为 `task.simple`，Cron 表达式 `*/10 * * * * ?` (每10秒)。
   - **任务二**：`JobHandler` 设为 `task.sharding`，Cron 表达式 `*/15 * * * * ?` (每15秒)，**路由策略** 选择 **分片广播**。

4. 运行 Go 程序：

   ```Bash
   go run main.go
   ```

5. 观察 Admin 界面，你会看到 `xxl-job-executor-golang` 执行器下有一个实例注册上来。

6. 等待任务触发，你可以在 Go 程序的控制台看到任务执行的日志，同时也可以在 Admin 的“调度日志”页面查看执行记录和详细日志。对于分片任务，如果你启动多个 Go 程序实例（修改端口号为 9998, 9997 等），你会看到任务被同时分发给了所有实例。



## 总结

XXL-JOB 以其“简单、可靠”的特质，完美地解决了分布式环境下的任务调度痛点。它的架构清晰，不引入过多复杂组件，使得开发者可以快速上手并将其集成到现有系统中。无论是简单的定时提醒，还是复杂的大数据分片处理，XXL-JOB 都能提供稳定、高效的调度服务。通过本文的学习，你不仅掌握了其核心原理，更具备了在 Go 生态中使用它的实战能力，为你的系统构建强大的后台调度能力。
