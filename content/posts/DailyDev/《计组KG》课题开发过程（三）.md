---
title: "《计组KG》课题开发过程（三）"
date: 2024-11-04
aliases: ["/Daily Dev"]
tags: ["Daily Dev", "Project Dev"]
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

## 牢骚

时隔一个多月才完成了基本的可视化系统的搭建，中间有着各种各样的原因：Vue3第一次用、Django-Ninja不熟悉等等再加上一些闲杂原因就一直拖到了现在，效率多少有点堪忧。

总之不算怎么说，也算是曲折的完成了原定计划，下面将从后端、前端的技术选择、功能介绍、成品展示等几个章节大致的讲述下。



## 后端

### 项目地址

[KurongTohsaka/PCCKGVisualization](https://github.com/KurongTohsaka/PCCKGVisualization/tree/main)

### 技术栈

Web 框架自然是选相对擅长且好用的 Django ，但是 Django 本身使用起来又过于繁重，不太适合开发Restful类型的接口，所以一般会搭配着 Django REST Framework (DRF) 使用。但是这一次我打算尝试一个新的 Django 扩展，它是 FastAPI 的 Django 版：Django-Ninja 。

Web 框架订好了以后，就该选数据库了。既然是存储图数据，那肯定是 Neo4j 。而网站数据就使用简单的 Sqlite 吧，主要就是省事。

### 功能与接口

**下面的所有接口的最上级请求路径：`pcc_kg_vs`**

功能主要分为两部分：认证和可视化。

首先是认证，包含以下基本功能：

- 用户认证：用户登陆、注册、登出
- API 鉴权：token 验证
- CSRF 验证：CSRF token 验证

下面是简易的接口标准：

- 登陆

  - 先检查该用户是否已注册，若注册则从数据库中根据信息查询到用户，然后登陆该用户。若未注册则登陆失败

  - ```
    {
      "method": "POST",
      "path": "/login",
      "params": {
        "username": "",
        "password": ""  
      },
      "return": {
        "successful": bool,
        "code": int,
        "token": str,  //登陆令牌
        'info': str
      }
    }
    ```

- 登出 HTTPBearer

  - 检查用户信息，然后登出该用户

  - ```
    {
      "method": "GET",
      "path": "/logout",
      "params": {},
      "return": {
        "successful": bool,
        "code": int,
      	"info": str
      }
    }
    ```

- 申请

  - 根据信息注册用户，注意不要和数据库中用户重叠。申请成功后直接登陆

  - ```
    {
      "method": "POST",
      "path": "/register",
      "params": {
        "username": str, 
        "password": str 
      },
      "return": {
        "successful": bool,
        "code": int,
        "token": str,
      	"info": str
      }
    }
    ```

然后是可视化功能，目前只包含知识图谱可视化。考虑到本次课题只涉及到了知识图谱的可视化，但是又不得不为以后的接口扩展做考虑，所以接口做成了一个通用接口。

下面是接口标准：

- 可视化 HTTPBearer

  - ```
    {
      "method": "POST",
      "path": "common_visual",
      "params": {
        "vs_type": "graph",  //可视化样式
        "data": []  //可视化数据
        ...
      },
      "return": {
            "successful": bool,
            "code": int,
            "chart": {
              "graph": ...  //vs_type对应的值
            },
          	"info": str
          }
    }
    ```

  - neo4j 的图可视化请求参数：

  - ```
    {
    	"vs_type": "graph",
      "data": {
        "node_ids" : [],  //要可视化的节点ID
        "relation_ids": []  //要可视化的关系ID
      }
    }
    ```



## 前端

### 项目地址

[KurongTohsaka/pcc-kg-vs-frontend](https://github.com/KurongTohsaka/pcc-kg-vs-frontend)

### 基本功能

包含了三个基本页面：

- 用户登陆
- 用户注册
- 仪表盘 Dashboard

> 没啥可介绍的，看看页面就能明白有啥用。

### 成品展示

- 用户登陆：

  ![](/img/DailyDev/img1.png)

- 用户注册：

  ![](/img/DailyDev/img2.png)

- 仪表盘：

  ![](/img/DailyDev/img3.png)



## 总结

系统还有许多可以优化的地方，在之后的几个月或许会不断的修修补补。

总之，先告一段落了，接下来重新回到算法优化和写作上啊（叹气）。
