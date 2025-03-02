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
searchHidden: false
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
    
    func (s *server) AddProduct(ctx context.Context, in *pb.Product) (*pb.ProductID, error) {
      // 业务逻辑
    }
    
    func (s *server) getProduct(ctx context.Context, in *pb.ProductID) (*pb.Product, error) {
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



## 开始使用 gRPC

### 创建服务定义

正如一开始的“gRPC 入门”篇章中的代码片段，先定义消息类型、服务接口。完成服务定义后，就是生成代码并进行实现了。

### Go 实现

- gRPC 服务端：

  - ```go
    package main
    
    import (
      "context"
      "errors"
      "log"
      
      "github.com/gofrs/uuid"
      pb "productinfo/service/ecommerce"
    )
    
    // 中间件交互的抽象
    type server struct {
      productMap map[string]*pb.Product
    }
    
    func (s *server) AddProduct(ctx context.Context, in *pb.Product) (*pb.ProductID, error) {
      out, err := uuid.NewV4()
      if err != nil {
        return nil, status.Errorf(codes.Internal, "Error while generating Product ID", err)
      }
      
      in.Id = out.String()
      if s.productMap != nil {
        s.ProductMap = make(map[string]*pb.Product)
      }
      s.productMap[in.Id] = in
      
      return &pb.ProductID{Value: in.Id}, status.New(codes.OK, "").Err()
    }
    ```

  - ```go
    package main
    
    import {
      "log"
      "net"
      
      pb "productinfo/service/ecommerce"
      "google.golang.org/grpc"
    }
    
    const (
      port = ":50051"
    )
    
    func main() {
      lis, err := net.listen("tcp". port)
      if err != nil {
        log.Fatalf("failed to listen: %v", err)
      }
      
      // 创建服务器并监听
      s := grpc.NewServer()
      pb.RegisterProductInfoServer(s, &server())
      log.Printf("Starting gRPC listener on port "+ port)
      
      if err := s.Serve(lis); err != nil {
        log.Fatalf("failed to serve: %v", err)
      }
    }
    ```

- gRPC 客户端：

  - ```go
    package main
    import (
      "context"
      "time"
      "log"
      
      "google.golang.org/grpc"
      pb "productinfo/service/ecommerce"
    )
    
    const (
      address = "localhost:50051"
    )
    
    func main() {
      conn, errr := grpc.Dial(address. grpc.WithInsecure())
      if err != nil {
        log.Fatalf("did not connect: %v", err)
      }
      defer conn.close()
      
      c := pb.NewProductInfoClient(conn)
      ctx, cancel := context.WithTimeout(context.Background(), time.Second)
      defer cancel()
      
      name := "Apple 11"
      description := "Apple 11"
      price := float32(1000.0)
      r, err := c.AddProduct(ctx, &pb.Product{Name: name, Description: description, Price: price})
      if err != nil {
        log.Fatalf("Could not add product: %v", err)
      }
      log.Printf("Product ID: %s added successfully", r.Value)
      
      product, err := c.GetProduct(ctx, &pb.ProductID(Value: r.Value))
      if err != nil {
        log.Fatalf("Could not get product: %v", err)
      }
      log.Printf("Product: ", Product.String())
    }
    ```



## gRPC 的通信模式

### 一元 RPC 模式

前面出现的示例就是一元 RPC 模式，一个服务端、一个客户端，通信时始终只有一个请求和一个响应。这种方式适用于大多数的进程间通信。

### 服务端流 RPC 模式

在服务端流 RPC 模式，服务端接收到客户端的请求消息后，会发回一个响应的序列，这个序列叫做“流”。在将所有的服务端响应发送结束后，服务端会以 trailer 元数据的形式将其状态发送给客户端，从而标记流的结束。

使用方式：

- 服务定义：返回是 stream 类型

  - ```protobuf
    service OrderManagement {
    	rpc searchOrders(google.protobuf.StringValue) returns (stream Order);
    }
    ```

- 服务端实现：

  - ```go
    func (s *server) SearchOrders(searchQuery *wrappers.StringValue, stream pb.OrderManagement_SearchOrdersServer) error {
      // 业务逻辑
      ...
      err := stream.Send(&order)  // 通过 send 发送到流中
     	return nil
    }
    ```

  - 接收 stream 的响应，最后返回 nil 来标记流已经结束。

- 客户端实现：

  - ```go
    ...
    c := pb.NewOrderManagementClient(conn)
    ...
    searchStream, _ := c.SearchOrders(ctx, &wrapper.StringValue{Value: "Google"})
    for {
      searchOrder, err := searchStream.Recv()
      if err == io.EOF {  // 流已经结束
        break
      }
      log.Printf("Search Result: ", searchOrder)
    }
    ```

### 客户端流 RPC 模式

在客户端流 RPC 模式，客户端会发送多个请求给服务端，服务端会发送一个响应给客户端。但是服务端不一定要等到客户端接收到所有消息后才发送响应。

使用方式：

- 服务定义：请求是 stream 类型

  -  ```protobuf
     service OrderManagement {
     	rpc updateOrders(stream Order) returns (google.protobuf.StringValue);
     }
     ```

- 服务端实现：

  - ```go
    func (s *server) UpdateOrders(stream pb.OrderManagement_UpdateOrdersServer) error {
      ...
      order, err := stream.Recv()
      ...
      return stream.SendAndClose(...)  // 服务端通过该方法发送响应
      ...
    }
    ```

- 客户端实现：

  - ```go
    ...
    c := pb.NewOrderManagementClient(conn)
    ...
    if err := updateStream.Send(&upOrder1); err != nil {  // 客户端通过 Send 连续发送多条请求
      log.Fatalf(...)
    }
    ...
    updateRes, err := updateStream.CloseAndRecv()  // 接收响应
    ...
    ```

### 双向流 RPC 模式

在双向流 RPC 模式中，客户端以消息流的形式发送请求到服务端，服务端也以消息流形式响应。

流的操作完全独立，客户端和服务端可以按照任意顺序进行读取和写入。一旦建立连接，通信模式就完全取决于客户端和服务端本身。



## gRPC 的底层原理

### RPC 流

一个基本的 gRPC 过程，以 getProduct 方法为例：

1. 客户端通过生成的存根调用 getProduct方法；
2. 客户端存根使用已编码的消息创建 POST 请求。在 gRPC 中所有的请求都是 PSOT，并且 content-type 的值为 application/grpc ；
3. HTTP 请求消息通过网络发送到服务端；
4. 接收到消息后，服务端检查消息头消息，确定要调用的方法，然后将消息传递给服务端骨架；
5. 服务端骨架将消息解析成特定的数据结构；
6. 借助解析后的消息，服务发起对 getProduct 方法的本地调用。

gRPC 的这些步骤与其他 RPC 方式很相似，而主要区别在与消息的编码方式。gRPC 采用 protocol buffers。

### 使用 protocol buffers 编码消息

对于 protocol buffers 中的消息定义：

- ```protobuf
  message ProductID {
  	string value = 1;
  }
  ```

- 消息在经过编码后的字节流：消息字段1、消息字段2、….、结束标志。

  - 每个消息字段由标签、值两部分组成，标签由字段索引和线路类型（wire type）组成，字段索引就是上面消息定义里的“1”，线路类型则是该消息字段所属数据结构的类型，如 string 属于 “2:基于长度分隔”，那么其线路类型就是 2。值的编码会根据数据类型而不同。

### 基于长度前缀的消息分帧

长度前缀分帧是指在写入消息本身之前，写入长度信息，来表明每条消息的大小。

在 gRPC 通信中，每条消息都有额外的4字节用来设置其大小，这表示 gRPC 通信可以处理大小不超过 4GB 的所有消息。

除了消息的大小外，还有单字节的无符号整数，用于表明数据是否进行了压缩。

### 基于 HTTP/2 的 gRPC

- 请求消息：请求消息包含请求头信息、以长度作为前缀的消息、流结束标记 EOS。远程调用会在客户端发送请求信息之后就会初始化，然后发送以长度作为前缀的消息，最后发送 EOS 标记。
- 响应消息：响应消息包含响应头消息、以长度作为前缀的消息、trailer，这个也是发送顺序。

那么，从这个角度再来重新看下 gRPC 的通信模式：

- 一元 RPC 模式：就是以上面消息原始的方式互相发出去；
- 服务端流 RPC 模式：请求消息不变，响应消息中以长度作为前缀的消息可以是多个；
- 客户端流 RPC 模式：响应消息不变，请求消息中以长度作为前缀的消息可以是多个；
- 双向流 RPC 模式：两种消息的以长度作为前缀的消息都可以是多个。

至此，gRPC 在 HTTP/2 层的原理介绍就到这了。



## gRPC：超越基础知识

这一部分就不看了，就简单的把 gRPC 还能干啥写出来：

- 拦截器
- 截止时间
- 取消
- 错误处理
- 多路复用
- 元数据
- 负载均衡





