# 01-Goroutine原理

[← 返回本章目录](./README.md) | [← 返回总目录](../README.md)

---

## 📋 目录

- [核心概念](#核心概念)
- [Goroutine创建](#goroutine创建)
- [Goroutine调度](#goroutine调度)
- [Goroutine状态](#goroutine状态)
- [实现原理](#实现原理)
- [代码示例](#代码示例)
- [常见问题](#常见问题)
- [面试题](#面试题)
- [最佳实践](#最佳实践)
- [参考资料](#参考资料)

---

## 核心概念

Goroutine 是 Go 语言实现并发的核心机制，是一种轻量级的用户态线程。

```
特点：
1. 轻量级：初始栈大小仅 2KB
2. 低开销：创建和销毁成本极低
3. 动态栈：栈大小可动态增长和收缩
4. 调度器：由 Go 运行时调度，非操作系统调度
5. 通信：通过 Channel 进行通信
```

### Goroutine vs 线程

| 特性 | Goroutine | 线程 |
|------|-----------|------|
| 创建开销 | ~2KB | ~1-2MB |
| 调度方式 | 用户态调度 | 内核态调度 |
| 切换开销 | 极低 | 较高 |
| 数量限制 | 百万级 | 千级 |
| 通信方式 | Channel | 共享内存 |

---

## Goroutine创建

### 1. 基本创建

```go
package main

import (
    "fmt"
    "time"
)

func sayHello() {
    fmt.Println("Hello from goroutine")
}

func main() {
    // 创建 goroutine
    go sayHello()
    
    // 主 goroutine 等待
    time.Sleep(time.Second)
    fmt.Println("Main goroutine")
}
```

### 2. 匿名函数

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    // 使用匿名函数
    go func() {
        fmt.Println("Anonymous goroutine")
    }()
    
    // 带参数的匿名函数
    go func(msg string) {
        fmt.Println(msg)
    }("Hello")
    
    time.Sleep(time.Second)
}
```

### 3. 批量创建

```go
package main

import (
    "fmt"
    "sync"
)

func main() {
    var wg sync.WaitGroup
    
    // 创建 10 个 goroutine
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            fmt.Printf("Goroutine %d\n", id)
        }(i)
    }
    
    wg.Wait()
}
```

---

## Goroutine调度

### 1. GMP 模型

```
G (Goroutine)：用户态线程
M (Machine)：操作系统线程
P (Processor)：逻辑处理器

调度流程：
1. G 需要在 M 上执行
2. M 必须持有 P 才能执行 G
3. P 维护一个本地 G 队列
4. 全局有一个 G 队列
```

### 2. 调度时机

```go
// 1. 主动让出 CPU
runtime.Gosched()

// 2. 系统调用
syscall.Read()

// 3. Channel 操作阻塞
<-ch

// 4. 锁操作阻塞
mu.Lock()

// 5. 垃圾回收
runtime.GC()
```

### 3. 调度示例

```go
package main

import (
    "fmt"
    "runtime"
    "time"
)

func main() {
    // 设置使用的 CPU 核心数
    runtime.GOMAXPROCS(2)
    
    go func() {
        for i := 0; i < 5; i++ {
            fmt.Println("Goroutine 1:", i)
            runtime.Gosched()  // 主动让出 CPU
        }
    }()
    
    go func() {
        for i := 0; i < 5; i++ {
            fmt.Println("Goroutine 2:", i)
            runtime.Gosched()
        }
    }()
    
    time.Sleep(time.Second)
}
```

---

## Goroutine状态

### 1. 状态转换

```
状态：
- _Gidle：刚刚被分配，还没有初始化
- _Grunnable：在运行队列中，等待被调度
- _Grunning：正在执行
- _Gsyscall：正在执行系统调用
- _Gwaiting：被阻塞（等待 Channel、锁等）
- _Gdead：刚刚退出或正在被初始化

状态转换：
_Gidle -> _Grunnable -> _Grunning -> _Gwaiting -> _Grunnable
                                  -> _Gsyscall -> _Grunnable
                                  -> _Gdead
```

### 2. 查看 Goroutine 信息

```go
package main

import (
    "fmt"
    "runtime"
    "time"
)

func main() {
    // 获取当前 goroutine 数量
    fmt.Println("初始 goroutine 数量:", runtime.NumGoroutine())
    
    for i := 0; i < 10; i++ {
        go func() {
            time.Sleep(time.Hour)
        }()
    }
    
    time.Sleep(time.Millisecond * 100)
    fmt.Println("创建后 goroutine 数量:", runtime.NumGoroutine())
}
```

---

## 实现原理

### 1. Goroutine 结构

```go
// runtime/runtime2.go
type g struct {
    stack       stack       // 栈信息
    stackguard0 uintptr     // 栈保护
    m           *m          // 当前绑定的 M
    sched       gobuf       // 调度信息
    atomicstatus uint32     // 状态
    goid        int64       // goroutine ID
    // ... 更多字段
}

type stack struct {
    lo uintptr  // 栈底
    hi uintptr  // 栈顶
}

type gobuf struct {
    sp   uintptr  // 栈指针
    pc   uintptr  // 程序计数器
    g    guintptr // goroutine
    ret  uintptr  // 返回值
    // ...
}
```

### 2. 栈管理

```go
// 初始栈大小：2KB
const _StackMin = 2048

// 栈增长
// 当栈空间不足时，会分配新的更大的栈
// 并将旧栈的内容复制到新栈

// 栈收缩
// 当栈使用率低于 1/4 时，会收缩栈
```

### 3. 调度循环

```go
// 简化的调度循环
func schedule() {
    gp := findRunnable()  // 查找可运行的 G
    execute(gp)           // 执行 G
    // G 执行完毕或被阻塞后，继续调度
    schedule()
}

func findRunnable() *g {
    // 1. 从本地队列获取
    // 2. 从全局队列获取
    // 3. 从网络轮询器获取
    // 4. 从其他 P 偷取（work stealing）
}
```

---

## 代码示例

### 示例 1: 并发计算

```go
package main

import (
    "fmt"
    "sync"
)

func sum(nums []int, result chan<- int) {
    sum := 0
    for _, n := range nums {
        sum += n
    }
    result <- sum
}

func main() {
    nums := []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}
    
    // 分成两部分并发计算
    result := make(chan int, 2)
    
    go sum(nums[:len(nums)/2], result)
    go sum(nums[len(nums)/2:], result)
    
    sum1, sum2 := <-result, <-result
    fmt.Println("总和:", sum1+sum2)
}
```

### 示例 2: 生产者-消费者

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

func producer(ch chan<- int, wg *sync.WaitGroup) {
    defer wg.Done()
    for i := 0; i < 5; i++ {
        ch <- i
        fmt.Println("生产:", i)
        time.Sleep(time.Millisecond * 100)
    }
    close(ch)
}

func consumer(ch <-chan int, wg *sync.WaitGroup) {
    defer wg.Done()
    for num := range ch {
        fmt.Println("消费:", num)
        time.Sleep(time.Millisecond * 200)
    }
}

func main() {
    ch := make(chan int, 3)
    var wg sync.WaitGroup
    
    wg.Add(2)
    go producer(ch, &wg)
    go consumer(ch, &wg)
    
    wg.Wait()
}
```

### 示例 3: 并发爬虫

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

type Fetcher interface {
    Fetch(url string) (body string, urls []string, err error)
}

func Crawl(url string, depth int, fetcher Fetcher, wg *sync.WaitGroup, visited map[string]bool, mu *sync.Mutex) {
    defer wg.Done()
    
    if depth <= 0 {
        return
    }
    
    mu.Lock()
    if visited[url] {
        mu.Unlock()
        return
    }
    visited[url] = true
    mu.Unlock()
    
    body, urls, err := fetcher.Fetch(url)
    if err != nil {
        fmt.Println(err)
        return
    }
    
    fmt.Printf("找到: %s %q\n", url, body)
    
    for _, u := range urls {
        wg.Add(1)
        go Crawl(u, depth-1, fetcher, wg, visited, mu)
    }
}

func main() {
    var wg sync.WaitGroup
    var mu sync.Mutex
    visited := make(map[string]bool)
    
    wg.Add(1)
    go Crawl("https://golang.org/", 4, fetcher, &wg, visited, &mu)
    wg.Wait()
}

// 模拟的 Fetcher
type fakeFetcher map[string]*fakeResult

type fakeResult struct {
    body string
    urls []string
}

func (f fakeFetcher) Fetch(url string) (string, []string, error) {
    if res, ok := f[url]; ok {
        return res.body, res.urls, nil
    }
    return "", nil, fmt.Errorf("未找到: %s", url)
}

var fetcher = fakeFetcher{
    "https://golang.org/": &fakeResult{
        "The Go Programming Language",
        []string{
            "https://golang.org/pkg/",
            "https://golang.org/cmd/",
        },
    },
    "https://golang.org/pkg/": &fakeResult{
        "Packages",
        []string{
            "https://golang.org/",
            "https://golang.org/cmd/",
        },
    },
}
```

---

## 常见问题

### 1. Goroutine 泄漏

```go
// ❌ 错误：goroutine 永远阻塞
func leak() {
    ch := make(chan int)
    go func() {
        val := <-ch  // 永远等待
        fmt.Println(val)
    }()
    // ch 没有发送数据，goroutine 泄漏
}

// ✅ 正确：使用 context 或 close
func noLeak() {
    ch := make(chan int)
    go func() {
        select {
        case val := <-ch:
            fmt.Println(val)
        case <-time.After(time.Second):
            fmt.Println("超时")
        }
    }()
}
```

### 2. 闭包陷阱

```go
// ❌ 错误：所有 goroutine 打印相同的值
func closureTrap() {
    for i := 0; i < 5; i++ {
        go func() {
            fmt.Println(i)  // 打印 5, 5, 5, 5, 5
        }()
    }
    time.Sleep(time.Second)
}

// ✅ 正确：传递参数
func closureCorrect() {
    for i := 0; i < 5; i++ {
        go func(n int) {
            fmt.Println(n)  // 打印 0, 1, 2, 3, 4
        }(i)
    }
    time.Sleep(time.Second)
}
```

### 3. 主 Goroutine 退出

```go
// ❌ 错误：主 goroutine 退出，子 goroutine 也会退出
func main() {
    go func() {
        time.Sleep(time.Second)
        fmt.Println("子 goroutine")
    }()
    // 主 goroutine 立即退出
}

// ✅ 正确：等待子 goroutine
func main() {
    var wg sync.WaitGroup
    wg.Add(1)
    go func() {
        defer wg.Done()
        time.Sleep(time.Second)
        fmt.Println("子 goroutine")
    }()
    wg.Wait()
}
```

---

## 面试题

### 1. Goroutine 和线程的区别？

**答案：**

| 特性 | Goroutine | 线程 |
|------|-----------|------|
| 内存占用 | 2KB 初始栈 | 1-2MB 固定栈 |
| 创建销毁 | 极快 | 较慢 |
| 调度 | 用户态（Go 调度器） | 内核态（OS 调度器） |
| 切换开销 | 纳秒级 | 微秒级 |
| 数量 | 百万级 | 千级 |

### 2. 如何控制 Goroutine 的数量？

**答案：**

```go
// 方法 1: 使用带缓冲的 channel
func limitGoroutines() {
    limit := make(chan struct{}, 10)  // 最多 10 个
    
    for i := 0; i < 100; i++ {
        limit <- struct{}{}  // 获取令牌
        go func(id int) {
            defer func() { <-limit }()  // 释放令牌
            // 执行任务
            fmt.Println(id)
        }(i)
    }
}

// 方法 2: 使用 worker pool
func workerPool() {
    jobs := make(chan int, 100)
    results := make(chan int, 100)
    
    // 创建 10 个 worker
    for w := 0; w < 10; w++ {
        go worker(jobs, results)
    }
    
    // 发送任务
    for j := 0; j < 100; j++ {
        jobs <- j
    }
    close(jobs)
    
    // 收集结果
    for r := 0; r < 100; r++ {
        <-results
    }
}

func worker(jobs <-chan int, results chan<- int) {
    for j := range jobs {
        results <- j * 2
    }
}
```

### 3. Goroutine 如何退出？

**答案：**

```go
// 1. 函数执行完毕自动退出
go func() {
    fmt.Println("执行完毕")
}()

// 2. 使用 channel 通知退出
done := make(chan struct{})
go func() {
    for {
        select {
        case <-done:
            return
        default:
            // 执行任务
        }
    }
}()
close(done)  // 通知退出

// 3. 使用 context 控制退出
ctx, cancel := context.WithCancel(context.Background())
go func() {
    for {
        select {
        case <-ctx.Done():
            return
        default:
            // 执行任务
        }
    }
}()
cancel()  // 通知退出
```

### 4. 下面代码输出什么？

```go
func main() {
    for i := 0; i < 5; i++ {
        go func() {
            fmt.Println(i)
        }()
    }
    time.Sleep(time.Second)
}
```

**答案：** 可能输出 5, 5, 5, 5, 5（闭包捕获的是变量 i 的引用）

**正确写法：**
```go
for i := 0; i < 5; i++ {
    go func(n int) {
        fmt.Println(n)
    }(i)
}
```

---

## 最佳实践

### 1. 避免 Goroutine 泄漏

```go
// ✅ 使用 context 控制生命周期
func doWork(ctx context.Context) {
    go func() {
        for {
            select {
            case <-ctx.Done():
                return
            default:
                // 执行任务
            }
        }
    }()
}
```

### 2. 合理控制并发数

```go
// ✅ 使用 worker pool
func processItems(items []Item) {
    const workers = 10
    jobs := make(chan Item, len(items))
    
    for w := 0; w < workers; w++ {
        go worker(jobs)
    }
    
    for _, item := range items {
        jobs <- item
    }
    close(jobs)
}
```

### 3. 使用 WaitGroup 等待

```go
// ✅ 确保所有 goroutine 完成
var wg sync.WaitGroup
for i := 0; i < 10; i++ {
    wg.Add(1)
    go func(id int) {
        defer wg.Done()
        // 执行任务
    }(i)
}
wg.Wait()
```

### 4. 传递参数而非闭包

```go
// ✅ 传递参数
for i := 0; i < 10; i++ {
    go func(n int) {
        fmt.Println(n)
    }(i)
}

// ❌ 闭包捕获
for i := 0; i < 10; i++ {
    go func() {
        fmt.Println(i)
    }()
}
```

---

## 参考资料

- [x] [Go 官方文档 - Goroutines](https://go.dev/tour/concurrency/1)
- [x] [Effective Go - Goroutines](https://go.dev/doc/effective_go#goroutines)
- [x] [Go by Example - Goroutines](https://gobyexample.com/goroutines)
- [x] [Go 调度器设计文档](https://golang.org/s/go11sched)

---

**下一节：** [02-Channel机制](./02-Channel机制.md)
