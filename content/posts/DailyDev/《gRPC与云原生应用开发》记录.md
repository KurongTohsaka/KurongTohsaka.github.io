---
title: "《gRPC与云原生应用开发》记录"
date: 2024-11-30
aliases: ["/Daily Dev"]
tags: ["RPC", "Cloud Native", "Golang"]
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

# 《gRPC与云原生应用开发》记录

## gRPC 入门

### gRPC 的定义

gRPC 是一项进程间通信技术，可以用来连接、调用、操作和调试分布式异构应用程序。

开发 gRPC 应用程序时，要先定义服务接口：消费者消费信息的方式、消费者能够远程调用的方法以及调用方法所使用的参数和消息格式等。在服务定义中使用的语言叫作接口定义语言（IDL）。

下面是使用 protocol buffers 作为 IDL 来定义服务接口：

- 服务定义

  - ```protobuf
    // ProductInfo.proto
    syntax = "proto3";
    package ecommerce;  // 防止协议消息类型之间发生命名冲突
    
    // 定义服务接口
    service ProductInfo {
    	rpc addProduct(Product) returns (ProductID);
    	rpc getProduct(ProductID) returns (Product);
    }
    
    // 定义消息格式
    message Product {
    	string id = 1;
    	string name = 2;
    	string description = 3;
    }
    
    message ProductID {
    	string value = 1;
    }
    ```

  - 上面定义完成之后，就使用 protocol buffers 编译器 protoc 生成服务器端和客户端代码。

- gRPC 服务端

  - ```go
    import (
    	"context"
      pb ".../proto"
      "google.golang.org/grpc"
    )
    
    // ProductInfo 的 go 实现
    
    func (s *server) AddProduct(ctx context.context, in *pb.Product) (*pb.ProductID, error) {
      // 业务逻辑
    }
    
    func (s *server) getProduct(ctx context.context, in *pb.ProductID) (*pb.Product, error) {
      // 业务逻辑
    }

  - gRPC 服务端要做两件事：

    - 通过重载服务基类，实现所生成的服务器端骨架的逻辑

    - 运行 gRPC 服务器，监听来自客户端的请求并返回服务响应

      ``` go
      func main() {
        lis, _ := net.listen("tcp". port)
        s := grpc.NewServer()
        pb.RegisterProductInfoServer(s, &server())
        if err := s.Serve(lis); err != nil {
          log.Fatalf("failed to serve: %v", err)
        }
      }
      ```

- gRPC 客户端

  - ```go
    // 使用远程服务器地址创建通道
    ManagedChannel channel = ManagedChannelBuiler.forAddress("localhost", 8080)
    	.usePlaintext(true)
    	.build()
    
    // 使用通道初始化阻塞式存根
    ProductInfoGrpc.ProductInfoBlockingStub stub = ProductInfoGrpc.newBlockingStub(channel)
    
    // 使用阻塞式存根调用远程方法
    StringValue ProductID = stub.addProduct(
      Product.newBuiler()
      .setName("Apple")
      .setDescription("Apple")
      .build()
    )
    ```

  - 可以使用服务定义生成客户端存根，它提供了与服务端类似的方法，供客户端进行调用。

- 客户端-服务端的消息流：当调用 gRPC服务时，客户端的 gRPC 库会使用 protpcol buffers，并将 RPC 的请求编排（marshal）为 protocol buffers 格式，然后将其通过 HTTP/2 进行发送。在服务端，请求会被解排（unmarshal），对应的过程调用会使用 protocol buffers 来执行。

### RPC 演进

- 传统的 RPC：通用对象请求代理体系结构（CORBA）和 Java 远程方法调用（RMI）。基于 TCP 构建，很复杂，且限制较多；
- 简单对象访问协议（SOAP）：在服务之间基于 XML 交换数据，并能基于任意的底层通信协议进行通信，如HTTP。但是消息格式复杂、规范复杂；
- 描述性状态迁移（REST）：基于 HTTP 实现。REST 架构风格已经成为构建微服务的标准方法了，但是其仍然有局限性：
  - 基于文本的低效消息协议：使用 json 等人类可读形式的文本类型在服务间通信时是低效的；
  - 应用程序之间缺乏强类型接口：不同语言构建的服务通过网络进行交互，导致缺乏明确定义和强类型的服务接口成为了使用 RESTful 服务的主要阻碍；
  - REST 架构风格难以强制实施：虽然有一些“好的实践”，而且只有遵循这些实践才能构建真正的 RESTful 服务。但是 REST 架构不是强制性的。且那些实践难以实施。
- gRPC：2015年由Google开源。真神降临 😊

gRPC 的优势所在：

- 高效的进程间通信：使用基于 HTTP/2 的 protocol buffers 进行进程间通信；
- 简单且定义良好的服务接口和模式：使用 IDL 定义服务接口，可以保证简单但一致、可靠且可扩展的开发体验；
- 属于强类型：明确了服务接口的输入、返回的消息类型，属于强类型；
- 支持多语言（语言中立）；
- 支持全双工：可以开发流服务或流客户端；
- 具备多种特性：支持认证、加密、弹性、负载均衡、服务发现等；
- 与云原生的集成：是 CNCF 的一部分，大多数现代框架和技术都提供了对 gRPC 的原生支持。
