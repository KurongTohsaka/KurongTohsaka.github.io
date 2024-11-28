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
