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

- 关于切片，切片就是没有长度的数组。
