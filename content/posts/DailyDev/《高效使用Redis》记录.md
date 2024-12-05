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

