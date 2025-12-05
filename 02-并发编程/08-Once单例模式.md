# 08-Once单例模式

[← 返回本章目录](./README.md) | [← 返回总目录](../README.md)

---

## 📋 目录

- [核心概念](#核心概念)
- [基本用法](#基本用法)
- [实现原理](#实现原理)
- [单例模式实现](#单例模式实现)
- [代码示例](#代码示例)
- [常见问题](#常见问题)
- [面试题](#面试题)
- [最佳实践](#最佳实践)
- [参考资料](#参考资料)

---

## 核心概念

`sync.Once` 是 Go 语言提供的用于确保某个操作只执行一次的同步原语。它常用于单例模式、资源初始化等场景。

```
Once 的核心特性：
1. 只执行一次：无论调用多少次，函数只会执行一次
2. 并发安全：多个 goroutine 同时调用也只执行一次
3. 阻塞等待：第一次调用执行时，其他调用会阻塞等待
4. 零值可用：var once sync.Once 即可使用
5. 不可复制：Once 不能被复制
6. 无返回值：Do 方法不返回任何值
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

var once sync.Once

func initialize() {
	fmt.Println("初始化操作，只执行一次")
}

func main() {
	for i := 0; i < 5; i++ {
		once.Do(initialize)
	}
	// 输出：初始化操作，只执行一次（只输出一次）
}
```

### 2. 并发场景

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

var once sync.Once

func initialize() {
	fmt.Println("开始初始化...")
	time.Sleep(2 * time.Second)
	fmt.Println("初始化完成")
}

func worker(id int, wg *sync.WaitGroup) {
	defer wg.Done()
	fmt.Printf("Worker %d 调用 Once\n", id)
	once.Do(initialize)
	fmt.Printf("Worker %d 完成\n", id)
}

func main() {
	var wg sync.WaitGroup

	for i := 1; i <= 5; i++ {
		wg.Add(1)
		go worker(i, &wg)
	}

	wg.Wait()
}
```

### 3. 带参数的初始化

```go
package main

import (
	"fmt"
	"sync"
)

var once sync.Once
var config map[string]string

func initConfig(filename string) {
	once.Do(func() {
		fmt.Printf("加载配置文件: %s\n", filename)
		config = map[string]string{
			"host": "localhost",
			"port": "8080",
		}
	})
}

func main() {
	initConfig("config.yaml")
	initConfig("other.yaml") // 不会执行

	fmt.Println(config)
}
```

---

## 实现原理

### 1. Once 数据结构

```go
// sync/once.go
type Once struct {
	done uint32      // 标记是否已执行（原子操作）
	m    Mutex       // 互斥锁
}
```

### 2. Do 方法实现

```go
// Do 确保函数 f 只执行一次
func (o *Once) Do(f func()) {
	// 快速路径：如果已经执行过，直接返回
	if atomic.LoadUint32(&o.done) == 0 {
		// 慢速路径：需要加锁
		o.doSlow(f)
	}
}

func (o *Once) doSlow(f func()) {
	o.m.Lock()
	defer o.m.Unlock()

	// 双重检查：再次检查是否已执行
	if o.done == 0 {
		// 先执行函数，再设置标记
		defer atomic.StoreUint32(&o.done, 1)
		f()
	}
}
```

### 3. 工作流程

```
初始状态：done = 0

第一个 goroutine 调用 Do(f)：
1. atomic.LoadUint32(&o.done) == 0 → 进入 doSlow
2. 获取锁
3. 再次检查 done == 0
4. 执行 f()
5. 设置 done = 1
6. 释放锁

其他 goroutine 调用 Do(f)：
- 如果在第一个 goroutine 执行期间：
  1. atomic.LoadUint32(&o.done) == 0 → 进入 doSlow
  2. 尝试获取锁（阻塞等待）
  3. 获取锁后检查 done == 1
  4. 直接返回

- 如果在第一个 goroutine 执行完成后：
  1. atomic.LoadUint32(&o.done) == 1 → 直接返回
```

### 4. 为什么使用双重检查？

```go
// 双重检查的原因：
// 1. 第一次检查（原子操作）：避免每次都加锁，提高性能
// 2. 第二次检查（加锁后）：确保只有一个 goroutine 执行函数

// 如果只有一次检查：
func (o *Once) Do(f func()) {
	o.m.Lock()
	defer o.m.Unlock()
	if o.done == 0 {
		defer atomic.StoreUint32(&o.done, 1)
		f()
	}
}
// 问题：每次调用都需要加锁，性能差

// 如果没有第二次检查：
func (o *Once) Do(f func()) {
	if atomic.LoadUint32(&o.done) == 0 {
		o.m.Lock()
		defer o.m.Unlock()
		// 没有第二次检查
		defer atomic.StoreUint32(&o.done, 1)
		f()
	}
}
// 问题：多个 goroutine 可能同时通过第一次检查，导致 f() 执行多次
```

---

## 单例模式实现

### 1. 懒汉式单例（使用 Once）

```go
package main

import (
	"fmt"
	"sync"
)

// Singleton 单例结构体
type Singleton struct {
	data string
}

var (
	instance *Singleton
	once     sync.Once
)

// GetInstance 获取单例实例
func GetInstance() *Singleton {
	once.Do(func() {
		fmt.Println("创建单例实例")
		instance = &Singleton{
			data: "单例数据",
		}
	})
	return instance
}

func main() {
	var wg sync.WaitGroup

	for i := 0; i < 10; i++ {
		wg.Add(1)
		go func(id int) {
			defer wg.Done()
			s := GetInstance()
			fmt.Printf("Goroutine %d: %p\n", id, s)
		}(i)
	}

	wg.Wait()
	// 所有 goroutine 获取的都是同一个实例（地址相同）
}
```

### 2. 饿汉式单例（包初始化）

```go
package main

import "fmt"

// Singleton 单例结构体
type Singleton struct {
	data string
}

// 包初始化时创建实例
var instance = &Singleton{
	data: "单例数据",
}

// GetInstance 获取单例实例
func GetInstance() *Singleton {
	return instance
}

func main() {
	s1 := GetInstance()
	s2 := GetInstance()
	fmt.Printf("s1: %p\n", s1)
	fmt.Printf("s2: %p\n", s2)
	fmt.Println(s1 == s2) // true
}
```

### 3. 带初始化参数的单例

```go
package main

import (
	"fmt"
	"sync"
)

// Config 配置单例
type Config struct {
	Host string
	Port int
}

var (
	config *Config
	once   sync.Once
)

// InitConfig 初始化配置（只能调用一次）
func InitConfig(host string, port int) {
	once.Do(func() {
		fmt.Println("初始化配置")
		config = &Config{
			Host: host,
			Port: port,
		}
	})
}

// GetConfig 获取配置
func GetConfig() *Config {
	if config == nil {
		panic("配置未初始化")
	}
	return config
}

func main() {
	InitConfig("localhost", 8080)
	InitConfig("example.com", 9090) // 不会执行

	cfg := GetConfig()
	fmt.Printf("Host: %s, Port: %d\n", cfg.Host, cfg.Port)
	// 输出：Host: localhost, Port: 8080
}
```

### 4. 多个单例管理

```go
package main

import (
	"fmt"
	"sync"
)

// Database 数据库连接
type Database struct {
	name string
}

// Cache 缓存
type Cache struct {
	name string
}

var (
	db       *Database
	cache    *Cache
	dbOnce   sync.Once
	cacheOnce sync.Once
)

// GetDatabase 获取数据库实例
func GetDatabase() *Database {
	dbOnce.Do(func() {
		fmt.Println("初始化数据库连接")
		db = &Database{name: "MySQL"}
	})
	return db
}

// GetCache 获取缓存实例
func GetCache() *Cache {
	cacheOnce.Do(func() {
		fmt.Println("初始化缓存连接")
		cache = &Cache{name: "Redis"}
	})
	return cache
}

func main() {
	db1 := GetDatabase()
	db2 := GetDatabase()
	fmt.Println(db1 == db2) // true

	cache1 := GetCache()
	cache2 := GetCache()
	fmt.Println(cache1 == cache2) // true
}
```

---

## 代码示例

### 示例 1: 资源初始化

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

// Resource 资源
type Resource struct {
	data string
}

var (
	resource *Resource
	once     sync.Once
)

// InitResource 初始化资源
func InitResource() {
	once.Do(func() {
		fmt.Println("开始初始化资源...")
		time.Sleep(2 * time.Second) // 模拟耗时操作
		resource = &Resource{data: "资源数据"}
		fmt.Println("资源初始化完成")
	})
}

// GetResource 获取资源
func GetResource() *Resource {
	InitResource()
	return resource
}

func main() {
	var wg sync.WaitGroup

	for i := 1; i <= 5; i++ {
		wg.Add(1)
		go func(id int) {
			defer wg.Done()
			fmt.Printf("Goroutine %d 获取资源\n", id)
			r := GetResource()
			fmt.Printf("Goroutine %d 得到: %s\n", id, r.data)
		}(i)
	}

	wg.Wait()
}
```

### 示例 2: 配置加载

```go
package main

import (
	"fmt"
	"sync"
)

// AppConfig 应用配置
type AppConfig struct {
	Database struct {
		Host string
		Port int
	}
	Redis struct {
		Host string
		Port int
	}
}

var (
	appConfig *AppConfig
	once      sync.Once
)

// LoadConfig 加载配置
func LoadConfig() *AppConfig {
	once.Do(func() {
		fmt.Println("加载配置文件...")
		appConfig = &AppConfig{}
		appConfig.Database.Host = "localhost"
		appConfig.Database.Port = 3306
		appConfig.Redis.Host = "localhost"
		appConfig.Redis.Port = 6379
		fmt.Println("配置加载完成")
	})
	return appConfig
}

func main() {
	var wg sync.WaitGroup

	for i := 1; i <= 3; i++ {
		wg.Add(1)
		go func(id int) {
			defer wg.Done()
			cfg := LoadConfig()
			fmt.Printf("Goroutine %d: DB=%s:%d\n",
				id, cfg.Database.Host, cfg.Database.Port)
		}(i)
	}

	wg.Wait()
}
```

### 示例 3: 日志初始化

```go
package main

import (
	"fmt"
	"sync"
)

// Logger 日志器
type Logger struct {
	level string
	file  string
}

var (
	logger *Logger
	once   sync.Once
)

// InitLogger 初始化日志器
func InitLogger(level, file string) {
	once.Do(func() {
		fmt.Printf("初始化日志器: level=%s, file=%s\n", level, file)
		logger = &Logger{
			level: level,
			file:  file,
		}
	})
}

// GetLogger 获取日志器
func GetLogger() *Logger {
	if logger == nil {
		panic("日志器未初始化")
	}
	return logger
}

// Log 记录日志
func (l *Logger) Log(msg string) {
	fmt.Printf("[%s] %s\n", l.level, msg)
}

func main() {
	// 初始化日志器
	InitLogger("INFO", "app.log")

	// 尝试再次初始化（不会执行）
	InitLogger("DEBUG", "debug.log")

	// 使用日志器
	logger := GetLogger()
	logger.Log("应用启动")
	logger.Log("处理请求")
}
```

### 示例 4: 连接池初始化

```go
package main

import (
	"fmt"
	"sync"
)

// ConnectionPool 连接池
type ConnectionPool struct {
	connections []string
	maxSize     int
}

var (
	pool *ConnectionPool
	once sync.Once
)

// InitPool 初始化连接池
func InitPool(maxSize int) *ConnectionPool {
	once.Do(func() {
		fmt.Printf("初始化连接池，大小: %d\n", maxSize)
		pool = &ConnectionPool{
			connections: make([]string, 0, maxSize),
			maxSize:     maxSize,
		}
		// 预创建连接
		for i := 0; i < maxSize; i++ {
			pool.connections = append(pool.connections,
				fmt.Sprintf("连接-%d", i+1))
		}
	})
	return pool
}

// GetConnection 获取连接
func (p *ConnectionPool) GetConnection() string {
	if len(p.connections) > 0 {
		conn := p.connections[0]
		p.connections = p.connections[1:]
		return conn
	}
	return ""
}

func main() {
	var wg sync.WaitGroup

	for i := 1; i <= 5; i++ {
		wg.Add(1)
		go func(id int) {
			defer wg.Done()
			pool := InitPool(3)
			conn := pool.GetConnection()
			fmt.Printf("Goroutine %d 获取: %s\n", id, conn)
		}(i)
	}

	wg.Wait()
}
```

---

## 常见问题

### 1. Once 可以重置吗？

```go
// ❌ 不可以：Once 不能重置
var once sync.Once

once.Do(func() {
	fmt.Println("第一次")
})

// 无法重置 once，以下代码不会执行
once.Do(func() {
	fmt.Println("第二次")
})

// ✅ 如果需要重置，使用新的 Once 实例
once = sync.Once{}
once.Do(func() {
	fmt.Println("新的 Once")
})
```

### 2. Do 方法可以返回值吗？

```go
// ❌ 不可以：Do 方法不返回值
var once sync.Once
var result int

once.Do(func() {
	result = 42 // 只能通过外部变量传递
})

// ✅ 正确做法：使用外部变量
var (
	once   sync.Once
	config *Config
	err    error
)

func LoadConfig() (*Config, error) {
	once.Do(func() {
		config, err = loadConfigFromFile()
	})
	return config, err
}
```

### 3. 如果 Do 中的函数 panic 会怎样？

```go
// ⚠️ 注意：如果函数 panic，Once 仍然认为已执行
var once sync.Once

func test() {
	defer func() {
		if r := recover(); r != nil {
			fmt.Println("捕获 panic:", r)
		}
	}()

	once.Do(func() {
		fmt.Println("执行中...")
		panic("出错了")
	})

	// 再次调用不会执行
	once.Do(func() {
		fmt.Println("这不会执行")
	})
}
```

### 4. Once 可以用于多个不同的函数吗？

```go
// ❌ 不推荐：Once 只应该用于一个特定的初始化操作
var once sync.Once

once.Do(func() {
	fmt.Println("初始化 A")
})

once.Do(func() {
	fmt.Println("初始化 B") // 不会执行
})

// ✅ 正确：为不同的操作使用不同的 Once
var (
	onceA sync.Once
	onceB sync.Once
)

onceA.Do(func() {
	fmt.Println("初始化 A")
})

onceB.Do(func() {
	fmt.Println("初始化 B")
})
```

---

## 面试题

### 1. Once 的实现原理是什么？

**答案：**

Once 使用**双重检查锁定**模式实现：

1. **第一次检查**：使用原子操作快速检查 `done` 标记
2. **加锁**：如果未执行，获取互斥锁
3. **第二次检查**：再次检查 `done` 标记（避免重复执行）
4. **执行函数**：执行传入的函数
5. **设置标记**：使用 defer 确保标记被设置

**优势：**
- 第一次检查避免了每次都加锁，提高性能
- 第二次检查确保只有一个 goroutine 执行函数
- 使用 defer 确保即使函数 panic 也会设置标记

### 2. Once 和 Mutex 有什么区别？

**答案：**

| 特性 | Once | Mutex |
|------|------|-------|
| 用途 | 确保操作只执行一次 | 保护临界区 |
| 执行次数 | 只执行一次 | 可以多次加锁/解锁 |
| 可重置 | 不可重置 | 可以重复使用 |
| 性能 | 第一次后无锁检查 | 每次都需要加锁 |
| 使用场景 | 初始化、单例 | 并发访问共享资源 |

### 3. 为什么 Once 使用双重检查？

**答案：**

**原因：**
1. **性能优化**：第一次原子检查避免每次都加锁
2. **正确性保证**：第二次检查确保只执行一次

**如果只有一次检查：**
- 只在锁内检查：每次调用都需要加锁，性能差
- 只在锁外检查：多个 goroutine 可能同时通过检查，导致重复执行

### 4. Once 适用于哪些场景？

**答案：**

**适用场景：**
1. **单例模式**：确保只创建一个实例
2. **资源初始化**：数据库连接、配置加载
3. **一次性设置**：日志初始化、全局变量设置
4. **延迟初始化**：只在需要时才初始化

**不适用场景：**
1. 需要多次执行的操作
2. 需要返回值的操作（需要通过外部变量）
3. 需要重置的操作

### 5. 如何实现可重置的 Once？

**答案：**

```go
// ResetOnce 可重置的 Once
type ResetOnce struct {
	done uint32
	m    sync.Mutex
}

// Do 执行函数
func (o *ResetOnce) Do(f func()) {
	if atomic.LoadUint32(&o.done) == 0 {
		o.doSlow(f)
	}
}

func (o *ResetOnce) doSlow(f func()) {
	o.m.Lock()
	defer o.m.Unlock()
	if o.done == 0 {
		defer atomic.StoreUint32(&o.done, 1)
		f()
	}
}

// Reset 重置状态
func (o *ResetOnce) Reset() {
	atomic.StoreUint32(&o.done, 0)
}
```

---

## 最佳实践

### 1. 为每个初始化操作使用独立的 Once

```go
// ✅ 推荐
var (
	dbOnce    sync.Once
	cacheOnce sync.Once
)

func initDB() {
	dbOnce.Do(func() {
		// 初始化数据库
	})
}

func initCache() {
	cacheOnce.Do(func() {
		// 初始化缓存
	})
}
```

### 2. 使用闭包传递参数

```go
// ✅ 推荐
func InitConfig(filename string) {
	once.Do(func() {
		loadConfig(filename)
	})
}
```

### 3. 处理初始化错误

```go
// ✅ 推荐
var (
	once   sync.Once
	config *Config
	err    error
)

func LoadConfig() (*Config, error) {
	once.Do(func() {
		config, err = loadConfigFromFile()
	})
	return config, err
}
```

### 4. 避免在 Once 中执行耗时操作

```go
// ❌ 不推荐：阻塞其他 goroutine
once.Do(func() {
	time.Sleep(10 * time.Second) // 耗时操作
})

// ✅ 推荐：异步初始化
once.Do(func() {
	go func() {
		// 耗时操作
	}()
})
```

### 5. 确保 Once 不被复制

```go
// ✅ 推荐：使用指针传递
func process(once *sync.Once) {
	once.Do(func() {
		// 初始化
	})
}

// ❌ 错误：值传递会复制 Once
func process(once sync.Once) {
	once.Do(func() {
		// 初始化
	})
}
```

---

## 参考资料

- [x] [Go 官方文档 - sync.Once](https://pkg.go.dev/sync#Once)
- [x] [Go Once 源码分析](https://github.com/golang/go/blob/master/src/sync/once.go)
- [x] [深入理解 Go Once](https://colobu.com/2018/12/18/dive-into-go-once/)
- [x] [Go 单例模式最佳实践](https://marcio.io/2015/07/singleton-pattern-in-go/)

---

**上一节：** [07-WaitGroup](./07-WaitGroup.md)  
**下一节：** [09-原子操作Atomic](./09-原子操作Atomic.md)
