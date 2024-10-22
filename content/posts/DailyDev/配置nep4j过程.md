---
title: "MAC上用docker配置neo4j以及apoc扩展的过程"
date: 2024-10-22
aliases: ["Daily Dev"]
tags: ["neo4j"]
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

## 拉取 neo4j Docker image

在 MAC 上用的 Docker Desktop，确实很方便操作。

直接在 Image Tab 页搜 Neo4j 就有一个官方 image，我选的 5.24.1版本，pull 下来。

这里之所以要选择一个特定的版本，原因是之后要启用的 apoc 扩展需要用对应 neo4j 版本的 jar 包。



## 配置容器

因为用的 Docker Desktop，很多没必要的步骤都可以直接忽略。配置好转发端口、挂载位置后，就启动容器。

Neo4j 容器启动后默认直接运行，登陆 Web 管理界面配置后用户名密码后就完成了基本配置。

> Docker 真 tm 好用（



## 配置 apoc

**这里需要注意一点，neo4j 5.x 之后的版本中，apoc 本身的相关配置需要写在 `neo4j/conf/apoc.conf` 里，而不是 `neo4j/conf/neo4j.conf` 中，不然在启动服务时会报错！！！**

首先是 apoc 的下载地址：[Index of /doc/neo4j-apoc/](http://doc.we-yun.com:1008/doc/neo4j-apoc/) 。

下载好后，把它放进 `neo4j/plugins`，并改名为 `apoc.jar` **（啥名都行，这里是方面后面改配置）**。

接下来修改 `neo4j/conf/neo4j.conf` ，修改以下两行并取消该行注释：

````conf
dbms.security.procedures.unrestricted=apoc.*
dbms.security.procedures.allowlist=apoc.coll.*,apoc.load.*
````

最后修改 `neo4j/conf/apoc.conf` ：

```conf
apoc.import.file.enabled=true
apoc.export.file.enabled=true
```

到这里就配置完所有的了，重启服务即可。

