---
title: "geeCache项目笔记"
date: 2022-04-13T21:31:05+08:00
draft: false
tags: [专业知识]
categories: [笔记]
---



# LRU缓存淘汰策略

LRU 认为，如果数据最近被访问过，那么将来被访问的概率也会更高。LRU 算法的实现非常简单，维护一个队列，如果某条记录被访问了，则移动到队尾，那么队首则是最近最少访问的数据，淘汰该条记录即可。

```go
package lru

import (
	"container/list"
)

type entry struct {
	key   string
	value Value
}

// Value Len()返回所占用的内存大小
type Value interface {
	Len() int
}

// Cache 是LRU缓存，并发不安全
type Cache struct {
	// 允许使用的最大内存
	maxBytes int64
	// 当前已经使用的内存
	nbytes int64
	// 双向链表list.List
	ll *list.List
	// 字典map
	mp map[string]*list.Element
	// 记录被移除时的回调函数，可以为nil
	OnEvicted func(key string, value Value)
}

func NewCache(maxBytes int64, onEvicted func(key string, value Value)) *Cache {
	return &Cache{
		maxBytes:  maxBytes,
		ll:        list.New(),
		mp:        make(map[string]*list.Element),
		OnEvicted: onEvicted,
	}
}

func (c *Cache) Get(key string) (Value, bool) {
	if ele, ok := c.mp[key]; ok {
		// 如果键对应的链表节点存在，则将对应节点移动到队首
		c.ll.MoveToFront(ele)
		kv := ele.Value.(*entry)
		return kv.value, ok
	}
	return nil, false
}

func (c *Cache) RemoveOldest() {
	// 取队尾节点，即最近最少访问的节点
	if ele := c.ll.Back(); ele != nil {
		kv := ele.Value.(*entry)
		// 从链表中删除该节点
		c.ll.Remove(ele)
		// 从字典cache中删除节点映射关系
		delete(c.mp, kv.key)
		// 更新当前所用内存
		c.nbytes -= int64(len(kv.key)) + int64(kv.value.Len())
		// 若回调函数不为nil，调用回调函数
		if c.OnEvicted != nil {
			c.OnEvicted(kv.key, kv.value)
		}
	}
}

// Add 同时实现新增和修改的功能
func (c *Cache) Add(key string, value Value) {
	if ele, ok := c.mp[key]; ok {
		// 若key已存在，修改
		c.ll.MoveToFront(ele)
		kv := ele.Value.(*entry)
		c.nbytes += int64(value.Len()) - int64(kv.value.Len())
		kv.value = value
	} else {
		// 若key不存在，新增
		ele := c.ll.PushFront(&entry{key, value})
		c.mp[key] = ele
		c.nbytes += int64(len(key)) + int64(value.Len())
	}

	for c.maxBytes != 0 && c.nbytes > c.maxBytes {
		c.RemoveOldest()
	}
}

func (c *Cache) Len() int {
	return c.ll.Len()
}
```

# 单机并发缓存

## 1 sync.Mutex

多个协程(goroutine)同时读写同一个变量，在并发度较高的情况下，会发生冲突。确保一次只有一个协程(goroutine)可以访问该变量以避免冲突，这称之为`互斥`，互斥锁可以解决这个问题。

> sync.Mutex 是一个互斥锁，可以由不同的协程加锁和解锁。

`sync.Mutex` 是 Go 语言标准库提供的一个互斥锁，当一个协程(goroutine)获得了这个锁的拥有权后，其它请求锁的协程(goroutine) 就会阻塞在 `Lock()` 方法的调用上，直到调用 `Unlock()` 锁被释放。

## 2 支持并发读写

抽象了一个只读数据结构 `ByteView` 用来表示缓存值，是 GeeCache 主要的数据结构之一。

```go
package geeCache

// A ByteView holds an immutable view of bytes.
type ByteView struct {
	b []byte
}

// Len returns the view's length
func (v ByteView) Len() int {
	return len(v.b)
}

// ByteSlice returns a copy of the data as a byte slice.
func (v ByteView) ByteSlice() []byte {
	return cloneBytes(v.b)
}

// String returns the data as a string, making a copy if necessary.
func (v ByteView) String() string {
	return string(v.b)
}

func cloneBytes(b []byte) []byte {
	c := make([]byte, len(b))
	copy(c, b)
	return c
}

```

- 实现 `Len() int` 方法，我们在 lru.Cache 的实现中，要求被缓存对象必须实现 Value 接口，即 `Len() int` 方法，返回其所占的内存大小。
- `b` 是只读的，使用 `ByteSlice()` 方法返回一个拷贝，防止缓存值被外部程序修改。

> `ByteView` 是只读的，不可修改。通过 `ByteSlice()` 或 `String()` 方法取到缓存值的副本。
>
> 只读属性，是设计 `ByteView` 的主要目的之一。

接下来为 lru.Cache 添加并发特性。

```go
package geeCache

import (
   "geeCache/lru"
   "sync"
)

type cache struct {
   mu         sync.Mutex
   lru        *lru.Cache
   cacheBytes int64
}

func (c *cache) add(key string, value ByteView) {
   c.mu.Lock()
   defer c.mu.Unlock()
   if c.lru == nil {
      c.lru = lru.NewCache(c.cacheBytes, nil)
   }
   c.lru.Add(key, value)
}

func (c *cache) get(key string) (value ByteView, ok bool) {
   c.mu.Lock()
   defer c.mu.Unlock()
   if c.lru == nil {
      return
   }
   if v, ok := c.lru.Get(key); ok {
      return v.(ByteView), ok
   }
   return
}
```

- `cache.go` 的实现非常简单，实例化 lru，封装 get 和 add 方法，并添加互斥锁 mu。

## 3 主体结构 Group

Group 是 GeeCache 最核心的数据结构，负责与用户的交互，并且控制缓存值存储和获取的流程。

```
                            是
接收 key --> 检查是否被缓存 -----> 返回缓存值 ⑴
                |  否                         是
                |-----> 是否应当从远程节点获取 -----> 与远程节点交互 --> 返回缓存值 ⑵
                            |  否
                            |-----> 调用`回调函数`，获取值并添加到缓存 --> 返回缓存值 ⑶
```

```
geecache/
    |--lru/
        |--lru.go  // lru 缓存淘汰策略
    |--byteview.go // 缓存值的抽象与封装
    |--cache.go    // 并发控制
    |--geecache.go // 负责与外部交互，控制缓存存储和获取的主流程
```

### 3.1 回调 Getter

如果缓存不存在，应从数据源（文件，数据库等）获取数据并添加到缓存中。

GeeCache 是否应该支持多种数据源的配置？不应该，一是数据源的种类太多，没办法一一实现；二是扩展性不好。

如何从源头获取数据，应该是用户决定的事情，我们就把这件事交给用户。因此，我们设计了一个回调函数(callback)，在缓存不存在时，调用这个函数，得到源数据。

[geeCache/geecache.go]()

```go
package geeCache

// A Getter loads data for a key.
type Getter interface {
   Get(key string) ([]byte, error)
}

// A GetterFunc implements Getter with a function.
type GetterFunc func(key string) ([]byte, error)

// Get implements Getter interface function
func (f GetterFunc) Get(key string) ([]byte, error) {
   return f(key)
}
```

- 定义接口 Getter 和 回调函数 `Get(key string)([]byte, error)`，参数是 key，返回值是 []byte。
- 定义函数类型 GetterFunc，并实现 Getter 接口的 `Get` 方法。
- 函数类型实现某一个接口，称之为接口型函数，方便使用者在调用时既能够传入函数作为参数，也能够传入实现了该接口的结构体作为参数。

> 了解接口型函数的使用场景，可以参考 [Go 接口型函数的使用场景 - 7days-golang Q & A](https://geektutu.com/post/7days-golang-q1.html)

我们可以写一个测试用例来保证回调函数能够正常工作。

```go
func TestGetter(t *testing.T) {
	var f Getter = GetterFunc(func(key string) ([]byte, error) {
		return []byte(key), nil
	})

	expect := []byte("key")
	if v, _ := f.Get("key"); !reflect.DeepEqual(v, expect) {
		t.Errorf("callback failed")
	}
}
```

- 这个测试用例借助 GetterFunc 的类型转换，将一个匿名回调函数转换成了接口 `f Getter`。
- 调用该接口的方法 `f.Get(key string)`，实际上就是在调用匿名回调函数本身。

> 定义一个函数类型 F，并且实现接口 A 的方法，然后在这个方法中调用自己。
>
> 这是 Go 语言中将其他函数（参数返回值定义与 F 一致）转换为接口 A 的常用技巧。

### 3.2 Group定义

接下来是最核心数据结构 Group 的定义

[geeCache/geecache.go]()

```go
var (
   mu     sync.RWMutex
   groups = make(map[string]*Group)
)

// Group 可以看成一个缓存的命名空间
type Group struct {
   name string
   // 缓存未命中时获取源数据的回调(callback)
   getter Getter
   // 并发缓存
   mainCache cache
}

func NewGroup(name string, getter Getter, cacheBytes int64) *Group {
   if getter == nil {
      panic("nil Getter")
   }
   mu.Lock()
   defer mu.Unlock()
   g := &Group{
      name:      name,
      getter:    getter,
      mainCache: cache{cacheBytes: cacheBytes},
   }
   groups[name] = g
   return g
}

func GetGroup(name string) *Group {
   mu.RLock()
   defer mu.RUnlock()
   g := groups[name]
   return g
}
```

- 一个 Group 可以认为是一个缓存的命名空间，每个 Group 拥有一个唯一的名称 `name`。比如可以创建三个 Group，缓存学生的成绩命名为 scores，缓存学生信息的命名为 info，缓存学生课程的命名为 courses。
- 第二个属性是 `getter Getter`，即缓存未命中时获取源数据的回调(callback)。
- 第三个属性是 `mainCache cache`，即一开始实现的并发缓存。
- 构建函数 `NewGroup` 用来实例化 Group，并且将 group 存储在全局变量 `groups` 中。
- `GetGroup` 用来特定名称的 Group，这里使用了只读锁 `RLock()`，因为不涉及任何冲突变量的写操作。

### 3.3 Group 的 Get 方法

[geeCache/geecache.go]()

```go
func (g *Group) Get(key string) (ByteView, error) {
   if key == "" {
      return ByteView{}, fmt.Errorf("key is required")
   }
   // 流程（1）：从 mainCache 中查找缓存，如果存在则返回缓存值。
   if v, ok := g.mainCache.get(key); ok {
      log.Panicln("[GeeCache] hit")
      return v, nil
   }
   // 流程（3）：缓存不存在，则调用 load 方法
   return g.load(key)
}

func (g *Group) load(key string) (value ByteView, err error) {
   return g.getLocally(key)
}

func (g *Group) getLocally(key string) (ByteView, error) {
   // 调用用户回调函数 g.getter.Get() 获取源数据
   bytes, err := g.getter.Get(key)
   if err != nil {
      return ByteView{}, err
   }
   value := ByteView{b: cloneBytes(bytes)}
   // 并且将源数据添加到缓存 mainCache 中
   g.populateCache(key, value)
   return value, nil
}

func (g *Group) populateCache(key string, value ByteView) {
   g.mainCache.add(key, value)
}
```

- Get 方法实现了上述所说的流程 ⑴ 和 ⑶。
- 流程 ⑴ ：从 mainCache 中查找缓存，如果存在则返回缓存值。
- 流程 ⑶ ：缓存不存在，则调用 load 方法，load 调用 getLocally（分布式场景下会调用 getFromPeer 从其他节点获取），getLocally 调用用户回调函数 `g.getter.Get()` 获取源数据，并且将源数据添加到缓存 mainCache 中（通过 populateCache 方法）

## 4 测试

用一个 map 模拟耗时的数据库。

```go
var db = map[string]string{
	"Tom":  "630",
	"Jack": "589",
	"Sam":  "567",
}
```

创建 group 实例，并测试 `Get` 方法

```go
func TestGet(t *testing.T) {
	loadCounts := make(map[string]int, len(db))
	gee := NewGroup("scores", 2<<10, GetterFunc(
		func(key string) ([]byte, error) {
			log.Println("[SlowDB] search key", key)
			if v, ok := db[key]; ok {
				if _, ok := loadCounts[key]; !ok {
					loadCounts[key] = 0
				}
				loadCounts[key] += 1
				return []byte(v), nil
			}
			return nil, fmt.Errorf("%s not exist", key)
		}))

	for k, v := range db {
		if view, err := gee.Get(k); err != nil || view.String() != v {
			t.Fatal("failed to get value of Tom")
		} // load from callback function
		if _, err := gee.Get(k); err != nil || loadCounts[k] > 1 {
			t.Fatalf("cache %s miss", k)
		} // cache hit
	}

	if view, err := gee.Get("unknown"); err == nil {
		t.Fatalf("the value of unknow should be empty, but %s got", view)
	}
}
```

- 在这个测试用例中，我们主要测试了 2 种情况
- 1）在缓存为空的情况下，能够通过回调函数获取到源数据。
- 2）在缓存已经存在的情况下，是否直接从缓存中获取，为了实现这一点，使用 `loadCounts` 统计某个键调用回调函数的次数，如果次数大于1，则表示调用了多次回调函数，没有缓存。

测试结果如下：

```shell
(base) PS E:\Workspace\Go\gee\gee-cache\geeCache> go test -run TestGet
2022/04/14 14:52:10 [SlowDB] search key Jack
2022/04/14 14:52:10 [GeeCache] hit
2022/04/14 14:52:10 [SlowDB] search key Sam
2022/04/14 14:52:10 [GeeCache] hit
2022/04/14 14:52:10 [SlowDB] search key Tom
2022/04/14 14:52:10 [GeeCache] hit
2022/04/14 14:52:10 [SlowDB] search key unknown
PASS
ok      geeCache        0.216s
```

可以很清晰地看到，缓存为空时，调用了回调函数，第二次访问时，则直接从缓存中读取。

# HTTP服务端

## 1 geeCache HTTP

分布式缓存需要实现节点间通信，建立基于 HTTP 的通信机制是比较常见和简单的做法。如果一个节点启动了 HTTP 服务，那么这个节点就可以被其他节点访问。今天我们就为单机节点搭建 HTTP Server。

不与其他部分耦合，我们将这部分代码放在新的 `http.go` 文件中，当前的代码结构如下：

```
geecache/
    |--lru/
        |--lru.go  // lru 缓存淘汰策略
    |--byteview.go // 缓存值的抽象与封装
    |--cache.go    // 并发控制
    |--geecache.go // 负责与外部交互，控制缓存存储和获取的主流程
	|--http.go     // 提供被其他节点访问的能力(基于http)
```

首先创建一个结构体 `HTTPPool`，作为承载节点间 HTTP 通信的核心数据结构（包括服务端和客户端，先只实现服务端）。

[geeCache/http.go]()

```go
const defaultBasePath = "/_geecache/"

// HTTPPool implements PeerPicker for a pool of HTTP peers.
type HTTPPool struct {
	// this peer's base URL, e.g. "https://example.net:8000"
	self     string
	basePath string
}

// NewHTTPPool initializes an HTTP pool of peers.
func NewHTTPPool(self string) *HTTPPool {
	return &HTTPPool{
		self:     self,
		basePath: defaultBasePath,
	}
}
```

- `HTTPPool` 只有 2 个参数，一个是 self，用来记录自己的地址，包括主机名/IP 和端口。
- 另一个是 basePath，作为节点间通讯地址的前缀，默认是 `/_geecache/`，那么 http://example.com/_geecache/ 开头的请求，就用于节点间的访问。因为一个主机上还可能承载其他的服务，加一段 Path 是一个好习惯。比如，大部分网站的 API 接口，一般以 `/api` 作为前缀。

接下来，实现最为核心的 `ServeHTTP` 方法。

[geeCache/http.go]()

```go
func (p *HTTPPool) Log(format string, v ...interface{}) {
   log.Printf("[Server %s] %s", p.self, fmt.Sprintf(format, v...))
}

func (p *HTTPPool) ServeHTTP(w http.ResponseWriter, r *http.Request) {
   // (1) 判断路径前缀是否为basePath
   if !strings.HasPrefix(r.URL.Path, p.basePath) {
      panic("HTTPPool serving unexpected path: " + r.URL.Path)
   }
   p.Log("%s %s", r.Method, r.URL.Path)

   // 约定访问路径格式为 /<basepath>/<groupname>/<key> required
   parts := strings.SplitN(r.URL.Path[len(p.basePath):], "/", 2)
   if len(parts) != 2 {
      http.Error(w, "bad request", http.StatusBadRequest)
      return
   }
   // 通过groupName得到group实例
   groupName := parts[0]
   group := GetGroup(groupName)
   if group == nil {
      http.Error(w, "no such group:"+groupName, http.StatusNotFound)
      return
   }
   // 通过group.Get(key)得到缓存数据
   key := parts[1]
   view, err := group.Get(key)
   if err != nil {
      http.Error(w, err.Error(), http.StatusInternalServerError)
      return
   }
   // 将缓存值作为httpResponse的body返回
   w.Header().Set("Content-Type", "application/octet-stream")
   w.Write(view.ByteSlice())
}
```

- ServeHTTP 的实现逻辑是比较简单的，首先判断访问路径的前缀是否是 `basePath`，不是返回错误。
- 我们约定访问路径格式为 `/<basepath>/<groupname>/<key>`，通过 groupname 得到 group 实例，再使用 `group.Get(key)` 获取缓存数据。
- 最终使用 `w.Write()` 将缓存值作为 httpResponse 的 body 返回。

## 2 测试

实现 main 函数，实例化 group，并启动 HTTP 服务。

```go
package gee_cache

import (
   "fmt"
   "geeCache"
   "log"
   "net/http"
)

var db = map[string]string{
   "Tom":  "630",
   "Jack": "589",
   "Sam":  "567",
}

func main() {
   geeCache.NewGroup("scores", geeCache.GetterFunc(
      func(key string) ([]byte, error) {
         log.Println("[SlowDB] search key", key)
         if v, ok := db[key]; ok {
            return []byte(v), nil
         }
         return nil, fmt.Errorf("%s not exist", key)
      }), 2<<10)

   addr := "localhost:9999"
   peers := geeCache.NewHTTPPool(addr)
   log.Println("geecache is running at", addr)
   log.Fatal(http.ListenAndServe(addr, peers))
}
```

- 同样地，我们使用 map 模拟了数据源 db。
- 创建一个名为 scores 的 Group，若缓存为空，回调函数会从 db 中获取数据并返回。
- 使用 http.ListenAndServe 在 9999 端口启动了 HTTP 服务。

> 需要注意的点：
> main.go 和 geecache/ 在同级目录，但 go modules 不再支持 import <相对路径>，相对路径需要在 go.mod 中声明：
> require geecache v0.0.0
> replace geecache => ./geecache

```shell
[root@VM-20-16-centos gee-cache]# curl http://localhost:9999/_geecache/scores/Tom
2022/04/14 15:51:32 [Server localhost:9999] GET /_geecache/scores/Tom
2022/04/14 15:51:32 [SlowDB] search key Tom
630
[root@VM-20-16-centos gee-cache]# curl http://localhost:9999/_geecache/scores/Tom
2022/04/14 15:52:17 [Server localhost:9999] GET /_geecache/scores/Tom
2022/04/14 15:52:17 [GeeCache] hit
630
[root@VM-20-16-centos gee-cache]# curl http://localhost:9999/_geecache/scores/kkk
2022/04/14 15:53:05 [Server localhost:9999] GET /_geecache/scores/kkk
2022/04/14 15:53:05 [SlowDB] search key kkk
kkk not exist
```

节点间的相互通信不仅需要 HTTP 服务端，还需要 HTTP 客户端，这是下一步需要做的事情。

# 一致性哈希

## 1 为什么使用一致性哈希

一致性哈希算法是 GeeCache 从单节点走向分布式节点的一个重要的环节。

### 1.1 我该访问谁？

对于分布式缓存来说，当一个节点接收到请求，如果该节点并没有存储缓存值，那么它面临的难题是，从谁那获取数据？假设包括自己在内一共有 10 个节点，当一个节点接收到请求时，随机选择一个节点，由该节点从数据源获取数据。假设第一次随机选取了节点 1 ，节点 1 从数据源获取到数据的同时缓存该数据；那第二次，只有 1/10 的可能性再次选择节点 1, 有 9/10 的概率选择了其他节点，如果选择了其他节点，就意味着需要再一次从数据源获取数据，一般来说，这个操作是很耗时的。

这样做，一是缓存效率低，二是各个节点上存储着相同的数据，浪费了大量的存储空间。有什么办法，对于给定的 key，每一次都选择同一个节点？

使用 hash 算法也能够做到这一点。那把 key 的每一个字符的 ASCII 码加起来，再除以 10 取余数。

### 1.2 节点数量变化了怎么办？

简单求Hash 解决了缓存性能的问题，但是没有考虑节点数量变化的场景。

假设，移除了其中一台节点，只剩下 9 个，那么之前 `hash(key) % 10` 变成了 `hash(key) % 9`，也就意味着几乎缓存值对应的节点都发生了改变。即几乎所有的缓存值都失效了。节点在接收到对应的请求时，均需要重新去数据源获取数据，容易引起 `缓存雪崩`。

> 缓存雪崩：缓存在同一时刻全部失效，造成瞬时DB请求量大、压力骤增，引起雪崩。常因为缓存服务器宕机，或缓存设置了相同的过期时间引起。

此时就需要一致性哈希算法。



## 2 算法原理

### 2.1 步骤

一致性哈希算法将 key 映射到 2^32 的空间中，将这个数字首尾相连，形成一个环。

- 计算节点/机器(通常使用节点的名称、编号和 IP 地址)的哈希值，放置在环上。
- 计算 key 的哈希值，放置在环上，顺时针寻找到的第一个节点，就是应选取的节点/机器。

![一致性哈希添加节点 consistent hashing add peer](https://geektutu.com/post/geecache-day4/add_peer.jpg)

环上有 peer2，peer4，peer6 三个节点，`key11`，`key2`，`key27` 均映射到 peer2，`key23` 映射到 peer4。此时，如果新增节点/机器 peer8，假设它新增位置如图所示，那么只有 `key27` 从 peer2 调整到 peer8，其余的映射均没有发生改变。

也就是说，一致性哈希算法，在新增/删除节点时，只需要重新定位该节点附近的一小部分数据，而不需要重新定位所有的节点，这就解决了上述的问题。

### 2.2 数据倾斜问题

如果服务器的节点过少，容易引起 key 的倾斜。例如上面例子中的 peer2，peer4，peer6 分布在环的上半部分，下半部分是空的。那么映射到环下半部分的 key 都会被分配给 peer2，key 过度向 peer2 倾斜，缓存节点间负载不均。

为了解决这个问题，引入了虚拟节点的概念，一个真实节点对应多个虚拟节点。

假设 1 个真实节点对应 3 个虚拟节点，那么 peer1 对应的虚拟节点是 peer1-1、 peer1-2、 peer1-3（通常以添加编号的方式实现），其余节点也以相同的方式操作。

- 第一步，计算虚拟节点的 Hash 值，放置在环上。
- 第二步，计算 key 的 Hash 值，在环上顺时针寻找到应选取的虚拟节点，例如是 peer2-1，那么就对应真实节点 peer2。

虚拟节点扩充了节点的数量，解决了节点较少的情况下数据容易倾斜的问题。而且代价非常小，只需要增加一个字典(map)维护真实节点与虚拟节点的映射关系即可。

## 3 实现

[geeCache/consistentHash/consistentHash.go]()

**首先，构造数据结构** 

```go
// Hash maps bytes to uint32
type Hash func(data []byte) uint32

// Map constains all hashed keys
type Map struct {
	hash     Hash
	replicas int
	keys     []int // Sorted
	hashMap  map[int]string
}

// New creates a Map instance
func New(replicas int, fn Hash) *Map {
	m := &Map{
		replicas: replicas,
		hash:     fn,
		hashMap:  make(map[int]string),
	}
	if m.hash == nil {
		m.hash = crc32.ChecksumIEEE
	}
	return m
}
```

- 定义函数类型 `Hash`，采取依赖注入的方式，允许用于替换成自定义的 Hash 函数，也方便测试时替换。

  默认为 `crc32.ChecksumIEEE` 算法。

- `Map` 是一致性哈希算法的主数据结构，包含 4 个成员变量：Hash 函数 `hash`；虚拟节点倍数 `replicas`；哈希环 `keys`；虚拟节点与真实节点的映射表 `hashMap`，键是虚拟节点的哈希值，值是真实节点的名称。

- 构造函数 `New()` 允许自定义虚拟节点倍数和 Hash 函数。

**接下来，实现添加真实节点/机器的 `Add()` 方法。**

```go
// Add adds some keys to the hash.
func (m *Map) Add(keys ...string) {
   for _, key := range keys {
      // 对每个真实节点，创建replicas个虚拟节点
      for i := 0; i < m.replicas; i++ {
         // 通过编号i的方式区分不同的虚拟节点，然后计算虚拟节点哈希值
         hash := int(m.hash([]byte(strconv.Itoa(i) + key)))
         // 把虚拟节点哈希值添加到环上
         m.keys = append(m.keys, hash)
         // 增加虚拟节点和真实节点的映射关系
         m.mp[hash] = key
      }
   }

   sort.Ints(m.keys)
}
```

- `Add` 函数允许传入 0 或 多个真实节点的名称。
- 对每一个真实节点 `key`，对应创建 `m.replicas` 个虚拟节点，虚拟节点的名称是：`strconv.Itoa(i) + key`，即通过添加编号的方式区分不同虚拟节点。
- 使用 `m.hash()` 计算虚拟节点的哈希值，使用 `append(m.keys, hash)` 添加到环上。
- 在 `hashMap` 中增加虚拟节点和真实节点的映射关系。
- 最后一步，环上的哈希值排序。

**最后一步，实现选择节点的 `Get()` 方法。**

```go
// Get gets the closest item in the hash to the provided key.
func (m *Map) Get(key string) string {
   if len(key) == 0 {
      return ""
   }
   // 计算key的哈希值
   hash := int(m.hash([]byte(key)))
   // 顺时针找到第一个匹配的虚拟节点的下标 idx
   idx := sort.Search(len(m.keys), func(i int) bool {
      return m.keys[i] >= hash
   })
   // m.keys是一个环形结构，所以使用取余数的方式
   return m.mp[m.keys[idx%len(m.keys)]]
}
```

- 选择节点就非常简单了，第一步，计算 key 的哈希值。
- 第二步，顺时针找到第一个匹配的虚拟节点的下标 `idx`，从 `m.keys` 中获取到对应的哈希值。如果 `idx == len(m.keys)`，说明应选择 `m.keys[0]`，因为 `m.keys` 是一个环状结构，所以用取余数的方式来处理这种情况。
- 第三步，通过 `hashMap` 映射得到真实的节点。

至此，整个一致性哈希算法就实现完成了。





# 分布式节点

## 1 流程回顾

```
                            是
接收 key --> 检查是否被缓存 -----> 返回缓存值 ⑴
                |  否                         是
                |-----> 是否应当从远程节点获取 -----> 与远程节点交互 --> 返回缓存值 ⑵
                            |  否
                            |-----> 调用`回调函数`，获取值并添加到缓存 --> 返回缓存值 ⑶
```

之前描述了 geecache 的流程。

在这之前已经实现了流程 ⑴ 和 ⑶，今天实现流程 ⑵，从远程节点获取缓存值。

进一步细化流程 ⑵：

```
使用一致性哈希选择节点        是                                    是
    |-----> 是否是远程节点 -----> HTTP 客户端访问远程节点 --> 成功？-----> 服务端返回返回值
                    |  否                                    ↓  否
                    |----------------------------> 回退到本地节点处理。
```

## 2 抽象 PeerPicker

[geeCache/peers.go]()

```go
type PeerPicker interface {
   PickPeer(key string) (peer PeerGetter, ok bool)
}

type PeerGetter interface {
   Get(group string, key string) ([]byte, error)
}
```

- 在这里，抽象出 2 个接口，PeerPicker 的 `PickPeer()` 方法用于根据传入的 key 选择相应节点 PeerGetter。

- 接口 PeerGetter 的 `Get()` 方法用于从对应 group 查找缓存值。

  PeerGetter 就对应于上述流程中的 HTTP 客户端。

## 3 节点选择与 HTTP 客户端

**首先创建具体的 HTTP 客户端类 `httpGetter`，实现 PeerGetter 接口。**

[geeCache/http.go]()

```go
// http客户端类
type httpGetter struct {
	baseURL string // 表示将要访问的远程节点的地址
	// e.g. http://example.com/_geecache/
}

func (h *httpGetter) Get(group string, key string) ([]byte, error) {
	u := fmt.Sprintf("%v%v/%v", h.baseURL, url.QueryEscape(group), url.QueryEscape(key))

	// 使用http.Get() 获取返回值
	res, err := http.Get(u)
	if err != nil {
		return nil, err
	}
	defer res.Body.Close()

	if res.StatusCode != http.StatusOK {
		return nil, fmt.Errorf("server returned: %v", res.Status)
	}
	// 并转换为[]byte类型
	bytes, err := ioutil.ReadAll(res.Body)
	if err != nil {
		return nil, fmt.Errorf("reading response body: %v", err)
	}

	return bytes, nil
}
```

- baseURL 表示将要访问的远程节点的地址，例如 `http://example.com/_geecache/`。
- 使用 `http.Get()` 方式获取返回值，并转换为 `[]bytes` 类型。

**第二步，为 HTTPPool 添加节点选择的功能。**

```go
const (
   defaultBasePath = "/_geecache/"
   defaultReplicas = 50
)

type HTTPPool struct {
   self        string // 用来记录自己的地址，包括主机名/IP和端口
   basePath    string // basePath 节点间通讯地址的前缀，
   mu          sync.Mutex
   peers       *consistentHash.Map    // 用来根据具体key选择节点
   httpGetters map[string]*httpGetter // 映射远程节点和对应的httpGetter
}
```

- 新增成员变量 `peers`，类型是一致性哈希算法的 `Map`，用来根据具体的 key 选择节点。
- 新增成员变量 `httpGetters`，映射远程节点与对应的 httpGetter。每一个远程节点对应一个 httpGetter，因为 httpGetter 与远程节点的地址 `baseURL` 有关。

**第三步，实现 PeerPicker 接口。**

```go
// Set updates the pool's list of peers.
func (p *HTTPPool) Set(peers ...string) {
	p.mu.Lock()
	defer p.mu.Unlock()
	// 实例化一致性哈希算法
	p.peers = consistentHash.New(defaultReplicas, nil)
	// 将传入的节点加入一致性哈希算法中
	p.peers.Add(peers...)
	// 并为每个节点创建一个对应的http客户端 httpGetter
	p.httpGetters = make(map[string]*httpGetter, len(peers))
	for _, peer := range peers {
		p.httpGetters[peer] = &httpGetter{baseURL: peer + p.basePath}
	}
}

// PickPeer picks a peer according to key
func (p *HTTPPool) PickPeer(key string) (PeerGetter, bool) {
	p.mu.Lock()
	defer p.mu.Unlock()
	// 包装了一致性哈希算法的Get()方法，根据具体的key，选择节点
	if peer := p.peers.Get(key); peer != "" && peer != p.self {
		p.Log("Pick peer %s", peer)
		// 返回节点对应的http客户端
		return p.httpGetters[peer], true
	}
	return nil, false
}
```

- `Set()` 方法实例化了一致性哈希算法，并且添加了传入的节点。
- 并为每一个节点创建了一个 HTTP 客户端 `httpGetter`。
- `PickerPeer()` 包装了一致性哈希算法的 `Get()` 方法，根据具体的 key，选择节点，返回节点对应的 HTTP 客户端。

至此，HTTPPool 既具备了提供 HTTP 服务的能力，也具备了根据具体的 key，创建 HTTP 客户端从远程节点获取缓存值的能力。

## 4 实现主流程

最后，我们需要将上述新增的功能集成在geecache.go中。

[geeCache/geecache.go]()

```go
// Group 可以看成一个缓存的命名空间
type Group struct {
   name      string
   getter    Getter // 缓存未命中时获取源数据的回调(callback)
   mainCache cache  // 并发缓存
   peers     PeerPicker
}

...

// RegisterPeers 将实现了PeerPicker接口的HTTPPool注入到Group中
func (g *Group) RegisterPeers(peers PeerPicker) {
	if g.peers != nil {
		panic("RegisterPeerPicker called more than once")
	}
	// 将实现了PeerPicker接口的HTTPPool注入到Group中
	g.peers = peers
}

func (g *Group) load(key string) (value ByteView, err error) {
	if g.peers != nil {
		// 使用PickPeer() 选择节点，若为ok则说明选择的节点为远程节点
		if peer, ok := g.peers.PickPeer(key); ok {
			// 调用getFromPeer获取缓存值
			if value, err := g.getFromPeer(peer, key); err == nil {
				return value, nil
			}
			log.Println("[GeeCache] Failed to get from peer", err)
		}
	}
	// !ok,说明选择远程节点失败或者选择的是本地节点
	return g.getLocally(key)
}

// getFromPeer 使用实现了PeerGetter接口的httpGetter访问远程节点，获取缓存值
func (g *Group) getFromPeer(peer PeerGetter, key string) (ByteView, error) {
	bytes, err := peer.Get(g.name, key)
	if err != nil {
		return ByteView{}, err
	}
	return ByteView{bytes}, nil
}
```

- 新增 `RegisterPeers()` 方法，将 实现了 PeerPicker 接口的 HTTPPool 注入到 Group 中。
- 新增 `getFromPeer()` 方法，使用实现了 PeerGetter 接口的 httpGetter 从访问远程节点，获取缓存值。
- 修改 load 方法，使用 `PickPeer()` 方法选择节点，若非本机节点，则调用 `getFromPeer()` 从远程获取。若是本机节点或失败，则回退到 `getLocally()`。

## 5 测试

```go
package main

import (
   "flag"
   "fmt"
   "geeCache"
   "log"
   "net/http"
)

var db = map[string]string{
   "Tom":  "630",
   "Jack": "589",
   "Sam":  "567",
}

func createGroup() *geeCache.Group {
   return geeCache.NewGroup("scores", geeCache.GetterFunc(
      func(key string) ([]byte, error) {
         log.Println("[SlowDB] search key", key)
         if v, ok := db[key]; ok {
            return []byte(v), nil
         }
         return nil, fmt.Errorf("%s not exist", key)
      }), 2<<10)
}

// 启动缓存服务器
func startCacheServer(addr string, addrs []string, gee *geeCache.Group) {
   // 创建HTTPPool
   peers := geeCache.NewHTTPPool(addr)
   // 添加节点信息
   peers.Set(addrs...)
   // 注册到gee中
   gee.RegisterPeers(peers)
   // 启动HTTP服务
   log.Println("geecache is running at", addr)
   log.Fatal(http.ListenAndServe(addr[7:], peers))
}

// 启动API服务（9999）
func startAPIServer(apiAddr string, gee *geeCache.Group) {
   http.Handle("/api", http.HandlerFunc(
      func(w http.ResponseWriter, r *http.Request) {
         key := r.URL.Query().Get("key")
         view, err := gee.Get(key)
         if err != nil {
            http.Error(w, err.Error(), http.StatusInternalServerError)
            return
         }
         w.Header().Set("Content-Type", "application/octet-stream")
         w.Write(view.ByteSlice())
      }))
   log.Println("fontend server is running at", apiAddr)
   log.Fatal(http.ListenAndServe(apiAddr[7:], nil))

}

func main() {
   var port int
   var api bool
   flag.IntVar(&port, "port", 8001, "Geecache server port")
   flag.BoolVar(&api, "api", false, "Start a api server?")
   flag.Parse()

   apiAddr := "http://localhost:9999"
   addrMap := map[int]string{
      8001: "http://localhost:8001",
      8002: "http://localhost:8002",
      8003: "http://localhost:8003",
   }

   var addrs []string
   for _, v := range addrMap {
      addrs = append(addrs, v)
   }

   gee := createGroup()
   if api {
      go startAPIServer(apiAddr, gee)
   }
   startCacheServer(addrMap[port], addrs, gee)
}
```

- `startCacheServer()` 用来启动缓存服务器：创建 HTTPPool，添加节点信息，注册到 gee 中，启动 HTTP 服务（共3个端口，8001/8002/8003），用户不感知。
- `startAPIServer()` 用来启动一个 API 服务（端口 9999），与用户进行交互，用户感知。
- `main()` 函数需要命令行传入 `port` 和 `api` 2 个参数，用来在指定端口启动 HTTP 服务。

为了方便，我们将启动的命令封装为一个 `shell` 脚本：

```sh
#!/bin/bash
trap "rm server;kill 0" EXIT

go build -o server
./server -port=8001 &
./server -port=8002 &
./server -port=8003 -api=1 &

sleep 2
echo ">>> start test"
curl "http://localhost:9999/api?key=Tom" &
curl "http://localhost:9999/api?key=Tom" &
curl "http://localhost:9999/api?key=Tom" &

wait
```

```sh
[root@VM-20-16-centos gee-cache]# ./d5.sh
2022/04/15 10:49:51 geecache is running at http://localhost:8003
2022/04/15 10:49:51 fontend server is running at http://localhost:9999
2022/04/15 10:49:51 geecache is running at http://localhost:8002
2022/04/15 10:49:51 geecache is running at http://localhost:8001
>>> start test
2022/04/15 10:49:53 [Server http://localhost:8003] Pick peer http://localhost:8001
2022/04/15 10:49:53 [Server http://localhost:8001] GET /_geecache/scores/Tom
2022/04/15 10:49:53 [SlowDB] search key Tom
630
2022/04/15 10:49:53 [Server http://localhost:8003] Pick peer http://localhost:8001
2022/04/15 10:49:53 [Server http://localhost:8001] GET /_geecache/scores/Tom
2022/04/15 10:49:53 [GeeCache] hit
630
2022/04/15 10:49:53 [Server http://localhost:8003] Pick peer http://localhost:8001
2022/04/15 10:49:53 [Server http://localhost:8001] GET /_geecache/scores/Tom
2022/04/15 10:49:53 [GeeCache] hit
630
```

此时打开一个新的 shell，进行测试：

```sh
[root@VM-20-16-centos gee-cache]# curl "http://localhost:9999/api?key=Tom"
2022/04/15 10:51:37 [Server http://localhost:8003] Pick peer http://localhost:8001
2022/04/15 10:51:37 [Server http://localhost:8001] GET /_geecache/scores/Tom
2022/04/15 10:51:37 [GeeCache] hit
630[root@VM-20-16-centos gee-cache]# curl "http://localhost:9999/api?key=kkk"
2022/04/15 10:51:47 [Server http://localhost:8003] Pick peer http://localhost:8001
2022/04/15 10:51:47 [Server http://localhost:8001] GET /_geecache/scores/kkk
2022/04/15 10:51:47 [SlowDB] search key kkk
2022/04/15 10:51:47 [GeeCache] Failed to get from peer <nil>
2022/04/15 10:51:47 [SlowDB] search key kkk
kkk not exist
```

测试并发了 3 个请求 `?key=Tom`，从日志中可以看到，三次均选择了节点 `8001`，这是一致性哈希算法的功劳。

但是有一个问题在于，同时向 `8001` 发起了 3 次请求。试想，假如有 10 万个在并发请求该数据呢？

那就会向 `8001` 同时发起 10 万次请求，如果 `8001` 又同时向数据库发起 10 万次查询请求，很容易导致缓存被击穿。



# 防止缓存击穿

## 1 缓存雪崩、缓存击穿、缓存穿透

> **缓存雪崩**：缓存在同一时刻全部失效，造成瞬时DB请求量大、压力骤增，引起雪崩。缓存雪崩通常因为缓存服务器宕机、缓存的 key 设置了相同的过期时间等引起。

> **缓存击穿**：一个存在的key，在缓存过期的一刻，同时有大量的请求，这些请求都会击穿到 DB ，造成瞬时DB请求量大、压力骤增。

> **缓存穿透**：查询一个不存在的数据，因为不存在则不会写到缓存中，所以每次都会去请求 DB，如果瞬间流量过大，穿透到 DB，导致宕机。

## 2 实现singleflight 

在「分布式节点」的测试中，我们并发了 N 个请求 `?key=Tom`，8003 节点向 8001 同时发起了 N 次请求。

假设对数据库的访问没有做任何限制的，很可能向数据库也发起 N 次请求，容易导致缓存击穿和穿透。即使对数据库做了防护，HTTP 请求是非常耗费资源的操作，针对相同的 key，8003 节点向 8001 发起三次请求也是没有必要的。

那这种情况下，我们如何做到只向远端节点发起一次请求呢？geecache 实现了一个名为 **singleflight** 的 package 来解决这个问题。

[geecache/singleflight/singleflight.go]()

**首先创建 `call` 和 `Group` 类型。**

```go
// 正在进行中、或者已经结束的请求
type call struct {
   wg  sync.WaitGroup // 避免重入
   val interface{}
   err error
}

// Group 主数据结构，管理不同key的请求（call）
type Group struct {
   mu sync.Mutex // protects m
   m  map[string]*call
}
```

- `call` 代表正在进行中，或已经结束的请求。使用 `sync.WaitGroup` 锁避免重入。
- `Group` 是 singleflight 的主数据结构，管理不同 key 的请求(call)。

**实现 `Do` 方法**

```go
func (g *Group) Do(key string, fn func() (interface{}, error)) (interface{}, error) {
	g.mu.Lock()
	if g.mp == nil {
		g.mp = make(map[string]*call)
	}
	if c, ok := g.mp[key]; ok {
		g.mu.Unlock()
		c.wg.Wait()         // 如果请求正在进行中，则等待
		return c.val, c.err // 请求结束，返回结果
	}
	c := new(call)
    c.wg.Add(1)   // 发起请求前加锁
	g.mp[key] = c // 添加到g.mp，表明key已经有对应的请求在处理
	g.mu.Unlock()
	c.val, c.err = fn() // 调用 fn，发起请求
	c.wg.Done()         // 请求结束

	g.mu.Lock()
	delete(g.mp, key) // 更新g.mp
	g.mu.Unlock()

	return c.val, c.err
}
```

- Do 方法，第一个参数是 `key`，第二个参数是一个函数 `fn`。

  Do 的作用就是，针对相同的 key，无论 Do 被调用多少次，函数 `fn` 都只会被调用一次，等待 `fn` 调用结束了，返回返回值或错误。

  - wg.Add(1) 锁加1。
  - wg.Wait() 阻塞，直到锁被释放。
  - wg.Done() 锁减1。

## 3 singleflight 的使用

[geeCache/geecache.go]()

**重写load**

```go
func (g *Group) load(key string) (value ByteView, err error) {
   // 每个key只会被请求一次
   view, err := g.loader.Do(key, func() (interface{}, error) {
      if g.peers != nil {
         // 使用PickPeer() 选择节点，若为ok则说明选择的节点为远程节点
         if peer, ok := g.peers.PickPeer(key); ok {
            // 调用getFromPeer获取缓存值
            if value, err := g.getFromPeer(peer, key); err == nil {
               return value, nil
            }
            log.Println("[GeeCache] Failed to get from peer", err)
         }
      }
      // !ok,说明选择远程节点失败或者选择的是本地节点
      return g.getLocally(key)
   })

   if err == nil {
      return view.(ByteView), nil
   }
   return
}
```

- 修改 `geecache.go` 中的 `Group`，添加成员变量 loader，并更新构建函数 `NewGroup`。
- 修改 `load` 函数，将原来的 load 的逻辑，使用 `g.loader.Do` 包裹起来即可，这样确保了并发场景下针对相同的 key，`load` 过程只会调用一次。

## 4 测试

执行 `run.sh` 就可以看到效果了。

```sh
[root@VM-20-16-centos gee-cache]# ./d5.sh
2022/04/17 16:26:09 geecache is running at http://localhost:8003
2022/04/17 16:26:09 fontend server is running at http://localhost:9999
2022/04/17 16:26:09 geecache is running at http://localhost:8002
2022/04/17 16:26:09 geecache is running at http://localhost:8001
>>> start test
2022/04/17 16:26:11 [Server http://localhost:8003] Pick peer http://localhost:8001
2022/04/17 16:26:11 [Server http://localhost:8001] GET /_geecache/scores/Tom
2022/04/17 16:26:11 [SlowDB] search key Tom
2022/04/17 16:26:11 [Server http://localhost:8003] Pick peer http://localhost:8001
2022/04/17 16:26:11 [Server http://localhost:8001] GET /_geecache/scores/Tom
2022/04/17 16:26:11 [GeeCache] hit
630630
2022/04/17 16:26:11 [Server http://localhost:8003] Pick peer http://localhost:8001
2022/04/17 16:26:11 [Server http://localhost:8001] GET /_geecache/scores/Tom
2022/04/17 16:26:11 [GeeCache] hit
630630630630630630630630630630630630
2022/04/17 16:26:11 [Server http://localhost:8003] Pick peer http://localhost:8001
2022/04/17 16:26:11 [Server http://localhost:8001] GET /_geecache/scores/Tom
2022/04/17 16:26:11 [GeeCache] hit
630630

```

可以看到，向 API 发起了三次并发请求，但8003 只向 8001 发起了一次请求，就搞定了。

如果并发度不够高，可能仍会看到向 8001 请求三次的场景。这种情况下三次请求是串行执行的，并没有触发 `singleflight` 的锁机制工作，可以加大并发数量再测试。即，将 `run.sh` 中的 `curl` 命令复制 N 次。

# Protobuf 通信

## 1 为什么要使用 protobuf

> protobuf 即 Protocol Buffers，Google 开发的一种数据描述语言，是一种轻便高效的结构化数据存储格式，与语言、平台无关，可扩展可序列化。protobuf 以二进制方式存储，占用空间小。

protobuf 的安装和使用教程 [Go Protobuf 简明教程](https://geektutu.com/post/quick-go-protobuf.html)。

protobuf 广泛地应用于远程过程调用(RPC) 的二进制传输，使用 protobuf 的目的非常简单，为了获得更高的性能。传输前使用 protobuf 编码，接收方再进行解码，可以显著地降低二进制传输的大小。另外一方面，protobuf 可非常适合传输结构化数据，便于通信字段的扩展。

使用 protobuf 一般分为以下 2 步：

- 按照 protobuf 的语法，在 `.proto` 文件中定义数据结构，并使用 `protoc` 生成 Go 代码（`.proto` 文件是跨平台的，还可以生成 C、Java 等其他源码文件）。
- 在项目代码中引用生成的 Go 代码。

## 2 使用 protobuf 通信

**新建 package `geecachepb`，定义 `geecachepb.proto`**

[geecache/geecachepb/geecachepb.proto]()

```go
syntax = "proto3";
option go_package = "./";
package geecachepb;

message Request {
  string group = 1;
  string key = 2;
}

message Response {
  bytes value = 1;
}

service GroupCache {
  rpc Get(Request) returns (Response);
}
```

- `Request` 包含 2 个字段， group 和 cache，这与我们之前定义的接口 `/_geecache/<group>/<name>` 所需的参数吻合。
- `Response` 包含 1 个字段，bytes，类型为 byte 数组，与之前吻合。

**接下来，修改 `peers.go` 中的 `PeerGetter` 接口，参数使用 `geecachepb.pb.go` 中的数据类型。**

```go
import pb "geeCache/geecachepb"

type PeerGetter interface {
	Get(in *pb.Request, out *pb.Response) error
}
```

**最后，修改 `geecache.go` 和 `http.go` 中使用了 `PeerGetter` 接口的地方。**

```go
// getFromPeer 使用实现了PeerGetter接口的httpGetter访问远程节点，获取缓存值
func (g *Group) getFromPeer(peer PeerGetter, key string) (ByteView, error) {
   req := &pb.Request{
      Group: g.name,
      Key:   key,
   }
   res := &pb.Response{}
   err := peer.Get(req, res)
   if err != nil {
      return ByteView{}, err
   }
   return ByteView{b: res.Value}, nil
}
```

```go
func (p *HTTPPool) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	// (1) 判断路径前缀是否为basePath
	if !strings.HasPrefix(r.URL.Path, p.basePath) {
		panic("HTTPPool serving unexpected path: " + r.URL.Path)
	}
	p.Log("%s %s", r.Method, r.URL.Path)

	// 约定访问路径格式为 /<basepath>/<groupname>/<key> required
	parts := strings.SplitN(r.URL.Path[len(p.basePath):], "/", 2)
	if len(parts) != 2 {
		http.Error(w, "bad request", http.StatusBadRequest)
		return
	}

	// 通过groupName得到group实例
	groupName := parts[0]
	group := GetGroup(groupName)
	if group == nil {
		http.Error(w, "no such group:"+groupName, http.StatusNotFound)
		return
	}

	// 通过group.Get(key)得到缓存数据
	key := parts[1]
	view, err := group.Get(key)
	if err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
		return
	}

	body, err := proto.Marshal(&pb.Response{Value: view.ByteSlice()})
	if err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
		return
	}
	// 将缓存值作为httpResponse的body返回
	w.Header().Set("Content-Type", "application/octet-stream")
	w.Write(body)

}

func (h *httpGetter) Get(in *pb.Request, out *pb.Response) error {
   u := fmt.Sprintf(
      "%v%v/%v",
      h.baseURL,
      url.QueryEscape(in.GetGroup()),
      url.QueryEscape(in.GetKey()),
   )

   // 使用http.Get() 获取返回值
   res, err := http.Get(u)
   if err != nil {
      return err
   }
   defer res.Body.Close()

   if res.StatusCode != http.StatusOK {
      return fmt.Errorf("server returned: %v", res.Status)
   }
   // 并转换为[]byte类型
   bytes, err := ioutil.ReadAll(res.Body)
   if err != nil {
      return fmt.Errorf("reading response body: %v", err)
   }

   if err = proto.Unmarshal(bytes, out); err != nil {
      return fmt.Errorf("decoding response body: %v", err)
   }

   return nil
}
```

- `ServeHTTP()` 中使用 `proto.Marshal()` 编码 HTTP 响应。
- `Get()` 中使用 `proto.Unmarshal()` 解码 HTTP 响应。

至此，我们已经将 HTTP 通信的中间载体替换成了 protobuf。运行 `run.sh` 即可以测试 GeeCache 能否正常工作。