---
title: "《Go微服务实战》记录"
date: 2025-01-18
aliases: ["/Daily Dev"]
tags: ["Golang", "微服务"]
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

# 《Go微服务实战》记录

本书包含很多的 Go 基础和部分进阶内容，这里只选取对现阶段有帮助的内容，毕竟 Go 已经入门过一遍了。

## Go 基础

- 关于指针，如果涉及到修改变量本身就使用指针作为函数输入变量等，典型的如单例模式下的变量都是指针。反之如果不修改变量本身，就直接使用复制变量作为函数输入变量等，函数返回值自然也是某个新的变量。

- 关于 Go 中的循环，请看 code：

  ```go
  package main
  
  func main() {
    for i := 0; i < 10; i++ {
      ...
    }  // 最标准的循环
    
    for {
      ...
    }  // 就是没有显式跳出条件的 while
    
    arr := []int{1, 2, 3, 4, 5}
    for i, v := range arr {
      ...
    }  // 这里的 range 是关键字，用法有些类似于 Python 中的 enumerate 函数
  }
  ```

- Go 的垃圾回收策略使用的三色标记法，具体内容会单独进行学习。



## 字符串与复合数据类型

- 关于字符串的相关操作，见代码：

  ```go
  package main
  
  import "fmt"
  
  func main() {
    s := "hello world"
    fmt.Println(len(s))  // 11
    fmt.Println(s[0])  // 101, 内存地址
    fmt.Println(s[0:1])  // h, 切片用法与 Python 一致 
  }
  ```

- 关于数组的相关操作，见代码：

  ```go
  package main
  
  import "fmt"
  
  func main() {
    a := [...]int{1, 2, 3, 4}  // ... 表示自适应长度
    b := &a
    var c [4]int  // 默认值为0
    var s = [...]string{"hello", "world"}
    for i, v range s {
      if i == 0 {
        fmt.Printf("s[%d]:%s\n", i, v)  // s[0]:hello
      }
      break
    }
  }
  ```

- 切片底层通过数组实现。切片可以通过切片操作、 `s := []int[1, 2, 3]` 、`ss := make([]int, 10)` 等方式来进行声明。关于切片的相关操作以及部分概念，见下代码：

  ```go
  package main
  
  import "fmt"
  
  func main() {
    a := [...]{1, 2, 3, 4 , 5}
    ss := a[1:3]
    fmt.Println(ss)  // [2 3]
    ss[0] = 666
    fmt.Println(ss)  // [666 3]
    fmt.Println(a)  // [1 666 3 4 5], 发现对切片的修改也应用到了原数组的对应元素, 说明切片只是数组的引用
    fmt.Println(cap(ss))  // 2, cap 用于查看切片的容量
    append(ss, 777)  // 尾部添加, [666 3 777]
    fmt.Println(cap(ss))  // 4, 自动扩容到原容量的2倍，以此类推
    append(ss, []int{888})  // 另一种尾部添加, [666 3 777 888]
    ss = append(ss[:0], ss[:3]...)  // 只保留前三个元素, [666 3 777]
    ss = append([]int{000}, ss...)  // 首部添加, [000 666 3 777]
    var temp = []int{}
    copy(temp, ss)  // 把 ss 复制给 temp, 需要特别注意 copy 只能用于切片
  }
  ```

  切片可以为空，有长度为0和容量不为0和长度容量均为0两种情况。但是两种都为0不可以用 `nil` 判断，要根据长度、容量进行判断。

- 关于映射相关操作，见代码：

  ```go
  package main
  
  import "fmt"
  
  func main() {
    m1 := make(map[string]int)
    m1["k1"] = 1
    m1["k2"] = 2
    print(m1)  // k1 1
    delete(m1, "k1")
    print(m1)  // k2 2
    
    key = "k1"
    val, ok := m1[key]  // 取值
    if ok != nil {
      fmt.Println(val)
    } else {
      fmt.Printf{"The key `%s` not exist.", key}
    }
    
    for k, v := range m1 {
      ...
    }  // 通过 range 取值
  }
  ```

- 关于结构体，直接看代码：

  ```go
  package main
  
  import "fmt"
  
  type Person struct {
    Name string
    Gender, Age int
  } 
  
  type Employee struct {
    p Person
    Salary int
  }  // 结构体可以通过组合实现类的继承,。直接通过 e.Age 访问属性, 而不是 e.Person.Age，后面的方法同理
  
  func AddAge(p Person) (p2 Person) {
    p.Age += 1  // 通过此种方式访问属性
    return p
  }
  
  func AddAgePlus(pp *Person) {
    pp.Age += 1
  }
  
  func main() {
    p1 := Person{Name: "Scott", Gender: 1, Age: 18}
    p2 := AddAge(p1)
    fmt.Println(p1)
    fmt.Println(p2)  // {Name: "Scott", Gender: 1, Age: 19}
    
    AddAgePlus(&p1)  // AddAgePlus 接收 Person 指针变量，所以这里使用 & 传递内存地址
    fmt.Println(p1)  // {Name: "Scott", Gender: 1, Age: 19}
    
    pp := new(Person)  // 使用 new 声明一个空结构体指针
    AddAgePlus(pp)
    fmt.Println(pp)  // &{0 1}
  }
  ```



## 函数、方法、接口和反射

- 关于函数的部分内容，见代码：

  ```go
  package main
  
  import "fmt"
  
  func main() {
    ...
  }
  
  func swap(a int, b int) (int, int) {  // 多返回值
    return b, a
  }
  
  func sum(a int, others ...int) int {  // 变长参数，什么类型都可以
    for _, v := range others {
      a += v
    }
    return int
  }
  ```

- 关于方法，可以绑定给结构体、接口。见代码：

  ```go
  package main
  
  import "fmt"
  
  type Rectangle struct {
    w, h float64
  }
  
  // 接收器为指针类型
  func (r *Rectangle) area() float64 {
    return r.w * r.h
  }
  
  // 接收器为结构体类型
  func (r Rectangle) area2() float64 {
    return r.w * r.h
  }
  
  func main() {
    p := &Rectangle{w: 2, h: 2}
    fmt.Println(p.area())
    
    p2 := Rectangle{w: 2, h: 2}
    fmt.Println(p2.area2())
    fmt.Println(&p2.area())
    fmt.Println(p2.area())  // 会隐式的加上 *p2
  }
  ```

- 关于接口，Go 中的接口是鸭子类型，即实现了接口约定的属性和方法就视为实现了该接口。这是一种隐式的实现方式，而不用如 Java 中的 `implements` 关键字一样显式声明实现某个接口。见代码：

  ```go
  package main
  
  import (
    "fmt"
    "math"
  )
  
  type ShapeDesc interface {
    Area() float64
    Perimeter() float64
  }
  
  type rectangle struct {
    H, W float64
  }
  
  func (r rectangle) Area() float64 {
    ...
  }
  
  func (r rectangle) Perimeter() float64 {
    ...
  }
  
  func main() {
    var s ShapeDesc  // 声明接口
    s = rectangle{H: 1, W: 2}  // 此时 rectangle 结构体已经隐式的实现了 ShapeDesc 接口
    fmt.Println(s.Area())
    fmt.Println(s.Perimeter())
  }
  ```

- 关于反射，可以动态的获取对象类型及结构信息。Go 中使用反射需要借助 `reflect` 包，其中 `reflect.Value` 存储任意值，`reflect.Type` 存储任意类型。使用方式见代码：

  ```go
  package main
  
  import (
  	"fmt"
    "reflect"
  )
  
  type X struct {
    I int
    F float64
    S string
  }
  
  type Person struct {
    Name string `json:"jname"`
    Gender int  `json:"jgender"`
    Age int     `json:"jage"`
  }
  
  func (x X) CompareStr(xx X) bool {
    rx1 := reflect.ValueOf(&x).Elem()  // 通过 ValueOf 获取变量内存地址，然后通过 Elem 获取地址指向的 Value
    rx2 := reflect.ValueOf(&xx).Elem()
    for i := 0; i < rx1.NumField(); i++ {   // NumField 返回 Value 中的字段个数
      if rx1.Field(i).Interface() != rx2.Field(i).Interface() {  // 通过 Field(index) 返回对应索引的字段，Interface 是将字段以接口类型返回
        return false
      }
    }
    return true
  }
  
  func (p Person) PrintTags() {
    for i := 0; i < reflect.TypeOf(p).NumField(); i++ {
      fmt.Println(reflect.TypeOf(p).Field(i).Tag.Get("json"))  // 通过 TypeOf 获取到类型的内存地址，Field 同上，Tag 是获取结构体中的 tag，Get 是返回对应键的值
    }
  }
  
  func main() {
    ...
  }
  ```



## 并发编程

- 协程（goroutine）是 Go 中的轻量级线程，使用 `go` 关键字启动。

- 见代码：

  ```go
  package main
  
  import (
  	"fmt"
  	"sync"
  	"time"
  )
  
  // worker 是一个模拟工作任务的函数，每个协程都会运行这个函数
  // id 是工作者的标识，wg 是一个指向 WaitGroup 的指针，用于通知主线程任务完成
  func worker(id int, wg *sync.WaitGroup) {
  	// defer 确保在函数结束时调用 wg.Done()，将 WaitGroup 的计数减 1
  	defer wg.Done()
  
  	fmt.Printf("Worker %d: starting\n", id)
  	time.Sleep(time.Second * time.Duration(id))
  	fmt.Printf("Worker %d: done\n", id)
  }
  
  func main() {
  	// 创建一个 WaitGroup 实例，用于等待所有协程完成任务
  	var wg sync.WaitGroup
  	numWorkers := 3
  
  	// 启动多个协程，每个协程都会运行 worker 函数
  	for i := 1; i <= numWorkers; i++ {
  		wg.Add(1) // 每启动一个协程，将 WaitGroup 的计数加 1
  		go worker(i, &wg) // 启动协程并传递工作者 ID 和 WaitGroup 指针
  	}
  
  	// 调用 wg.Wait() 阻塞主线程，直到所有协程完成任务
  	wg.Wait()
  	fmt.Println("All workers completed.")
  }
  
  ```

-  关于通道，见代码：

  ```go
  package main
  
  import (
  	"fmt"
    "time"
  )
  
  func worker(id int, jobs <-chan int, results chan<- int) {  // 定义了单向通道作为函数参数
    for job := range jobs {  // 通过 range 关键字循环取（万能的 range）
      fmt.Printf("Worker %d started job %d\n", id, job)
  		time.Sleep(time.Second)
  		fmt.Printf("Worker %d finished job %d\n", id, job)
      results <- job * 2
    }
  }
  
  func main() {
    // 缓冲通道
    jobs := make(chan int, 5)
    results := make(chan int, 5)
    
    for i := 1; i <= 2; i++ {
      go worker(i, jobs, results)
    }
    
    // 向 jobs channel 中发送任务
  	for j := 1; j <= 5; j++ {
  		jobs <- j
  	}
    close(jobs) // 关闭 jobs channel，通知 workers 不再有更多任务
    
    done := make(chan struct{}) // 用于通知完成
    go func() {
      for i := 1; i <= 5; i++ {
        select {
          // 以下两步的 case 会同时开始执行，哪一个先成功就执行后续的操作然后跳出 select
          case result := <- results:
          	fmt.Printf("Result received: %d\n", result)
          case <- time.After(2 * time.Second):  // 超时检查，如果第一个操作超过了 time.After 规定的时间就超时
          	fmt.Println("Timeout waiting for result")
        }
      }
      done <- struct{}{}  // 通知完成
    }
    <-done  // 等待完成，通道为空时该操作会阻塞
    fmt.Println("All jobs processed.")
  }
  ```



## 并发编程进阶

### Go 的并发模式与goroutine

在一般的编程语言中，使用的都是内存共享式的并发模式。Go 则不然，它所推崇的并发模式叫做 CSP（Communicating Sequential Process），通过 goroutine 和 channel 实现。

> 当然，Go 也支持内存共享式并发模式，只是不推荐。

goroutine 的运行模型是 M:N ，即多个 goroutine 映射到多个操作系统线程上。下面是 goroutine 的调度模型 MGP：

- 调度由 Go 运行时负责，核心结构如下：
  - M（Machine）：操作系统线程，负责执行 goroutine 。
  - P（Processor）：逻辑处理器，负责分配 goroutine 到 M 。
  - G（Goroutine）：协程，由 Go 运行时管理。

goroutine 相比线程的优势：

- goroutine 的初始栈大小仅为 2KB，运行时会根据需要动态增长和缩小，最大支持 1GB，但通常会保持较小的内存占用，单个 Go 程序可以轻松创建数百万个 goroutine 。
- goroutine 的阻塞操作由 Go 的调度器转换为非阻塞操作。
- goroutine 切换的开销比线程低，因为只涉及用户态上下文切换。

### sync中的同步方式

sync 提供了多种用于同步的方式：

- `Mutex`：独占锁。
- `RWMutex`：允许多个读锁并发执行，写锁互斥。
- `Once`：确保某段代码只执行一次，通常用于初始化。
- `Cond`：用于条件变量的等待和通知。
- `Pool`：用于对象的复用以减少 GC 压力。
- `Map`：提供线程安全的 Map 操作，不需要额外加锁。

见代码：

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	// Mutex 示例
	var mu sync.Mutex
	counter := 0

	// RWMutex 示例
	var rwMu sync.RWMutex
	sharedData := "Initial Data"

	// Once 示例
	var once sync.Once
	initialize := func() {
		fmt.Println("This will run only once")
	}

	// Cond 示例
	var cond = sync.NewCond(&sync.Mutex{})
	var ready bool

	// Pool 示例
	pool := sync.Pool{
		New: func() interface{} {  // New 方法用于自动创建新对象
			return "New Value"
		},
	}

	// Map 示例
	var m sync.Map

	// Start goroutines to demonstrate functionality
	wg := sync.WaitGroup{}

	// Mutex: Protect the counter
	wg.Add(1)
	go func() {
		defer wg.Done()
		mu.Lock()
		defer mu.Unlock()
		counter++
		fmt.Println("Counter incremented:", counter)
	}()

	// RWMutex: Read/Write protection
	wg.Add(2)
	go func() {
		defer wg.Done()
		rwMu.RLock()
		defer rwMu.RUnlock()
		fmt.Println("Read shared data:", sharedData)
	}()
	go func() {
		defer wg.Done()
		rwMu.Lock()
		defer rwMu.Unlock()
		sharedData = "Updated Data"
		fmt.Println("Write shared data:", sharedData)
	}()

	// Once: Run a function only once
	wg.Add(2)
	go func() {
		defer wg.Done()
		once.Do(initialize)
	}()
	go func() {
		defer wg.Done()
		once.Do(initialize) // This call will be ignored
	}()

	// Cond: Notify waiting goroutines
	wg.Add(2)
	go func() {
		defer wg.Done()
		cond.L.Lock()
		for !ready {
			cond.Wait()  // Wait 会自动解锁
		}
		cond.L.Unlock()
		fmt.Println("Condition satisfied")
	}()
	go func() {
		defer wg.Done()
		cond.L.Lock()
		ready = true
		cond.L.Unlock()
		cond.Broadcast()  // 广播，通知所有满足条件的协程
	}()

	// Pool: Get/Put reusable objects
	wg.Add(1)
	go func() {
		defer wg.Done()
		value := pool.Get().(string)
		fmt.Println("Got from pool:", value)
		pool.Put("Reusable Value")
	}()

	// Map: Concurrent-safe map operations
	wg.Add(2)
	go func() {
		defer wg.Done()
		m.Store("key1", "value1")
		fmt.Println("Stored in map: key1 -> value1")
	}()
	go func() {
		defer wg.Done()
		value, ok := m.Load("key1")
		if ok {
			fmt.Println("Loaded from map:", value)
		}
	}()

	// Wait for all goroutines to complete
	wg.Wait()
}
```

### context

context 包有两个主要功能：控制请求的生命周期和传递上下文。

每个子 `Context` 都引用其父 `Context`，形成树状结构。当父 `Context` 被取消，所有子 `Context` 会自动取消。

context 中的重要函数：

- `context.Background()` ：返回根上下文，常用作顶层起点。
- `context.TODO()` ：用于不确定具体上下文的占位。
- `context.WithCancel(parent Context)` ：创建一个可取消的子上下文。
- `context.WithDeadline(parent Context, deadline time.Time)` ：创建一个在指定时间点超时的子上下文。
- `context.WithTimeout(parent Context, timeout time.Duration)` ：创建一个在指定超时时长后自动超时的子上下文。
- `context.WithValue(parent Context, key, val any)` ：返回携带特定键值对的子上下文。

读取 `Context` 的方法：

- `ctx.Done()` ：返回一个 channel，当上下文被取消或超时时，该 channel 被关闭。父级通过关闭 `Done` channel 通知子级。
- `ctx.Err()` ：返回上下文被取消或超时的具体原因，取消或超时。
- `ctx.Value(key)` ：获取上下文中的值。

context 的应用：

- 超时控制：

  ```go
  package main
  
  import (
  	"context"
    "fmt"
    "time"
  )
  
  func main() {
    ctx, cancel := context.WithTimeout(context.Background(), 2 * time.Second)
    defer cancel()
    
    select {
      case <-time.After(3 * time.Second):  // 执行时表示指定时间到了，任务正常完成
      	fmt.Println("Operation completed")
      case <-ctx.Done():
  			fmt.Println("Timeout:", ctx.Err())  // 执行时表示任务被取消或超时
    }
  }
  ```

- 并发取消：

  ```go
  package main
  
  import (
  	"fmt"
    "context"
    "time"
  )
  
  func worker(ctx context.Context, id int) {
    for {
      select {
        case <-ctx.Done():
        	fmt.Printf("Worker %d stopping: %s\n", id, ctx.Err())
  				return
        default:
        	fmt.Printf("Worker %d working...\n", id)
  				time.Sleep(500 * time.Millisecond)
      }
    }
  }
  
  func main() {
    ctx, cancel := context.WithCancel(context.Background())
    go worker(ctx, 1)
  	go worker(ctx, 2)
  
  	time.Sleep(2 * time.Second)
  	cancel() // 取消所有子任务
  }
  ```

- 数据传递：

  ```go
  package main
  
  import (
  	"context"
  	"fmt"
  )
  
  func handler(ctx context.Context) {
  	if user := ctx.Value("user"); user != nil {
  		fmt.Println("User:", user)
  	} else {
  		fmt.Println("No user in context")
  	}
  }
  
  func main() {
  	ctx := context.WithValue(context.Background(), "user", "Alice")
  	handler(ctx)
  }
  ```

### 工作池

工作池（Worker Pool） 是一种常见的并发模式，常用于控制任务的并发数量，以限制资源使用并提高程序效率。通过工作池，可以在一定范围内高效处理大量任务，同时避免过多的 goroutine 创建导致的系统资源压力。见代码：

```go
package main

import (
	"fmt"
	"sync"
)

func worker(id int, jobs <-chan int, results chan<- int, wg *sync.WaitGroup) {
	defer wg.Done() // 确保 Goroutine 执行完成后通知 WaitGroup
	for job := range jobs {
		fmt.Printf("Worker %d started job %d\n", id, job)
		results <- job * job // 任务执行结果
		fmt.Printf("Worker %d finished job %d\n", id, job)
	}
}

func main() {
	const numWorkers = 3
	const numJobs = 10

	jobs := make(chan int, numJobs)
	results := make(chan int, numJobs)
	var wg sync.WaitGroup // WaitGroup 用于等待所有任务完成

	// 启动工作者，创建固定数量的协程就是工作池的基本原理
	for i := 1; i <= numWorkers; i++ {
		wg.Add(1)
		go worker(i, jobs, results, &wg)
	}

	// 提交任务
	for j := 1; j <= numJobs; j++ {
		jobs <- j
	}
	close(jobs) // 所有任务已提交

	// 等待所有工作者完成任务
	wg.Wait()
	close(results)

	// 处理结果
	for result := range results {
		fmt.Println("Result:", result)
	}
}
```



## 微服务

微服务是一种细粒度的分布式解决方案，这些细粒度的服务独立性强并且会协同工作。

需要注意，微服务架构和微服务概念不同。微服务架构是一种具体的设计实现，而微服务是通过这种实现或方案最终完成的服务。

微服务的特征：

- 职责唯一性：微服务中的每个服务都是单一的，即高内聚、低耦合。
- 通信轻量级：服务之间的通信采用轻量级的实现，如 Restful、Json、grpc等。
- 独立性：每个服务在开发、测试和部署过程中都是独立的，不受其他服务影响，也不会影响到其他服务。
- 进程隔离：每个微服务都运行在自己独立的进程中，有独立的运行时环境。

基于以上特征，相比于传统的单体架构、垂直架构等，明显有以下优势：

- 开发效率高：每个微服务功能单一，易于理解和方便开发。
- 方便部署：单个微服务的部署不影响全部。
- 新增需求响应快：每个新的服务开发都非常高效，适合敏捷开发。

但是，微服务也是存在缺点的：

- 运维难度增加。
- 分布式部署难度增加。
- 接口修改成本高。
- 部分代码重复。



## 微服务化策略

### 微服务架构风格

有两种：

- 每个服务都拥有独立的数据库：这种方式让整个系统以松耦合的方式进行整合。
- 基于 API 的模块化：通过 API 进行模块化可以避免随着应用的增大而导致内部关系复杂。

### 微服务化进程中的重点问题

- 微服务的通信：使用何种通信方式、服务发现、通信可靠性、如何将业务事件与中间件进行结合、客户端如何与微服务通信。
- 事务管理的一致性：数据库一致性、缓存一致性等中间件的事务一致性问题。
- 微服务数据查询：对于复杂查询可能涉及到多个 API ，可以把需要的 API 依次调用然后聚合结果，也可以使用命令查询职责分离的方式。
- 微服务部署：现在最常用的是单容器单服务模式，一般是一个 docker 容器一个服务。还有一种比较流行的无服务器模式（Serverless），开发者可以专注于应用程序逻辑，而无需关心底层服务器的管理和资源分配，这种模式通常由云服务提供商管理。
- 微服务生产环境监测：监控 API 状态、日志、请求追踪、异常追踪、运行指标。
- 微服务的自动化测试。

### 微服务的拆分

微服务的拆分有两条基本原则：

- 单一职责：微服务职责单一，满足小、高内聚、低耦合的特点。
- 闭包：当需要改变一个微服务的时候，所有以来都在这个微服务的组件内，不需要修改其他微服务。

这两条基本原则是理想情况下的，下面是实际在工程中应用的方式：

- 依据业务能力拆分：将一个系统根据不同的业务场景进行拆分，这种方式严重依赖于架构师的经验和业务能力，但是仍然常用。
- 依据领域驱动设计拆分：领域驱动设计（Domain-Driven Design, DDD）是一种复杂业务系统的设计方法论，通过以业务为核心进行建模和设计，从而实现系统的功能与业务领域的高度契合。有以下基本概念：
  - 领域：指系统所要解决问题的业务范围或业务背景。
  - 子域：将领域进一步划分为更小的业务范围。分为核心子域（核心业务价值的体现）、支撑子域（辅助业务流程）、通用子域（非核心的共性业务，如认证）。
  - 限界上下文：描述一个特定的模型或业务语义的边界，是 DDD 中的核心概念。每个限界上下文内有明确的职责和独立的数据模型。
  - 领域模型：反映领域内的核心业务逻辑的抽象。包括实体（Entity）、值对象（Value Object）、聚合（Aggregate）、领域服务（Domain Service）等。



## 微服务中的进程间通信

本书中的进程间通信使用的是 grpc，介绍请看对应的文章[《gRPC与云原生应用开发》记录 | KurongBlog](https://kurongtohsaka.github.io/posts/dailydev/grpc与云原生应用开发记录/)。

微服务发现用的 consul，和 etcd 一样是以键值存储服务及服务地址。相比于书编写时（2021年），现在（2025年） etcd 用的更多些。



## 微服务中的分布式事务管理

### 微服务下的事务管理

事务管理即维护数据库的 ACID 特性。对于微服务开发，应使用单一存储库原则（SRP），即每个微服务维护自己的数据库，并且任何服务都不应直接访问其他微服务的数据库。实际开发也可以使用折衷方案，把所有微服务的表放在一个数据库内，通过表名进行区分，表于表之间不允许有主外键关系。

### 微服务中处理事务的方式

- 避免跨微服务的事务
- 基于 XA 协议的两阶段提交协议

最终一致性和补偿

### Saga模式

Saga 模式最受欢迎。



## 领域驱动设计的Go实现





## 生产环境的微服务安全





## 日志与监控





## CI/CD





## 使用 go-kit 框架构建微服务

