# 03-Select多路复用

[← 返回本章目录](./README.md) | [← 返回总目录](../README.md)

---

## 📋 目录

- [核心概念](#核心概念)
- [基本语法](#基本语法)
- [使用场景](#使用场景)
- [实现原理](#实现原理)
- [代码示例](#代码示例)
- [常见问题](#常见问题)
- [面试题](#面试题)
- [最佳实践](#最佳实践)
- [参考资料](#参考资料)

---

## 核心概念

Select 是 Go 语言中用于处理多个 Channel 操作的控制结构，类似于 switch 语句，但专门用于 Channel 通信。

```
核心特性：
1. 多路复用：同时监听多个 Channel 操作
2. 随机选择：多个 case 同时就绪时随机选择一个执行
3. 非阻塞：配合 default 实现非阻塞操作
4. 超时控制：配合 time.After 实现超时机制
5. 退出控制：配合 context 或 done channel 实现优雅退出

语法结构：
select {
case <-ch1:
    // 从 ch1 接收数据
case ch2 <- value:
    // 向 ch2 发送数据
case value := <-ch3:
    // 从 ch3 接收数据并赋值
default:
    // 所有 case 都不满足时执行
}
```

---

## 基本语法

### 1. 基本用法

```go
package main

import "fmt"

func main() {
    ch1 := make(chan int)
    ch2 := make(chan string)
    
    go func() {
        ch1 <- 42
    }()
    
    go func() {
        ch2 <- "hello"
    }()
    
    // select 会阻塞，直到某个 case 可以执行
    select {
    case v1 := <-ch1:
        fmt.Println("从 ch1 接收:", v1)
    case v2 := <-ch2:
        fmt.Println("从 ch2 接收:", v2)
    }
}
```

### 2. 带 default 的非阻塞操作

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    ch := make(chan int)
    
    // 非阻塞接收
    select {
    case v := <-ch:
        fmt.Println("接收到:", v)
    default:
        fmt.Println("没有数据可接收")
    }
    
    // 非阻塞发送
    select {
    case ch <- 42:
        fmt.Println("发送成功")
    default:
        fmt.Println("无法发送")
    }
}
```

### 3. 超时控制

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    ch := make(chan int)
    
    // 使用 time.After 实现超时
    select {
    case v := <-ch:
        fmt.Println("接收到:", v)
    case <-time.After(2 * time.Second):
        fmt.Println("超时")
    }
}
```

### 4. 多个 case 同时就绪

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    ch1 := make(chan int, 1)
    ch2 := make(chan int, 1)
    
    // 两个 channel 都有数据
    ch1 <- 1
    ch2 <- 2
    
    // 随机选择一个执行
    select {
    case v := <-ch1:
        fmt.Println("从 ch1 接收:", v)
    case v := <-ch2:
        fmt.Println("从 ch2 接收:", v)
    }
}
```

---

## 使用场景

### 1. 超时控制

```go
package main

import (
    "fmt"
    "time"
)

// 带超时的请求
func requestWithTimeout(ch <-chan string, timeout time.Duration) (string, error) {
    select {
    case result := <-ch:
        return result, nil
    case <-time.After(timeout):
        return "", fmt.Errorf("请求超时")
    }
}

func main() {
    ch := make(chan string)
    
    // 模拟耗时操作
    go func() {
        time.Sleep(3 * time.Second)
        ch <- "操作完成"
    }()
    
    result, err := requestWithTimeout(ch, 2*time.Second)
    if err != nil {
        fmt.Println("错误:", err)
    } else {
        fmt.Println("结果:", result)
    }
}
```

### 2. 非阻塞通信

```go
package main

import "fmt"

// 非阻塞发送
func trySend(ch chan<- int, value int) bool {
    select {
    case ch <- value:
        return true
    default:
        return false
    }
}

// 非阻塞接收
func tryReceive(ch <-chan int) (int, bool) {
    select {
    case value := <-ch:
        return value, true
    default:
        return 0, false
    }
}

func main() {
    ch := make(chan int, 1)
    
    // 尝试发送
    if trySend(ch, 42) {
        fmt.Println("发送成功")
    }
    
    // 尝试接收
    if value, ok := tryReceive(ch); ok {
        fmt.Println("接收到:", value)
    }
}
```

### 3. 多路复用

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    ch1 := make(chan int)
    ch2 := make(chan string)
    done := make(chan struct{})
    
    // 生产者 1
    go func() {
        for i := 0; i < 5; i++ {
            ch1 <- i
            time.Sleep(time.Millisecond * 500)
        }
    }()
    
    // 生产者 2
    go func() {
        for i := 0; i < 5; i++ {
            ch2 <- fmt.Sprintf("msg-%d", i)
            time.Sleep(time.Millisecond * 700)
        }
    }()
    
    // 定时器
    go func() {
        time.Sleep(3 * time.Second)
        close(done)
    }()
    
    // 消费者
    for {
        select {
        case v := <-ch1:
            fmt.Println("从 ch1 接收:", v)
        case v := <-ch2:
            fmt.Println("从 ch2 接收:", v)
        case <-done:
            fmt.Println("退出")
            return
        }
    }
}
```

### 4. 优雅退出

```go
package main

import (
    "context"
    "fmt"
    "time"
)

func worker(ctx context.Context, id int) {
    for {
        select {
        case <-ctx.Done():
            fmt.Printf("Worker %d 退出\n", id)
            return
        default:
            fmt.Printf("Worker %d 工作中...\n", id)
            time.Sleep(time.Second)
        }
    }
}

func main() {
    ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
    defer cancel()
    
    for i := 1; i <= 3; i++ {
        go worker(ctx, i)
    }
    
    <-ctx.Done()
    fmt.Println("主程序退出")
}
```

---

## 实现原理

### 1. Select 数据结构

```go
// runtime/select.go
type scase struct {
    c    *hchan         // channel
    elem unsafe.Pointer // 数据元素指针
}

// select 执行流程
func selectgo(cas0 *scase, order0 *uint16, ncases int) (int, bool) {
    // 1. 锁定所有 channel
    // 2. 按顺序检查每个 case
    // 3. 如果有 case 可以执行，执行并返回
    // 4. 如果没有 case 可以执行：
    //    - 有 default：执行 default
    //    - 无 default：阻塞等待
    // 5. 解锁所有 channel
}
```

### 2. 执行流程

```
Select 执行步骤：

1. 编译阶段：
   - 将 select 语句转换为 selectgo 函数调用
   - 生成 case 数组和顺序数组

2. 运行阶段：
   a. 锁定阶段：
      - 按地址顺序锁定所有涉及的 channel
      - 避免死锁
   
   b. 轮询阶段：
      - 按随机顺序检查每个 case
      - 检查是否有可以立即执行的 case
   
   c. 等待阶段（无 default 且无就绪 case）：
      - 将当前 goroutine 加入所有 channel 的等待队列
      - 阻塞当前 goroutine
   
   d. 唤醒阶段：
      - 某个 channel 操作就绪
      - 唤醒 goroutine
      - 从其他 channel 的等待队列中移除
   
   e. 解锁阶段：
      - 按地址顺序解锁所有 channel
```

### 3. 随机选择机制

```go
// select 使用随机顺序遍历 case
// 避免饥饿问题

func main() {
    ch1 := make(chan int, 1)
    ch2 := make(chan int, 1)
    
    ch1 <- 1
    ch2 <- 2
    
    // 多次运行，输出顺序不同
    for i := 0; i < 10; i++ {
        select {
        case v := <-ch1:
            fmt.Println("ch1:", v)
            ch1 <- 1
        case v := <-ch2:
            fmt.Println("ch2:", v)
            ch2 <- 2
        }
    }
}
```

---

## 代码示例

### 示例 1: 心跳检测

```go
package main

import (
    "fmt"
    "time"
)

func heartbeat(interval time.Duration) <-chan struct{} {
    heartbeat := make(chan struct{})
    go func() {
        ticker := time.NewTicker(interval)
        defer ticker.Stop()
        for {
            select {
            case <-ticker.C:
                heartbeat <- struct{}{}
            }
        }
    }()
    return heartbeat
}

func main() {
    hb := heartbeat(time.Second)
    timeout := time.After(5 * time.Second)
    
    for {
        select {
        case <-hb:
            fmt.Println("心跳:", time.Now().Format("15:04:05"))
        case <-timeout:
            fmt.Println("超时退出")
            return
        }
    }
}
```

### 示例 2: 任务调度器

```go
package main

import (
    "fmt"
    "time"
)

// Task 任务接口
type Task struct {
    ID   int
    Data string
}

// Scheduler 调度器
type Scheduler struct {
    tasks   chan Task
    results chan Task
    done    chan struct{}
}

// NewScheduler 创建调度器
func NewScheduler(workers int) *Scheduler {
    s := &Scheduler{
        tasks:   make(chan Task, 100),
        results: make(chan Task, 100),
        done:    make(chan struct{}),
    }
    
    // 启动 worker
    for i := 0; i < workers; i++ {
        go s.worker(i)
    }
    
    return s
}

// worker 工作协程
func (s *Scheduler) worker(id int) {
    for {
        select {
        case task := <-s.tasks:
            fmt.Printf("Worker %d 处理任务 %d\n", id, task.ID)
            time.Sleep(time.Second)
            s.results <- task
        case <-s.done:
            fmt.Printf("Worker %d 退出\n", id)
            return
        }
    }
}

// Submit 提交任务
func (s *Scheduler) Submit(task Task) {
    s.tasks <- task
}

// GetResult 获取结果
func (s *Scheduler) GetResult() Task {
    return <-s.results
}

// Stop 停止调度器
func (s *Scheduler) Stop() {
    close(s.done)
}

func main() {
    scheduler := NewScheduler(3)
    
    // 提交任务
    for i := 1; i <= 5; i++ {
        scheduler.Submit(Task{ID: i, Data: fmt.Sprintf("data-%d", i)})
    }
    
    // 获取结果
    for i := 1; i <= 5; i++ {
        result := scheduler.GetResult()
        fmt.Printf("完成任务 %d\n", result.ID)
    }
    
    scheduler.Stop()
    time.Sleep(time.Second)
}
```

### 示例 3: 限流器

```go
package main

import (
    "fmt"
    "time"
)

// RateLimiter 限流器
type RateLimiter struct {
    rate     int           // 每秒允许的请求数
    tokens   chan struct{} // 令牌桶
    done     chan struct{} // 退出信号
}

// NewRateLimiter 创建限流器
func NewRateLimiter(rate int) *RateLimiter {
    rl := &RateLimiter{
        rate:   rate,
        tokens: make(chan struct{}, rate),
        done:   make(chan struct{}),
    }
    
    // 填充令牌桶
    for i := 0; i < rate; i++ {
        rl.tokens <- struct{}{}
    }
    
    // 定期补充令牌
    go rl.refill()
    
    return rl
}

// refill 补充令牌
func (rl *RateLimiter) refill() {
    ticker := time.NewTicker(time.Second / time.Duration(rl.rate))
    defer ticker.Stop()
    
    for {
        select {
        case <-ticker.C:
            select {
            case rl.tokens <- struct{}{}:
            default:
                // 令牌桶已满
            }
        case <-rl.done:
            return
        }
    }
}

// Allow 检查是否允许请求
func (rl *RateLimiter) Allow() bool {
    select {
    case <-rl.tokens:
        return true
    default:
        return false
    }
}

// Wait 等待令牌
func (rl *RateLimiter) Wait() {
    <-rl.tokens
}

// Stop 停止限流器
func (rl *RateLimiter) Stop() {
    close(rl.done)
}

func main() {
    limiter := NewRateLimiter(5) // 每秒 5 个请求
    defer limiter.Stop()
    
    // 模拟 20 个请求
    for i := 1; i <= 20; i++ {
        if limiter.Allow() {
            fmt.Printf("请求 %d 通过 - %s\n", i, time.Now().Format("15:04:05.000"))
        } else {
            fmt.Printf("请求 %d 被限流\n", i)
            limiter.Wait()
            fmt.Printf("请求 %d 重试通过 - %s\n", i, time.Now().Format("15:04:05.000"))
        }
        time.Sleep(time.Millisecond * 100)
    }
}
```

---

## 常见问题

### 1. Select 死锁

```go
// ❌ 错误：所有 case 都阻塞，没有 default
func deadlock() {
    ch1 := make(chan int)
    ch2 := make(chan int)
    
    select {
    case <-ch1:
    case <-ch2:
    }
    // fatal error: all goroutines are asleep - deadlock!
}

// ✅ 正确：添加 default 或超时
func noDeadlock() {
    ch1 := make(chan int)
    ch2 := make(chan int)
    
    select {
    case <-ch1:
    case <-ch2:
    case <-time.After(time.Second):
        fmt.Println("超时")
    }
}
```

### 2. 空 Select

```go
// 空 select 会永久阻塞
func emptySelect() {
    select {}
    // 永远不会执行到这里
}

// 用途：阻止 main 函数退出
func main() {
    go func() {
        // 后台任务
    }()
    
    select {} // 阻止退出
}
```

### 3. Nil Channel

```go
func nilChannel() {
    var ch chan int // nil channel
    
    select {
    case <-ch: // 永远不会执行
        fmt.Println("接收")
    case ch <- 1: // 永远不会执行
        fmt.Println("发送")
    default:
        fmt.Println("default 执行")
    }
}

// 利用 nil channel 禁用某个 case
func disableCase() {
    ch1 := make(chan int)
    ch2 := make(chan int)
    
    // 禁用 ch2
    ch2 = nil
    
    select {
    case v := <-ch1:
        fmt.Println(v)
    case v := <-ch2: // 永远不会执行
        fmt.Println(v)
    }
}
```

---

## 面试题

### 1. Select 的执行顺序是什么？

**答案：**

1. 如果有多个 case 同时就绪，select 会**随机**选择一个执行
2. 如果没有 case 就绪且有 default，执行 default
3. 如果没有 case 就绪且没有 default，阻塞等待

```go
// 验证随机性
func main() {
    ch := make(chan int, 2)
    ch <- 1
    ch <- 2
    
    for i := 0; i < 10; i++ {
        select {
        case v := <-ch:
            fmt.Print(v, " ")
            ch <- v
        }
    }
}
// 输出可能是：1 2 1 2 1 2... 或 2 1 2 1 2 1...
```

### 2. 下面代码会输出什么？

```go
func main() {
    ch := make(chan int, 1)
    ch <- 1
    
    select {
    case ch <- 2:
        fmt.Println("发送 2")
    case v := <-ch:
        fmt.Println("接收", v)
    }
}
```

**答案：**

可能输出 "发送 2" 或 "接收 1"，因为两个 case 都就绪，随机选择。

### 3. 如何实现超时重试？

**答案：**

```go
func retryWithTimeout(fn func() error, timeout time.Duration, maxRetries int) error {
    for i := 0; i < maxRetries; i++ {
        errCh := make(chan error, 1)
        
        go func() {
            errCh <- fn()
        }()
        
        select {
        case err := <-errCh:
            if err == nil {
                return nil
            }
            fmt.Printf("重试 %d 失败: %v\n", i+1, err)
        case <-time.After(timeout):
            fmt.Printf("重试 %d 超时\n", i+1)
        }
    }
    return fmt.Errorf("达到最大重试次数")
}
```

### 4. Select 和 Switch 的区别？

**答案：**

| 特性 | Select | Switch |
|------|--------|--------|
| 用途 | Channel 操作 | 条件判断 |
| case 类型 | Channel 操作 | 表达式 |
| 执行顺序 | 随机（多个就绪时） | 顺序 |
| default | 非阻塞 | 默认分支 |
| 阻塞 | 可能阻塞 | 不阻塞 |

---

## 最佳实践

### 1. 使用 Context 控制生命周期

```go
// ✅ 推荐
func worker(ctx context.Context) {
    for {
        select {
        case <-ctx.Done():
            return
        default:
            // 工作逻辑
        }
    }
}
```

### 2. 避免在循环中使用 time.After

```go
// ❌ 错误：每次循环都创建新的 timer，导致内存泄漏
func badLoop() {
    for {
        select {
        case <-time.After(time.Second):
            // 处理
        }
    }
}

// ✅ 正确：使用 time.NewTicker
func goodLoop() {
    ticker := time.NewTicker(time.Second)
    defer ticker.Stop()
    
    for {
        select {
        case <-ticker.C:
            // 处理
        }
    }
}
```

### 3. 合理使用 Default

```go
// ✅ 非阻塞操作
func tryReceive(ch <-chan int) (int, bool) {
    select {
    case v := <-ch:
        return v, true
    default:
        return 0, false
    }
}
```

### 4. 使用 Nil Channel 动态控制

```go
// ✅ 动态禁用某个 case
func dynamicSelect() {
    ch1 := make(chan int)
    ch2 := make(chan int)
    
    for {
        select {
        case v := <-ch1:
            fmt.Println("ch1:", v)
            if v > 10 {
                ch1 = nil // 禁用 ch1
            }
        case v := <-ch2:
            fmt.Println("ch2:", v)
        }
    }
}
```

---

## 参考资料

- [x] [Go 官方文档 - Select](https://go.dev/ref/spec#Select_statements)
- [x] [Effective Go - Select](https://go.dev/doc/effective_go#select)
- [x] [Go by Example - Select](https://gobyexample.com/select)
- [x] [Go Concurrency Patterns - Select](https://go.dev/blog/pipelines)

---

**上一节：** [02-Channel机制](./02-Channel机制.md)  
**下一节：** [04-Mutex与RWMutex](./04-Mutex与RWMutex.md)
