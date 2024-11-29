---
title: "《深入RabbitMQ》记录"
date: 2024-11-28
aliases: ["/Daily Dev"]
tags: ["RabbitMQ"]
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

# 《深入RabbitMQ》记录

## RabbitMQ 基础

### AMQP 协议

AMQP，即高级消息队列协议（Advanced Meesage Queuing Protocal）。它定义了一个 AMQ 应该有以下三部分：

- 交换器：一个接收器接收发送到 MQ 中的消息并决定把它们投递到何处；
- 队列：存储接收到的消息；
- 绑定：定义队列和交换器之间的关系。在 RabbitMQ 中，绑定或者叫绑定键即告知一个叫交换器应该将消息投递到哪些队列中。



## 使用 AMQ 协议与 Rabbit 进行交互

### AMQP 帧类型

AMQP 规定了五种类型的帧：

- 协议头帧：用于连接到 RabbitMQ，仅使用一次；
- 方法帧：携带发送到 RabbitMQ 或从 RabbitMQ 接收到的 RPC 请求或响应；
- 内容头帧：包含一条消息的大小和属性；
- 消息体帧：包含消息的内容；
- 心跳帧：确保连接，一种校验机制。

### 将消息编组成帧

首先是方法帧、内容头帧，这两种帧比较好理解。

而消息体帧会根据消息体的大小切分成一到多个消息体帧。

要发送消息，首先发送的是方法帧，之后是内容帧，最后是若干个消息体帧。

### 帧结构

方法帧携带构建 RPC 请求所需的类、方法和参数。

内容头帧包含了消息的大小和其他对消息起描述作用的属性。

消息体帧可以传输很多类型的数据，可以是二进制、文本，也可以是二进制化的图片和序列化后的json、xml。

### 使用协议

包含下面一个基本流程：

- 声明交换器
- 声明队列
- 绑定队列到交换器
- 发布消息到 RabbitMQ
- 从 RabbitMQ 消费消息



## 消息属性详解

### 使用 content-type 创建显式的消息契约

在不同的语言的消费者框架中，框架可以自动的根据 content-type 类型将其反序列化为该语言中的数据结构。

### 使用 gzip 和 content-encoding 压缩消息大小

通过指定 content-encoding 属性可以在消息体上应用特殊的编码。

### 使用 message-id 和 correlation-id 引用消息 

correlation-id 用于指定该消息是另一个消息的响应。

### 创建时间 timestamp

### 消息自动过期 expiration

### 通过 delivery-mode 平衡速度和安全性

delivery-mode 可以用来指定在消息被消费前是否要持久化消息。

### 使用 app-id 和 user-id 验证消息来源

在处理消息之前检查 app-id 允许应用丢弃那些来源不明或不受支持的消息。app-id 可以是应用版本、API类等等。

user-id 即发送消息的用户 id。

### 使用 type 获取明细

可以使用 type 来描述消息的内容。

### 使用 headers 自定义属性

headers 是一个键值对表，可以存储自定义的属性。

### 优先级 priority



## 消息发布的性能权衡

- 消息确认机制很重要，但同时也会影响性能，有时会关掉消息确认已保证实时性；
- 使用 mandatory 设置，MQ 将不再接受不可路由的消息；
- 将发布者确认机制作为事务的轻量级替代（二者不可并存）；
- 使用备用交换器处理无法路由的消息；
- 使用事务来批量处理消息；
- 使用高可用队列（HA）避免节点故障
- 谨慎考虑消息持久化机制；
- 在负载程度大时，MQ 会对狂发消息的客户端使用消息回推机制，一段时间内 MQ 不会再接收该客户端的消息。



## 消费信息，避免拉取

### 对比 Basic.Get 和 Basic.Consume

Basic.Get 用于查询队列中是否存有消息，不要用于“获取”消息并消费。

要想获取信息并消费应该使用的是 Basic.Consume 。

RabbitMQ 采用发布-订阅模式，客户端发送命令将自己注册到 RabbitMQ 中，这样才能开始获取消息。RabbitMQ 通过唯一标识的消费者标签来分辨不同消费者。

### 优化消费者性能

- 使用 no-ack 模式；
- 通过服务质量（QoS）控制消费者预取；
- 使用事务批量获取、消费。

### 拒绝消息

使用 Basic.Reject 通知代理服务器无法对所投递的消息进行处理，但只能用于单个消息的拒绝。使用 Basic.Nack 拒绝多个消息。

被拒绝的消息可以作为死信消息由死信交换器进行路由到另一个队列。

### 控制队列

可以控制队列是临时的、永久的，队列还可以只允许单个消费者、自动过期、设置消息自动过期、最大队列长度等操作。



## 消息路由模式

### 通过 direct 交换器路由消息

当需要投递的消息有一个或多个确定的目标时，direct 交换器就是适用场景。它会检查绑定时会比较字符串是否相等，此时不允许使用任何类型的模式匹配。

### 使用 fanout 交换器广播消息

所有发往 fanout 交换器的消息会被投递到所有绑定到该交换器的队列中，这也意味着这些队列对应的消费者都能消费该广播的消息。

### 使用 topic 交换器有选择性地路由消息

topic 交换器会将消息路由到匹配路由键的任一队列中，但是通过采用句点分隔的形式，队列可以通过基于通配符的模式匹配的方法来绑定到路由键上。通过 * 号会匹配路由键中下一个句点前的所有字符，# 号会匹配接下来的所有字符，包括句点。

topic 交换器时消息路由的极佳选择，它使得单一用途的消费者能够对消息进行不同的处理。

### 使用 headers 交换器有选择地路由消息

headers 交换器通过消息属性中的 headers 表支持任意的路由策略。

### 交换器性能对比

四种交换器性能几乎一致 😓

