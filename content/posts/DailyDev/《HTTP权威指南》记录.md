---
title: "《HTTP权威指南》记录"
date: 2025-03-14
aliases: ["/Daily Dev"]
tags: ["HTTP"]
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

# 《HTTP权威指南》记录

**本书中的 HTTP 协议版本为 HTTP/1.1 。**

## URL 与资源

### URL 语法

- 在 URL 中以分号 `;` 作为参数的分隔符，如下：

  ```url
  http://xxx.xxxx.com/hammers;sale=false/index.html;graphics=true
  ```

- 有些资源需要通过查询字符串来缩小查找范围，查询字符串通过 `?` 分割，如常见的 `GET` 请求中的查询参数。

- URL 支持使用片段组建来表示一个资源内部的片段，以 `#` 进行分割：

  ```url
  http://xxx.xxx.com/index.html#head
  ```



## 连接管理



