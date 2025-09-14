---
title: "golang单元测试与冒烟测试入门"
date: 2025-09-14
aliases: ["/Daily Dev"]
tags: ["Golang", "Unit Testing", "Mock Testing"]
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

## 概念解析：单元测试 vs. 冒烟测试

### 1. 单元测试 (Unit Testing)

**单元测试**是针对程序中最小的可测试单元进行的测试。在 Go 语言中，这个“单元”通常指一个函数（Function）或一个方法（Method）。

- **核心目标**：验证单个函数或方法的逻辑是否正确。它确保给定特定的输入，该函数能返回预期的输出。
- **关键原则**：**隔离**。单元测试应该与其依赖项（如数据库、文件系统、外部API等）隔离开来。如果一个函数依赖了这些外部服务，我们通常会使用“模拟（Mock）”技术来创建一个假的依赖项，从而保证测试的稳定性和速度。
- **特点**：
  - **速度快**：因为不涉及外部I/O操作，通常毫秒级就能完成。
  - **粒度细**：精确到每个函数，能够快速定位问题。
  - **编写频繁**：是开发过程中最常编写的测试类型。

### 2. 冒烟测试 (Smoke Testing)

**冒烟测试**，又称“构建验证测试（Build Verification Test）”，是一种非常基础的测试，用于确定软件的关键功能是否能够正常工作。它像是在给一个新设备通电后，看看它是否会冒烟——如果不冒烟，说明最基本的功能没问题，可以进行更详细的测试。

- **核心目标**：验证应用程序的核心、关键流程是否能够跑通，确保程序在主要路径上不会崩溃。
- **关键原则**：**快速、宽泛、但不深入**。它不关心业务逻辑的细枝末节，只关心“主干”是否健康。
- **特点**：
  - **速度较快**：比完整的端到端测试快，但比单元测试慢。
  - **覆盖面广**：覆盖的是最关键的业务流程。
  - **执行时机**：通常在新版本构建完成后、进行全面测试之前执行。对于后端服务来说，可能是在服务启动后，立即调用几个核心API（如健康检查`/health`，登录接口等）看看是否返回200 OK。



## Go 中的测试库

### 标准库

Go 语言内置了强大的测试支持，主要通过 `testing` 包来实现。这是所有 Go 测试的基础，你必须首先掌握它。

- `testing`: 提供了编写测试的基本框架，例如 `*testing.T` 类型用于报告测试状态。
- `net/http/httptest`: 专门用于测试 HTTP 客户端和服务器，无需真正监听端口即可模拟 HTTP 请求和响应，非常适合后端 API 测试。

### 流行的第三方库

虽然标准库很强大，但在某些方面，社区提供了更高效、更易读的工具。

- **`stretchr/testify`**: 这是 Go 社区中最受欢迎的断言库。它提供了丰富的断言函数（如 `assert.Equal()`, `assert.NoError()`），让你的测试代码更简洁、更具可读性。
- **`golang/mock` (Gomock)**: Google 官方开发的 Mock 框架。当你需要隔离测试单元与它的依赖时，Gomock 可以根据接口自动生成模拟对象，让你完全控制依赖项的行为。
- **`h2non/gock`**: 如果你的代码需要调用外部 HTTP API，这个库可以轻松地模拟这些外部请求，让你的测试不依赖于网络和第三方服务的稳定性。

------

## 从单元测试到冒烟测试入门

### Part 1: 什么是单元测试？为什么需要它？

想象一下你正在组装一台复杂的机器，比如一台汽车。单元测试就像是分别检查每一个零件（螺丝、齿轮、活塞）是否合格。只有确保每个最小零件都工作正常，你才能对最终组装好的汽车有信心。

在 Go 中，这个“零件”就是你的**函数**或**方法**。

**单元测试（Unit Testing）** 的核心思想是：

> 验证程序中最小的可测试单元（一个函数）在给定输入时，能否产生预期输出。

它最重要的原则是**隔离**。测试一个函数时，我们不希望受到数据库、网络请求或文件系统的干扰。这保证了测试的**速度**和**稳定性**。

### Part 2: 如何编写你的第一个 Go 单元测试

Go 的测试生态非常原生和简洁，你只需要遵循几个简单规则：

1. 测试文件必须以 `_test.go` 结尾（例如 `math.go` 的测试文件是 `math_test.go`）。
2. 测试函数必须以 `Test` 开头，并接收一个 `*testing.T` 类型的参数（例如 `func TestAdd(t *testing.T)`）。
3. 测试文件和源文件通常放在同一个包（package）下。

#### 示例：使用标准库 `testing`

假设我们有一个简单的加法函数：

**`math.go`**

```go
package main

// Add returns the sum of two integers.
func Add(a, b int) int {
	return a + b
}
```

现在，我们为它编写一个单元测试：

**`math_test.go`**

```go
package main

import "testing"

// TestAdd is a unit test for the Add function.
func TestAdd(t *testing.T) {
	// Arrange: set up the test data
	a := 2
	b := 3
	expected := 5

	// Act: call the function being tested
	result := Add(a, b)

	// Assert: check if the result is what we expect
	if result != expected {
		// t.Errorf will report an error but continue the test.
		// t.Fatalf will report an error and stop the test immediately.
		t.Errorf("Add(%d, %d) = %d; want %d", a, b, result, expected)
	}
}
```

在终端中运行 `go test` 命令，Go 会自动找到并执行这个测试。

### Part 3: 让测试更优雅 - `stretchr/testify`

虽然标准库够用，但 `if result != expected` 这样的断言写多了会有些繁琐。`stretchr/testify` 库提供了更丰富的语义化断言，让代码意图一目了然。

安装 `testify`：

```bash
go get github.com/stretchr/testify
```

用 `testify/assert` 重写上面的测试：

**`math_test.go` (testify version)**

```go
package main

import (
	"testing"
	"github.com/stretchr/testify/assert"
)

func TestAddWithAssert(t *testing.T) {
	// assert.Equal(t, expected, actual, "error message")
	assert.Equal(t, 5, Add(2, 3), "2 + 3 should be 5")
	assert.NotEqual(t, 6, Add(2, 3), "2 + 3 should not be 6")
}
```

是不是清晰多了？`assert` 包提供了海量的断言函数，强烈推荐你在项目中使用。

### Part 4: 什么是冒烟测试？它和单元测试有何不同？

如果说单元测试是检查汽车的每个零件，那么**冒烟测试（Smoke Testing）** 就是在汽车组装好后，拧动钥匙，看看引擎是否能启动、车灯是否会亮。它不关心引擎内部的复杂运作，只关心**最核心的功能是否正常**。

对于一个后端服务来说，冒烟测试通常覆盖以下场景：

- 服务能否成功启动？
- 能否连接到数据库？
- 健康检查接口（如 `/health`）是否返回 200 OK？
- 一个最关键的业务接口（如用户登录）是否能走通？

它的目的是**快速失败**：如果连冒烟测试都通不过，说明新构建的版本有严重问题，应立即打回，无需投入更多资源进行详细测试。

### Part 5: 如何编写一个后端的冒烟测试

对于 Web 后端，`net/http/httptest` 标准库是编写冒烟测试（或任何 HTTP 测试）的利器。它能让你在不监听真实端口的情况下测试 `http.Handler`。

#### 示例：测试一个健康检查接口

假设我们有这样一个 HTTP Handler：

**`server.go`**

```go
package main

import (
	"fmt"
	"net/http"
)

// HealthCheckHandler responds with a simple health status.
func HealthCheckHandler(w http.ResponseWriter, r *http.Request) {
	w.WriteHeader(http.StatusOK)
	w.Header().Set("Content-Type", "application/json")
	fmt.Fprintln(w, `{"status": "ok"}`)
}

// NewRouter creates a new router and registers the handlers.
func NewRouter() http.Handler {
    mux := http.NewServeMux()
    mux.HandleFunc("/health", HealthCheckHandler)
    return mux
}
```

我们的冒烟测试就是要确保 `/health` 接口能返回 `200 OK`。

**`server_test.go`**

```go
package main

import (
	"net/http"
	"net/http/httptest"
	"testing"

	"github.com/stretchr/testify/assert"
)

// TestSmokeHealthCheck is a smoke test for the health check endpoint.
func TestSmokeHealthCheck(t *testing.T) {
	// Arrange: create a new router and a test request
	router := NewRouter()
	req, err := http.NewRequest("GET", "/health", nil)
	assert.NoError(t, err, "Failed to create request")

	// Create a ResponseRecorder to record the response.
	rr := httptest.NewRecorder()

	// Act: serve the HTTP request to our recorder
	router.ServeHTTP(rr, req)

	// Assert: check the status code and response body
	assert.Equal(t, http.StatusOK, rr.Code, "Handler should return status 200 OK")
	assert.JSONEq(t, `{"status": "ok"}`, rr.Body.String(), "Response body should be correct")
}
```

这个测试启动了我们的路由，模拟了一次对 `/health` 的真实HTTP请求，并检查了响应码和响应体。它验证了“HTTP服务启动并能正确响应核心接口”这一关键链路，是一个典型的冒烟测试。
