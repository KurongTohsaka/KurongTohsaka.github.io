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



## Web 服务器

Web 服务器应该做的：

1. 接受客户端连接：客户端收到一条连接之后，那么它将会把新连接添加到现存web服务器连接列表中，用于监视当前连接上的数据传输情况。期间服务器还应该做到通过一定的设备机制阻止未认证或已知恶意黑名客户端的连接。
2. 接收请求报文：
   - 这一步需要解析请求报文：
     - 解析请求行，查找请求方式、URI 、版本号以及 CRLF 分隔符。
     - 读取以 CRLF 结尾的报文首部。
     - 检测到以 CRLF 结尾的，标识首部结尾的空行。
     - 解析得到请求体。
3. 处理请求：其他章节会详解。
4. 对资源的映射及访问：找到客户端请求资源在服务器的上的目录路径。
   - Web 服务器存放内容的文件夹称为文档的根目录（doc root），Web 服务器会从请求报文中获取 URI ，并将其附加在 doc root 的后面。
   - 在一个服务器上挂载多个 Web 站点，那么这样当请求的资源路径相同时，服务器应该从请求报文首部的 HOST 和 URI 字段找出真正的资源目录，这些目录都可以更改配置。
5. 构建响应：
   - 正确设置响应主体的长度（content-length）。
   - 设置报文的 MIME 类型（content-type）。
   - 有时候资源不在原地，需要进行重定向。
6. 发送响应。
7. 记录事务日志。



## 代理

### Web 的中间实体

- Web 代理服务器是代表客户端对事务请求处理的中间人，代理分为私有代理（只代理一个客户端）和公共代理（代理多个客户端）。
- 代理和网关的对比：代理的两端使用相同的协议，而网关的两端使用不同的协议，网关负责协议转换。

### 代理应用

- 内容过滤。
- 文档访问控制。
- 安全防火墙。
- Web 缓存：缓存资源的副本。
- 反向代理：反向代理伪装成原始服务器，不过与服务器不同的是反向代理还可以向其他服务器发送请求，以便实现按需定位所请求的内容。
- …….

### 代理去往何处

根据部署代理的位置，可以把其分为：

- 出口代理：部署在本地网络，保护本地网络或限制带宽。
- 访问代理：将响应缓存起来。
- 反向代理：部署在服务端上，用于实现更精确的请求并提供更优性能。
- 网络交换代理：部署在网络上，用于检测流浪等问题。

### 追踪报文

- 需要一种机制来追踪报文经过了那些节点，此时报文中的 `via` 字段就是一个描述报文在代理中逐级传输的过程中所经过代理的方式，格式如下：

```http
	GET /index.html HTTP/1.0
	Accept: text/html
	Host: www.joes-hardware.com
	Via: 1.1 proxy-62.irenes-isp.net,1.0 cache.joes-hardware.com
```

`via` 字符告诉我们报文流经了两个代理。这个字符串说明第一个代理名为 proxy-62.irenes-isp.net ,它实现了 HTTP/1.1 ，第二个代理被称为 cache.joes-hardware.com, 实现了 HTTP/1.0 。



## 缓存

### 带宽瓶颈与瞬间拥塞

- 很多网络为本地客户端配置的带宽要比远程服务器配置的带宽更大，如果在这种状况下客户端去请求远程服务器，那么客户端将会以一种的较低的速度去请求服务端（受到服务端限制），从而没有发挥出客户端大带宽的长处。如果在客户端方向配置一个高速缓存服务器，那么就可以很快的得到响应。

- 瞬间拥塞：当发生一个爆炸性的新闻和热点事件，如果没有配置缓存，那么在短时间之内服务器会收到大量请求，负荷会爆炸性增长。但是有了缓存，可以大大分担服务器的负载数量。

### HTTP 再验证机制

原始响应内容是在变化的，所以缓存应该在文档过期时间之后去验证缓存的副本是不是新鲜的。这一过程称之为 HTTP 缓存再验证。

- 如果再验证之后得知缓存副本是新鲜的，那么原始服务器返回  `304 not modified` ，此时，称之为再验证命中或缓存慢命中。
- 如果得知缓存不是新鲜的，那么服务器返回 `200 ok` 。此时，称之为再验证未命中。
- 如果原始对象已经被删除了，返回 `404 not found` 响应，相应地缓存副本要删除。

> 虽然再验证命中需要跟原始服务器沟通一次，但是它与直接请求服务器相比还是要快一点，因为再验证命中只是返回了一些新的过期时间有关的新首部，并没有发送主体对象。

### 缓存的处理步骤

- 简单概括，即：

  ```
  接收————解析————查询————新鲜度检测————创建响应————发送————记入日志
  ```

### 保持缓存副本的新鲜度

过期时间通过 HTTP Header 中的 `cache-control` 和 `expire` 字段设置。

新鲜度检测通过以下条件进行验证：

- 两个特殊的 HTTP Header `If-Modified-Since` 和 `If-None-Match` 。前者根据修改时间判断文档新鲜度，后者根据实体标签进行再验证。



## 集成点：网关、隧道及中继

- Web 隧道允许用户通过 HTTP 连接发送非 HTTP 流量，这样就可以在 HTTP 上捎带其他协议数据了。
- 使用 Web 隧道最常见的原因就是要在 HTTP 连接中嵌入 HTTP 流量，然后这类流量就可以穿过只允许 Web 流量通过的防火墙了。
- HTTP 中继是没有完全遵循 HTTP 规范的简单代理。中继负责处理 HTTP 中建立连接的部分，然后对字节进行盲转发。



## 客户端识别与 Cookie 机制

### http 报文中承载用户信息的首部

- From：用户邮件地址。
- User-Agent：用户代理或爬虫标记。
- Referer：从哪个页面跳转而来。
- Authorization：用户的认证信息。
- Client-IP：客户端 IP 。
- X-Forwarded-For：客户端 IP 。
- Cookie：认证信息。

### Cookie

Cookie 分为会话 Cookie 和持久 Cookie ，前者在会话关闭时就会销毁，后者则会保存到存储中。

用户第一次请求服务器时，服务器返回一个带 Set-Cookie（Set-Cookie1）首部的报文，值为键值对，描述了cookie的名字、值、域、路径等信息，然后客户端接下来每次访问服务器的时候都会带上一个有 Cookie 首部的报文，它的值刚好是前面响应报文返回的名字键值对，从而达到验证用户身份的信息。

Cookie 中有以下属性：

- domain：cookie 的域。
- allh：哪些书籍可以使用此 cookie 。
- path：哪些路径可以使用 cookie 。
- secure：是否在发送 https 报文时使用 cookie 。
- expires：过期时间。
- name：名字。
- value：值。

Cookie 不能任意使用，只能在它自己设定的域、主机、路径里面使用。

Cookie 有两个版本：

- 网景公司版本：服务器返回 Set-Cookie 首部，而客户端请求发送 Cookie 首部。客户端发送请求时，会将所有与域、路径和安全过滤器相匹配的未过期 cookie 都发送给这个站点，且所有 cookie 都被组合到一个 Cookie 首部中。
- RFC 2965 版本：这个版本标准引入了 Set-Cookie2 、 Cookie2 首部，但它也能与之前的系统进行互操作。两个版本的主要区别在于，服务器发送的是 Set-Cookie 首部，客户端发送的是 Cookie 首部。
