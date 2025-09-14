---
title: "Kafka简单入门"
date: 2025-09-13
aliases: ["/Daily Dev"]
tags: ["Kafka", "Golang"]
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

Apache Kafka 是一个开源的分布式事件流平台，被数千家公司用于高性能数据管道、流分析、数据集成和关键任务应用。无论你是后端开发者、数据工程师，还是希望构建实时数据系统的架构师，掌握 Kafka 都是一项至关重要的技能。

本文将带你系统地学习 Kafka，从核心概念到命令行实战，再到面试高频考点，最后用 Go 语言构建一个简单的生产者和消费者应用。

## 一、Kafka 核心概念解析

要理解 Kafka，首先必须掌握其核心术语。我们可以把 Kafka 想象成一个高度有序、可分区、可复制的数字“邮局系统”。

#### 1. Broker

Kafka 集群中的一台服务器就是一个 Broker。你可以把它想象成邮局系统中的一个“邮局分拣中心”。多个 Broker 共同组成一个 Kafka 集群，协同工作，提供高可用和负载均衡。

#### 2. Topic（主题）

Topic 是消息的类别。所有发布到 Kafka 集群的消息都有一个指定的 Topic。这就像邮局里的不同“信件类型”，比如“平信”、“挂号信”或“国际邮件”。生产者将消息发送到特定的 Topic，消费者订阅特定的 Topic 来接收消息。

#### 3. Partition（分区）

每个 Topic 都可以被分成一个或多个 Partition。Partition 是 Kafka 实现高吞吐量和水平扩展的关键。你可以将 Partition 理解为某个信件类型（Topic）的专用“处理通道”。

- **有序性**：在单个 Partition 内，消息是严格有序的。消息被追加到 Partition 的末尾，并被分配一个唯一的、递增的 ID，称为 **Offset**。
- **并发性**：不同的 Partition 可以分布在不同的 Broker 上，这使得多个消费者可以并行地从一个 Topic 中读取数据，极大地提高了读取效率。

#### 4. Offset（偏移量）

Offset 是 Partition 中每条消息的唯一标识符，是一个单调递增的整数。消费者通过 Offset 来追踪自己已经消费到 Partition 中的哪个位置。这个机制非常强大，因为它允许消费者自由地控制消费进度，可以重复消费或跳过某些消息。

#### 5. Producer（生产者）

负责创建消息并将其发布（发送）到 Kafka Topic 的应用程序。生产者在发送消息时，可以指定 Topic，也可以指定消息的 Key。如果指定了 Key，Kafka 会使用哈希算法将这个 Key 映射到某个特定的 Partition，确保所有具有相同 Key 的消息都进入同一个 Partition，从而保证了这部分消息的顺序性。

#### 6. Consumer（消费者）

从 Kafka Topic 订阅并处理消息的应用程序。消费者以“拉（Pull）”模式从 Broker 中拉取消息。

#### 7. Consumer Group（消费者组）

多个消费者可以组成一个 Consumer Group 来共同消费一个 Topic。这是 Kafka 实现高可扩展消费的另一大法宝。

- 一个 Topic 的一个 Partition 在同一时间只能被同一个 Consumer Group 中的一个 Consumer 消费。
- 如果 Consumer 的数量多于 Partition 的数量，多余的 Consumer 将会闲置。
- 如果 Consumer 的数量少于 Partition 的数量，某些 Consumer 将会消费多个 Partition。
- 当组内成员发生变化（新增、崩溃）时，会触发 **Rebalance（再均衡）**，Partition 会被重新分配给组内的消费者。

#### 8. Zookeeper / KRaft

- **Zookeeper（传统模式）**：在旧版本中，Kafka 严重依赖 Zookeeper 来管理集群元数据，例如 Broker 列表、Topic 配置、Controller 选举等。
- **KRaft（新模式）**：从 Kafka 2.8 版本开始，社区引入了 KRaft 协议，旨在彻底移除对 Zookeeper 的依赖。Kafka 集群自己选举出一个 Controller 节点来管理元数据，从而简化了部署、运维，并提高了性能和可扩展性。目前，KRaft 模式已在生产环境中广泛推荐使用。



## 二、Kafka 基本用法（命令行）

在本地安装并启动 Kafka 后，你可以使用自带的命令行工具快速体验其工作流程。

#### 1. 创建一个 Topic

创建一个名为 `golang-test` 的 Topic，它有 3 个分区和 1 个副本。

```Bash
# For Kraft mode
kafka-topics.sh --create --topic golang-test --partitions 3 --bootstrap-server localhost:9092

# For Zookeeper mode
# kafka-topics.sh --create --topic golang-test --partitions 3 --replication-factor 1 --zookeeper localhost:2181
```

#### 2. 启动一个生产者发送消息

在终端启动一个生产者，然后你可以输入一些消息，每行一条。

```Bash
kafka-console-producer.sh --topic golang-test --bootstrap-server localhost:9092
>Hello Kafka
>This is a message from CLI
>Learning Golang with Kafka
```

#### 3. 启动一个消费者接收消息

打开另一个终端，启动一个消费者来接收 `golang-test` Topic 的消息。

```bash
kafka-console-consumer.sh --topic golang-test --from-beginning --bootstrap-server localhost:9092
```

`--from-beginning` 参数会让消费者从头开始消费该 Topic 的所有消息。你会看到生产者发送的三条消息被打印出来。



## 三、Kafka 面试高频考点

以下是面试中经常被问到的关于 Kafka 的问题，理解它们能帮你更好地掌握 Kafka 的核心设计。

#### 1. Kafka 如何保证消息不丢失？

这涉及到 Kafka 的数据可靠性机制，主要从三个层面保证：

- **生产者（Producer）层面**：
  - 使用 `acks` 配置。`acks=0`（不等待 Broker 确认）、`acks=1`（Leader 确认即可）、`acks=all`（所有 ISR 中的副本都确认）。通常设置为 `all` 以获得最高的数据保证。
  - 生产者发送失败后可以配置重试机制。
- **Broker 层面**：
  - **副本机制（Replication）**：每个 Partition 都有多个副本（Replica），分布在不同的 Broker 上。其中一个是 Leader，其余是 Follower。
  - **ISR（In-Sync Replicas）**：一个与 Leader 保持同步的 Follower 副本集合。当生产者设置 `acks=all` 时，消息必须被写入所有 ISR 副本后，才算成功。如果 Leader 宕机，Kafka 会从 ISR 中选举一个新的 Leader，保证数据不丢失。
- **消费者（Consumer）层面**：
  - 消费者在消费完消息后，才会向 Broker 提交（Commit）Offset。通过关闭自动提交（`enable.auto.commit=false`）并采用手动提交的方式，可以确保消息被成功处理后再更新 Offset，避免了消息在处理过程中丢失。

#### 2. Kafka 的消息投递语义是什么？

- **At most once（最多一次）**：消息可能会丢失，但绝不会重复。例如，消费者先提交 Offset 再处理消息，如果处理过程中宕机，这条消息就丢失了。
- **At least once（至少一次）**：消息绝不会丢失，但可能会重复。例如，消费者先处理消息再提交 Offset，如果在提交前宕机，重启后会重新消费这条消息。
- **Exactly once（精确一次）**：消息既不丢失也不重复，每条消息都被精确处理一次。Kafka 从 0.11 版本开始，通过 **幂等性生产者** 和 **事务（Transactions）** 实现了端到端的 Exactly-once 语义。



#### 3. 如何保证消息的顺序性？

Kafka 只能保证 **单个 Partition 内的消息是有序的**。

- 如果你的业务需要全局有序，那么只能将 Topic 设置为只有一个 Partition，但这会牺牲吞吐量。
- 如果你的业务只需要局部有序（例如，同一个用户的所有订单操作需要有序），可以将用户的 ID 作为消息的 Key。这样，所有该用户的消息都会被发送到同一个 Partition，从而保证了顺序性。

#### 4. Kafka 为什么这么快？

- **顺序 I/O**：Kafka 将消息追加到磁盘上的日志文件中，这是顺序写操作。相比于随机 I/O，磁盘的顺序 I/O 性能非常高，甚至可以媲美内存操作。
- **零拷贝（Zero-Copy）**：在将数据从磁盘发送到网络时，Kafka 利用了操作系统的零拷贝技术（如 `sendfile`），数据直接从内核空间的页缓存（Page Cache）发送到网卡，避免了在用户空间和内核空间之间的多次数据拷贝，极大地提升了数据传输效率。
- **批量处理（Batching）**：生产者可以批量发送消息，消费者可以批量拉取消息。这大大减少了网络开销，提高了吞- 吐量。
- **数据压缩**：生产者可以对消息进行压缩（如 Gzip, Snappy, LZ4），减少网络传输的数据量和磁盘存储空间。



## 四、Golang 实战：构建一个简单的 Kafka 应用

我们将使用 `confluent-kafka-go` 这个库，它是 `librdkafka`（一个高性能的 C 语言 Kafka 客户端）的 Go 语言封装，性能和稳定性都非常出色。

#### 准备工作

首先，安装 Go 语言环境和 Kafka。然后，获取 `confluent-kafka-go` 库：

```bash
go get github.com/confluentinc/confluent-kafka-go/v2/kafka
```

#### 1. 生产者（Producer）代码

创建一个名为 `producer.go` 的文件。

```go
// producer.go
package main

import (
	"fmt"
	"log"
	"strconv"
	"time"

	"github.com/confluentinc/confluent-kafka-go/v2/kafka"
)

func main() {
	// --- Producer Configuration ---
	// Create a new producer instance with a configuration map.
	// 'bootstrap.servers' points to the Kafka broker(s).
	p, err := kafka.NewProducer(&kafka.ConfigMap{"bootstrap.servers": "localhost:9092"})
	if err != nil {
		log.Fatalf("Failed to create producer: %s", err)
	}
	defer p.Close()

	// --- Delivery Report Goroutine ---
	// This goroutine handles delivery reports from Kafka.
	// It's crucial for knowing if messages were successfully sent.
	go func() {
		for e := range p.Events() {
			switch ev := e.(type) {
			case *kafka.Message:
				if ev.TopicPartition.Error != nil {
					// Message failed to be delivered.
					fmt.Printf("Delivery failed: %v\n", ev.TopicPartition)
				} else {
					// Message was successfully delivered.
					fmt.Printf("Delivered message to %v\n", ev.TopicPartition)
				}
			}
		}
	}()

	// --- Message Production ---
	// Define the topic to which we want to produce messages.
	topic := "golang-test"
	for i := 0; i < 10; i++ {
		// Prepare the message key and value.
		key := "key-" + strconv.Itoa(i)
		value := "hello world from golang producer - message " + strconv.Itoa(i)

		// Produce the message. This is an asynchronous operation.
		// The delivery report is handled by the goroutine above.
		err := p.Produce(&kafka.Message{
			TopicPartition: kafka.TopicPartition{Topic: &topic, Partition: kafka.PartitionAny},
			Key:            []byte(key),
			Value:          []byte(value),
		}, nil)

		if err != nil {
			log.Printf("Failed to produce message: %v\n", err)
		}
		
		// Wait for a short duration before sending the next message.
		time.Sleep(500 * time.Millisecond)
	}

	// --- Flush and Close ---
	// Wait for all messages in the producer's queue to be delivered.
	// The timeout is set to 15 seconds.
	p.Flush(15 * 1000)
	fmt.Println("Producer has finished sending messages.")
}
```

#### 2. 消费者（Consumer）代码

创建一个名为 `consumer.go` 的文件。

```go
// consumer.go
package main

import (
	"fmt"
	"log"
	"os"
	"os/signal"
	"syscall"
	"time"

	"github.com/confluentinc/confluent-kafka-go/v2/kafka"
)

func main() {
	// --- Consumer Configuration ---
	// Create a new consumer instance with a configuration map.
	// 'bootstrap.servers' points to the Kafka broker(s).
	// 'group.id' is essential for consumer groups.
	// 'auto.offset.reset' determines the starting point when no offset is stored.
	// 'earliest' means start from the beginning of the topic.
	c, err := kafka.NewConsumer(&kafka.ConfigMap{
		"bootstrap.servers": "localhost:9092",
		"group.id":          "my-golang-group",
		"auto.offset.reset": "earliest",
	})
	if err != nil {
		log.Fatalf("Failed to create consumer: %s", err)
	}
	defer c.Close()

	// --- Topic Subscription ---
	// Subscribe to the topic.
	topic := "golang-test"
	err = c.SubscribeTopics([]string{topic}, nil)
	if err != nil {
		log.Fatalf("Failed to subscribe to topic: %s", err)
	}

	// --- Graceful Shutdown Setup ---
	// Set up a channel to listen for OS signals (like Ctrl+C).
	sigchan := make(chan os.Signal, 1)
	signal.Notify(sigchan, syscall.SIGINT, syscall.SIGTERM)

	// --- Message Consumption Loop ---
	run := true
	for run {
		select {
		case sig := <-sigchan:
			// Received a shutdown signal.
			fmt.Printf("Caught signal %v: terminating\n", sig)
			run = false
		default:
			// Poll for a message. The timeout is set to 100 milliseconds.
			// This is a non-blocking call.
			ev := c.Poll(100)
			if ev == nil {
				continue // No event, continue polling.
			}

			switch e := ev.(type) {
			case *kafka.Message:
				// Received a message.
				fmt.Printf("Received message on %s: Key: %s | Value: %s\n", e.TopicPartition, string(e.Key), string(e.Value))
			case kafka.Error:
				// Encountered an error.
				// This could be a transient error or a fatal one.
				fmt.Fprintf(os.Stderr, "%% Error: %v: %v\n", e.Code(), e)
				if e.IsFatal() {
					run = false
				}
			default:
				// Other event types (e.g., stats).
				fmt.Printf("Ignored event: %v\n", e)
			}
		}
	}
	fmt.Println("Consumer is shutting down.")
}
```

#### 如何运行 Demo

1. 确保你的 Kafka 服务正在运行。

2. 打开一个终端，启动消费者：

   Bash

   ```
   go run consumer.go
   ```

   消费者会启动并等待消息。

3. 打开另一个终端，启动生产者：

   Bash

   ```
   go run producer.go
   ```

   生产者会发送 10 条消息。

4. 切换回消费者的终端，你会看到它接收并打印了生产者发送的所有消息。



## 总结

Apache Kafka 是一个功能强大且复杂的系统，但其核心设计思想却相当优雅。通过理解其 Broker、Topic、Partition 等基本概念，掌握其数据可靠性和高性能的原理，你就能在系统设计和面试中游刃有余。最后，结合 Go 语言的强大并发能力和 `confluent-kafka-go` 这样的高性能库，你可以轻松地构建出稳定、高效的实时数据应用。

希望这篇博客能成为你学习 Kafka 之路上的一个坚实起点。祝你学习愉快！
