---
title: "《Go微服务实战》记录"
date: 2025-01-18
aliases: ["/Daily De"]
tags: ["Golang", "微服务"]
categories: ["Daily De"]
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

- 协程（goroutine）是 Go 中的轻量级线程，使用 `go` 关键字启动。见代码：

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





## Go Web编程





## 微服务



## 微服务化策略





## 微服务中的进程间通信





## 微服务中的分布式事务管理





## 领域驱动设计的Go实现





## 生产环境的微服务安全





## 日志与监控





## CI/CD





## 使用 go-kit 框架构建微服务

