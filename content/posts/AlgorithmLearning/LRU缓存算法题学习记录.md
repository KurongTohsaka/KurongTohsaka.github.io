---
title: "LRU缓存算法题学习记录"
date: 2025-09-20
aliases: ["/Algorithm Learning"]
tags: ["LRU"]
categories: ["Algorithm Learning"]
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

## LRU 缓存到底要做什么？

在动手写代码之前，最重要的一步是明确我们的目标。LRU 缓存机制的核心是：**当缓存满了以后，优先淘汰最久没有被使用过的数据。**

我们来分解一下 `LRUCache` 需要实现的功能：

1. **`Constructor(capacity int)`**: 初始化一个固定容量的缓存。
2. **`Get(key int)`**:
   - 如果 `key` 存在，返回对应的 `value`。
   - **关键点**：这次访问使得该 `key` 成为了“最近使用的”，所以需要更新它的位置。
   - 如果 `key` 不存在，返回 `-1`。
3. **`Put(key int, value int)`**:
   - 如果 `key` 存在，更新 `value`。
   - **关键点**：这次更新也使得该 `key` 成为了“最近使用的”，需要更新它的位置。
   - 如果 `key` 不存在，插入这个新的键值对。
   - **关键点**：插入后，如果缓存的尺寸超过了 `capacity`，则必须淘汰“最久未使用的”那个元素。

总结一下，我们需要一个数据结构，它必须同时满足以下几个要求：

- **快速查找**：`Get` 和 `Put` 都需要快速地判断一个 `key` 是否存在。
- **有序存储**：需要一种机制来记录所有 `key` 的“新旧”顺序，方便我们找到“最久未使用的”那个。
- **快速增删**：当一个 `key` 被访问（`Get` 或 `Put`）时，我们要能快速地将它移动到“最新”的位置。当缓存满了，也要能快速地删除“最旧”的那个。



## 哈希表 + 双向链表

这是本题的**核心套路**。没有任何单一的基础数据结构能同时满足上述所有要求，所以需要组合。

- **为什么需要哈希表 (`map`)？**
  - 为了实现 **O(1)** 复杂度的快速查找。我们可以用 `map[int]*Node` 来存储 `key` 到链表节点的映射。
- **为什么需要链表？**
  - 为了维护元素的“新旧”顺序。链表天然有序。我们可以规定，越靠近链表头部的元素，就是“最近使用”的；越靠近链表尾部的，就是“最久未使用”的。
- **为什么必须是双向链表，而不是单向？**
  - 因为我们需要 **O(1)** 复杂度的删除操作。想象一下，当我们要删除一个节点（比如缓存满了要淘汰尾部节点，或者 `Get` 了一个中间节点要把它移动到头部），我们需要修改这个节点**前面那个节点**的 `Next` 指针。在单向链表中，找到前驱节点需要从头遍历，时间复杂度是 O(n)。而双向链表每个节点都存了 `Pre` 和 `Next` 指针，使得我们可以在 O(1) 的时间内完成任意节点的删除。

**结论**：**哈希表 (`map`) + 双向链表 (`Doubly Linked List`)** 是解决此问题的完美组合。

- **哈希表** 负责快速定位 (`Get`）：`map[key] -> Node`。
- **双向链表** 负责维护顺序和快速增删 (`Put`/淘汰）：`head <-> node1 <-> node2 <-> tail`。



## 代码实现

#### 1. 定义数据结构

```go
// LRUCache holds the core logic for the cache.
type LRUCache struct {
    cache      map[int]*Node // Hash map for O(1) lookups: key -> Node pointer
    head, tail *Node          // Dummy head and tail nodes to handle edge cases
    cap        int           // The capacity of the cache
}

// Node represents a node in the doubly linked list.
type Node struct {
    Val, Key  int           // Store both key and value
    Pre, Next *Node         // Pointers to the previous and next nodes
}
```

#### 2. 初始化 `Constructor`

```go
// Constructor initializes a new LRUCache with a given capacity.
func Constructor(capacity int) LRUCache {
    head, tail := &Node{}, &Node{}
    head.Next = tail
    tail.Pre = head
    return LRUCache{
       cache: make(map[int]*Node),
       head:  head,
       tail:  tail,
       cap:   capacity,
    }
}
```

#### 3. 编写核心辅助函数

为了让 `Get` 和 `Put` 的逻辑更清晰，我们把链表操作封装成独立的辅助函数。

- `removeNode(node *Node)`: 从链表中移除一个节点。
- `addToHead(node *Node)`: 将一个节点添加到链表头部（`head` 之后）。
- `moveToHead(node *Node)`: 组合技，先移除再添加到头部，用于更新节点为“最新”。

```go
// removeNode detaches a node from the doubly linked list.
func (this *LRUCache) removeNode(node *Node) {
    node.Pre.Next = node.Next
    node.Next.Pre = node.Pre
}

// addToHead adds a node right after the dummy head.
func (this *LRUCache) addToHead(node *Node) {
    node.Pre = this.head
    node.Next = this.head.Next
    this.head.Next.Pre = node
    this.head.Next = node
}

// moveToHead moves an existing node to the head of the list.
func (this *LRUCache) moveToHead(node *Node) {
    this.removeNode(node)
    this.addToHead(node)
}
```

#### 4. 实现 `Get` 和 `Put`

有了辅助函数，`Get` 和 `Put` 的逻辑就变得非常直观了。

**`Get(key int)` 逻辑:**

1. 用哈希表查找 `key`。
2. 如果不存在，返回 `-1`。
3. 如果存在，我们拿到了节点 `node`。这个节点刚刚被访问，所以要把它更新为“最新”。直接调用 `moveToHead(node)`。
4. 返回 `node.Val`。

```go
func (this *LRUCache) Get(key int) int {
    if node, ok := this.cache[key]; ok {
       // The node was accessed, move it to the head to mark it as recently used.
       this.moveToHead(node)
       return node.Val
    }
    // Key not found.
    return -1
}
```

**`Put(key int, value int)` 逻辑:**

1. 先用哈希表查找 `key`，看是“更新”还是“插入”。
2. **情况一：Key 已存在**。
   - 更新 `node` 的 `Val`。
   - 因为它被访问了，调用 `moveToHead(node)` 将其更新为最新。
3. **情况二：Key 不存在**。
   - **检查容量**：`if len(this.cache) == this.cap`，如果缓存满了：
     - 找到最久未使用的节点，也就是哨兵 `tail` 的前一个节点 `tail.Pre`。
     - 从链表中删除它：`removeNode(tail.Pre)`。
     - 从哈希表中也删除它：`delete(this.cache, tail.Pre.Key)`。
   - 创建一个新的 `Node`。
   - 将新节点添加到链表头部：`addToHead(newNode)`。
   - 将新节点存入哈希表：`this.cache[key] = newNode`。

```go
func (this *LRUCache) Put(key int, value int) {
    // Case 1: The key already exists.
    if node, ok := this.cache[key]; ok {
       node.Val = value
       this.moveToHead(node) // Mark it as recently used.
       return
    }

    // Case 2: The key does not exist.
    // Check if the cache is full.
    if len(this.cache) == this.cap {
       // Evict the least recently used element, which is the one before the tail.
       tail := this.tail.Pre
       this.removeNode(tail)
       delete(this.cache, tail.Key)
    }

    // Add the new node.
    newNode := &Node{Key: key, Val: value}
    this.addToHead(newNode)
    this.cache[key] = newNode
}
```



## LRU + 过期时间 (TTL)

### 需求变更分析

1. **`Constructor(capacity int, ttl int)`**: 构造函数现在需要接收一个 `ttl` 参数，单位通常是秒。
2. **`Put(key, value)`**:
   - 当插入一个**新**的键值对时，需要记录它的过期时间点，即 `当前时间 + ttl`。
   - 当更新一个**已存在**的键值对时，同样需要**重置**它的过期时间点。
3. **`Get(key)`**:
   - 在返回一个值之前，**必须检查**它是否已经过期。
   - 如果已过期，应将其从缓存中**删除**，并返回 `-1` (表现得好像它不存在一样)。
   - 如果未过期，才执行标准的 LRU 逻辑：把它移动到链表头部并返回值。

### 核心设计决策：如何处理过期数据？

这里有两种主流策略：

1. **被动删除 (Lazy Eviction)**:
   - 我们不主动去监控哪些 key 过期了。
   - 只在 `Get` 一个 key 的时候，才去检查它的时间戳。如果发现它过期了，就地删除它。
   - **优点**：实现简单，没有额外的性能开销。
   - **缺点**：如果一个过期的 key 再也不被访问，它会一直占据着内存，直到被 LRU 规则淘汰出去。
2. **主动删除 (Eager Eviction)**:
   - 可以通过一个后台的定时任务，周期性地扫描整个缓存，删除所有过期的 key。
   - 或者，使用一个额外的数据结构（如最小堆或另一个按过期时间排序的链表）来专门管理过期，这样可以快速找到下一个要过期的元素。
   - **优点**：可以更及时地释放内存。
   - **缺点**：实现复杂，会引入额外的复杂度和性能开销（例如，后台扫描可能导致锁竞争）。

**面试中的选择**：在面试的有限时间内，**被动删除 (Lazy Eviction)** 是最推荐的方案。它清晰地展示了你处理核心逻辑的能力，而不会陷入过于复杂的实现细节中。

所以，我们的策略是：在 `Get` 时检查过期，并在 `Put` 时更新过期时间。

### 代码实现

#### 1. 升级数据结构

`Node` 结构体需要增加一个字段来存储它的“死亡时间”。

```go
import "time"

// Node now includes an expiration timestamp.
type Node struct {
    Val, Key  int
    Pre, Next *Node
    ExpiresAt int64 // Unix timestamp (seconds) when this entry expires.
}

// The main struct now includes the TTL duration.
type LRUCacheWithTTL struct {
    cache      map[int]*Node
    head, tail *Node
    cap        int
    ttl        int64 // Time-to-live in seconds for each entry.
}
```

#### 2. 修改 `Constructor`

```go
// Constructor now accepts a TTL value.
func Constructor(capacity int, ttl int) LRUCacheWithTTL {
    head, tail := &Node{}, &Node{}
    head.Next = tail
    tail.Pre = head
    return LRUCacheWithTTL{
       cache: make(map[int]*Node),
       head:  head,
       tail:  tail,
       cap:   capacity,
       ttl:   int64(ttl),
    }
}
```

#### 3. 修改 `Get` 方法

这是变化最大的地方。在找到节点后，我们必须先检查它的存活状态。

```go
// Get must now check for expiration before returning a value.
func (this *LRUCacheWithTTL) Get(key int) int {
    if node, ok := this.cache[key]; ok {
       // Check if the node has expired.
       if time.Now().Unix() > node.ExpiresAt {
           // It's expired. Remove it from the cache.
           this.removeNode(node)
           delete(this.cache, key)
           return -1 // Act as if it doesn't exist.
       }

       // Not expired, proceed with standard LRU logic.
       this.moveToHead(node)
       return node.Val
    }

    return -1
}
```

#### 4. 修改 `Put` 方法

`Put` 的逻辑也需要更新，无论是插入还是更新，都要设置新的过期时间。

```go
// Put must now set or reset the expiration time.
func (this *LRUCacheWithTTL) Put(key int, value int) {
    // Calculate the new expiration time.
    newExpiresAt := time.Now().Unix() + this.ttl

    // Case 1: The key already exists.
    if node, ok := this.cache[key]; ok {
       node.Val = value
       node.ExpiresAt = newExpiresAt // Reset its expiration time.
       this.moveToHead(node)
       return
    }

    // Case 2: The key does not exist.
    // Check for capacity and evict the LRU element if full.
    if len(this.cache) == this.cap {
       tail := this.tail.Pre
       this.removeNode(tail)
       delete(this.cache, tail.Key)
    }

    // Create and add the new node.
    newNode := &Node{
        Key:       key,
        Val:       value,
        ExpiresAt: newExpiresAt,
    }
    this.addToHead(newNode)
    this.cache[key] = newNode
}
```
