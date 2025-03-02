---
title: "《高效使用Redis》记录"
date: 2024-12-05
aliases: ["/Daily Dev"]
tags: ["Redis"]
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

# 《高效使用Redis》记录

## 基础数据结构解析

### 对象

Redis中的 RedisObject 的 c 定义：

```c
#define LRU_BITS 24
typedef struct redisObject {
  unsigned type:4;  // 数据类型
  unsigned encoding:4;  // 底层数据结构
  unsigned lru:LRU_BITS;  // 缓存淘汰时使用
  int refcount;  // 引用计数
  void *ptr;  // 指向实际存储位置
} robj;
```

RedisObject 是 Redis 中数据结构的一个抽象，用它来存储所有的 key-value 数据。

下面是结构体中各个属性的。说明：

- type：用来表示对象类型；

- encoding：表示当前对象底层存储所采用的数据结构，如int、字典、压缩列表、跳跃表等等；

- lru：用于在配置文件中通过 maxmemory-policy 配置已用内存到最大内存限制时的缓存淘汰策略。

  - 以 GET 命令为例，简单看看 Redis 中的 LRU 实现。使用 GET 后，会执行这段代码更新对象的 lru 属性：

    ```c
    if (server.maxmemory_policy & MAXMEMORY_FLAG_LRU) {
      uodateLFU(val);
    } else {
      val->lru = LRU_CLOCK();
    }
    ```

    LRU_CLOCK 用于获取当前时间，但并不是实时获取。Redis 每 1s 执行系统调用来获取精确时间，然后存储在全局变量 server.lruclock 中，LRU_CLOCK 只是获取该缓存时间。updateLFU 用于更新对象的上次访问时间和访问次数。

- refcount：存储当前对象的引用次数，用于内存回收；

- ptr：指向实际存储的某一种数据结构。然而，当 robj 存储的数据类型可以用 long 类型表示，数据会直接存储在 ptr 中。

### 字符串

简单动态字符串（sds）是 Redis 的基本数据结构，用于存储字符串和整型数据。

首先是 sds 的一个简单设计：

```c
struct sds {
  int len;
  int free;
  char buf[];
}
```

在 Redis 3.2 之前，sds 就是这样设计的：

- 内容存储在柔性数组 buf 中，因为柔性数组的地址和结构体是连续的，所以一是可以满足随机存取的要求，二是可以直接通过首地址偏移得到结构体的首地址，进而获取其他变量；
- 由于有 len ，在读写字符串时不依赖“\\0”终止符，保证了二进制安全。

但是这种存储方式内存利用率不高，所以 Redis 按照长度把字符串分为 5 种类型：sdshdr5、sdshdr8、sdshdr16、sdshdr32、sdshdr64 ，会用 1 个字节的低 3 位标识字符串类型，高 5 位用于存储字符串长度。以 sdshdr5 为例：

- 结构体：

  ```c
  struct __attribute__ ((__packed__))sdshdr5 {
    unsigned char flags;  // 低 3 位标识字符串类型，高 5 位用于存储字符串长度
    char buf[];  // 柔性数组，存放实际内容 
  }
  ```

对于长度大于 31B 的字符串，Redis 会把 len 和 free 单独存放，所以除了 sdshdr5 之外的字符串类型结构相同。以 sdshdr16 为例：

```c
struct __attribute__ ((__packed__))sdshdr16 {
  uint16_t len;  // 2B 存储 buf 中已占用字节数
  uint16_t alloc;  // 2B 存储 buf 已分配字节数
  unsigned char flags;
  char buf[];
}
```

### 列表

Redis 列表对象的底层数据结构是 quicklist。在 Redis 3.2 之前 Redis 用 ziplist（压缩链表） 及 adlist（双向链表） 作为 list 的底层实现。当元素较少且元素长度较小时，Redis 会用 ziplist ，其他条件会用 adlist。而 quicklist 由 list 和 ziplist 结合而成。

- list：Redis 用的双向非循环链表，3.2 之后加入了表头。

- ziplist：ziplist 是一个字节数组，是为了节约内存而设计的数据结构。它的基本结构：zlbytes、zltail、zllen、entry1…、zlend。

  - zlbytes：列表字节长度，4B；
  - zltail：表尾元素相对于起始地址的偏移量，4B；
  - zlen：元素数目，2B；
  - entry：元素，可以是字节数组或整数；
  - zlend：结尾，1B。

  entry 的编码结构：previous_entry_length、encoding、content：

  - previous_entry_length：表示前一个元素的长度。元素长度小于 254B 时占 1B ，否则占 5B；
  - encoding：当前元素的编码类型，是可变的；
  - content：内容。

  ziplist 在两个元素之间插入新元素时会有连锁更新现象，效率很低。

- quicklist：是一个由 ziplist 作为节点的双向链表。

  结构体定义：

  ```c
  typedef struct quicklist {
    quicklistNode *head;
    quicklistnode *tail;
    unsigned long count;
    unsigned long len;
    int fill:16;  // 指明每个节点的 ziplist 长度
    unsigned int compress:16;  // 节点压缩深度
    // ...
  } quicklist;
  ```

  节点结构体定义：

  ```c
  typedef struct quicklistNode {
    struct quicklistNode *prev;
    struct quicklistNode *next;
    unsigned char *zl;  // 该节点对应的 ziplist
    unsigned int sz;  // 整个 ziplist 的大小
    unsigned int count:16; 
    unsigned int encoding:2;  // 表示是否采用 LZF 压缩算法
    unsigned int container:2;  // 为 quicklistNode 节点 zl 指向的容器类型：1为 none，2为使用 ziplist
    unsigned int recompress:1;  // 用于表示该节点之前是否被压缩过
  } quicklistNode;
  ```

  当使用 LZF 算法压缩时，quicklistNode 节点指向的结构是 quciklistLZF：

  ```c
  typedef struct quicklistLZF {
    unsigned int sz;  // 被 LZF 算法压缩后的 ziplist 大小
    char compressed[];  // 保存压缩后的数组
  } quicklistLZF;
  ```

  关于 LZF 算法内容，略。

### 字典

Redis 的字典是通过 Hash 来实现的。

结构体定义：

```c
typedef struct dictht {
  dictEntry **table;  // 指针数组，存储键值对
  unsigned long size;  // table 数组的大小
  unsigned long sizemask;  // 恒定值，掩码 = size - 1
  unsigned long used;  // 已存储的元素个数 
}
```

sizemask 属性用来计算 key 的索引值，值恒为 size - 1。

Hash 元素结构体定义：

```c
typedef struct dictEntry {
  void *key;
  union {
    void *val;  // db.dict 中的 val
    uint64_t u64;
    int64_t s64;  // db.expires 中存储了过期时间
    double d;
  } v;
  struct dictEntry *next;  // Hash 冲突时指向冲突的元素
} dictEntry;
```

dict 结构体定义：

```c
typedef struct dict {
  dictType *type;  // 该字典对应的特定操作函数
  void *privdata;  // 该字典所依赖的数据
  dictht ht[2];  // Hash 表
  long rehasidx;  // rehash 标志，默认值为 -1，表示不进行rehash操作
  int16_t pauserehash;  // 当前运行的安全迭代器数
}
```

关于字典的扩容：

1. 申请一块新内存，初次申请默认大小为4个 dictEntry，非首次申请的大小则为当前 Hash 表容量的 1 倍；
2. 把新申请的内存地址赋值给 ht[1] ，并把字典的 rehashidx 标志位改为 0，表示之后需要进行 rehash 操作。

### 集合

Redis 的集合底层基于 dict 和 intset 实现，当集合类型的元素都处于 64 位有符号整数范围之内时用 intset 存储，其他情况用 dict 存储。

intset 是一个有序的、存储整型数据的结构，结构体定义如下：

```c
typedef struct intset {
  uint32_t encoding;  // 编码类型，决定每个元素占用几个字节
  uint32_t length;  // 元素个数
  int8_t contents[];  // 柔性数组
} intset;
```

intset 会根据待插入的值决定是否需要进行扩容，扩容会修改 encoding 属性的值，进而需要扩展 contents 中原来的元素。可以得到集合的一个简单结论，只要引起了扩容，这个待插入值一定是最值。

### 有序集合

Redis 的有序集合没有使用平衡树或红黑树实现，而是跳跃表。跳跃表的效率与红黑树相似，但实现要简单很多。

跳跃表是一种有序链表的升级版，通过额外的“跳跃层”提高查找效率，接近于二分查找的性能。它的基本结构是：

- 跳跃表由多层链表组成，底层是一个普通的有序链表（包含所有数据）。
- 上层链表是底层链表的“索引”，每一层跳跃表包含一部分关键节点，链接在更高层级。
- 每层的节点数量逐级减少，形成一种“金字塔”结构，层数越高一般节点越少。

关于层数的生成，每个节点通过随机算法决定是否出现在上一层。这个随机性使跳跃表的性能接近二分查找。

跳跃表从顶层开始向右查找，直到超过目标值（或者到达链表尾部）。如果当前层找不到，就下移一层继续，直到到底层链表。

查找、插入、删除的平均时间复杂度是 $O(log⁡n)$ 。

它具有以下的优势：

- 动态性好：插入和删除时只需局部调整，不像平衡二叉树需要全局旋转或调整。

- 实现简单：不需要复杂的树平衡操作，代码更易维护。

- 空间效率高：相比平衡树，跳跃表额外的索引层占用较小空间。

跳跃表的结构体定义：
```c
typedef struct zskiplistNode {
  sds ele;  // 存储字符串类型数据
  double score;  // 存储用于排序的分值
  struct zskiplistNode *backward;  // 回溯指针，指向当前节点最底层的前一个节点。从后向前遍历时使用
  struct zskiplistLevel {
    struct zskiplistNode *forward;  // 指向本层下一个节点
    unsigned int span;  // forward 指针指向的节点与本节点之间的元素个数，值越大，跳过的节点个数越多
  } level[];  // level 数组，柔性数组
} zskiplistNode;

// 表头
typedef struct zskiplist {
  struct zskiplistNode *header, *tail;
  unsigned long length;  // 跳跃表长度
  int level;  // 跳跃表的层高
}
```

zskiplist 的 header 指针指向跳跃表的 head 节点。head 节点的 level 数组元素个数为 32，head 节点不存储任何 member、score值，ele 值为 NULL，score 值为 0 ，也不计入总长度的计算。



## stream 简介

Redis 的 Stream 是一种高效的、功能强大的数据结构，适用于消息队列、日志管理和实时数据处理等场景。它在 Redis 5.0 中引入，提供了一种基于时间序列的数据模型，类似于 Kafka 等消息队列系统，但具有 Redis 的高性能和易用性。

stream 的基本特性：

- 日志结构：

  - Stream 是按时间排序的日志记录，每条记录包含一个唯一的 ID 和对应的数据。

  - ID 通常是自动生成的时间戳（形如 `1669872150488-0`），包含毫秒时间戳和序号。

- 消费者组：
  - 支持消费者组 (Consumer Groups)，允许多个消费者分摊处理消息，提高并行处理能力。
  - 消息在消费者组中只会被一个消费者消费，避免重复处理。
- 轻量级：
  - Stream 是内置的数据类型，无需额外配置，易于上手。

Redis 的消息队列功能在一些场景中被广泛使用，特别是在轻量级、低延迟的实时消息处理场景中。相比专业的消息队列中间件（如 RabbitMQ），Redis 的消息队列有其独特的优势和局限性。

stream 的优势

1. 高性能、低延迟：
   - Redis 是内存数据库，消息的发布和消费速度非常快，特别适合对延迟敏感的场景。
2. 简化架构：
   - Redis 集成了多种功能（如缓存、存储、队列等），不需要引入额外的消息队列中间件，降低了运维和管理成本。
3. 简单易用：
   - 通过 PUBLISH/SUBSCRIBE 或 Stream 数据结构即可实现消息队列功能，开发和配置简单。
4. 轻量级任务队列：
   - 对于简单任务分发或轻量级的数据流处理，Redis 是一个理想的选择。



## Redis 启动流程与事件驱动模型

### server 启动过程

1. 初始化配置，包含用户可配置的参数以及命令表的初始化，入口函数为 initServerConfig；
2. 加载并解析配置文件，入口函数为 loadServerConfig；
3. 初始化服务端内部变量，入口函数 initServer；
4. 创建事件循环 eventLoop，入口为 aeEventLoop；
5. 创建 socket 并启动监听，入口为 listenToPort；
6. 创建文件事件与时间事件；
7. 开启事件循环。

### 事件处理

事件都封装在 aeEventLoop 中，它的定义如下：

```c
typedef struct aeEventLoop {
  int stop;  // 标志事件结束
  aeFileEvent *events;  // 文件事件数组
  aeFiredEvent *fired;  // 已触发事件数组
  aeTimeEvent *timeEventHead;  // 时间事件链表头节点
  void *apidata;  // 多路复用的底层实现相关数据
  aeBeforeSleepProc *beforesleep;  // 事件循环前执行的钩子函数
  aeBeforeSleepProc *aftersleep;  // 事件循环后执行的钩子函数
} aeEventLoop;
```

Redis 的事件驱动模型是基于事件循环的，它将所有任务分为两类：文件事件（File Event）和时间事件（Time Event），并通过事件循环依次处理这些事件。

Redis 的事件循环通过函数 aeMain 实现，大致流程如下：

1. 检查是否需要退出：通过 stop 标志判断是否需要退出事件循环；
2. 处理已触发的文件事件：
   - 调用多路复用器（如 epoll_wait）等待文件事件触发；
   - 收集触发的事件，执行相应的回调函数。
3. 处理到期的时间事件：
   - 遍历时间事件链表，检查当前时间是否已达到某个事件的触发时间；
   - 如果是，则执行该时间事件的回调函数。
4. 执行钩子函数：
   - 在进入或退出休眠状态时，执行 beforesleep 和 aftersleep 钩子函数，完成一些特定的任务（如清理数据或更新状态）。
5. 重复上述步骤，直到事件循环终止。

### I/O 多路复用机制

Redis 支持以下几种 I/O 多路复用技术：

- epoll（linux）：

  - 性能最优，适合大规模并发连接。

  - 采用事件通知方式，避免了 select 的轮询开销。

- kqueue（macOS）：

  - 类似于 epoll，性能优越。

- select（跨平台）：

  - 简单但性能差，适合小规模连接。

- poll（linux、unix）：

  - 比 select 稍好，但性能较差。

Redis 使用抽象层封装了多路复用的具体实现，核心接口包括：

- **`aeApiCreate`**：创建多路复用上下文。
- **`aeApiAddEvent`**：注册文件描述符及其感兴趣的事件类型（读/写）。
- **`aeApiPoll`**：阻塞等待事件触发，并将触发的事件写入 fired 数组。
- **`aeApiDelEvent`**：注销文件描述符上的某个事件类型。
- **`aeApiFree`**：释放多路复用器资源。

### 文件事件

在 Redis 的事件循环机制中，"文件事件（File Event）" 是对可读写的文件描述符（File Descriptor, FD）进行异步处理的核心手段。虽然名为“文件事件”，但实际应用中，它更多用于处理网络套接字（如客户端连接的 TCP Socket）的读写，这与传统的 UNIX 网络编程中“文件描述符既可代表文件又可代表套接字”的概念相呼应。Redis 基本上是通过文件事件来实现非阻塞、单线程的网络 I/O 处理，从而在不依赖多线程的情况下实现高吞吐和低延迟。

Redis 单线程运行中，为了同时处理众多客户端请求，需要在不阻塞主线程的前提下对 I/O 操作（如读写套接字）进行管理。文件事件模型就是为了解决这个问题而设计的。通过将每一个套接字（客户端连接）注册到事件循环中，当该套接字可读或可写时，事件循环会获悉并调用相应的回调函数进行处理。文件事件基于操作系统的多路复用机制（select、epoll、kqueue），以非阻塞方式高效地管理多个 I/O 通道。

aeFileEvent 是文件事件的核心结构，它的结构体定义：

```c
typedef struct aeFileEvent {
    int mask;                 // 事件类型掩码，可以是 AE_READABLE 或 AE_WRITABLE 或两者的组合
    aeFileProc *rfileProc;    // 可读事件的回调函数
    aeFileProc *wfileProc;    // 可写事件的回调函数
    void *clientData;         // 用户数据，可以传递给回调函数
} aeFileEvent;
```

文件事件有几个实现的关键点：

- 事件注册与管理：
  - aeEventLoop 的 events 属性中，每个 FD 对应一个 aeFileEvent ；
  - 注册文件事件时，调用 aeCreateFileEvent 将 FD 与事件类型绑定，同时更新底层多路复用器的监听列表。
- 事件监听与触发：
  - Redis 使用多路复用器监听所有注册的 FD 的状态变化，多路复用器由 aeApiPoll 实现；
  - 多路复用器会在事件发生时返回触发的 FD 和事件类型。
- 事件分发与处理：
  - Redis 将触发的文件事件存入 aeEventLoop 的 fired 数组；
  - 遍历 fired 数组，根据 FD 查找对应的 aeFileEvent 并执行绑定的回调函数。

一个较完整的文件事件的流程：

1. 初始化事件循环 aeEventLoop ；
2. 用 aeCreateFileEvent 注册文件事件；
3. 事件循环：
   - Redis 使用 aeProcessEvents 处理事件：
     1. 等待事件发生：调用 aeApiPoll 阻塞等待 FD 的状态变化；
     2. 收集触发事件：将触发的事件放入 fired 数组；
     3. 执行回调函数：遍历 fired 数组，调用回调函数。
4. 注销文件事件：当 FD 不再需要被监听了，调用aeDeleteFileEvent 注销事件并更新多路复用器。

### 时间事件

时间事件是基于定时触发的回调机制，用于在特定时间点或周期性地执行特定任务。例如，Redis 中的一些周期性操作（如服务器内部维护、过期键清理、统计信息更新）都是通过时间事件来实现的。

Redis 使用一个单向链表管理所有注册的时间事件，头指针就是 aeEventLoop 中的 timeEventHead 属性。时间事件的核心结构是 aeTimeEvent：

```c
typedef struct aeTimeEvent {
    long long id;  // 时间事件的唯一标识符
    long whenSec;  // 事件触发的时间（秒级）
    long whenMs;   // 事件触发的时间（毫秒级，与 whenSec 合用可表示一个精确时间点）
    aeTimeProc *timeProc;  // 时间事件的回调函数
    aeEventFinalizerProc *finalizerProc;  // 可选的清理回调函数，当时间事件删除时调用
    struct aeTimeEvent *next;  // 下一个时间事件的指针
} aeTimeEvent;

```

时间事件的触发与处理流程：

1. 事件检查与触发时机：

   时间事件并非由操作系统机制直接触发，而是由 Redis 在事件循环每一轮迭代（aeProcessEvents 调用）中自行检查当前时间与事件的触发时间是否匹配：

   - Redis 在执行 aeProcessEvents 时，先处理文件事件，然后调用 aeProcessTimeEvents 遍历 timeEventHead 链表；
   - 对于每个时间事件，Redis 获取当前系统时间并与 whenSec、whenMs 对比。如果当前时间已达或超过事件设定的触发时间，则执行 timeProc 回调函数。

2. 执行回调函数与事件重设：

   当执行 timeProc 时：

   - 若返回 -1，该时间事件时一次性执行，执行结束后立即删除该事件；
   - 若返回整数（毫秒数），则 Redis 会在当前执行时间基础上增加该毫秒作为下次触发时间，这样就实现了周期性执行的逻辑。

3. 事件清理：如果事件执行结束或调用 aeDeleteTimeEvent 删除，Redis 会调用 finalizerProc 进行清理。



## Redis 多线程模型

### I/O 多线程

在 Redis 6.0 之前，读取客户端命令、执行命令以及向客户端返回结果都是在主线程完成的。Redis 执行命令速度非常快，但是另外两个操作，也就是网络 I/O 就成了性能瓶颈。所以 6.0 之后，Redis 加入了多线程 I/O 。

Redis 的 I/O 多线程并不是让整个 Redis 服务都运行在多线程模式下，而是专注于网络 I/O（例如读取客户端请求和发送响应）。其他任务（如命令执行、键空间操作等）仍然由主线程完成。

具体过程：

1. 主线程监听网络事件：
   - 主线程接收客户端连接和网络事件，分配这些事件给线程池中的 I/O 线程。
2. I/O 线程并行处理网络数据：
   - 每个线程处理一组客户端连接，负责从 socket 中读取数据或将数据写回 socket。
3. 主线程处理命令逻辑：
   - 当 I/O 线程完成数据的读取后，主线程解析命令并执行。
   - 结果由主线程统一交给 I/O 线程写回客户端。

### I/O 线程管理

Redis 通过配置文件和启动参数管理 I/O 线程。默认情况下，Redis 的多线程支持是关闭的，且线程数量设置为 1（即单线程模式）。

配置参数：

1. io-threads：

   - 用于设置 Redis 的 I/O 线程数量。
   - 值必须大于等于 1（默认值为 1 表示单线程）。

   示例：

   ```
   io-threads 4
   ```

2. io-threads-do-reads：

   - 控制是否使用多线程处理读取操作。
   - 值为 yes 时，启用多线程读取；no 时，读取操作仍由主线程完成（默认值为 no）。

   示例：

   ```
   io-threads-do-reads yes
   ```

线程池管理：

- Redis 的线程池在启动时创建，线程数量固定。
- Redis 不会动态调整线程池的大小，因此需要根据工作负载合理配置线程数量。

### I/O 线程同步

Redis 的 I/O 线程同步主要体现在两个阶段：

1. I/O 线程之间的同步（多线程并行读取/写入时的安全）。
2. I/O 线程和主线程的同步（确保主线程正确处理数据）。

关于 I/O 线程之间的同步，Redis 采用任务分片和线程本地数据隔离的方式：

- 任务分片：每个 I/O 线程负责一组客户端连接，客户端连接的分配由主线程完成，各线程的任务互不重叠。
- 线程本地缓冲区：Redis 为每个 I/O 线程分配自己的输入/输出缓冲区，线程在处理网络 I/O 时，只会操作属于自己的缓冲区，不会访问其他线程的数据。
- 分阶段处理：所有线程在同一个阶段完成任务，等到所有线程完成后，才进入下一阶段（如命令解析）。

而 I/O 线程与主线程之间的同步是在两个节点上进行的：

- I/O 线程完成数据读取后，通知主线程解析和执行命令。
- 主线程执行完命令后，通知 I/O 线程将结果写回客户端。

为此，Redis 使用了以下机制：

- 主线程全局控制权：主线程是唯一负责命令解析和执行的线程。
- 分阶段锁：主线程会等待所有 I/O 线程完成当前阶段的任务（如读取或写入）后，才进入下一阶段。阶段结束的标志由主线程统一检查。
- 轻量级同步标志：Redis 使用轻量级的同步机制（如原子操作和状态标志）协调主线程和 I/O 线程的状态，而不是引入复杂的锁机制。

Redis 在线程同步中会使用以下几种锁机制：

1. 互斥锁（Mutex）

​	Redis 使用操作系统提供的互斥锁（pthread_mutex）来保护关键资源。这种锁是重量级锁，只有在不可避免的情况下才会使用。

- 使用场景：
  - 保护客户端连接列表的修改。
  - 保护任务队列的操作。
  - 确保线程安全的统计信息更新。
- 优缺点：
  - 优点：简单可靠，适用于临界区较少的场景。
  - 缺点：加锁和解锁需要上下文切换，可能增加线程之间的等待时间。

2. 自旋锁（Spinlock）

​	Redis 更倾向于使用自旋锁（pthread_spinlock）来减少线程等待的开销。在自旋锁中，线程不会进入睡眠状态，而是不断轮询锁的状态，直到获取锁为止。

- 使用场景：
  - 临界区非常小且操作耗时短的场景，例如更新统计信息或同步标志。
  - 自旋锁适用于高并发、锁持有时间短的情况，可以避免线程切换的开销。
- 优缺点：
  - 优点：高效，适合临界区操作非常快的场景。
  - 缺点：如果锁被长时间持有，会浪费 CPU 资源。

3. 原子操作（Atomic Operation）

​	Redis 使用原子操作（例如 __sync_fetch_and_add 或 stdatomic.h 提供的接口）来同步一些简单的共享数据，例如计数器或标志位。

- 使用场景：
  - 更新 I/O 线程的完成计数。
  - 统计信息（如处理的命令数量）的更新。
- 优缺点：
  - 优点：无需显式加锁，性能极高。
  - 缺点：只适用于简单的原子性操作，无法保护复杂的共享资源。



## 持久化

（这本书中关于持久化的内容全是围绕源码讲的，对于现阶段没有帮助，所以此章内容为参考网络资料整理而来。）

### RDB

RDB（Redis Database） 是通过在特定时间间隔或满足特定触发条件时，将内存中的数据集状态“快照”（Snapshot）地保存到磁盘文件（一般为 dump.rdb）中。该文件使用二进制格式，包含 Redis 数据库在某个时间点上的所有数据状态。

Redis 的 RDB 持久化通常通过两种方式触发：

1. 后台定时保存（自动快照）：在 `redis.conf` 中，通过配置 `save` 命令设置保存条件。例如：

   ```
   save 900 1    # 900秒（15分钟）内如果至少有1个key被修改就触发一次RDB保存
   save 300 10   # 300秒（5分钟）内如果至少有10个key被修改就触发一次RDB保存
   save 60 10000 # 60秒（1分钟）内如果至少有10000个key被修改就触发一次RDB保存
   ```

   当满足任意一个条件时，Redis 会触发一次 RDB 快照。

2. 手动触发：可以在客户端执行 `BGSAVE` 命令（推荐）或 `SAVE` 命令。

   - `BGSAVE` 会在后台子进程中创建一个快照，这个过程中并不会阻塞主线程的命令处理，因此对服务影响较小。
   - `SAVE` 则是在当前进程中执行快照生成，在保存期间阻塞所有客户端请求，一般不推荐在线上环境使用。

RDB 持久化的特点

- 优点：
  - 文件紧凑：RDB 文件是二进制压缩格式，体积较小，便于传输和备份。
  - 恢复快速：RDB 文件包含某个时刻的全量数据快照，重启时直接加载就能快速恢复到该时刻的状态。
  - 性能影响相对较小：在使用 `BGSAVE` 时，快照工作由子进程完成，主进程依旧可以响应请求，整体对服务性能影响相对可控。
- 缺点：
  - 潜在数据丢失：因为 RDB 不是实时保存数据，而是以一定周期或条件触发，那么在上一次 RDB 之后的修改操作若 Redis 意外崩溃，这些修改将丢失。
  - 高并发下的开销：在进行 RDB 快照时需要 fork 子进程，复制内存页，若数据量很大，可能造成一定的性能抖动。

### AOF

AOF 模式通过记录 每一次对 Redis 数据进行修改的命令 来实现持久化。它会将 Redis 的写操作（如 `SET`、`HSET`、`INCR` 等）以文本协议的形式追加到 AOF 文件（默认 `appendonly.aof`）末尾。

AOF 的写策略可以通过 `appendfsync` 参数控制：

- `appendfsync always`：每次有写操作都同步（fsync）到磁盘，安全性最高但性能开销最大。
- `appendfsync everysec`：每秒强制进行一次磁盘同步，是默认策略，也是性能和安全的较好平衡点。在极端崩溃情况下，最多丢失1秒的数据。
- `appendfsync no`：不主动调用 fsync 由操作系统决定磁盘刷新时间，性能最好但可能丢失更多数据。

由于 AOF 通过不断追加命令记录，随着时间推移文件会越来越大。为解决文件过大的问题，Redis 提供 AOF 重写机制。当 AOF 文件达到一定大小门限（可配置），Redis 会自动触发 AOF 重写（`BGREWRITEAOF`），过程如下：

1. Redis fork 出一个子进程。
2. 子进程根据当前内存数据状态生成一份最精简的命令集，将这些命令写入一个新的 AOF 临时文件。
3. 在此过程中，主进程仍在接受新写入操作，这些操作会同时被写入内存缓冲区，并在子进程完成重写后，追加到新的 AOF 文件末尾。
4. 当子进程完成重写后，用新的 AOF 文件替换旧文件，从而实现 AOF 文件的压缩精简。

通过 AOF 重写机制，可以保证 AOF 文件不会无限增长。

AOF 的特点

- 优点：
  - 数据安全性高：AOF 可以在更高频率（如每秒甚至每次写入）下同步数据到磁盘，从而最大限度地减少数据丢失。
  - 文件易读性：AOF 是文本协议格式，打开文件就能看到具体的 Redis 命令，有利于故障排查。
- 缺点：
  - 文件可能较大：相较于 RDB，AOF 的文件会因为不断记录写操作而变得更大。不过可以通过重写来缓解。
  - 恢复速度稍慢：在恢复数据时，Redis 需要从头按顺序执行 AOF 文件中的所有写操作命令，处理耗时可能比加载 RDB 文件长（不过通过 AOF 重写可有效缩短恢复时间）。

### 混合持久化

Redis 4.0 及以上版本引入了混合持久化的机制，即在进行 AOF 重写时，可以将当前内存数据的 RDB 格式和后续增量的命令相结合，从而在恢复时既能快速加载 RDB 部分，又能保证数据的完整性。Redis 5.0 开始默认开启混合持久化。

混合持久化在进行 AOF 重写时，先写入一段 RDB 格式的二进制数据（表示当前快照状态），然后在此之后附加增量的 AOF 命令部分。这样恢复数据时先读 RDB 部分（极快恢复大部分数据），然后再应用后面的 AOF 命令（补齐最新写入的数据），实现恢复性能与数据完整性间的平衡。

