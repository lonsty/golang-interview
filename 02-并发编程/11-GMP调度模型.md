# 11-GMP调度模型

[← 返回本章目录](./README.md) | [← 返回总目录](../README.md)

---

## 📋 目录

- [核心概念](#核心概念)
- [GMP模型详解](#gmp模型详解)
- [调度策略](#调度策略)
- [实现原理](#实现原理)
- [性能优化](#性能优化)
- [常见问题](#常见问题)
- [面试题](#面试题)
- [最佳实践](#最佳实践)
- [参考资料](#参考资料)

---

## 核心概念

GMP 是 Go 语言运行时（runtime）的调度器模型，用于管理和调度 goroutine 的执行。GMP 分别代表：

```
G (Goroutine)：Go 协程，是 Go 语言的执行单元
M (Machine)：操作系统线程，真正执行计算的资源
P (Processor)：处理器，调度的上下文，维护 G 的队列

GMP 模型的核心思想：
1. 用户态调度：在用户态完成调度，避免内核态切换开销
2. M:N 调度：M 个 goroutine 映射到 N 个 OS 线程
3. 工作窃取：空闲的 P 可以从其他 P 窃取 G
4. 抢占式调度：防止某个 G 长时间占用 CPU
```

---

## GMP模型详解

### 1. G (Goroutine)

```go
// runtime/runtime2.go
type g struct {
	stack       stack   // 栈内存：[stack.lo, stack.hi)
	stackguard0 uintptr // 栈溢出检测
	
	m         *m      // 当前绑定的 M
	sched     gobuf   // 调度信息（PC、SP 等）
	atomicstatus uint32 // 状态
	
	goid         int64  // goroutine ID
	waitsince    int64  // 等待时间
	waitreason   string // 等待原因
}

// Goroutine 的状态
const (
	_Gidle      = iota // 0: 刚刚分配，还未初始化
	_Grunnable         // 1: 在运行队列中，等待执行
	_Grunning          // 2: 正在执行
	_Gsyscall          // 3: 正在执行系统调用
	_Gwaiting          // 4: 被阻塞（IO、channel、锁等）
	_Gdead             // 6: 已经执行完毕
	_Gcopystack        // 8: 正在复制栈
	_Gpreempted        // 9: 被抢占
)
```

### 2. M (Machine)

```go
// runtime/runtime2.go
type m struct {
	g0      *g     // 用于执行调度的特殊 g
	curg    *g     // 当前正在执行的 g
	p       puintptr // 绑定的 P
	nextp   puintptr // 下一个要绑定的 P
	
	spinning bool   // 是否处于自旋状态
	blocked  bool   // 是否被阻塞
	
	park     note   // 休眠/唤醒机制
	alllink  *m     // 全局 M 链表
}

// M 的特点：
// 1. 数量可以动态增长（最多 10000 个）
// 2. 执行系统调用时会阻塞
// 3. 可以被复用
```

### 3. P (Processor)

```go
// runtime/runtime2.go
type p struct {
	id          int32
	status      uint32 // P 的状态
	link        puintptr
	m           muintptr // 绑定的 M
	
	// 本地运行队列
	runqhead uint32
	runqtail uint32
	runq     [256]guintptr // 本地队列（最多 256 个 G）
	
	runnext guintptr // 下一个要运行的 G（优先级最高）
	
	// 空闲 G 列表
	gFree struct {
		gList
		n int32
	}
}

// P 的状态
const (
	_Pidle    = iota // 0: 空闲
	_Prunning        // 1: 运行中
	_Psyscall        // 2: 系统调用中
	_Pgcstop         // 3: GC 停止
	_Pdead           // 4: 已死亡
)

// P 的数量：
// 默认等于 CPU 核心数（GOMAXPROCS）
// 可以通过 runtime.GOMAXPROCS() 设置
```

### 4. 全局队列

```go
// 全局运行队列
var (
	sched struct {
		lock mutex
		
		// 全局运行队列
		runq     gQueue
		runqsize int32
		
		// 空闲的 P 列表
		pidle      puintptr
		npidle     uint32
		
		// 空闲的 M 列表
		midle      muintptr
		nmidle     int32
		
		// 自旋的 M 数量
		nmspinning uint32
	}
)
```

---

## 调度策略

### 1. 调度流程

```
1. 创建 Goroutine
   go func() { ... }
   ↓
2. 放入 P 的本地队列（或全局队列）
   ↓
3. M 从 P 获取 G 执行
   ↓
4. G 执行完毕或被阻塞
   ↓
5. M 继续获取下一个 G

调度时机：
1. go 关键字创建新的 goroutine
2. GC
3. 系统调用
4. 同步操作（channel、锁等）
5. 主动让出 CPU（runtime.Gosched()）
```

### 2. 获取 G 的顺序

```go
// runtime/proc.go
func findrunnable() (gp *g, inheritTime bool) {
	// 1. 从 P 的 runnext 获取（优先级最高）
	if gp := _p_.runnext; gp != nil {
		return gp
	}
	
	// 2. 从 P 的本地队列获取
	if gp := runqget(_p_); gp != nil {
		return gp
	}
	
	// 3. 从全局队列获取
	if sched.runqsize != 0 {
		gp := globrunqget(_p_, 0)
		if gp != nil {
			return gp
		}
	}
	
	// 4. 从网络轮询器获取
	if netpollinited() {
		gp := netpoll(false)
		if gp != nil {
			return gp
		}
	}
	
	// 5. 工作窃取：从其他 P 窃取
	for i := 0; i < 4; i++ {
		for enum := stealOrder.start(); !enum.done(); enum.next() {
			p2 := allp[enum.position()]
			if gp := runqsteal(_p_, p2); gp != nil {
				return gp
			}
		}
	}
	
	// 6. 再次检查全局队列和网络轮询器
	// ...
	
	return nil
}
```

### 3. 工作窃取（Work Stealing）

```
工作窃取算法：
1. 当 P 的本地队列为空时
2. 随机选择一个其他的 P
3. 从其本地队列尾部窃取一半的 G
4. 放入自己的本地队列

优势：
- 负载均衡
- 减少全局队列的竞争
- 提高 CPU 利用率

示例：
P1: [G1, G2, G3, G4, G5, G6]
P2: []

窃取后：
P1: [G1, G2, G3]
P2: [G4, G5, G6]
```

### 4. 抢占式调度

```go
// Go 1.14 之前：协作式抢占
// - 在函数调用时检查抢占标记
// - 问题：无函数调用的循环无法被抢占

// Go 1.14+：基于信号的抢占
// - 使用 SIGURG 信号
// - 可以抢占任意执行中的 G

// 抢占时机：
// 1. G 执行超过 10ms
// 2. GC 需要停止所有 G
// 3. 系统监控检测到长时间运行的 G

// runtime/proc.go
func retake(now int64) uint32 {
	for i := 0; i < len(allp); i++ {
		_p_ := allp[i]
		pd := &_p_.sysmontick
		s := _p_.status
		
		if s == _Prunning || s == _Psyscall {
			// 检查是否运行超过 10ms
			t := int64(_p_.schedtick)
			if int64(pd.schedtick) != t {
				pd.schedtick = uint32(t)
				pd.schedwhen = now
			} else if pd.schedwhen+forcePreemptNS <= now {
				// 抢占
				preemptone(_p_)
			}
		}
	}
}
```

### 5. 系统调用处理

```
场景 1: 阻塞系统调用（如 read、write）
1. G 进入系统调用
2. M 和 P 解绑
3. P 寻找其他空闲的 M 或创建新的 M
4. 系统调用返回后，M 尝试重新获取 P
5. 如果获取失败，G 放入全局队列

场景 2: 非阻塞系统调用（如 epoll）
1. 使用网络轮询器（netpoller）
2. G 不会阻塞 M
3. 当 IO 就绪时，G 被唤醒

示例：
M1-P1-G1 → 系统调用
    ↓
M1-G1 (阻塞)
P1 → 寻找 M2
    ↓
M2-P1-G2 (继续执行其他 G)
```

---

## 实现原理

### 1. Goroutine 创建

```go
// 创建 goroutine
go func() {
	fmt.Println("Hello")
}()

// 编译器转换为：
runtime.newproc(size, fn)

// runtime/proc.go
func newproc(siz int32, fn *funcval) {
	// 1. 获取参数
	argp := add(unsafe.Pointer(&fn), sys.PtrSize)
	
	// 2. 获取当前 G 和 P
	gp := getg()
	pc := getcallerpc()
	
	// 3. 创建新的 G
	newg := newproc1(fn, argp, siz, gp, pc)
	
	// 4. 放入 P 的运行队列
	runqput(_p_, newg, true)
	
	// 5. 如果有空闲的 P，唤醒或创建 M
	if atomic.Load(&sched.npidle) != 0 && atomic.Load(&sched.nmspinning) == 0 {
		wakep()
	}
}
```

### 2. 调度循环

```go
// runtime/proc.go
func schedule() {
	_g_ := getg()
	
top:
	// 1. 检查是否需要 GC
	if sched.gcwaiting != 0 {
		gcstopm()
		goto top
	}
	
	// 2. 获取下一个要执行的 G
	var gp *g
	if gp == nil {
		gp, inheritTime = findrunnable() // 阻塞直到找到 G
	}
	
	// 3. 执行 G
	execute(gp, inheritTime)
}

func execute(gp *g, inheritTime bool) {
	_g_ := getg()
	
	// 1. 绑定 G 和 M
	_g_.m.curg = gp
	gp.m = _g_.m
	
	// 2. 切换到 G 的栈并执行
	gogo(&gp.sched)
}
```

### 3. 栈管理

```go
// Goroutine 栈：
// - 初始大小：2KB（Go 1.4+）
// - 最大大小：1GB（64位）/ 250MB（32位）
// - 动态增长：栈不够时自动扩容
// - 栈收缩：栈使用率低时自动收缩

// 栈扩容
func newstack() {
	// 1. 分配新栈（2倍大小）
	// 2. 复制旧栈数据到新栈
	// 3. 调整指针
	// 4. 释放旧栈
}
```

---

## 性能优化

### 1. GOMAXPROCS 设置

```go
package main

import (
	"fmt"
	"runtime"
)

func main() {
	// 获取当前 GOMAXPROCS
	fmt.Println("当前 GOMAXPROCS:", runtime.GOMAXPROCS(0))
	
	// 设置 GOMAXPROCS
	runtime.GOMAXPROCS(4)
	
	// 通常设置为 CPU 核心数
	runtime.GOMAXPROCS(runtime.NumCPU())
}
```

### 2. 减少 Goroutine 切换

```go
// ❌ 不推荐：频繁创建 goroutine
for i := 0; i < 1000000; i++ {
	go func() {
		// 简单操作
	}()
}

// ✅ 推荐：使用 worker pool
func workerPool() {
	jobs := make(chan int, 100)
	results := make(chan int, 100)
	
	// 固定数量的 workers
	for w := 1; w <= 10; w++ {
		go worker(jobs, results)
	}
	
	// 发送任务
	for j := 1; j <= 100; j++ {
		jobs <- j
	}
	close(jobs)
}
```

### 3. 避免 Goroutine 泄漏

```go
// ❌ 错误：goroutine 泄漏
func leak() {
	ch := make(chan int)
	go func() {
		val := <-ch // 永远阻塞
		fmt.Println(val)
	}()
	// ch 没有发送数据，goroutine 永远不会退出
}

// ✅ 正确：使用 context 或 done channel
func noLeak(ctx context.Context) {
	ch := make(chan int)
	go func() {
		select {
		case val := <-ch:
			fmt.Println(val)
		case <-ctx.Done():
			return
		}
	}()
}
```

### 4. 使用 sync.Pool 复用对象

```go
package main

import (
	"fmt"
	"sync"
)

var bufferPool = sync.Pool{
	New: func() interface{} {
		return make([]byte, 1024)
	},
}

func processData() {
	// 从池中获取
	buf := bufferPool.Get().([]byte)
	defer bufferPool.Put(buf) // 归还到池中
	
	// 使用 buf
	fmt.Println(len(buf))
}
```

---

## 常见问题

### 1. 为什么需要 P？

**答案：**

如果只有 G 和 M：
- M 需要频繁访问全局队列（需要加锁）
- 无法实现工作窃取
- 调度效率低

引入 P 后：
- 每个 P 有本地队列（无锁）
- 可以实现工作窃取
- 减少全局队列的竞争

### 2. M 的数量如何确定？

**答案：**

- 默认最多 10000 个
- 实际数量根据需要动态创建
- 通常略多于 P 的数量
- 系统调用会创建新的 M

### 3. Goroutine 和线程的区别？

**答案：**

| 特性 | Goroutine | 线程 |
|------|-----------|------|
| 创建成本 | ~2KB | ~2MB |
| 切换成本 | ~200ns | ~1-2μs |
| 调度方式 | 用户态 | 内核态 |
| 数量限制 | 百万级 | 千级 |

---

## 面试题

### 1. 什么是 GMP 模型？

**答案：**

GMP 是 Go 运行时的调度器模型：
- **G (Goroutine)**：Go 协程，轻量级线程
- **M (Machine)**：操作系统线程
- **P (Processor)**：处理器，调度上下文

**核心思想：** M:N 调度，M 个 goroutine 映射到 N 个 OS 线程

### 2. Go 调度器的调度时机有哪些？

**答案：**

1. 使用 `go` 关键字创建新 goroutine
2. GC
3. 系统调用
4. 同步操作（channel、锁等）
5. 主动让出 CPU（`runtime.Gosched()`）
6. 抢占式调度（执行超过 10ms）

### 3. 什么是工作窃取？

**答案：**

当 P 的本地队列为空时，从其他 P 的本地队列尾部窃取一半的 G。

**优势：**
- 负载均衡
- 减少全局队列竞争
- 提高 CPU 利用率

### 4. Go 1.14 的抢占式调度有什么改进？

**答案：**

**Go 1.14 之前：** 协作式抢占
- 在函数调用时检查抢占标记
- 无法抢占无函数调用的循环

**Go 1.14+：** 基于信号的抢占
- 使用 SIGURG 信号
- 可以抢占任意执行中的 G
- 解决了长时间运行的 G 无法被抢占的问题

### 5. 如何查看 Goroutine 的数量？

**答案：**

```go
// 方法 1: runtime.NumGoroutine()
fmt.Println("Goroutine 数量:", runtime.NumGoroutine())

// 方法 2: pprof
import _ "net/http/pprof"
// 访问 http://localhost:6060/debug/pprof/goroutine

// 方法 3: runtime.Stack()
buf := make([]byte, 1<<16)
runtime.Stack(buf, true)
fmt.Printf("%s", buf)
```

---

## 最佳实践

### 1. 合理设置 GOMAXPROCS

```go
// 通常设置为 CPU 核心数
runtime.GOMAXPROCS(runtime.NumCPU())

// 容器环境需要注意配额限制
```

### 2. 避免创建过多 Goroutine

```go
// ✅ 使用 worker pool 限制并发数
sem := make(chan struct{}, 100)
for i := 0; i < 10000; i++ {
	sem <- struct{}{}
	go func() {
		defer func() { <-sem }()
		// 工作
	}()
}
```

### 3. 及时释放资源

```go
// ✅ 使用 defer 确保资源释放
func process() {
	file, err := os.Open("file.txt")
	if err != nil {
		return
	}
	defer file.Close()
	
	// 处理文件
}
```

### 4. 使用 Context 控制生命周期

```go
// ✅ 使用 context 控制 goroutine
func worker(ctx context.Context) {
	for {
		select {
		case <-ctx.Done():
			return
		default:
			// 工作
		}
	}
}
```

### 5. 监控 Goroutine 泄漏

```go
// 定期检查 goroutine 数量
ticker := time.NewTicker(10 * time.Second)
go func() {
	for range ticker.C {
		fmt.Println("Goroutines:", runtime.NumGoroutine())
	}
}()
```

---

## 参考资料

- [x] [Go 调度器设计文档](https://golang.org/s/go11sched)
- [x] [Go 运行时源码](https://github.com/golang/go/tree/master/src/runtime)
- [x] [深入理解 Go 调度器](https://www.ardanlabs.com/blog/2018/08/scheduling-in-go-part1.html)
- [x] [Go 抢占式调度](https://github.com/golang/proposal/blob/master/design/24543-non-cooperative-preemption.md)

---

**上一节：** [10-并发模式](./10-并发模式.md)  
**下一节：** [12-并发陷阱与最佳实践](./12-并发陷阱与最佳实践.md)
