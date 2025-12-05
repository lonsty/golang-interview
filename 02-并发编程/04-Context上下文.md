# 06-Context上下文

[← 返回本章目录](./README.md) | [← 返回总目录](../README.md)

---

## 📋 目录

- [核心概念](#核心概念)
- [Context接口](#context接口)
- [Context类型](#context类型)
- [使用场景](#使用场景)
- [实现原理](#实现原理)
- [代码示例](#代码示例)
- [常见问题](#常见问题)
- [面试题](#面试题)
- [最佳实践](#最佳实践)
- [参考资料](#参考资料)

---

## 核心概念

Context（上下文）是 Go 1.7 引入的标准库，用于在 goroutine 之间传递截止时间、取消信号和请求范围的值。

```
Context 的三大功能：
1. 取消信号传播：父 Context 取消时，所有子 Context 自动取消
2. 超时控制：设置操作的截止时间
3. 值传递：在调用链中传递请求范围的数据
```

---

## Context接口

### 1. 接口定义

```go
// context.Context 接口定义
type Context interface {
    // Deadline 返回 context 的截止时间
    Deadline() (deadline time.Time, ok bool)
    
    // Done 返回一个 channel，当 context 被取消或超时时关闭
    Done() <-chan struct{}
    
    // Err 返回 context 结束的原因
    Err() error
    
    // Value 返回 context 中的值
    Value(key interface{}) interface{}
}
```

### 2. 错误类型

```go
var Canceled = errors.New("context canceled")
var DeadlineExceeded error = deadlineExceededError{}

// Err() 可能返回的错误：
// - nil：context 未取消
// - Canceled：context 被取消
// - DeadlineExceeded：context 超时
```

---

## Context类型

### 1. Background 和 TODO

```go
package main

import (
    "context"
    "fmt"
)

func main() {
    // Background：根 context，永不取消，没有值，没有截止时间
    // 通常用于 main 函数、初始化和测试
    ctx := context.Background()
    fmt.Printf("Background: %v\n", ctx)
    
    // TODO：当不确定使用什么 context 时使用
    // 通常用于重构时的临时占位
    ctx2 := context.TODO()
    fmt.Printf("TODO: %v\n", ctx2)
}
```

### 2. WithCancel

```go
package main

import (
    "context"
    "fmt"
    "time"
)

func main() {
    // 创建可取消的 context
    ctx, cancel := context.WithCancel(context.Background())
    
    go func() {
        for {
            select {
            case <-ctx.Done():
                fmt.Println("goroutine 收到取消信号")
                return
            default:
                fmt.Println("工作中...")
                time.Sleep(time.Millisecond * 500)
            }
        }
    }()
    
    time.Sleep(time.Second * 2)
    cancel() // 取消 context
    time.Sleep(time.Second)
}
```

### 3. WithTimeout

```go
package main

import (
    "context"
    "fmt"
    "time"
)

func main() {
    // 创建带超时的 context
    ctx, cancel := context.WithTimeout(context.Background(), time.Second*2)
    defer cancel() // 确保资源释放
    
    go func() {
        select {
        case <-ctx.Done():
            fmt.Println("超时:", ctx.Err())
        }
    }()
    
    time.Sleep(time.Second * 3)
}
```

### 4. WithDeadline

```go
package main

import (
    "context"
    "fmt"
    "time"
)

func main() {
    // 创建带截止时间的 context
    deadline := time.Now().Add(time.Second * 2)
    ctx, cancel := context.WithDeadline(context.Background(), deadline)
    defer cancel()
    
    go func() {
        select {
        case <-ctx.Done():
            fmt.Println("到达截止时间:", ctx.Err())
        }
    }()
    
    time.Sleep(time.Second * 3)
}
```

### 5. WithValue

```go
package main

import (
    "context"
    "fmt"
)

// 定义 key 类型，避免冲突
type contextKey string

const (
    userIDKey contextKey = "userID"
    traceIDKey contextKey = "traceID"
)

func main() {
    // 创建带值的 context
    ctx := context.WithValue(context.Background(), userIDKey, "12345")
    ctx = context.WithValue(ctx, traceIDKey, "trace-abc")
    
    // 获取值
    if userID, ok := ctx.Value(userIDKey).(string); ok {
        fmt.Println("User ID:", userID)
    }
    
    if traceID, ok := ctx.Value(traceIDKey).(string); ok {
        fmt.Println("Trace ID:", traceID)
    }
}
```

---

## 使用场景

### 1. HTTP 请求超时控制

```go
package main

import (
    "context"
    "fmt"
    "net/http"
    "time"
)

func fetchData(ctx context.Context, url string) error {
    req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
    if err != nil {
        return err
    }
    
    client := &http.Client{}
    resp, err := client.Do(req)
    if err != nil {
        return err
    }
    defer resp.Body.Close()
    
    fmt.Println("请求成功:", resp.Status)
    return nil
}

func main() {
    ctx, cancel := context.WithTimeout(context.Background(), time.Second*3)
    defer cancel()
    
    if err := fetchData(ctx, "https://www.google.com"); err != nil {
        fmt.Println("请求失败:", err)
    }
}
```

### 2. 数据库查询超时

```go
package main

import (
    "context"
    "database/sql"
    "fmt"
    "time"
)

func queryDatabase(ctx context.Context, db *sql.DB) error {
    // 使用 context 控制查询超时
    rows, err := db.QueryContext(ctx, "SELECT * FROM users WHERE id = ?", 1)
    if err != nil {
        return err
    }
    defer rows.Close()
    
    for rows.Next() {
        var id int
        var name string
        if err := rows.Scan(&id, &name); err != nil {
            return err
        }
        fmt.Printf("ID: %d, Name: %s\n", id, name)
    }
    
    return rows.Err()
}

func main() {
    // 模拟数据库连接
    // db, _ := sql.Open("mysql", "user:password@/dbname")
    // defer db.Close()
    
    ctx, cancel := context.WithTimeout(context.Background(), time.Second*5)
    defer cancel()
    
    // if err := queryDatabase(ctx, db); err != nil {
    //     fmt.Println("查询失败:", err)
    // }
}
```

### 3. 并发任务取消

```go
package main

import (
    "context"
    "fmt"
    "sync"
    "time"
)

func worker(ctx context.Context, id int, wg *sync.WaitGroup) {
    defer wg.Done()
    
    for {
        select {
        case <-ctx.Done():
            fmt.Printf("Worker %d 收到取消信号\n", id)
            return
        default:
            fmt.Printf("Worker %d 工作中...\n", id)
            time.Sleep(time.Millisecond * 500)
        }
    }
}

func main() {
    ctx, cancel := context.WithCancel(context.Background())
    var wg sync.WaitGroup
    
    // 启动多个 worker
    for i := 1; i <= 3; i++ {
        wg.Add(1)
        go worker(ctx, i, &wg)
    }
    
    // 2秒后取消所有 worker
    time.Sleep(time.Second * 2)
    cancel()
    
    wg.Wait()
    fmt.Println("所有 worker 已停止")
}
```

---

## 实现原理

### 1. Context 树形结构

```go
// context 形成树形结构
// Background/TODO (根节点)
//     |
//     +-- WithCancel
//     |       |
//     |       +-- WithTimeout
//     |       |       |
//     |       |       +-- WithValue
//     |       |
//     |       +-- WithValue
//     |
//     +-- WithDeadline

// 父节点取消时，所有子节点自动取消
```

### 2. cancelCtx 实现

```go
// cancelCtx 可以被取消
type cancelCtx struct {
    Context
    
    mu       sync.Mutex            // 保护以下字段
    done     chan struct{}         // 延迟创建，第一次取消时关闭
    children map[canceler]struct{} // 第一次取消时设为 nil
    err      error                 // 第一次取消时设置
}

// cancel 方法
func (c *cancelCtx) cancel(removeFromParent bool, err error) {
    if err == nil {
        panic("context: internal error: missing cancel error")
    }
    c.mu.Lock()
    if c.err != nil {
        c.mu.Unlock()
        return // 已经被取消
    }
    c.err = err
    if c.done == nil {
        c.done = closedchan
    } else {
        close(c.done)
    }
    // 取消所有子节点
    for child := range c.children {
        child.cancel(false, err)
    }
    c.children = nil
    c.mu.Unlock()
    
    if removeFromParent {
        removeChild(c.Context, c)
    }
}
```

### 3. timerCtx 实现

```go
// timerCtx 带有定时器和截止时间
type timerCtx struct {
    cancelCtx
    timer *time.Timer // 在 cancelCtx.mu 下
    
    deadline time.Time
}

// 到达截止时间时自动取消
func (c *timerCtx) cancel(removeFromParent bool, err error) {
    c.cancelCtx.cancel(false, err)
    if removeFromParent {
        removeChild(c.cancelCtx.Context, c)
    }
    c.mu.Lock()
    if c.timer != nil {
        c.timer.Stop()
        c.timer = nil
    }
    c.mu.Unlock()
}
```

### 4. valueCtx 实现

```go
// valueCtx 携带键值对
type valueCtx struct {
    Context
    key, val interface{}
}

// Value 方法：递归查找
func (c *valueCtx) Value(key interface{}) interface{} {
    if c.key == key {
        return c.val
    }
    return c.Context.Value(key)
}
```

---

## 代码示例

### 示例 1: HTTP 服务器优雅关闭

```go
package main

import (
    "context"
    "fmt"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    "time"
)

func main() {
    server := &http.Server{
        Addr: ":8080",
        Handler: http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            // 模拟耗时操作
            time.Sleep(time.Second * 2)
            fmt.Fprintf(w, "Hello, World!")
        }),
    }
    
    // 启动服务器
    go func() {
        fmt.Println("服务器启动在 :8080")
        if err := server.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            fmt.Printf("服务器错误: %v\n", err)
        }
    }()
    
    // 等待中断信号
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit
    
    fmt.Println("正在关闭服务器...")
    
    // 创建超时 context
    ctx, cancel := context.WithTimeout(context.Background(), time.Second*5)
    defer cancel()
    
    // 优雅关闭
    if err := server.Shutdown(ctx); err != nil {
        fmt.Printf("服务器强制关闭: %v\n", err)
    }
    
    fmt.Println("服务器已关闭")
}
```

### 示例 2: 管道处理与取消

```go
package main

import (
    "context"
    "fmt"
    "time"
)

// 生成器
func generator(ctx context.Context) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        n := 1
        for {
            select {
            case <-ctx.Done():
                return
            case out <- n:
                n++
                time.Sleep(time.Millisecond * 100)
            }
        }
    }()
    return out
}

// 处理器
func processor(ctx context.Context, in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for {
            select {
            case <-ctx.Done():
                return
            case n, ok := <-in:
                if !ok {
                    return
                }
                select {
                case <-ctx.Done():
                    return
                case out <- n * 2:
                }
            }
        }
    }()
    return out
}

func main() {
    ctx, cancel := context.WithTimeout(context.Background(), time.Second*2)
    defer cancel()
    
    // 构建管道
    numbers := generator(ctx)
    doubled := processor(ctx, numbers)
    
    // 消费数据
    for n := range doubled {
        fmt.Println(n)
    }
    
    fmt.Println("管道已关闭")
}
```

### 示例 3: 请求链路追踪

```go
package main

import (
    "context"
    "fmt"
    "math/rand"
    "time"
)

type contextKey string

const (
    requestIDKey contextKey = "requestID"
    userIDKey    contextKey = "userID"
)

// 生成请求 ID
func generateRequestID() string {
    return fmt.Sprintf("req-%d", rand.Int63())
}

// 中间件：注入请求 ID
func withRequestID(ctx context.Context) context.Context {
    return context.WithValue(ctx, requestIDKey, generateRequestID())
}

// 中间件：注入用户 ID
func withUserID(ctx context.Context, userID string) context.Context {
    return context.WithValue(ctx, userIDKey, userID)
}

// 获取请求 ID
func getRequestID(ctx context.Context) string {
    if reqID, ok := ctx.Value(requestIDKey).(string); ok {
        return reqID
    }
    return "unknown"
}

// 获取用户 ID
func getUserID(ctx context.Context) string {
    if userID, ok := ctx.Value(userIDKey).(string); ok {
        return userID
    }
    return "anonymous"
}

// 业务函数
func handleRequest(ctx context.Context) {
    reqID := getRequestID(ctx)
    userID := getUserID(ctx)
    
    fmt.Printf("[%s] 处理用户 %s 的请求\n", reqID, userID)
    
    // 调用其他服务
    callService(ctx)
}

func callService(ctx context.Context) {
    reqID := getRequestID(ctx)
    userID := getUserID(ctx)
    
    fmt.Printf("[%s] 服务调用，用户: %s\n", reqID, userID)
}

func main() {
    // 创建根 context
    ctx := context.Background()
    
    // 注入请求 ID
    ctx = withRequestID(ctx)
    
    // 注入用户 ID
    ctx = withUserID(ctx, "user-123")
    
    // 处理请求
    handleRequest(ctx)
}
```

---

## 常见问题

### 1. Context 值传递的问题

```go
// ❌ 错误：不要用 context 传递可选参数
func badFunction(ctx context.Context) {
    timeout := ctx.Value("timeout").(int) // 不推荐
}

// ✅ 正确：使用函数参数
func goodFunction(ctx context.Context, timeout int) {
    // 推荐
}
```

### 2. Context 应该作为第一个参数

```go
// ❌ 错误
func badFunction(name string, ctx context.Context) {}

// ✅ 正确
func goodFunction(ctx context.Context, name string) {}
```

### 3. 不要存储 Context

```go
// ❌ 错误：不要在结构体中存储 context
type BadStruct struct {
    ctx context.Context
}

// ✅ 正确：作为参数传递
type GoodStruct struct {
    name string
}

func (g *GoodStruct) DoSomething(ctx context.Context) {
    // 使用 ctx
}
```

### 4. 必须调用 cancel 函数

```go
// ❌ 错误：忘记调用 cancel
func badCancel() {
    ctx, cancel := context.WithCancel(context.Background())
    // 忘记调用 cancel，导致资源泄漏
    _ = cancel
}

// ✅ 正确：使用 defer 确保调用
func goodCancel() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel() // 确保资源释放
    
    // 使用 ctx
}
```

---

## 面试题

### 1. Context 的作用是什么？

**答案：**

Context 主要有三个作用：
1. **取消信号传播**：父 Context 取消时，所有子 Context 自动取消
2. **超时控制**：设置操作的截止时间，防止 goroutine 泄漏
3. **值传递**：在调用链中传递请求范围的数据（如请求 ID、用户信息）

### 2. WithCancel、WithTimeout、WithDeadline 的区别？

**答案：**

- **WithCancel**：手动取消，调用 cancel() 函数取消
- **WithTimeout**：相对时间，从现在开始计算超时时间
- **WithDeadline**：绝对时间，指定具体的截止时间点

```go
// WithCancel：手动取消
ctx, cancel := context.WithCancel(parent)
cancel() // 手动调用

// WithTimeout：2秒后自动取消
ctx, cancel := context.WithTimeout(parent, 2*time.Second)

// WithDeadline：指定时间点取消
deadline := time.Now().Add(2*time.Second)
ctx, cancel := context.WithDeadline(parent, deadline)
```

### 3. Context.Value 应该存储什么类型的数据？

**答案：**

Context.Value 应该只存储**请求范围的数据**，不应该用于传递可选参数。

**适合存储的数据：**
- 请求 ID、追踪 ID
- 用户身份信息
- 请求的元数据

**不适合存储的数据：**
- 函数的可选参数
- 业务逻辑数据
- 配置信息

### 4. 为什么 Context 要作为函数的第一个参数？

**答案：**

这是 Go 的约定俗成的规范，原因包括：
1. **一致性**：所有使用 Context 的函数都遵循相同的模式
2. **可读性**：一眼就能看出函数支持取消和超时
3. **工具支持**：静态分析工具可以检查 Context 的使用

---

## 最佳实践

### 1. 不要在结构体中存储 Context

```go
// ❌ 不推荐
type Server struct {
    ctx context.Context
}

// ✅ 推荐
type Server struct {
    name string
}

func (s *Server) Serve(ctx context.Context) error {
    // 使用 ctx
    return nil
}
```

### 2. Context 作为第一个参数

```go
// ✅ 推荐
func DoSomething(ctx context.Context, arg1 string, arg2 int) error {
    // ...
    return nil
}
```

### 3. 使用 defer 确保 cancel 被调用

```go
// ✅ 推荐
func process() {
    ctx, cancel := context.WithTimeout(context.Background(), time.Second*5)
    defer cancel() // 确保资源释放
    
    // 使用 ctx
}
```

### 4. 使用自定义类型作为 key

```go
// ✅ 推荐：避免 key 冲突
type contextKey string

const userIDKey contextKey = "userID"

ctx := context.WithValue(parent, userIDKey, "123")
```

### 5. 不要传递 nil Context

```go
// ❌ 不推荐
func bad() {
    doSomething(nil) // 不要传递 nil
}

// ✅ 推荐
func good() {
    doSomething(context.Background())
    // 或
    doSomething(context.TODO())
}
```

---

## 参考资料

- [x] [Go 官方文档 - context](https://pkg.go.dev/context)
- [x] [Go Concurrency Patterns: Context](https://go.dev/blog/context)
- [x] [Context isn't for cancellation](https://dave.cheney.net/2017/08/20/context-isnt-for-cancellation)
- [x] [How to correctly use context.Context](https://go.dev/blog/context-and-structs)

---

**上一节：** [05-WaitGroup与Once](./05-WaitGroup与Once.md)  
**下一节：** [07-原子操作](./07-原子操作.md)
