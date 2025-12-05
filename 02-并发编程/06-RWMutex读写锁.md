# 06-RWMutex读写锁

[← 返回本章目录](./README.md) | [← 返回总目录](../README.md)

---

## 📋 目录

- [核心概念](#核心概念)
- [基本用法](#基本用法)
- [实现原理](#实现原理)
- [性能对比](#性能对比)
- [代码示例](#代码示例)
- [常见问题](#常见问题)
- [面试题](#面试题)
- [最佳实践](#最佳实践)
- [参考资料](#参考资料)

---

## 核心概念

RWMutex（读写锁）是 Go 语言提供的一种特殊的互斥锁，允许多个读操作同时进行，但写操作是互斥的。适用于读多写少的场景。

```
RWMutex 的核心特性：
1. 读锁共享：多个 goroutine 可以同时持有读锁
2. 写锁互斥：写锁与读锁、写锁互斥
3. 写优先：当有写锁等待时，新的读锁会被阻塞
4. 零值可用：var mu sync.RWMutex 即可使用
5. 不可复制：RWMutex 不能被复制
```

---

## 基本用法

### 1. 简单示例

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

var (
    data   map[string]int
    rwLock sync.RWMutex
)

func init() {
    data = make(map[string]int)
}

// 读操作
func read(key string) int {
    rwLock.RLock()
    defer rwLock.RUnlock()
    return data[key]
}

// 写操作
func write(key string, value int) {
    rwLock.Lock()
    defer rwLock.Unlock()
    data[key] = value
}

func main() {
    // 写入数据
    write("age", 18)
    
    // 多个 goroutine 并发读取
    var wg sync.WaitGroup
    for i := 0; i < 5; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            value := read("age")
            fmt.Printf("Goroutine %d read: %d\n", id, value)
        }(i)
    }
    
    wg.Wait()
}
```

### 2. 线程安全的缓存

```go
package main

import (
    "fmt"
    "sync"
)

// Cache 线程安全的缓存
type Cache struct {
    mu   sync.RWMutex
    data map[string]string
}

// NewCache 创建缓存
func NewCache() *Cache {
    return &Cache{
        data: make(map[string]string),
    }
}

// Get 读取缓存
func (c *Cache) Get(key string) (string, bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()
    value, ok := c.data[key]
    return value, ok
}

// Set 设置缓存
func (c *Cache) Set(key, value string) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.data[key] = value
}

// Delete 删除缓存
func (c *Cache) Delete(key string) {
    c.mu.Lock()
    defer c.mu.Unlock()
    delete(c.data, key)
}

// Len 获取缓存大小
func (c *Cache) Len() int {
    c.mu.RLock()
    defer c.mu.RUnlock()
    return len(c.data)
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
    
    fmt.Println("cache size:", cache.Len())
}
```

### 3. 配置管理器

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

// Config 配置管理器
type Config struct {
    mu     sync.RWMutex
    config map[string]interface{}
}

// NewConfig 创建配置管理器
func NewConfig() *Config {
    return &Config{
        config: make(map[string]interface{}),
    }
}

// Get 获取配置
func (c *Config) Get(key string) (interface{}, bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()
    value, ok := c.config[key]
    return value, ok
}

// Set 设置配置
func (c *Config) Set(key string, value interface{}) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.config[key] = value
}

// GetAll 获取所有配置（返回副本）
func (c *Config) GetAll() map[string]interface{} {
    c.mu.RLock()
    defer c.mu.RUnlock()
    
    // 返回副本，避免外部修改
    result := make(map[string]interface{}, len(c.config))
    for k, v := range c.config {
        result[k] = v
    }
    return result
}

// Reload 重新加载配置
func (c *Config) Reload(newConfig map[string]interface{}) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.config = newConfig
}

func main() {
    config := NewConfig()
    
    // 初始化配置
    config.Set("host", "localhost")
    config.Set("port", 8080)
    config.Set("timeout", 30*time.Second)
    
    // 多个 goroutine 并发读取配置
    var wg sync.WaitGroup
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            if host, ok := config.Get("host"); ok {
                fmt.Printf("Goroutine %d: host=%v\n", id, host)
            }
        }(i)
    }
    
    // 一个 goroutine 更新配置
    wg.Add(1)
    go func() {
        defer wg.Done()
        time.Sleep(10 * time.Millisecond)
        config.Set("host", "127.0.0.1")
        fmt.Println("Config updated")
    }()
    
    wg.Wait()
}
```

---

## 实现原理

### 1. RWMutex 的结构

```go
type RWMutex struct {
    w           Mutex  // 写锁
    writerSem   uint32 // 写等待信号量
    readerSem   uint32 // 读等待信号量
    readerCount int32  // 读者数量
    readerWait  int32  // 写锁需要等待的读者数量
}

const rwmutexMaxReaders = 1 << 30 // 最大读者数量
```

### 2. 读锁（RLock）流程

```go
// RLock 获取读锁
func (rw *RWMutex) RLock() {
    // 读者数量+1
    if atomic.AddInt32(&rw.readerCount, 1) < 0 {
        // 有写锁等待，阻塞当前读者
        runtime_SemacquireMutex(&rw.readerSem, false, 0)
    }
}

// RUnlock 释放读锁
func (rw *RWMutex) RUnlock() {
    // 读者数量-1
    if r := atomic.AddInt32(&rw.readerCount, -1); r < 0 {
        // 可能有写锁在等待
        rw.rUnlockSlow(r)
    }
}

func (rw *RWMutex) rUnlockSlow(r int32) {
    // 检查是否重复解锁
    if r+1 == 0 || r+1 == -rwmutexMaxReaders {
        throw("sync: RUnlock of unlocked RWMutex")
    }
    // 如果是最后一个读者，唤醒写者
    if atomic.AddInt32(&rw.readerWait, -1) == 0 {
        runtime_Semrelease(&rw.writerSem, false, 1)
    }
}
```

### 3. 写锁（Lock）流程

```go
// Lock 获取写锁
func (rw *RWMutex) Lock() {
    // 先获取互斥锁
    rw.w.Lock()
    
    // 将 readerCount 减去 rwmutexMaxReaders，表示有写锁等待
    // 新的读者会被阻塞
    r := atomic.AddInt32(&rw.readerCount, -rwmutexMaxReaders) + rwmutexMaxReaders
    
    // 如果还有读者，等待所有读者完成
    if r != 0 && atomic.AddInt32(&rw.readerWait, r) != 0 {
        runtime_SemacquireMutex(&rw.writerSem, false, 0)
    }
}

// Unlock 释放写锁
func (rw *RWMutex) Unlock() {
    // 恢复 readerCount
    r := atomic.AddInt32(&rw.readerCount, rwmutexMaxReaders)
    
    // 检查是否重复解锁
    if r >= rwmutexMaxReaders {
        throw("sync: Unlock of unlocked RWMutex")
    }
    
    // 唤醒所有等待的读者
    for i := 0; i < int(r); i++ {
        runtime_Semrelease(&rw.readerSem, false, 0)
    }
    
    // 释放互斥锁
    rw.w.Unlock()
}
```

### 4. 状态转换图

```
初始状态：readerCount = 0

读锁获取：
readerCount++
如果 readerCount < 0，说明有写锁等待，阻塞

写锁获取：
1. 获取 w 互斥锁
2. readerCount -= rwmutexMaxReaders（标记有写锁等待）
3. 如果还有读者，等待所有读者完成

写锁释放：
1. readerCount += rwmutexMaxReaders（恢复）
2. 唤醒所有等待的读者
3. 释放 w 互斥锁
```

---

## 性能对比

### 1. Mutex vs RWMutex

```go
package main

import (
    "fmt"
    "sync"
    "testing"
    "time"
)

var (
    data     = make(map[string]int)
    mutex    sync.Mutex
    rwMutex  sync.RWMutex
)

// 使用 Mutex 的读操作
func readWithMutex(key string) int {
    mutex.Lock()
    defer mutex.Unlock()
    return data[key]
}

// 使用 RWMutex 的读操作
func readWithRWMutex(key string) int {
    rwMutex.RLock()
    defer rwMutex.RUnlock()
    return data[key]
}

// 基准测试：Mutex 读操作
func BenchmarkMutexRead(b *testing.B) {
    data["key"] = 100
    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            readWithMutex("key")
        }
    })
}

// 基准测试：RWMutex 读操作
func BenchmarkRWMutexRead(b *testing.B) {
    data["key"] = 100
    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            readWithRWMutex("key")
        }
    })
}
```

**测试结果：**
```
BenchmarkMutexRead-8      10000000    150 ns/op
BenchmarkRWMutexRead-8    50000000     30 ns/op
```

**结论：** 在读多写少的场景下，RWMutex 性能明显优于 Mutex。

### 2. 读写比例影响

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

func testRWMutex(readRatio float64, duration time.Duration) {
    var (
        rwMutex    sync.RWMutex
        data       int
        readCount  int64
        writeCount int64
    )
    
    done := make(chan struct{})
    time.AfterFunc(duration, func() {
        close(done)
    })
    
    // 启动多个 goroutine
    var wg sync.WaitGroup
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for {
                select {
                case <-done:
                    return
                default:
                    // 根据读写比例决定操作
                    if time.Now().UnixNano()%100 < int64(readRatio*100) {
                        // 读操作
                        rwMutex.RLock()
                        _ = data
                        rwMutex.RUnlock()
                        readCount++
                    } else {
                        // 写操作
                        rwMutex.Lock()
                        data++
                        rwMutex.Unlock()
                        writeCount++
                    }
                }
            }
        }()
    }
    
    wg.Wait()
    fmt.Printf("读写比例 %.0f%%, 读操作: %d, 写操作: %d\n", 
        readRatio*100, readCount, writeCount)
}

func main() {
    testRWMutex(0.9, 1*time.Second) // 90% 读操作
    testRWMutex(0.5, 1*time.Second) // 50% 读操作
    testRWMutex(0.1, 1*time.Second) // 10% 读操作
}
```

---

## 代码示例

### 示例 1: 统计信息收集器

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

// Stats 统计信息
type Stats struct {
    mu     sync.RWMutex
    counts map[string]int64
}

// NewStats 创建统计信息收集器
func NewStats() *Stats {
    return &Stats{
        counts: make(map[string]int64),
    }
}

// Inc 增加计数
func (s *Stats) Inc(key string) {
    s.mu.Lock()
    defer s.mu.Unlock()
    s.counts[key]++
}

// Get 获取计数
func (s *Stats) Get(key string) int64 {
    s.mu.RLock()
    defer s.mu.RUnlock()
    return s.counts[key]
}

// Snapshot 获取快照
func (s *Stats) Snapshot() map[string]int64 {
    s.mu.RLock()
    defer s.mu.RUnlock()
    
    result := make(map[string]int64, len(s.counts))
    for k, v := range s.counts {
        result[k] = v
    }
    return result
}

func main() {
    stats := NewStats()
    var wg sync.WaitGroup
    
    // 多个 goroutine 并发写入
    for i := 0; i < 5; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            for j := 0; j < 100; j++ {
                stats.Inc(fmt.Sprintf("metric_%d", id))
                time.Sleep(time.Millisecond)
            }
        }(i)
    }
    
    // 一个 goroutine 定期读取
    wg.Add(1)
    go func() {
        defer wg.Done()
        ticker := time.NewTicker(100 * time.Millisecond)
        defer ticker.Stop()
        
        for i := 0; i < 10; i++ {
            <-ticker.C
            snapshot := stats.Snapshot()
            fmt.Printf("Snapshot %d: %v\n", i, snapshot)
        }
    }()
    
    wg.Wait()
}
```

### 示例 2: 服务注册中心

```go
package main

import (
    "fmt"
    "sync"
)

// Service 服务信息
type Service struct {
    Name    string
    Address string
    Port    int
}

// Registry 服务注册中心
type Registry struct {
    mu       sync.RWMutex
    services map[string]*Service
}

// NewRegistry 创建注册中心
func NewRegistry() *Registry {
    return &Registry{
        services: make(map[string]*Service),
    }
}

// Register 注册服务
func (r *Registry) Register(service *Service) {
    r.mu.Lock()
    defer r.mu.Unlock()
    r.services[service.Name] = service
}

// Unregister 注销服务
func (r *Registry) Unregister(name string) {
    r.mu.Lock()
    defer r.mu.Unlock()
    delete(r.services, name)
}

// Get 获取服务
func (r *Registry) Get(name string) (*Service, bool) {
    r.mu.RLock()
    defer r.mu.RUnlock()
    service, ok := r.services[name]
    return service, ok
}

// List 列出所有服务
func (r *Registry) List() []*Service {
    r.mu.RLock()
    defer r.mu.RUnlock()
    
    result := make([]*Service, 0, len(r.services))
    for _, service := range r.services {
        result = append(result, service)
    }
    return result
}

func main() {
    registry := NewRegistry()
    
    // 注册服务
    registry.Register(&Service{
        Name:    "user-service",
        Address: "localhost",
        Port:    8001,
    })
    
    registry.Register(&Service{
        Name:    "order-service",
        Address: "localhost",
        Port:    8002,
    })
    
    // 查询服务
    if service, ok := registry.Get("user-service"); ok {
        fmt.Printf("Found service: %s at %s:%d\n", 
            service.Name, service.Address, service.Port)
    }
    
    // 列出所有服务
    services := registry.List()
    fmt.Printf("Total services: %d\n", len(services))
}
```

---

## 常见问题

### 1. 什么时候使用 RWMutex？

```
使用 RWMutex 的场景：
✅ 读操作远多于写操作（读写比例 > 10:1）
✅ 读操作耗时较长
✅ 需要提高并发读性能

使用 Mutex 的场景：
✅ 读写操作比例接近
✅ 操作非常快速
✅ 代码简单性优先
```

### 2. RWMutex 可以升级/降级吗？

```go
// ❌ 错误：不能从读锁升级到写锁
func upgrade() {
    rwMutex.RLock()
    // 尝试升级到写锁
    rwMutex.Lock() // 死锁！
    rwMutex.RUnlock()
    rwMutex.Unlock()
}

// ✅ 正确：先释放读锁，再获取写锁
func correct() {
    rwMutex.RLock()
    data := readData()
    rwMutex.RUnlock()
    
    if needUpdate(data) {
        rwMutex.Lock()
        updateData()
        rwMutex.Unlock()
    }
}
```

### 3. 写锁是否优先？

**答案：** 是的。当有写锁等待时，新的读锁会被阻塞，避免写锁饥饿。

```go
// 写锁等待时，新的读锁会被阻塞
func example() {
    // Goroutine 1: 持有读锁
    rwMutex.RLock()
    time.Sleep(1 * time.Second)
    rwMutex.RUnlock()
    
    // Goroutine 2: 等待写锁
    rwMutex.Lock()
    // 写操作
    rwMutex.Unlock()
    
    // Goroutine 3: 新的读锁会被阻塞，直到写锁完成
    rwMutex.RLock()
    rwMutex.RUnlock()
}
```

---

## 面试题

### 1. RWMutex 的实现原理是什么？

**答案：**

RWMutex 使用以下字段实现：
- **w (Mutex)**：保护写操作的互斥锁
- **readerCount (int32)**：当前读者数量
  - 正数：读者数量
  - 负数：有写锁等待（实际读者数 = readerCount + rwmutexMaxReaders）
- **readerWait (int32)**：写锁需要等待的读者数量
- **writerSem/readerSem**：信号量，用于阻塞和唤醒

**读锁流程：**
1. readerCount++
2. 如果 < 0，说明有写锁等待，阻塞

**写锁流程：**
1. 获取 w 互斥锁
2. readerCount -= rwmutexMaxReaders（标记有写锁）
3. 等待所有读者完成

### 2. RWMutex 和 Mutex 的性能差异？

**答案：**

| 场景 | Mutex | RWMutex | 推荐 |
|------|-------|---------|------|
| 读多写少（90%读） | 慢 | 快 | RWMutex |
| 读写均衡（50%读） | 快 | 慢 | Mutex |
| 写多读少（10%读） | 快 | 慢 | Mutex |

**原因：** RWMutex 需要维护更多状态，开销更大。只有在读操作占主导时才有优势。

### 3. 如何避免 RWMutex 死锁？

**答案：**

```go
// ❌ 死锁场景 1：读锁升级
rwMutex.RLock()
rwMutex.Lock() // 死锁！

// ❌ 死锁场景 2：重复加锁
rwMutex.Lock()
rwMutex.Lock() // 死锁！

// ✅ 正确做法
rwMutex.RLock()
data := readData()
rwMutex.RUnlock()

if needUpdate(data) {
    rwMutex.Lock()
    updateData()
    rwMutex.Unlock()
}
```

---

## 最佳实践

### 1. 使用 defer 释放锁

```go
// ✅ 推荐
func read() {
    rwMutex.RLock()
    defer rwMutex.RUnlock()
    // 读操作
}
```

### 2. 最小化锁的范围

```go
// ✅ 推荐
rwMutex.RLock()
data := copyData()
rwMutex.RUnlock()

// 在锁外处理数据
process(data)
```

### 3. 返回数据副本

```go
// ✅ 推荐：返回副本
func GetAll() map[string]int {
    rwMutex.RLock()
    defer rwMutex.RUnlock()
    
    result := make(map[string]int, len(data))
    for k, v := range data {
        result[k] = v
    }
    return result
}
```

### 4. 选择合适的锁类型

```go
// 读多写少：使用 RWMutex
type Cache struct {
    mu   sync.RWMutex
    data map[string]string
}

// 读写均衡：使用 Mutex
type Counter struct {
    mu    sync.Mutex
    count int
}
```

---

## 参考资料

- [x] [Go 官方文档 - sync.RWMutex](https://pkg.go.dev/sync#RWMutex)
- [x] [Go RWMutex 源码分析](https://github.com/golang/go/blob/master/src/sync/rwmutex.go)
- [x] [深入理解 Go RWMutex](https://colobu.com/2019/10/10/dive-into-go-rwmutex/)

---

**上一节：** [05-Mutex互斥锁](./05-Mutex互斥锁.md)  
**下一节：** [07-WaitGroup](./07-WaitGroup.md)
