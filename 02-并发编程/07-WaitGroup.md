# 07-WaitGroup

[← 返回本章目录](./README.md) | [← 返回总目录](../README.md)

---

## 📋 目录

- [核心概念](#核心概念)
- [基本用法](#基本用法)
- [实现原理](#实现原理)
- [高级用法](#高级用法)
- [代码示例](#代码示例)
- [常见问题](#常见问题)
- [面试题](#面试题)
- [最佳实践](#最佳实践)
- [参考资料](#参考资料)

---

## 核心概念

WaitGroup 是 Go 语言提供的用于等待一组 goroutine 完成的同步原语。它提供了一个简单的计数器机制，用于协调多个 goroutine 的执行。

```
WaitGroup 的核心特性：
1. 计数器机制：内部维护一个计数器
2. Add(delta)：增加或减少计数器
3. Done()：计数器减 1（等价于 Add(-1)）
4. Wait()：阻塞直到计数器为 0
5. 零值可用：var wg sync.WaitGroup 即可使用
6. 不可复制：WaitGroup 不能被复制
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

func worker(id int, wg *sync.WaitGroup) {
	defer wg.Done() // 完成时调用 Done()

	fmt.Printf("Worker %d 开始工作\n", id)
	time.Sleep(time.Second)
	fmt.Printf("Worker %d 完成工作\n", id)
}

func main() {
	var wg sync.WaitGroup

	for i := 1; i <= 5; i++ {
		wg.Add(1) // 增加计数
		go worker(i, &wg)
	}

	wg.Wait() // 等待所有 goroutine 完成
	fmt.Println("所有任务完成")
}
```

### 2. 批量添加

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	var wg sync.WaitGroup

	tasks := []string{"任务A", "任务B", "任务C", "任务D", "任务E"}

	// 一次性添加所有任务
	wg.Add(len(tasks))

	for _, task := range tasks {
		task := task // 避免闭包问题
		go func() {
			defer wg.Done()
			fmt.Println("执行:", task)
		}()
	}

	wg.Wait()
	fmt.Println("所有任务完成")
}
```

### 3. 错误处理

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	var wg sync.WaitGroup
	errChan := make(chan error, 5)

	for i := 1; i <= 5; i++ {
		wg.Add(1)
		go func(id int) {
			defer wg.Done()

			// 模拟可能出错的操作
			if id%2 == 0 {
				errChan <- fmt.Errorf("任务 %d 失败", id)
				return
			}

			fmt.Printf("任务 %d 成功\n", id)
		}(i)
	}

	// 等待所有任务完成
	wg.Wait()
	close(errChan)

	// 收集错误
	for err := range errChan {
		fmt.Println("错误:", err)
	}
}
```

---

## 实现原理

### 1. WaitGroup 数据结构

```go
// sync/waitgroup.go
type WaitGroup struct {
	noCopy noCopy      // 防止复制
	state1 [3]uint32   // 状态数组
}

// state1 包含三个字段（64位对齐）：
// - counter：计数器（高32位）
// - waiter：等待者数量（低32位）
// - sema：信号量（32位）
```

### 2. Add 方法

```go
// Add 增加或减少计数器
func (wg *WaitGroup) Add(delta int) {
	// 1. 获取状态指针
	statep, semap := wg.state()

	// 2. 原子操作增加计数器
	state := atomic.AddUint64(statep, uint64(delta)<<32)
	v := int32(state >> 32) // 计数器值
	w := uint32(state)      // 等待者数量

	// 3. 检查计数器是否为负数
	if v < 0 {
		panic("sync: negative WaitGroup counter")
	}

	// 4. 检查是否在 Wait 之后调用 Add
	if w != 0 && delta > 0 && v == int32(delta) {
		panic("sync: WaitGroup misuse: Add called concurrently with Wait")
	}

	// 5. 如果计数器为 0 且有等待者，唤醒所有等待者
	if v == 0 && w != 0 {
		// 重置状态
		*statep = 0
		// 唤醒所有等待的 goroutine
		for ; w != 0; w-- {
			runtime_Semrelease(semap, false, 0)
		}
	}
}
```

### 3. Done 方法

```go
// Done 计数器减 1
func (wg *WaitGroup) Done() {
	wg.Add(-1)
}
```

### 4. Wait 方法

```go
// Wait 阻塞直到计数器为 0
func (wg *WaitGroup) Wait() {
	statep, semap := wg.state()

	for {
		state := atomic.LoadUint64(statep)
		v := int32(state >> 32) // 计数器值
		w := uint32(state)      // 等待者数量

		// 如果计数器已经为 0，直接返回
		if v == 0 {
			return
		}

		// 增加等待者计数
		if atomic.CompareAndSwapUint64(statep, state, state+1) {
			// 阻塞等待信号量
			runtime_Semacquire(semap)

			// 被唤醒后检查状态
			if *statep != 0 {
				panic("sync: WaitGroup is reused before previous Wait has returned")
			}
			return
		}
	}
}
```

### 5. 工作流程

```
初始状态：counter = 0, waiter = 0

Add(3)：
counter = 3

启动 3 个 goroutine，每个完成后调用 Done()：
Goroutine 1 Done(): counter = 2
Goroutine 2 Done(): counter = 1
Goroutine 3 Done(): counter = 0 → 唤醒所有等待者

Wait()：
如果 counter > 0：waiter++，阻塞
如果 counter == 0：直接返回
```

---

## 高级用法

### 1. 限制并发数

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

func main() {
	var wg sync.WaitGroup
	maxConcurrent := 3
	sem := make(chan struct{}, maxConcurrent)

	for i := 1; i <= 10; i++ {
		wg.Add(1)
		go func(id int) {
			defer wg.Done()

			// 获取信号量
			sem <- struct{}{}
			defer func() { <-sem }()

			fmt.Printf("任务 %d 开始\n", id)
			time.Sleep(time.Second)
			fmt.Printf("任务 %d 完成\n", id)
		}(i)
	}

	wg.Wait()
	fmt.Println("所有任务完成")
}
```

### 2. 超时控制

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

func waitWithTimeout(wg *sync.WaitGroup, timeout time.Duration) bool {
	done := make(chan struct{})

	go func() {
		wg.Wait()
		close(done)
	}()

	select {
	case <-done:
		return true
	case <-time.After(timeout):
		return false
	}
}

func main() {
	var wg sync.WaitGroup

	for i := 1; i <= 3; i++ {
		wg.Add(1)
		go func(id int) {
			defer wg.Done()
			time.Sleep(time.Duration(id) * time.Second)
			fmt.Printf("任务 %d 完成\n", id)
		}(i)
	}

	if waitWithTimeout(&wg, 2*time.Second) {
		fmt.Println("所有任务按时完成")
	} else {
		fmt.Println("超时！")
	}
}
```

### 3. 结果收集

```go
package main

import (
	"fmt"
	"sync"
)

// Result 结果结构体
type Result struct {
	ID    int
	Value int
	Error error
}

func main() {
	var wg sync.WaitGroup
	results := make(chan Result, 5)

	for i := 1; i <= 5; i++ {
		wg.Add(1)
		go func(id int) {
			defer wg.Done()

			// 模拟计算
			value := id * id
			results <- Result{ID: id, Value: value}
		}(i)
	}

	// 等待所有任务完成后关闭 channel
	go func() {
		wg.Wait()
		close(results)
	}()

	// 收集结果
	for result := range results {
		fmt.Printf("任务 %d: 结果 = %d\n", result.ID, result.Value)
	}
}
```

---

## 代码示例

### 示例 1: 并发下载

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

func download(url string, wg *sync.WaitGroup) {
	defer wg.Done()

	fmt.Printf("开始下载: %s\n", url)
	time.Sleep(time.Second) // 模拟下载
	fmt.Printf("完成下载: %s\n", url)
}

func main() {
	urls := []string{
		"https://example.com/file1.zip",
		"https://example.com/file2.zip",
		"https://example.com/file3.zip",
		"https://example.com/file4.zip",
		"https://example.com/file5.zip",
	}

	var wg sync.WaitGroup
	wg.Add(len(urls))

	for _, url := range urls {
		go download(url, &wg)
	}

	wg.Wait()
	fmt.Println("所有文件下载完成")
}
```

### 示例 2: 并发爬虫

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

// Page 页面结构体
type Page struct {
	URL   string
	Title string
}

func crawl(url string, wg *sync.WaitGroup, results chan<- Page) {
	defer wg.Done()

	fmt.Printf("爬取: %s\n", url)
	time.Sleep(500 * time.Millisecond) // 模拟爬取

	results <- Page{
		URL:   url,
		Title: fmt.Sprintf("标题-%s", url),
	}
}

func main() {
	urls := []string{
		"https://example.com/page1",
		"https://example.com/page2",
		"https://example.com/page3",
	}

	var wg sync.WaitGroup
	results := make(chan Page, len(urls))

	wg.Add(len(urls))
	for _, url := range urls {
		go crawl(url, &wg, results)
	}

	// 等待所有爬取完成
	go func() {
		wg.Wait()
		close(results)
	}()

	// 收集结果
	for page := range results {
		fmt.Printf("收到: %s - %s\n", page.URL, page.Title)
	}
}
```

### 示例 3: 并发数据处理

```go
package main

import (
	"fmt"
	"sync"
)

func process(data int, wg *sync.WaitGroup, results chan<- int) {
	defer wg.Done()

	// 模拟数据处理
	result := data * data
	results <- result
}

func main() {
	data := []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}

	var wg sync.WaitGroup
	results := make(chan int, len(data))

	wg.Add(len(data))
	for _, d := range data {
		go process(d, &wg, results)
	}

	// 等待所有处理完成
	go func() {
		wg.Wait()
		close(results)
	}()

	// 收集结果
	sum := 0
	for result := range results {
		sum += result
	}

	fmt.Printf("总和: %d\n", sum)
}
```

### 示例 4: 并发任务处理器

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

// Task 任务接口
type Task interface {
	Execute() error
}

// SimpleTask 简单任务
type SimpleTask struct {
	ID   int
	Name string
}

// Execute 执行任务
func (t *SimpleTask) Execute() error {
	fmt.Printf("执行任务 %d: %s\n", t.ID, t.Name)
	time.Sleep(time.Millisecond * 500)
	return nil
}

// TaskProcessor 任务处理器
type TaskProcessor struct {
	wg     sync.WaitGroup
	tasks  []Task
	errors []error
	mu     sync.Mutex
}

// NewTaskProcessor 创建任务处理器
func NewTaskProcessor() *TaskProcessor {
	return &TaskProcessor{
		tasks:  make([]Task, 0),
		errors: make([]error, 0),
	}
}

// AddTask 添加任务
func (p *TaskProcessor) AddTask(task Task) {
	p.tasks = append(p.tasks, task)
}

// Process 处理所有任务
func (p *TaskProcessor) Process() {
	for _, task := range p.tasks {
		p.wg.Add(1)
		go func(t Task) {
			defer p.wg.Done()

			if err := t.Execute(); err != nil {
				p.mu.Lock()
				p.errors = append(p.errors, err)
				p.mu.Unlock()
			}
		}(task)
	}
}

// Wait 等待所有任务完成
func (p *TaskProcessor) Wait() []error {
	p.wg.Wait()
	return p.errors
}

func main() {
	processor := NewTaskProcessor()

	// 添加任务
	for i := 1; i <= 10; i++ {
		processor.AddTask(&SimpleTask{
			ID:   i,
			Name: fmt.Sprintf("任务-%d", i),
		})
	}

	// 处理任务
	processor.Process()

	// 等待完成
	errors := processor.Wait()

	if len(errors) > 0 {
		fmt.Printf("有 %d 个任务失败\n", len(errors))
	} else {
		fmt.Println("所有任务成功完成")
	}
}
```

---

## 常见问题

### 1. 为什么不能复制 WaitGroup？

```go
// ❌ 错误：复制 WaitGroup
func bad() {
	var wg1 sync.WaitGroup
	wg1.Add(1)

	wg2 := wg1 // 错误：复制了 WaitGroup

	go func() {
		wg2.Done() // 只会影响 wg2
	}()

	wg1.Wait() // 永远阻塞
}

// ✅ 正确：使用指针
func good() {
	var wg sync.WaitGroup
	wg.Add(1)

	go func(wg *sync.WaitGroup) {
		defer wg.Done()
		// 工作
	}(&wg)

	wg.Wait()
}
```

### 2. Add 和 Done 必须配对吗？

```go
// ❌ 错误：Add 和 Done 不配对
func bad() {
	var wg sync.WaitGroup
	wg.Add(2)

	go func() {
		defer wg.Done()
		// 工作
	}()

	// 缺少一个 Done()
	wg.Wait() // 永远阻塞
}

// ✅ 正确：确保配对
func good() {
	var wg sync.WaitGroup
	wg.Add(2)

	go func() {
		defer wg.Done()
		// 工作 1
	}()

	go func() {
		defer wg.Done()
		// 工作 2
	}()

	wg.Wait()
}
```

### 3. 可以在 Wait 之后调用 Add 吗？

```go
// ❌ 错误：Wait 之后调用 Add
func bad() {
	var wg sync.WaitGroup

	go func() {
		wg.Wait()
		wg.Add(1) // panic: WaitGroup misuse
	}()
}

// ✅ 正确：在 Wait 之前调用 Add
func good() {
	var wg sync.WaitGroup
	wg.Add(1)

	go func() {
		defer wg.Done()
		// 工作
	}()

	wg.Wait()
}
```

### 4. 可以在 goroutine 内部调用 Add 吗？

```go
// ❌ 不推荐：可能导致竞态条件
func bad() {
	var wg sync.WaitGroup

	for i := 0; i < 10; i++ {
		go func() {
			wg.Add(1) // 可能在 Wait() 之后执行
			defer wg.Done()
			// 工作
		}()
	}

	wg.Wait()
}

// ✅ 推荐：在启动 goroutine 前调用 Add
func good() {
	var wg sync.WaitGroup

	for i := 0; i < 10; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			// 工作
		}()
	}

	wg.Wait()
}
```

---

## 面试题

### 1. WaitGroup 的实现原理是什么？

**答案：**

WaitGroup 使用一个 64 位的状态字段实现：
- **高 32 位**：计数器（counter）
- **低 32 位**：等待者数量（waiter）
- **信号量**：用于阻塞和唤醒

**工作流程：**
1. **Add(n)**：原子操作将计数器加 n
2. **Done()**：原子操作将计数器减 1，如果变为 0 且有等待者，唤醒所有等待者
3. **Wait()**：如果计数器不为 0，增加等待者计数并阻塞；否则直接返回

### 2. WaitGroup 可以重用吗？

**答案：**

可以，但必须等待上一次的 Wait() 返回后才能重用。

```go
// ✅ 正确：等待 Wait 返回后重用
var wg sync.WaitGroup

// 第一次使用
wg.Add(1)
go func() { defer wg.Done(); /* 工作 */ }()
wg.Wait()

// 第二次使用
wg.Add(1)
go func() { defer wg.Done(); /* 工作 */ }()
wg.Wait()
```

### 3. WaitGroup 和 Channel 如何选择？

**答案：**

| 场景 | 选择 | 原因 |
|------|------|------|
| 只需要等待完成 | WaitGroup | 简单直接 |
| 需要传递数据 | Channel | 更灵活 |
| 需要超时控制 | Channel + select | 支持超时 |
| 大量 goroutine | WaitGroup | 性能更好 |

### 4. 如何实现带超时的 WaitGroup？

**答案：**

```go
func waitWithTimeout(wg *sync.WaitGroup, timeout time.Duration) bool {
	done := make(chan struct{})

	go func() {
		wg.Wait()
		close(done)
	}()

	select {
	case <-done:
		return true
	case <-time.After(timeout):
		return false
	}
}
```

### 5. WaitGroup 计数器为负数会怎样？

**答案：**

会触发 panic: "sync: negative WaitGroup counter"

```go
var wg sync.WaitGroup
wg.Add(1)
wg.Done()
wg.Done() // panic: sync: negative WaitGroup counter
```

---

## 最佳实践

### 1. 使用 defer 调用 Done

```go
// ✅ 推荐
func worker(wg *sync.WaitGroup) {
	defer wg.Done()
	// 工作逻辑
}
```

### 2. 在启动 goroutine 前调用 Add

```go
// ✅ 推荐
wg.Add(1)
go func() {
	defer wg.Done()
	// 工作
}()

// ❌ 不推荐
go func() {
	wg.Add(1) // 可能导致竞态条件
	defer wg.Done()
	// 工作
}()
```

### 3. 传递指针而非值

```go
// ✅ 推荐
func worker(wg *sync.WaitGroup) {
	defer wg.Done()
	// 工作
}

// ❌ 错误
func worker(wg sync.WaitGroup) { // 复制了 WaitGroup
	defer wg.Done()
	// 工作
}
```

### 4. 确保 Add 和 Done 配对

```go
// ✅ 推荐
wg.Add(len(tasks))
for _, task := range tasks {
	go func(t Task) {
		defer wg.Done()
		process(t)
	}(task)
}
```

### 5. 避免在循环中直接使用循环变量

```go
// ❌ 错误：闭包问题
for _, task := range tasks {
	wg.Add(1)
	go func() {
		defer wg.Done()
		process(task) // task 是循环变量，可能被修改
	}()
}

// ✅ 正确：传递参数或创建局部变量
for _, task := range tasks {
	wg.Add(1)
	task := task // 创建局部变量
	go func() {
		defer wg.Done()
		process(task)
	}()
}

// 或者
for _, task := range tasks {
	wg.Add(1)
	go func(t Task) {
		defer wg.Done()
		process(t)
	}(task)
}
```

---

## 参考资料

- [x] [Go 官方文档 - sync.WaitGroup](https://pkg.go.dev/sync#WaitGroup)
- [x] [Go WaitGroup 源码分析](https://github.com/golang/go/blob/master/src/sync/waitgroup.go)
- [x] [深入理解 Go WaitGroup](https://colobu.com/2018/12/23/dive-into-go-waitgroup/)
- [x] [Effective Go - Concurrency](https://go.dev/doc/effective_go#concurrency)

---

**上一节：** [06-RWMutex读写锁](./06-RWMutex读写锁.md)  
**下一节：** [08-Once单例模式](./08-Once单例模式.md)
