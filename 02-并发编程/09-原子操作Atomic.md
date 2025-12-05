# 09-原子操作Atomic

[← 返回本章目录](./README.md) | [← 返回总目录](../README.md)

---

## 📋 目录

- [核心概念](#核心概念)
- [基本操作](#基本操作)
- [实现原理](#实现原理)
- [高级用法](#高级用法)
- [代码示例](#代码示例)
- [常见问题](#常见问题)
- [面试题](#面试题)
- [最佳实践](#最佳实践)
- [参考资料](#参考资料)

---

## 核心概念

原子操作（Atomic Operations）是指不可被中断的操作，在执行过程中不会被其他线程干扰。Go 语言通过 `sync/atomic` 包提供了一组原子操作函数。

```
原子操作的核心特性：
1. 不可分割：操作要么完全执行，要么完全不执行
2. 无锁实现：基于 CPU 指令，性能优于互斥锁
3. 并发安全：多个 goroutine 同时操作不会出现竞态条件
4. 内存可见性：保证操作结果对所有 goroutine 可见
5. 支持类型：int32、int64、uint32、uint64、uintptr、unsafe.Pointer
```

---

## 基本操作

### 1. 加载（Load）

```go
package main

import (
	"fmt"
	"sync/atomic"
)

func main() {
	var count int64 = 100

	// 原子加载
	value := atomic.LoadInt64(&count)
	fmt.Println("当前值:", value) // 100
}
```

### 2. 存储（Store）

```go
package main

import (
	"fmt"
	"sync/atomic"
)

func main() {
	var count int64

	// 原子存储
	atomic.StoreInt64(&count, 100)
	fmt.Println("存储后:", atomic.LoadInt64(&count)) // 100
}
```

### 3. 增加（Add）

```go
package main

import (
	"fmt"
	"sync"
	"sync/atomic"
)

func main() {
	var count int64

	var wg sync.WaitGroup
	for i := 0; i < 1000; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			atomic.AddInt64(&count, 1)
		}()
	}

	wg.Wait()
	fmt.Println("最终值:", count) // 1000
}
```

### 4. 交换（Swap）

```go
package main

import (
	"fmt"
	"sync/atomic"
)

func main() {
	var count int64 = 100

	// 原子交换：返回旧值，设置新值
	old := atomic.SwapInt64(&count, 200)
	fmt.Println("旧值:", old)   // 100
	fmt.Println("新值:", count) // 200
}
```

### 5. 比较并交换（CompareAndSwap，CAS）

```go
package main

import (
	"fmt"
	"sync/atomic"
)

func main() {
	var count int64 = 100

	// CAS：如果当前值等于 old，则设置为 new
	swapped := atomic.CompareAndSwapInt64(&count, 100, 200)
	fmt.Println("是否交换:", swapped) // true
	fmt.Println("当前值:", count)    // 200

	// 再次尝试（失败）
	swapped = atomic.CompareAndSwapInt64(&count, 100, 300)
	fmt.Println("是否交换:", swapped) // false
	fmt.Println("当前值:", count)    // 200
}
```

---

## 实现原理

### 1. CPU 指令支持

```
原子操作基于 CPU 提供的原子指令：

x86/x64 架构：
- LOCK 前缀：确保指令的原子性
- CMPXCHG：比较并交换
- XADD：交换并加
- XCHG：交换

ARM 架构：
- LDREX/STREX：加载独占/存储独占
- DMB：数据内存屏障

这些指令由硬件保证原子性，无需软件锁
```

### 2. 内存模型

```go
// 原子操作保证的内存顺序：

// 1. Load：获取最新值
value := atomic.LoadInt64(&count)

// 2. Store：立即对所有 goroutine 可见
atomic.StoreInt64(&count, 100)

// 3. Add：原子增加并返回新值
new := atomic.AddInt64(&count, 1)

// 4. CAS：比较并交换
swapped := atomic.CompareAndSwapInt64(&count, old, new)
```

### 3. CAS 实现原理

```go
// CAS 的伪代码实现：
func CompareAndSwap(addr *int64, old, new int64) bool {
	// 以下操作是原子的
	if *addr == old {
		*addr = new
		return true
	}
	return false
}

// 实际实现使用 CPU 的 CMPXCHG 指令
```

---

## 高级用法

### 1. 原子计数器

```go
package main

import (
	"fmt"
	"sync"
	"sync/atomic"
)

// Counter 原子计数器
type Counter struct {
	value int64
}

// Increment 增加
func (c *Counter) Increment() int64 {
	return atomic.AddInt64(&c.value, 1)
}

// Decrement 减少
func (c *Counter) Decrement() int64 {
	return atomic.AddInt64(&c.value, -1)
}

// Get 获取当前值
func (c *Counter) Get() int64 {
	return atomic.LoadInt64(&c.value)
}

// Set 设置值
func (c *Counter) Set(value int64) {
	atomic.StoreInt64(&c.value, value)
}

func main() {
	counter := &Counter{}

	var wg sync.WaitGroup
	for i := 0; i < 1000; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			counter.Increment()
		}()
	}

	wg.Wait()
	fmt.Println("最终值:", counter.Get()) // 1000
}
```

### 2. 原子布尔值

```go
package main

import (
	"fmt"
	"sync/atomic"
)

// AtomicBool 原子布尔值
type AtomicBool struct {
	value int32
}

// Set 设置为 true
func (b *AtomicBool) Set() {
	atomic.StoreInt32(&b.value, 1)
}

// Unset 设置为 false
func (b *AtomicBool) Unset() {
	atomic.StoreInt32(&b.value, 0)
}

// IsSet 是否为 true
func (b *AtomicBool) IsSet() bool {
	return atomic.LoadInt32(&b.value) == 1
}

// SetTo 设置为指定值
func (b *AtomicBool) SetTo(value bool) {
	if value {
		atomic.StoreInt32(&b.value, 1)
	} else {
		atomic.StoreInt32(&b.value, 0)
	}
}

func main() {
	flag := &AtomicBool{}

	flag.Set()
	fmt.Println("是否设置:", flag.IsSet()) // true

	flag.Unset()
	fmt.Println("是否设置:", flag.IsSet()) // false
}
```

### 3. 自旋锁

```go
package main

import (
	"fmt"
	"runtime"
	"sync"
	"sync/atomic"
)

// SpinLock 自旋锁
type SpinLock struct {
	state int32
}

// Lock 加锁
func (s *SpinLock) Lock() {
	for !atomic.CompareAndSwapInt32(&s.state, 0, 1) {
		runtime.Gosched() // 让出 CPU
	}
}

// Unlock 解锁
func (s *SpinLock) Unlock() {
	atomic.StoreInt32(&s.state, 0)
}

func main() {
	var lock SpinLock
	var count int

	var wg sync.WaitGroup
	for i := 0; i < 1000; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			lock.Lock()
			count++
			lock.Unlock()
		}()
	}

	wg.Wait()
	fmt.Println("最终值:", count) // 1000
}
```

### 4. 原子指针

```go
package main

import (
	"fmt"
	"sync/atomic"
	"unsafe"
)

// Config 配置
type Config struct {
	Host string
	Port int
}

var configPtr unsafe.Pointer

// LoadConfig 加载配置
func LoadConfig() *Config {
	return (*Config)(atomic.LoadPointer(&configPtr))
}

// StoreConfig 存储配置
func StoreConfig(config *Config) {
	atomic.StorePointer(&configPtr, unsafe.Pointer(config))
}

func main() {
	// 初始配置
	StoreConfig(&Config{
		Host: "localhost",
		Port: 8080,
	})

	// 读取配置
	cfg := LoadConfig()
	fmt.Printf("配置: %s:%d\n", cfg.Host, cfg.Port)

	// 更新配置
	StoreConfig(&Config{
		Host: "example.com",
		Port: 9090,
	})

	// 读取新配置
	cfg = LoadConfig()
	fmt.Printf("新配置: %s:%d\n", cfg.Host, cfg.Port)
}
```

### 5. 无锁队列（简化版）

```go
package main

import (
	"fmt"
	"sync/atomic"
	"unsafe"
)

// Node 节点
type Node struct {
	value interface{}
	next  unsafe.Pointer // *Node
}

// LockFreeQueue 无锁队列
type LockFreeQueue struct {
	head unsafe.Pointer // *Node
	tail unsafe.Pointer // *Node
}

// NewLockFreeQueue 创建队列
func NewLockFreeQueue() *LockFreeQueue {
	node := unsafe.Pointer(&Node{})
	return &LockFreeQueue{
		head: node,
		tail: node,
	}
}

// Enqueue 入队
func (q *LockFreeQueue) Enqueue(value interface{}) {
	node := &Node{value: value}
	for {
		tail := (*Node)(atomic.LoadPointer(&q.tail))
		next := atomic.LoadPointer(&tail.next)

		if next == nil {
			if atomic.CompareAndSwapPointer(&tail.next, nil, unsafe.Pointer(node)) {
				atomic.CompareAndSwapPointer(&q.tail, unsafe.Pointer(tail), unsafe.Pointer(node))
				return
			}
		} else {
			atomic.CompareAndSwapPointer(&q.tail, unsafe.Pointer(tail), next)
		}
	}
}

// Dequeue 出队
func (q *LockFreeQueue) Dequeue() (interface{}, bool) {
	for {
		head := (*Node)(atomic.LoadPointer(&q.head))
		tail := (*Node)(atomic.LoadPointer(&q.tail))
		next := (*Node)(atomic.LoadPointer(&head.next))

		if head == tail {
			if next == nil {
				return nil, false
			}
			atomic.CompareAndSwapPointer(&q.tail, unsafe.Pointer(tail), unsafe.Pointer(next))
		} else {
			value := next.value
			if atomic.CompareAndSwapPointer(&q.head, unsafe.Pointer(head), unsafe.Pointer(next)) {
				return value, true
			}
		}
	}
}

func main() {
	queue := NewLockFreeQueue()

	// 入队
	queue.Enqueue(1)
	queue.Enqueue(2)
	queue.Enqueue(3)

	// 出队
	for {
		value, ok := queue.Dequeue()
		if !ok {
			break
		}
		fmt.Println("出队:", value)
	}
}
```

---

## 代码示例

### 示例 1: 并发计数

```go
package main

import (
	"fmt"
	"sync"
	"sync/atomic"
)

func main() {
	var (
		counter int64
		wg      sync.WaitGroup
	)

	// 启动 100 个 goroutine，每个增加 1000 次
	for i := 0; i < 100; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for j := 0; j < 1000; j++ {
				atomic.AddInt64(&counter, 1)
			}
		}()
	}

	wg.Wait()
	fmt.Println("最终计数:", counter) // 100000
}
```

### 示例 2: 配置热更新

```go
package main

import (
	"fmt"
	"sync/atomic"
	"time"
	"unsafe"
)

// ServerConfig 服务器配置
type ServerConfig struct {
	MaxConnections int
	Timeout        time.Duration
}

var configPtr unsafe.Pointer

// GetConfig 获取配置
func GetConfig() *ServerConfig {
	return (*ServerConfig)(atomic.LoadPointer(&configPtr))
}

// UpdateConfig 更新配置
func UpdateConfig(config *ServerConfig) {
	atomic.StorePointer(&configPtr, unsafe.Pointer(config))
}

func main() {
	// 初始配置
	UpdateConfig(&ServerConfig{
		MaxConnections: 100,
		Timeout:        30 * time.Second,
	})

	// 模拟服务运行
	go func() {
		for i := 0; i < 5; i++ {
			cfg := GetConfig()
			fmt.Printf("当前配置: MaxConn=%d, Timeout=%v\n",
				cfg.MaxConnections, cfg.Timeout)
			time.Sleep(time.Second)
		}
	}()

	// 2 秒后更新配置
	time.Sleep(2 * time.Second)
	UpdateConfig(&ServerConfig{
		MaxConnections: 200,
		Timeout:        60 * time.Second,
	})

	time.Sleep(4 * time.Second)
}
```

### 示例 3: 状态标志

```go
package main

import (
	"fmt"
	"sync/atomic"
	"time"
)

// Server 服务器
type Server struct {
	running int32
}

// Start 启动服务器
func (s *Server) Start() bool {
	if !atomic.CompareAndSwapInt32(&s.running, 0, 1) {
		return false // 已经在运行
	}

	go func() {
		fmt.Println("服务器启动")
		for atomic.LoadInt32(&s.running) == 1 {
			fmt.Println("服务器运行中...")
			time.Sleep(time.Second)
		}
		fmt.Println("服务器停止")
	}()

	return true
}

// Stop 停止服务器
func (s *Server) Stop() bool {
	if !atomic.CompareAndSwapInt32(&s.running, 1, 0) {
		return false // 已经停止
	}
	return true
}

// IsRunning 是否运行中
func (s *Server) IsRunning() bool {
	return atomic.LoadInt32(&s.running) == 1
}

func main() {
	server := &Server{}

	// 启动服务器
	if server.Start() {
		fmt.Println("启动成功")
	}

	// 等待 3 秒
	time.Sleep(3 * time.Second)

	// 停止服务器
	if server.Stop() {
		fmt.Println("停止成功")
	}

	time.Sleep(time.Second)
}
```

### 示例 4: 限流器

```go
package main

import (
	"fmt"
	"sync"
	"sync/atomic"
	"time"
)

// RateLimiter 限流器
type RateLimiter struct {
	rate     int64 // 每秒允许的请求数
	tokens   int64 // 当前令牌数
	lastTime int64 // 上次更新时间（纳秒）
}

// NewRateLimiter 创建限流器
func NewRateLimiter(rate int64) *RateLimiter {
	return &RateLimiter{
		rate:     rate,
		tokens:   rate,
		lastTime: time.Now().UnixNano(),
	}
}

// Allow 是否允许请求
func (r *RateLimiter) Allow() bool {
	now := time.Now().UnixNano()
	last := atomic.LoadInt64(&r.lastTime)

	// 计算应该添加的令牌数
	elapsed := now - last
	tokensToAdd := elapsed * r.rate / int64(time.Second)

	if tokensToAdd > 0 {
		// 更新时间和令牌数
		atomic.StoreInt64(&r.lastTime, now)
		newTokens := atomic.LoadInt64(&r.tokens) + tokensToAdd
		if newTokens > r.rate {
			newTokens = r.rate
		}
		atomic.StoreInt64(&r.tokens, newTokens)
	}

	// 尝试获取令牌
	for {
		tokens := atomic.LoadInt64(&r.tokens)
		if tokens <= 0 {
			return false
		}
		if atomic.CompareAndSwapInt64(&r.tokens, tokens, tokens-1) {
			return true
		}
	}
}

func main() {
	limiter := NewRateLimiter(10) // 每秒 10 个请求

	var wg sync.WaitGroup
	for i := 0; i < 20; i++ {
		wg.Add(1)
		go func(id int) {
			defer wg.Done()
			if limiter.Allow() {
				fmt.Printf("请求 %d 通过\n", id)
			} else {
				fmt.Printf("请求 %d 被限流\n", id)
			}
		}(i)
	}

	wg.Wait()
}
```

---

## 常见问题

### 1. 原子操作和互斥锁的区别？

```go
// 原子操作：无锁，性能高
var count int64
atomic.AddInt64(&count, 1)

// 互斥锁：有锁，性能较低
var (
	count int64
	mu    sync.Mutex
)
mu.Lock()
count++
mu.Unlock()
```

**选择建议：**
- 简单的计数、标志：使用原子操作
- 复杂的临界区：使用互斥锁

### 2. 为什么原子操作需要指针？

```go
// ✅ 正确：传递指针
var count int64
atomic.AddInt64(&count, 1)

// ❌ 错误：不能传递值
// atomic.AddInt64(count, 1) // 编译错误
```

**原因：** 原子操作需要直接修改内存地址的值

### 3. 原子操作支持哪些类型？

```go
// ✅ 支持的类型
var (
	i32 int32
	i64 int64
	u32 uint32
	u64 uint64
	ptr uintptr
	up  unsafe.Pointer
)

atomic.AddInt32(&i32, 1)
atomic.AddInt64(&i64, 1)
atomic.AddUint32(&u32, 1)
atomic.AddUint64(&u64, 1)
atomic.LoadPointer(&up)

// ❌ 不支持的类型
var (
	i   int     // 不支持
	str string  // 不支持
	b   bool    // 不支持（需要用 int32 模拟）
)
```

### 4. CAS 操作失败怎么办？

```go
// 方案 1: 重试
for {
	old := atomic.LoadInt64(&count)
	new := old + 1
	if atomic.CompareAndSwapInt64(&count, old, new) {
		break
	}
}

// 方案 2: 使用 Add
atomic.AddInt64(&count, 1)
```

---

## 面试题

### 1. 什么是原子操作？

**答案：**

原子操作是不可被中断的操作，在执行过程中不会被其他线程干扰。它基于 CPU 提供的原子指令实现，无需加锁即可保证并发安全。

**特点：**
- 不可分割
- 无锁实现
- 性能高
- 保证内存可见性

### 2. CAS 的 ABA 问题是什么？

**答案：**

**ABA 问题：** 值从 A 变为 B，再变回 A，CAS 操作无法检测到中间的变化。

```go
// 线程 1
old := atomic.LoadInt64(&value) // A
// ... 被中断 ...
atomic.CompareAndSwapInt64(&value, old, new) // 成功，但值已经变化过

// 线程 2（在线程 1 中断期间）
atomic.StoreInt64(&value, B) // A -> B
atomic.StoreInt64(&value, A) // B -> A
```

**解决方案：**
- 使用版本号
- 使用时间戳
- 使用双重 CAS

### 3. 原子操作的性能如何？

**答案：**

| 操作 | 性能 | 适用场景 |
|------|------|----------|
| 原子操作 | 最快 | 简单计数、标志 |
| 自旋锁 | 较快 | 临界区很小 |
| 互斥锁 | 较慢 | 临界区较大 |
| Channel | 最慢 | 需要通信 |

**性能对比：**
- 原子操作：~10ns
- 互斥锁：~20-30ns
- Channel：~100ns

### 4. 什么时候使用原子操作？

**答案：**

**适用场景：**
1. 简单的计数器
2. 状态标志
3. 配置热更新
4. 无锁数据结构

**不适用场景：**
1. 复杂的临界区
2. 需要多个变量同步
3. 需要条件等待

### 5. 如何实现原子布尔值？

**答案：**

```go
type AtomicBool struct {
	value int32
}

func (b *AtomicBool) Set() {
	atomic.StoreInt32(&b.value, 1)
}

func (b *AtomicBool) Unset() {
	atomic.StoreInt32(&b.value, 0)
}

func (b *AtomicBool) IsSet() bool {
	return atomic.LoadInt32(&b.value) == 1
}
```

---

## 最佳实践

### 1. 优先使用原子操作而非锁

```go
// ✅ 推荐：简单计数使用原子操作
var count int64
atomic.AddInt64(&count, 1)

// ❌ 不推荐：简单计数使用锁
var (
	count int64
	mu    sync.Mutex
)
mu.Lock()
count++
mu.Unlock()
```

### 2. 使用 CAS 实现无锁算法

```go
// ✅ 推荐：CAS 重试
for {
	old := atomic.LoadInt64(&value)
	new := compute(old)
	if atomic.CompareAndSwapInt64(&value, old, new) {
		break
	}
}
```

### 3. 注意内存对齐

```go
// ✅ 推荐：确保 64 位对齐
type Counter struct {
	value int64 // 放在结构体开头
	_     [56]byte // 缓存行填充
}

// ❌ 不推荐：可能未对齐
type Counter struct {
	flag  bool
	value int64 // 可能未对齐
}
```

### 4. 避免过度使用原子操作

```go
// ❌ 不推荐：复杂逻辑使用原子操作
atomic.AddInt64(&a, 1)
atomic.AddInt64(&b, 1)
atomic.AddInt64(&c, 1)

// ✅ 推荐：使用互斥锁
mu.Lock()
a++
b++
c++
mu.Unlock()
```

### 5. 使用 atomic.Value 存储任意类型

```go
// ✅ 推荐：使用 atomic.Value
var config atomic.Value
config.Store(&Config{})
cfg := config.Load().(*Config)
```

---

## 参考资料

- [x] [Go 官方文档 - sync/atomic](https://pkg.go.dev/sync/atomic)
- [x] [Go atomic 源码分析](https://github.com/golang/go/tree/master/src/sync/atomic)
- [x] [深入理解 Go 原子操作](https://colobu.com/2018/12/10/dive-into-go-atomic/)
- [x] [无锁编程实践](https://preshing.com/20120612/an-introduction-to-lock-free-programming/)

---

**上一节：** [08-Once单例模式](./08-Once单例模式.md)  
**下一节：** [10-并发模式](./10-并发模式.md)
