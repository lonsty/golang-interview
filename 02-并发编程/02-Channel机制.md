# 02-Channel机制

[← 返回本章目录](./README.md) | [← 返回总目录](../README.md)

---

## 📋 目录

- [核心概念](#核心概念)
- [Channel类型](#channel类型)
- [Channel操作](#channel操作)
- [Channel关闭](#channel关闭)
- [实现原理](#实现原理)
- [代码示例](#代码示例)
- [常见问题](#常见问题)
- [面试题](#面试题)
- [最佳实践](#最佳实践)
- [参考资料](#参考资料)

---

## 核心概念

Channel 是 Go 语言中用于 Goroutine 之间通信的核心机制，遵循 CSP（Communicating Sequential Processes）并发模型。

```
核心理念：
"不要通过共享内存来通信，而应该通过通信来共享内存"
Don't communicate by sharing memory; share memory by communicating.

特点：
1. 类型安全：Channel 是类型安全的
2. 阻塞机制：发送和接收操作可能阻塞
3. 同步原语：可用于 Goroutine 同步
4. FIFO：先进先出的数据结构
5. 线程安全：多个 Goroutine 可安全操作
```

---

## Channel类型

### 1. 无缓冲 Channel

```go
package main

import "fmt"

func main() {
    // 创建无缓冲 channel
    ch := make(chan int)
    
    // 发送和接收必须同时准备好
    go func() {
        ch <- 42  // 发送操作会阻塞，直到有接收者
    }()
    
    value := <-ch  // 接收操作会阻塞，直到有发送者
    fmt.Println(value)
}
```

**特点：**
- 发送操作阻塞，直到有接收者准备好
- 接收操作阻塞，直到有发送者准备好
- 提供强同步保证

### 2. 有缓冲 Channel

```go
package main

import "fmt"

func main() {
    // 创建容量为 3 的缓冲 channel
    ch := make(chan int, 3)
    
    // 缓冲区未满时，发送不阻塞
    ch <- 1
    ch <- 2
    ch <- 3
    
    // 接收数据
    fmt.Println(<-ch)  // 1
    fmt.Println(<-ch)  // 2
    fmt.Println(<-ch)  // 3
}
```

**特点：**
- 缓冲区未满时，发送不阻塞
- 缓冲区不为空时，接收不阻塞
- 缓冲区满时，发送阻塞
- 缓冲区空时，接收阻塞

### 3. 单向 Channel

```go
package main

import "fmt"

// 只能发送的 channel
func send(ch chan<- int) {
    ch <- 42
}

// 只能接收的 channel
func receive(ch <-chan int) {
    value := <-ch
    fmt.Println(value)
}

func main() {
    ch := make(chan int)
    
    go send(ch)
    receive(ch)
}
```

**类型：**
- `chan<- T`：只能发送的 channel
- `<-chan T`：只能接收的 channel
- `chan T`：可发送可接收的 channel

---

## Channel操作

### 1. 发送操作

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    ch := make(chan int, 1)
    
    // 基本发送
    ch <- 42
    
    // 非阻塞发送
    select {
    case ch <- 100:
        fmt.Println("发送成功")
    default:
        fmt.Println("channel 已满")
    }
    
    // 超时发送
    select {
    case ch <- 200:
        fmt.Println("发送成功")
    case <-time.After(time.Second):
        fmt.Println("发送超时")
    }
}
```

### 2. 接收操作

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    ch := make(chan int, 2)
    ch <- 1
    ch <- 2
    close(ch)
    
    // 基本接收
    value := <-ch
    fmt.Println(value)
    
    // 检查 channel 是否关闭
    value, ok := <-ch
    if ok {
        fmt.Println("接收到:", value)
    } else {
        fmt.Println("channel 已关闭")
    }
    
    // 非阻塞接收
    select {
    case v := <-ch:
        fmt.Println("接收到:", v)
    default:
        fmt.Println("channel 为空")
    }
    
    // 超时接收
    select {
    case v := <-ch:
        fmt.Println("接收到:", v)
    case <-time.After(time.Second):
        fmt.Println("接收超时")
    }
}
```

### 3. 遍历 Channel

```go
package main

import "fmt"

func main() {
    ch := make(chan int, 5)
    
    // 发送数据
    go func() {
        for i := 0; i < 5; i++ {
            ch <- i
        }
        close(ch)  // 必须关闭，否则 range 会永久阻塞
    }()
    
    // 使用 range 遍历
    for value := range ch {
        fmt.Println(value)
    }
}
```

---

## Channel关闭

### 1. 关闭规则

```go
package main

import "fmt"

func main() {
    ch := make(chan int, 2)
    
    // 发送数据
    ch <- 1
    ch <- 2
    
    // 关闭 channel
    close(ch)
    
    // 关闭后仍可接收数据
    fmt.Println(<-ch)  // 1
    fmt.Println(<-ch)  // 2
    
    // 接收零值和关闭状态
    value, ok := <-ch
    fmt.Printf("value: %d, ok: %v\n", value, ok)  // value: 0, ok: false
}
```

### 2. 关闭注意事项

```go
package main

func main() {
    ch := make(chan int)
    
    // ✅ 正确：发送者关闭 channel
    go func() {
        ch <- 1
        close(ch)
    }()
    
    <-ch
    
    // ❌ 错误：向已关闭的 channel 发送数据会 panic
    // ch <- 2  // panic: send on closed channel
    
    // ❌ 错误：重复关闭 channel 会 panic
    // close(ch)  // panic: close of closed channel
    
    // ❌ 错误：关闭 nil channel 会 panic
    var nilCh chan int
    // close(nilCh)  // panic: close of nil channel
}
```

### 3. 优雅关闭模式

```go
package main

import (
    "fmt"
    "sync"
)

// 模式 1: 单个发送者，多个接收者
func singleSenderMultiReceiver() {
    ch := make(chan int, 10)
    var wg sync.WaitGroup
    
    // 单个发送者
    go func() {
        for i := 0; i < 10; i++ {
            ch <- i
        }
        close(ch)  // 发送者关闭
    }()
    
    // 多个接收者
    for i := 0; i < 3; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            for value := range ch {
                fmt.Printf("接收者 %d: %d\n", id, value)
            }
        }(i)
    }
    
    wg.Wait()
}

// 模式 2: 多个发送者，单个接收者
func multiSenderSingleReceiver() {
    ch := make(chan int, 10)
    done := make(chan struct{})
    var wg sync.WaitGroup
    
    // 多个发送者
    for i := 0; i < 3; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            for j := 0; j < 5; j++ {
                select {
                case ch <- id*10 + j:
                case <-done:
                    return
                }
            }
        }(i)
    }
    
    // 等待所有发送者完成后关闭
    go func() {
        wg.Wait()
        close(ch)
    }()
    
    // 单个接收者
    for value := range ch {
        fmt.Println(value)
    }
}

// 模式 3: 多个发送者，多个接收者
func multiSenderMultiReceiver() {
    ch := make(chan int, 10)
    done := make(chan struct{})
    var wg sync.WaitGroup
    
    // 多个发送者
    for i := 0; i < 3; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            for j := 0; j < 5; j++ {
                select {
                case ch <- id*10 + j:
                case <-done:
                    return
                }
            }
        }(i)
    }
    
    // 等待所有发送者完成后关闭
    go func() {
        wg.Wait()
        close(ch)
    }()
    
    // 多个接收者
    var recvWg sync.WaitGroup
    for i := 0; i < 2; i++ {
        recvWg.Add(1)
        go func(id int) {
            defer recvWg.Done()
            for value := range ch {
                fmt.Printf("接收者 %d: %d\n", id, value)
            }
        }(i)
    }
    
    recvWg.Wait()
}

func main() {
    fmt.Println("=== 单发送者多接收者 ===")
    singleSenderMultiReceiver()
    
    fmt.Println("\n=== 多发送者单接收者 ===")
    multiSenderSingleReceiver()
    
    fmt.Println("\n=== 多发送者多接收者 ===")
    multiSenderMultiReceiver()
}
```

---

## 实现原理

### 1. Channel 数据结构

```go
// runtime/chan.go
type hchan struct {
    qcount   uint           // 队列中的元素个数
    dataqsiz uint           // 循环队列的大小
    buf      unsafe.Pointer // 指向循环队列的指针
    elemsize uint16         // 元素大小
    closed   uint32         // 是否关闭
    elemtype *_type         // 元素类型
    sendx    uint           // 发送索引
    recvx    uint           // 接收索引
    recvq    waitq          // 接收等待队列
    sendq    waitq          // 发送等待队列
    lock     mutex          // 互斥锁
}

type waitq struct {
    first *sudog  // 等待队列的头部
    last  *sudog  // 等待队列的尾部
}
```

### 2. 发送流程

```
发送数据到 channel 的流程：

1. 加锁
2. 检查 channel 是否关闭
3. 如果有等待接收的 goroutine：
   - 直接将数据传递给接收者
   - 唤醒接收 goroutine
4. 如果缓冲区未满：
   - 将数据复制到缓冲区
   - sendx++
5. 如果缓冲区已满：
   - 将当前 goroutine 加入发送等待队列
   - 阻塞当前 goroutine
6. 解锁
```

### 3. 接收流程

```
从 channel 接收数据的流程：

1. 加锁
2. 如果有等待发送的 goroutine：
   - 直接从发送者接收数据
   - 唤醒发送 goroutine
3. 如果缓冲区不为空：
   - 从缓冲区读取数据
   - recvx++
4. 如果缓冲区为空：
   - 将当前 goroutine 加入接收等待队列
   - 阻塞当前 goroutine
5. 解锁
```

### 4. 关闭流程

```
关闭 channel 的流程：

1. 加锁
2. 检查是否已关闭（避免重复关闭）
3. 设置 closed 标志
4. 释放所有接收等待队列中的 goroutine
5. 释放所有发送等待队列中的 goroutine（会 panic）
6. 解锁
```

---

## 代码示例

### 示例 1: 生产者-消费者模式

```go
package main

import (
    "fmt"
    "math/rand"
    "sync"
    "time"
)

// Producer 生产者
func Producer(id int, ch chan<- int, wg *sync.WaitGroup) {
    defer wg.Done()
    for i := 0; i < 5; i++ {
        value := rand.Intn(100)
        ch <- value
        fmt.Printf("生产者 %d 生产: %d\n", id, value)
        time.Sleep(time.Millisecond * time.Duration(rand.Intn(500)))
    }
}

// Consumer 消费者
func Consumer(id int, ch <-chan int, wg *sync.WaitGroup) {
    defer wg.Done()
    for value := range ch {
        fmt.Printf("消费者 %d 消费: %d\n", id, value)
        time.Sleep(time.Millisecond * time.Duration(rand.Intn(1000)))
    }
}

func main() {
    rand.Seed(time.Now().UnixNano())
    ch := make(chan int, 10)
    var producerWg, consumerWg sync.WaitGroup
    
    // 启动 3 个生产者
    for i := 1; i <= 3; i++ {
        producerWg.Add(1)
        go Producer(i, ch, &producerWg)
    }
    
    // 启动 2 个消费者
    for i := 1; i <= 2; i++ {
        consumerWg.Add(1)
        go Consumer(i, ch, &consumerWg)
    }
    
    // 等待所有生产者完成
    producerWg.Wait()
    close(ch)
    
    // 等待所有消费者完成
    consumerWg.Wait()
}
```

### 示例 2: 扇出扇入模式

```go
package main

import (
    "fmt"
    "sync"
)

// 扇出：一个输入，多个输出
func fanOut(input <-chan int, workers int) []<-chan int {
    outputs := make([]<-chan int, workers)
    for i := 0; i < workers; i++ {
        outputs[i] = worker(input, i)
    }
    return outputs
}

// Worker 处理数据
func worker(input <-chan int, id int) <-chan int {
    output := make(chan int)
    go func() {
        defer close(output)
        for value := range input {
            result := value * value
            fmt.Printf("Worker %d: %d -> %d\n", id, value, result)
            output <- result
        }
    }()
    return output
}

// 扇入：多个输入，一个输出
func fanIn(inputs ...<-chan int) <-chan int {
    output := make(chan int)
    var wg sync.WaitGroup
    
    for _, input := range inputs {
        wg.Add(1)
        go func(ch <-chan int) {
            defer wg.Done()
            for value := range ch {
                output <- value
            }
        }(input)
    }
    
    go func() {
        wg.Wait()
        close(output)
    }()
    
    return output
}

func main() {
    // 创建输入 channel
    input := make(chan int)
    
    // 发送数据
    go func() {
        for i := 1; i <= 10; i++ {
            input <- i
        }
        close(input)
    }()
    
    // 扇出到 3 个 worker
    outputs := fanOut(input, 3)
    
    // 扇入到一个 channel
    result := fanIn(outputs...)
    
    // 收集结果
    for value := range result {
        fmt.Println("结果:", value)
    }
}
```

### 示例 3: 管道模式

```go
package main

import "fmt"

// 生成器
func generator(nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for _, n := range nums {
            out <- n
        }
    }()
    return out
}

// 平方
func square(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for n := range in {
            out <- n * n
        }
    }()
    return out
}

// 过滤偶数
func filterEven(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for n := range in {
            if n%2 == 0 {
                out <- n
            }
        }
    }()
    return out
}

func main() {
    // 构建管道
    nums := generator(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
    squared := square(nums)
    filtered := filterEven(squared)
    
    // 消费结果
    for n := range filtered {
        fmt.Println(n)
    }
}
```

---

## 常见问题

### 1. Channel 死锁

```go
// ❌ 错误：死锁
func deadlock1() {
    ch := make(chan int)
    ch <- 1  // 阻塞，没有接收者
    <-ch
}

// ❌ 错误：死锁
func deadlock2() {
    ch := make(chan int)
    <-ch  // 阻塞，没有发送者
}

// ✅ 正确：使用 goroutine
func noDeadlock() {
    ch := make(chan int)
    go func() {
        ch <- 1
    }()
    <-ch
}
```

### 2. Channel 泄漏

```go
// ❌ 错误：goroutine 泄漏
func leak() {
    ch := make(chan int)
    go func() {
        value := <-ch  // 永远阻塞
        fmt.Println(value)
    }()
    // ch 没有发送数据
}

// ✅ 正确：使用超时或 context
func noLeak() {
    ch := make(chan int)
    go func() {
        select {
        case value := <-ch:
            fmt.Println(value)
        case <-time.After(time.Second):
            fmt.Println("超时")
        }
    }()
}
```

### 3. 向已关闭的 Channel 发送

```go
// ❌ 错误：panic
func sendToClosed() {
    ch := make(chan int)
    close(ch)
    ch <- 1  // panic: send on closed channel
}

// ✅ 正确：检查是否关闭
func safeSend() {
    ch := make(chan int, 1)
    done := make(chan struct{})
    
    go func() {
        defer close(done)
        select {
        case ch <- 1:
            fmt.Println("发送成功")
        case <-done:
            fmt.Println("channel 已关闭")
        }
    }()
}
```

---

## 面试题

### 1. 无缓冲 Channel 和有缓冲 Channel 的区别？

**答案：**

| 特性 | 无缓冲 Channel | 有缓冲 Channel |
|------|---------------|---------------|
| 创建 | `make(chan T)` | `make(chan T, size)` |
| 发送阻塞 | 必须有接收者 | 缓冲区满时阻塞 |
| 接收阻塞 | 必须有发送者 | 缓冲区空时阻塞 |
| 同步性 | 强同步 | 异步（缓冲区内） |
| 使用场景 | 需要严格同步 | 削峰填谷 |

### 2. 如何判断 Channel 是否关闭？

**答案：**

```go
// 方法 1: 使用 ok 模式
value, ok := <-ch
if !ok {
    fmt.Println("channel 已关闭")
}

// 方法 2: 使用 range（自动检测关闭）
for value := range ch {
    fmt.Println(value)
}
// range 会在 channel 关闭时自动退出
```

### 3. 下面代码会输出什么？

```go
func main() {
    ch := make(chan int, 3)
    ch <- 1
    ch <- 2
    close(ch)
    
    for i := 0; i < 5; i++ {
        value, ok := <-ch
        fmt.Printf("value: %d, ok: %v\n", value, ok)
    }
}
```

**答案：**
```
value: 1, ok: true
value: 2, ok: true
value: 0, ok: false
value: 0, ok: false
value: 0, ok: false
```

关闭后的 channel 可以继续接收，返回零值和 false。

### 4. 如何实现超时控制？

**答案：**

```go
// 方法 1: 使用 time.After
select {
case value := <-ch:
    fmt.Println(value)
case <-time.After(time.Second):
    fmt.Println("超时")
}

// 方法 2: 使用 context
ctx, cancel := context.WithTimeout(context.Background(), time.Second)
defer cancel()

select {
case value := <-ch:
    fmt.Println(value)
case <-ctx.Done():
    fmt.Println("超时")
}
```

---

## 最佳实践

### 1. 由发送者关闭 Channel

```go
// ✅ 正确：发送者关闭
func sender(ch chan<- int) {
    for i := 0; i < 10; i++ {
        ch <- i
    }
    close(ch)  // 发送者关闭
}

func receiver(ch <-chan int) {
    for value := range ch {
        fmt.Println(value)
    }
}
```

### 2. 使用缓冲 Channel 避免阻塞

```go
// ✅ 合理使用缓冲
func withBuffer() {
    ch := make(chan int, 10)  // 缓冲区大小根据实际需求
    
    go func() {
        for i := 0; i < 10; i++ {
            ch <- i  // 不会阻塞
        }
        close(ch)
    }()
    
    for value := range ch {
        fmt.Println(value)
    }
}
```

### 3. 使用 select 实现非阻塞操作

```go
// ✅ 非阻塞发送
select {
case ch <- value:
    fmt.Println("发送成功")
default:
    fmt.Println("channel 已满")
}

// ✅ 非阻塞接收
select {
case value := <-ch:
    fmt.Println(value)
default:
    fmt.Println("channel 为空")
}
```

### 4. 使用 Context 控制生命周期

```go
// ✅ 使用 context 优雅退出
func worker(ctx context.Context, ch <-chan int) {
    for {
        select {
        case value := <-ch:
            fmt.Println(value)
        case <-ctx.Done():
            fmt.Println("退出")
            return
        }
    }
}
```

---

## 参考资料

- [x] [Go 官方文档 - Channels](https://go.dev/tour/concurrency/2)
- [x] [Effective Go - Channels](https://go.dev/doc/effective_go#channels)
- [x] [Go by Example - Channels](https://gobyexample.com/channels)
- [x] [Go Concurrency Patterns](https://go.dev/blog/pipelines)

---

**上一节：** [01-Goroutine原理](./01-Goroutine原理.md)  
**下一节：** [03-Select多路复用](./03-Select多路复用.md)
