# 07-Map

[← 返回本章目录](./README.md) | [← 返回总目录](../README.md)

---

## 📋 目录

- [核心概念](#核心概念)
- [Map 声明和初始化](#map-声明和初始化)
- [Map 操作](#map-操作)
- [Map 底层实现](#map-底层实现)
- [Map 并发安全](#map-并发安全)
- [代码示例](#代码示例)
- [常见问题](#常见问题)
- [面试题](#面试题)
- [最佳实践](#最佳实践)
- [参考资料](#参考资料)

---

## 核心概念

Map 是 Go 语言中的哈希表实现，用于存储键值对。

```
特点：
1. 无序集合
2. 引用类型
3. 键必须可比较
4. 非并发安全
5. 动态扩容
```

### Map 的特性

- **键类型限制**：键必须支持 == 和 != 操作
- **值类型无限制**：值可以是任意类型
- **零值是 nil**：nil map 不能直接赋值
- **遍历无序**：每次遍历顺序可能不同

---

## Map 声明和初始化

### 1. 基本声明

```go
package main

import "fmt"

func main() {
    // 方式 1: 声明 nil map
    var m1 map[string]int
    fmt.Println("m1:", m1)        // map[]
    fmt.Println("m1 == nil:", m1 == nil)  // true
    // m1["key"] = 1  // panic: assignment to entry in nil map
    
    // 方式 2: 使用 make
    m2 := make(map[string]int)
    m2["age"] = 25
    fmt.Println("m2:", m2)  // map[age:25]
    
    // 方式 3: 使用 make 指定容量
    m3 := make(map[string]int, 10)
    fmt.Println("m3:", m3)  // map[]
    
    // 方式 4: 使用字面量
    m4 := map[string]int{
        "age":    25,
        "score":  90,
    }
    fmt.Println("m4:", m4)  // map[age:25 score:90]
}
```

### 2. 不同类型的 Map

```go
package main

import "fmt"

func main() {
    // 字符串 -> 整数
    ages := map[string]int{
        "Alice": 25,
        "Bob":   30,
    }
    
    // 整数 -> 字符串
    names := map[int]string{
        1: "Alice",
        2: "Bob",
    }
    
    // 结构体作为键
    type Point struct {
        X, Y int
    }
    points := map[Point]string{
        {0, 0}: "origin",
        {1, 1}: "diagonal",
    }
    
    // 切片作为值
    groups := map[string][]string{
        "admin": {"Alice", "Bob"},
        "user":  {"Charlie", "David"},
    }
    
    fmt.Println(ages, names, points, groups)
}
```

---

## Map 操作

### 1. 添加和修改

```go
package main

import "fmt"

func main() {
    m := make(map[string]int)
    
    // 添加元素
    m["age"] = 25
    m["score"] = 90
    fmt.Println(m)  // map[age:25 score:90]
    
    // 修改元素
    m["age"] = 30
    fmt.Println(m)  // map[age:30 score:90]
}
```

### 2. 获取元素

```go
package main

import "fmt"

func main() {
    m := map[string]int{
        "age":   25,
        "score": 90,
    }
    
    // 方式 1: 直接获取
    age := m["age"]
    fmt.Println("age:", age)  // 25
    
    // 获取不存在的键（返回零值）
    height := m["height"]
    fmt.Println("height:", height)  // 0
    
    // 方式 2: 检查键是否存在
    if age, ok := m["age"]; ok {
        fmt.Println("age exists:", age)
    }
    
    if height, ok := m["height"]; !ok {
        fmt.Println("height not exists")
    }
}
```

### 3. 删除元素

```go
package main

import "fmt"

func main() {
    m := map[string]int{
        "age":   25,
        "score": 90,
    }
    
    fmt.Println("删除前:", m)
    
    // 删除元素
    delete(m, "age")
    fmt.Println("删除后:", m)  // map[score:90]
    
    // 删除不存在的键（不会 panic）
    delete(m, "height")
    fmt.Println("删除不存在的键:", m)
}
```

### 4. 遍历 Map

```go
package main

import "fmt"

func main() {
    m := map[string]int{
        "age":   25,
        "score": 90,
        "level": 5,
    }
    
    // 遍历键值对
    for k, v := range m {
        fmt.Printf("%s: %d\n", k, v)
    }
    
    // 只遍历键
    for k := range m {
        fmt.Println("key:", k)
    }
    
    // 只遍历值
    for _, v := range m {
        fmt.Println("value:", v)
    }
}
```

### 5. Map 长度

```go
package main

import "fmt"

func main() {
    m := map[string]int{
        "age":   25,
        "score": 90,
    }
    
    // 获取长度
    fmt.Println("len:", len(m))  // 2
    
    // 清空 map（重新赋值）
    m = make(map[string]int)
    fmt.Println("len:", len(m))  // 0
}
```

---

## Map 底层实现

### 1. 数据结构

```go
// Map 的底层结构（简化版）
type hmap struct {
    count     int              // 元素个数
    flags     uint8            // 状态标志
    B         uint8            // 桶数量的对数（2^B 个桶）
    noverflow uint16           // 溢出桶数量
    hash0     uint32           // 哈希种子
    buckets   unsafe.Pointer   // 桶数组
    oldbuckets unsafe.Pointer  // 扩容时的旧桶
    nevacuate uintptr          // 扩容进度
}

// 桶结构
type bmap struct {
    tophash [8]uint8          // 哈希值的高 8 位
    // 后面跟着 8 个 key 和 8 个 value
    // 最后是一个指向溢出桶的指针
}
```

### 2. 哈希冲突解决

```go
// Go 使用链地址法解决哈希冲突
// 每个桶可以存储 8 个键值对
// 如果桶满了，会创建溢出桶
```

### 3. 扩容机制

```go
// 扩容条件：
// 1. 负载因子 > 6.5（元素数量 / 桶数量）
// 2. 溢出桶数量过多

// 扩容方式：
// 1. 翻倍扩容：桶数量翻倍
// 2. 等量扩容：桶数量不变，重新整理

// 渐进式扩容：
// 不是一次性完成，而是在访问时逐步迁移
```

### 4. Map 的遍历

```go
package main

import "fmt"

func main() {
    m := map[int]string{
        1: "a",
        2: "b",
        3: "c",
    }
    
    // Map 遍历是无序的
    for i := 0; i < 3; i++ {
        fmt.Print("第", i+1, "次遍历: ")
        for k, v := range m {
            fmt.Printf("%d:%s ", k, v)
        }
        fmt.Println()
    }
    
    // 每次遍历顺序可能不同
    // 这是 Go 故意设计的，防止依赖遍历顺序
}
```

---

## Map 并发安全

### 1. Map 不是并发安全的

```go
package main

import (
    "fmt"
    "sync"
)

func main() {
    m := make(map[int]int)
    
    // 并发写入会 panic
    var wg sync.WaitGroup
    for i := 0; i < 100; i++ {
        wg.Add(1)
        go func(i int) {
            defer wg.Done()
            m[i] = i  // fatal error: concurrent map writes
        }(i)
    }
    wg.Wait()
    
    fmt.Println(m)
}
```

### 2. 使用互斥锁

```go
package main

import (
    "fmt"
    "sync"
)

type SafeMap struct {
    mu sync.RWMutex
    m  map[int]int
}

func (sm *SafeMap) Set(key, value int) {
    sm.mu.Lock()
    defer sm.mu.Unlock()
    sm.m[key] = value
}

func (sm *SafeMap) Get(key int) (int, bool) {
    sm.mu.RLock()
    defer sm.mu.RUnlock()
    value, ok := sm.m[key]
    return value, ok
}

func main() {
    sm := &SafeMap{m: make(map[int]int)}
    
    var wg sync.WaitGroup
    for i := 0; i < 100; i++ {
        wg.Add(1)
        go func(i int) {
            defer wg.Done()
            sm.Set(i, i)
        }(i)
    }
    wg.Wait()
    
    fmt.Println("len:", len(sm.m))
}
```

### 3. 使用 sync.Map

```go
package main

import (
    "fmt"
    "sync"
)

func main() {
    var m sync.Map
    
    // 存储
    m.Store("age", 25)
    m.Store("score", 90)
    
    // 读取
    if value, ok := m.Load("age"); ok {
        fmt.Println("age:", value)
    }
    
    // 删除
    m.Delete("age")
    
    // 读取或存储
    actual, loaded := m.LoadOrStore("age", 30)
    fmt.Println("actual:", actual, "loaded:", loaded)
    
    // 遍历
    m.Range(func(key, value interface{}) bool {
        fmt.Printf("%v: %v\n", key, value)
        return true  // 返回 false 停止遍历
    })
}
```

---

## 代码示例

### 示例 1: 统计单词频率

```go
package main

import (
    "fmt"
    "strings"
)

func wordCount(text string) map[string]int {
    words := strings.Fields(text)
    count := make(map[string]int)
    
    for _, word := range words {
        count[word]++
    }
    
    return count
}

func main() {
    text := "hello world hello go go go"
    count := wordCount(text)
    
    for word, freq := range count {
        fmt.Printf("%s: %d\n", word, freq)
    }
}
```

### 示例 2: 两数之和

```go
package main

import "fmt"

func twoSum(nums []int, target int) []int {
    m := make(map[int]int)
    
    for i, num := range nums {
        if j, ok := m[target-num]; ok {
            return []int{j, i}
        }
        m[num] = i
    }
    
    return nil
}

func main() {
    nums := []int{2, 7, 11, 15}
    target := 9
    result := twoSum(nums, target)
    fmt.Println(result)  // [0 1]
}
```

### 示例 3: 分组

```go
package main

import "fmt"

type Person struct {
    Name string
    Age  int
    City string
}

func groupByCity(people []Person) map[string][]Person {
    groups := make(map[string][]Person)
    
    for _, p := range people {
        groups[p.City] = append(groups[p.City], p)
    }
    
    return groups
}

func main() {
    people := []Person{
        {"Alice", 25, "Beijing"},
        {"Bob", 30, "Shanghai"},
        {"Charlie", 35, "Beijing"},
        {"David", 40, "Shanghai"},
    }
    
    groups := groupByCity(people)
    
    for city, persons := range groups {
        fmt.Printf("%s: %v\n", city, persons)
    }
}
```

### 示例 4: LRU 缓存

```go
package main

import (
    "container/list"
    "fmt"
)

type LRUCache struct {
    capacity int
    cache    map[int]*list.Element
    list     *list.List
}

type entry struct {
    key   int
    value int
}

func NewLRUCache(capacity int) *LRUCache {
    return &LRUCache{
        capacity: capacity,
        cache:    make(map[int]*list.Element),
        list:     list.New(),
    }
}

func (c *LRUCache) Get(key int) int {
    if elem, ok := c.cache[key]; ok {
        c.list.MoveToFront(elem)
        return elem.Value.(*entry).value
    }
    return -1
}

func (c *LRUCache) Put(key, value int) {
    if elem, ok := c.cache[key]; ok {
        c.list.MoveToFront(elem)
        elem.Value.(*entry).value = value
        return
    }
    
    if c.list.Len() >= c.capacity {
        oldest := c.list.Back()
        if oldest != nil {
            c.list.Remove(oldest)
            delete(c.cache, oldest.Value.(*entry).key)
        }
    }
    
    elem := c.list.PushFront(&entry{key, value})
    c.cache[key] = elem
}

func main() {
    cache := NewLRUCache(2)
    
    cache.Put(1, 1)
    cache.Put(2, 2)
    fmt.Println(cache.Get(1))  // 1
    
    cache.Put(3, 3)            // 淘汰 key 2
    fmt.Println(cache.Get(2))  // -1
    
    cache.Put(4, 4)            // 淘汰 key 1
    fmt.Println(cache.Get(1))  // -1
    fmt.Println(cache.Get(3))  // 3
    fmt.Println(cache.Get(4))  // 4
}
```

---

## 常见问题

### 1. Map 的键可以是什么类型？

**答案：** 键必须是可比较的类型

```go
// ✅ 可以作为键
int, float64, string, bool, pointer, channel, interface, array, struct

// ❌ 不能作为键
slice, map, function

// 示例
type Point struct {
    X, Y int
}

// ✅ 结构体可以作为键（所有字段可比较）
m1 := make(map[Point]string)

// ❌ 包含切片的结构体不能作为键
type Data struct {
    Values []int
}
// m2 := make(map[Data]string)  // 编译错误
```

### 2. nil map 和空 map 的区别？

```go
// nil map
var m1 map[string]int
fmt.Println(m1 == nil)  // true
// m1["key"] = 1  // panic

// 空 map
m2 := make(map[string]int)
fmt.Println(m2 == nil)  // false
m2["key"] = 1  // ✅

m3 := map[string]int{}
fmt.Println(m3 == nil)  // false
```

### 3. Map 的遍历为什么是无序的？

**答案：** 

1. **哈希表本质**：Map 底层是哈希表，元素存储位置由哈希函数决定
2. **故意设计**：Go 故意在每次遍历时随机化起始位置，防止程序依赖遍历顺序
3. **扩容影响**：扩容时元素位置会改变

```go
// 如需有序遍历，先排序键
keys := make([]string, 0, len(m))
for k := range m {
    keys = append(keys, k)
}
sort.Strings(keys)

for _, k := range keys {
    fmt.Println(k, m[k])
}
```

### 4. Map 的扩容机制？

**答案：**

```go
// 扩容条件：
// 1. 负载因子 > 6.5
//    负载因子 = 元素数量 / 桶数量
//    例如：13 个元素，2 个桶，负载因子 = 6.5

// 2. 溢出桶过多
//    说明哈希冲突严重，需要重新整理

// 扩容方式：
// 1. 翻倍扩容：桶数量 * 2
// 2. 等量扩容：桶数量不变，重新整理

// 渐进式扩容：
// 不是一次性完成，而是在访问时逐步迁移
// 避免一次性扩容导致的性能抖动
```

---

## 面试题

### 1. Map 是并发安全的吗？

**答案：** 不是

```go
// ❌ 并发读写会 panic
m := make(map[int]int)
go func() {
    m[1] = 1
}()
go func() {
    _ = m[1]
}()

// ✅ 解决方案：
// 1. 使用互斥锁
// 2. 使用 sync.Map
// 3. 使用 channel
```

### 2. 如何实现一个并发安全的 Map？

**答案：**

```go
type SafeMap struct {
    mu sync.RWMutex
    m  map[string]interface{}
}

func (sm *SafeMap) Set(key string, value interface{}) {
    sm.mu.Lock()
    defer sm.mu.Unlock()
    sm.m[key] = value
}

func (sm *SafeMap) Get(key string) (interface{}, bool) {
    sm.mu.RLock()
    defer sm.mu.RUnlock()
    value, ok := sm.m[key]
    return value, ok
}
```

### 3. Map 的底层实现原理？

**答案：**

1. **哈希表**：使用哈希表实现
2. **桶数组**：2^B 个桶，每个桶存储 8 个键值对
3. **链地址法**：哈希冲突时使用溢出桶
4. **渐进式扩容**：负载因子 > 6.5 时扩容
5. **遍历随机化**：每次遍历起始位置随机

### 4. delete 一个不存在的键会怎样？

**答案：** 不会 panic，什么都不做

```go
m := make(map[string]int)
delete(m, "not-exist")  // ✅ 不会 panic
```

### 5. Map 的 value 可以取地址吗？

**答案：** 不可以

```go
m := map[string]int{"age": 25}

// ❌ 不能取地址
// p := &m["age"]  // 编译错误

// 原因：Map 扩容时元素会移动，地址会变化

// 解决方案：使用指针作为 value
m2 := map[string]*int{"age": new(int)}
*m2["age"] = 25
p := m2["age"]  // ✅
```

---

## 最佳实践

### 1. 预分配容量

```go
// ✅ 推荐
m := make(map[string]int, 100)

// ❌ 不推荐
m := make(map[string]int)
```

### 2. 检查键是否存在

```go
// ✅ 推荐
if value, ok := m["key"]; ok {
    // 键存在
}

// ❌ 不推荐
value := m["key"]
if value != 0 {  // 无法区分零值和不存在
    // ...
}
```

### 3. 并发访问使用锁

```go
// ✅ 推荐
type SafeMap struct {
    mu sync.RWMutex
    m  map[string]int
}

// ❌ 不推荐
m := make(map[string]int)
// 并发读写
```

### 4. 有序遍历先排序

```go
// ✅ 推荐
keys := make([]string, 0, len(m))
for k := range m {
    keys = append(keys, k)
}
sort.Strings(keys)
for _, k := range keys {
    fmt.Println(k, m[k])
}
```

### 5. 使用 sync.Map 的场景

```go
// sync.Map 适用于：
// 1. 读多写少
// 2. 多个 goroutine 读写不同的键

// 普通 map + 锁 适用于：
// 1. 读写比例均衡
// 2. 多个 goroutine 读写相同的键
```

---

## 参考资料

- [x] [Go 官方文档 - Maps](https://go.dev/blog/maps)
- [x] [Effective Go - Maps](https://go.dev/doc/effective_go#maps)
- [x] [Go by Example - Maps](https://gobyexample.com/maps)
- [x] [Go Map 源码分析](https://github.com/golang/go/blob/master/src/runtime/map.go)
- [x] [深入理解 Go Map](https://dave.cheney.net/2018/05/29/how-the-go-runtime-implements-maps-efficiently-without-generics)

---

**下一节：** [08-结构体](./08-结构体.md)
