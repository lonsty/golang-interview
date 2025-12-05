# 05-Mutex互斥锁

[← 返回本章目录](./README.md) | [← 返回总目录](../README.md)

---

## 📋 目录

- [核心概念](#核心概念)
- [基本用法](#基本用法)
- [实现原理](#实现原理)
- [锁的状态](#锁的状态)
- [代码示例](#代码示例)
- [常见问题](#常见问题)
- [面试题](#面试题)
- [最佳实践](#最佳实践)
- [参考资料](#参考资料)

---

## 核心概念

Mutex（互斥锁）是 Go 语言提供的最基本的同步原语，用于保护共享资源的并发访问，确保同一时刻只有一个 goroutine 可以访问临界区。

```
Mutex 的核心特性：
1. 互斥性：同一时刻只有一个 goroutine 可以持有锁
2. 不可重入：同一个 goroutine 不能重复加锁（会死锁）
3. 零值可用：var mu sync.Mutex 即可使用
4. 不可复制：Mutex 不能被复制，应该通过指针传递
```

---

## 基本用法

### 1. 简单示例

```go
package main

import (
    "fmt"
    "sync"
)

var (
    counter int
    mu      sync.Mutex
)

func increment() {
    mu.Lock()
    defer mu.Unlock()
    counter++
}

func main() {
    var wg sync.WaitGroup
    
    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            increment()
        }()
    }
    
    wg.Wait()
    fmt.Println("Counter:", counter) // 输出: Counter: 1000
}
```

### 2. 保护结构体

```go
package main

import (
    "fmt"
    "sync"
)

// SafeCounter 线程安全的计数器
type SafeCounter struct {
    mu    sync.Mutex
    count int
}

// Inc 增加计数
func (c *SafeCounter) Inc() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.count++
}

// Value 获取计数值
func (c *SafeCounter) Value() int {
    c.mu.Lock()
    defer c.mu.Unlock()
    return c.count
}

func main() {
    counter := &SafeCounter{}
    var wg sync.WaitGroup
    
    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            counter.Inc()
        }()
    }
    
    wg.Wait()
    fmt.Println("Final count:", counter.Value())
}
```

### 3. 使用 defer 释放锁

```go
package main

import (
    "fmt"
    "sync"
)

var (
    data map[string]int
    mu   sync.Mutex
)

func init() {
    data = make(map[string]int)
}

// Set 设置值
func Set(key string, value int) {
    mu.Lock()
    defer mu.Unlock() // 确保锁一定会被释放
    data[key] = value
}

// Get 获取值
func Get(key string) (int, bool) {
    mu.Lock()
    defer mu.Unlock()
    value, ok := data[key]
    return value, ok
}

func main() {
    Set("age", 18)
    if value, ok := Get("age"); ok {
        fmt.Println("age:", value)
    }
}
```

---

## 实现原理

### 1. Mutex 的结构

```go
type Mutex struct {
    state int32  // 锁的状态
    sema  uint32 // 信号量
}

// state 的位含义：
// 第 0 位：是否被锁定（mutexLocked）
// 第 1 位：是否被唤醒（mutexWoken）
// 第 2 位：是否处于饥饿模式（mutexStarving）
// 其余位：等待的 goroutine 数量
```

### 2. 两种模式

```
正常模式（Normal Mode）：
- 等待的 goroutine 按照 FIFO 顺序排队
- 被唤醒的 goroutine 与新到达的 goroutine 竞争锁
- 新到达的 goroutine 更容易获得锁（已经在 CPU 上运行）
- 被唤醒的 goroutine 可能获取不到锁，继续等待

饥饿模式（Starvation Mode）：
- 当一个 goroutine 等待超过 1ms 时，切换到饥饿模式
- 在饥饿模式下，锁直接交给等待队列最前面的 goroutine
- 新到达的 goroutine 不会尝试获取锁，直接加入队列尾部
- 当最后一个等待者获得锁或等待时间小于 1ms 时，切换回正常模式
```

### 3. Lock 流程

```go
// Lock 的简化流程
func (m *Mutex) Lock() {
    // 快速路径：尝试直接获取锁
    if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
        return
    }
    
    // 慢速路径：自旋或阻塞等待
    m.lockSlow()
}

func (m *Mutex) lockSlow() {
    var waitStartTime int64
    starving := false
    awoke := false
    iter := 0
    old := m.state
    
    for {
        // 1. 自旋尝试获取锁（正常模式且满足自旋条件）
        if old&(mutexLocked|mutexStarving) == mutexLocked && runtime_canSpin(iter) {
            // 设置 mutexWoken 标志
            if !awoke && old&mutexWoken == 0 && old>>mutexWaiterShift != 0 &&
                atomic.CompareAndSwapInt32(&m.state, old, old|mutexWoken) {
                awoke = true
            }
            runtime_doSpin()
            iter++
            old = m.state
            continue
        }
        
        // 2. 计算新状态
        new := old
        if old&mutexStarving == 0 {
            new |= mutexLocked // 尝试加锁
        }
        if old&(mutexLocked|mutexStarving) != 0 {
            new += 1 << mutexWaiterShift // 等待者+1
        }
        if starving && old&mutexLocked != 0 {
            new |= mutexStarving // 设置饥饿模式
        }
        if awoke {
            new &^= mutexWoken // 清除唤醒标志
        }
        
        // 3. 尝试更新状态
        if atomic.CompareAndSwapInt32(&m.state, old, new) {
            // 获取到锁，返回
            if old&(mutexLocked|mutexStarving) == 0 {
                break
            }
            
            // 阻塞等待
            runtime_SemacquireMutex(&m.sema, queueLifo, 1)
            
            // 被唤醒，检查是否进入饥饿模式
            starving = starving || runtime_nanotime()-waitStartTime > starvationThresholdNs
            old = m.state
            
            // 饥饿模式下直接获得锁
            if old&mutexStarving != 0 {
                delta := int32(mutexLocked - 1<<mutexWaiterShift)
                if !starving || old>>mutexWaiterShift == 1 {
                    delta -= mutexStarving // 退出饥饿模式
                }
                atomic.AddInt32(&m.state, delta)
                break
            }
            awoke = true
            iter = 0
        } else {
            old = m.state
        }
    }
}
```

### 4. Unlock 流程

```go
// Unlock 的简化流程
func (m *Mutex) Unlock() {
    // 快速路径：直接释放锁
    new := atomic.AddInt32(&m.state, -mutexLocked)
    if new != 0 {
        m.unlockSlow(new)
    }
}

func (m *Mutex) unlockSlow(new int32) {
    // 检查是否重复解锁
    if (new+mutexLocked)&mutexLocked == 0 {
        throw("sync: unlock of unlocked mutex")
    }
    
    // 正常模式
    if new&mutexStarving == 0 {
        old := new
        for {
            // 没有等待者或已经有 goroutine 被唤醒
            if old>>mutexWaiterShift == 0 || old&(mutexLocked|mutexWoken|mutexStarving) != 0 {
                return
            }
            // 唤醒一个等待者
            new = (old - 1<<mutexWaiterShift) | mutexWoken
            if atomic.CompareAndSwapInt32(&m.state, old, new) {
                runtime_Semrelease(&m.sema, false, 1)
                return
            }
            old = m.state
        }
    } else {
        // 饥饿模式：直接唤醒队首的 goroutine
        runtime_Semrelease(&m.sema, true, 1)
    }
}
```

---

## 锁的状态

### 1. 未锁定状态

```go
// state = 0
// 二进制: 00000000 00000000 00000000 00000000
// - 未被锁定
// - 没有等待者
// - 正常模式
```

### 2. 已锁定状态

```go
// state = 1
// 二进制: 00000000 00000000 00000000 00000001
// - 已被锁定
// - 没有等待者
// - 正常模式
```

### 3. 有等待者

```go
// state = 5 (二进制: 101)
// - 已被锁定（第 0 位为 1）
// - 有 1 个等待者（高 29 位为 1）
// - 正常模式
```

### 4. 饥饿模式

```go
// state = 5 (二进制: 101)
// - 已被锁定（第 0 位为 1）
// - 饥饿模式（第 2 位为 1）
// - 有等待者
```

---

## 代码示例

### 示例 1: 银行账户

```go
package main

import (
    "fmt"
    "sync"
)

// Account 银行账户
type Account struct {
    mu      sync.Mutex
    balance int
}

// Deposit 存款
func (a *Account) Deposit(amount int) {
    a.mu.Lock()
    defer a.mu.Unlock()
    a.balance += amount
}

// Withdraw 取款
func (a *Account) Withdraw(amount int) bool {
    a.mu.Lock()
    defer a.mu.Unlock()
    
    if a.balance >= amount {
        a.balance -= amount
        return true
    }
    return false
}

// Balance 查询余额
func (a *Account) Balance() int {
    a.mu.Lock()
    defer a.mu.Unlock()
    return a.balance
}

func main() {
    account := &Account{balance: 1000}
    var wg sync.WaitGroup
    
    // 10 个 goroutine 并发存款
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            account.Deposit(100)
        }()
    }
    
    // 5 个 goroutine 并发取款
    for i := 0; i < 5; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            account.Withdraw(50)
        }()
    }
    
    wg.Wait()
    fmt.Println("最终余额:", account.Balance())
}
```

### 示例 2: 缓存

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

// Cache 简单缓存
type Cache struct {
    mu   sync.Mutex
    data map[string]string
}

// NewCache 创建缓存
func NewCache() *Cache {
    return &Cache{
        data: make(map[string]string),
    }
}

// Set 设置缓存
func (c *Cache) Set(key, value string) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.data[key] = value
}

// Get 获取缓存
func (c *Cache) Get(key string) (string, bool) {
    c.mu.Lock()
    defer c.mu.Unlock()
    value, ok := c.data[key]
    return value, ok
}

// Delete 删除缓存
func (c *Cache) Delete(key string) {
    c.mu.Lock()
    defer c.mu.Unlock()
    delete(c.data, key)
}

func main() {
    cache := NewCache()
    
    // 写入
    cache.Set("name", "Alice")
    cache.Set("age", "18")
    
    // 读取
    if value, ok := cache.Get("name"); ok {
        fmt.Println("name:", value)
    }
    
    // 删除
    cache.Delete("age")
    
    if _, ok := cache.Get("age"); !ok {
        fmt.Println("age 已删除")
    }
}
```

---

## 常见问题

### 1. 为什么 Mutex 不可重入？

```go
// ❌ 错误：重复加锁会死锁
func recursive() {
    mu.Lock()
    defer mu.Unlock()
    
    // 再次加锁会死锁
    mu.Lock() // 死锁！
    defer mu.Unlock()
}

// ✅ 正确：避免重复加锁
func correct() {
    mu.Lock()
    defer mu.Unlock()
    // 不要再次加锁
}
```

### 2. 为什么要使用 defer 释放锁？

```go
// ❌ 不推荐：可能忘记释放锁
func bad() {
    mu.Lock()
    // 如果这里 panic，锁不会被释放
    doSomething()
    mu.Unlock()
}

// ✅ 推荐：使用 defer 确保释放
func good() {
    mu.Lock()
    defer mu.Unlock()
    // 即使 panic，锁也会被释放
    doSomething()
}
```

### 3. Mutex 可以复制吗？

```go
// ❌ 错误：不能复制 Mutex
type Counter struct {
    mu    sync.Mutex
    count int
}

func main() {
    c1 := Counter{}
    c2 := c1 // 错误：复制了 Mutex
}

// ✅ 正确：使用指针
func main() {
    c1 := &Counter{}
    c2 := c1 // 正确：共享同一个 Mutex
}
```

---

## 面试题

### 1. Mutex 的实现原理是什么？

**答案：**

Mutex 使用 state 和 sema 两个字段实现：

**state 字段（int32）：**
- 第 0 位：是否被锁定
- 第 1 位：是否被唤醒
- 第 2 位：是否处于饥饿模式
- 其余位：等待的 goroutine 数量

**两种模式：**
1. **正常模式**：等待者按 FIFO 排队，但新到达的 goroutine 可以插队
2. **饥饿模式**：等待超过 1ms 切换，锁直接交给队首等待者

**Lock 流程：**
1. 快速路径：CAS 尝试获取锁
2. 自旋：在多核且等待者少时自旋
3. 阻塞：加入等待队列并阻塞

### 2. 为什么需要饥饿模式？

**答案：**

**问题：** 在正常模式下，新到达的 goroutine 更容易获得锁（已经在 CPU 上运行），导致等待队列中的 goroutine 可能长时间获取不到锁（饥饿）。

**解决：** 引入饥饿模式：
- 当等待超过 1ms 时切换到饥饿模式
- 饥饿模式下锁直接交给队首等待者
- 保证公平性，避免 goroutine 饥饿

### 3. Mutex 和 Channel 如何选择？

**答案：**

| 场景 | 选择 | 原因 |
|------|------|------|
| 保护共享状态 | Mutex | 简单直接 |
| 传递数据 | Channel | 更符合 Go 哲学 |
| 简单计数器 | Mutex | 性能更好 |
| 协调多个 goroutine | Channel | 更灵活 |

**原则：** 通过通信共享内存，而不是通过共享内存通信。

---

## 最佳实践

### 1. 使用 defer 释放锁

```go
// ✅ 推荐
func process() {
    mu.Lock()
    defer mu.Unlock()
    // 处理逻辑
}
```

### 2. 最小化锁的范围

```go
// ✅ 推荐：最小化锁的范围
mu.Lock()
data := sharedData
mu.Unlock()

// 在锁外处理数据
process(data)
```

### 3. 避免在持有锁时调用外部函数

```go
// ❌ 不推荐
mu.Lock()
externalFunc() // 可能导致死锁
mu.Unlock()

// ✅ 推荐
mu.Lock()
data := getData()
mu.Unlock()
externalFunc(data)
```

### 4. 使用组合而非继承

```go
// ✅ 推荐：将 Mutex 作为字段
type SafeMap struct {
    mu   sync.Mutex
    data map[string]int
}
```

---

## 参考资料

- [x] [Go 官方文档 - sync.Mutex](https://pkg.go.dev/sync#Mutex)
- [x] [Go Mutex 源码分析](https://github.com/golang/go/blob/master/src/sync/mutex.go)
- [x] [深入理解 Go Mutex](https://colobu.com/2018/12/18/dive-into-go-mutex/)

---

**上一节：** [04-Context上下文](./04-Context上下文.md)  
**下一节：** [06-RWMutex读写锁](./06-RWMutex读写锁.md)
