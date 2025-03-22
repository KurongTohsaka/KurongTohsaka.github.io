---
title: "《MongoDB 权威指南》记录"
date: 2025-03-22
aliases: ["/Daily Dev"]
tags: ["MongoDB"]
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

# 《MongoDB 权威指南》记录

  ## MongoDB 简介

MongoDB 是面向文档的非关系型数据库。

MongoDB 具有以下特点：

- 易于使用：没有“行记录”的概念，而使用“文档”模型取代之。同时也没有预定义模式，文档键值的类型和大小是动态的。
- 易于扩展：MongoDB 采用了横向扩展方式，面向文档的数据模型使得跨多台服务器拆分数据更加容易。
- 性能强大：使用了机会锁，提高并发和吞吐量。同时会尽可能多的使用内存作为缓存，并尝试为查询自动选择合适的索引。



## 入门指南

### 文档

文档是一组有序键值的集合，如以下形式：

```python
{"q": "ans"}
```

### 集合

集合就是一组文档，相当于关系型数据库的表结构。

集合具有动态模式的特性，每个文档的长度大小、键数量都可以不一样。不过开发中并不推荐这样做，还是将“形状”相同的文档存在同一个集合中好，因为这样查询起来性能不会受到影响。

在 MongoDB 中通常会用 `.` 字符分隔不同命名空间的子集合，如 `blog.posts` 。

### 数据库

MongoDB 使用数据库对集合进行分组。

### 数据类型

- 基本数据类型：除了 int 等，还有 Regex、Array、ObjectID（12字节的文档的唯一ID 标识）、Bytes、JSCode。
- Array： `{"things": ["pie", 3.14]}`，数组内可以混有不同类型的数据。
- 内嵌文档：文档作为值。
- ObjectID：每个文档都有一个 `_id` ，默认是 ObjectID ，这个 ID 在集合内是唯一的。可以看到这个 ID 不是自增的，类似于 UUID，这说明 MongoDB 在设计之初就是作为分布式数据库存在的。以下是它的生成规则：
  - 前 4 字节为时间戳，中间 5 个字节是随机值，后 3 个字节是计数器（起始值随机）。



## 索引

MongoDB 中可以设置单一索引和复合索引，方式如下：

```shell
db.users.createIndex({"age": 1, "username": 1})  # 复合索引
```

MongoDB 的默认存储引擎是 WiredTiger ，索引是 B+ 树实现的。所以在索引的优化上，可以参照相关关系型数据库的索引优化经验与原则。下面是部分经验：

- 等值过滤的键放在最前面。
- 用于排序的键应该在多个值的字段之前。
- 多值过滤的键应该在最后面。

同时遵守以下原则：

- 选择键的方向：只有基于多个查询条件进行排序时，索引方向就重要了。
- 覆盖查询：尽可能的使一个索引覆盖到查询会用到的所有字段。
- 使用 in 条件查询，而不是 or 。

MongoDB 还有以下类型的索引：

- 唯一索引：可以单独设置，其中 `_id` 的唯一索引会自动创建。
- 部分索引：MongoDB 的部分索引（Partial Index） 是一种特殊类型的索引，它只对满足特定条件的文档建立索引，而不是对整个集合中的所有文档建立索引。这种索引可以显著减少索引的存储空间，并提高查询性能，尤其是在处理大型集合时。
  - 部分索引与稀疏索引不同，后者只对存在该字段的文档建立索引、不支持复杂查询条件。



## 特殊的索引和集合类型

 ### 全文搜索索引

全文搜索索引（Full-Text Search Index） 是 MongoDB 中用于支持文本搜索功能的一种特殊索引类型。它允许用户对集合中的文本字段进行高效的全文搜索，支持关键词匹配、词干提取、停用词过滤等功能，类似于搜索引擎的查询方式。

MongoDB 会为每个匹配的文档计算一个文本搜索评分（Text Score），表示文档与查询的相关性。

同时 MongoDB 支持多种语言的全文搜索，默认语言为英语。可以通过 `default_language` 参数指定语言。

### 固定集合和 TTL 索引

固定集合是一种特殊类型的集合，它的大小是固定的，达到上限后会自动覆盖最旧的文档。文档按插入顺序进行存储，类似于队列。由于数据按顺序写入且不可修改，固定集合的写入和读取性能非常高。

TTL 索引是一种特殊类型的索引，用于自动删除集合中过期的文档。通过为文档的某个日期字段创建 TTL 索引，MongoDB 会自动删除该字段值超过指定时间段的文档。固定集合就使用了 TTL 索引。

### GridFS

GridFS 是 MongoDB 提供的一种用于存储和检索大文件的规范。它通过将大文件分割成多个小块（chunks）来存储，解决了 MongoDB 单个文档大小限制（16MB）的问题。GridFS 非常适合存储大型二进制文件（如图片、视频、音频等）。

GridFS 将文件分成以下两部分存储在两个集合中：

1. `fs.files` 集合：存储文件的元数据（如文件名、大小、MIME 类型等）。
2. `fs.chunks` 集合：存储文件的实际数据，每个 chunk 默认大小为 255KB。

