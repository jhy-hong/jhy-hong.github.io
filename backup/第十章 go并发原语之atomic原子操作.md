

## 原子操作的概念

之所以叫原子操作，是因为一个原子在执行的时候，其它线程不会看到执行一半的操作结果。在其它线程看来，原子操作要么执行完了，要么还没有执行，就像一个最小的粒子 - 原子一样，不可分割。

> 在**不加锁**的情况下，对**单个变量**进行**读 / 写 / 修改**，并保证
>  **原子性（Atomicity） + 内存可见性（Happens-Before）**

它**只解决“某一个变量”的并发安全问题**，不解决：

- 多变量一致性
- 复杂业务临界区
- 结构体整体状态

## atomic 能操作哪些数据类型

基础整型：

```go
int32 / int64
uint32 / uint64
uintptr
```

指针：

```go
atomic.Pointer[T]
```

通用类型：

```go
atomic.Value
```

## 基本api及操作示例

### Load/Store

Load 方法会取出 addr 地址中的值，即使在多处理器、多核、有 CPU cache 的情况下，这个操作也能保证 Load 是一个原子操作。

Store 方法会把一个值存入到指定的 addr 地址中，即使在多处理器、多核、有 CPU cache 的情况下，这个操作也能保证 Store 是一个原子操作。别的 goroutine 通过 Load 读取出来，不会看到存取了一半的值。

```go

var enabled int32

func Enable() {
	atomic.StoreInt32(&enabled, 1)
}

func Disable() {
	atomic.StoreInt32(&enabled, 0)
}

func IsEnabled() bool {
	return atomic.LoadInt32(&enabled) == 1
}

```

**适合场景**

- 配置开关
- 状态位
- 是否关闭 / 是否就绪

### Add

Add 方法就是给第一个参数地址中的值增加一个 delta 值。

对于有符号的整数来说，delta 可以是一个负数，相当于减去一个值。对于无符号的整数和 uinptr 类型来说，

可以利用计算机补码的规则，把减法变成加法。

以 uint32 类型为例：

```go
AddUint32(&x, ^uint32(c-1)).
```

尤其是减 1 这种特殊的操作，我们可以简化为：

```go
AddUint32(&x, ^uint32(0))
```

```go
package main

import (
	"fmt"
	"sync"
	"sync/atomic"
)

var end int64

func Sum(num int64, wg *sync.WaitGroup) {
	atomic.AddInt64(&end, num)
	defer wg.Done()

}
func main() {
	wg := &sync.WaitGroup{}
	wg.Add(10000)
	for i := 1; i <= 10000; i++ {

		go Sum(int64(i), wg)
	}
	wg.Wait()
	fmt.Println(end)
}
```

**典型用途**

- QPS 统计
- 请求数
- 成功 / 失败计数

### Swap

不需要比较旧值，只是比较粗暴地替换的话，就可以使用 Swap 方法，它替换后还可以返回旧值

```go
package main

import (
	"fmt"
	"sync/atomic"
)

/*
Swap 适合：

“我要拿到旧值，并且立即替换为新值”

典型包括：

状态切换

一次性取走资源

无锁队列中的指针摘除

抢占式消费
*/

var state int32

func SetRunning() (old int32) {
	return atomic.SwapInt32(&state, 1)
}

func main() {
	state = 8
	old := SetRunning()
	fmt.Println("old value is", old)
	fmt.Println("new value is", atomic.LoadInt32(&state))
}

```

**用途**

- 状态机切换
- 旧值判断 + 更新一步完成

### CAS（Compare-And-Swap，核心能力）

这个方法会比较当前 addr 地址里的值是不是 old，如果不等于 old，就返回 false；如果等于 old，就把此地址的值替换成 new 值，返回 true。这就相当于“判断相等才替换”。

> CAS = **乐观锁的基础**

```go
package main

import (
	"fmt"
	"sync/atomic"
)

var inited int32

func Init() {
	if atomic.CompareAndSwapInt32(&inited, 3, 1) {
		// 只有一个 goroutine 能进来
		doInit()
	}
}

func doInit() {
	fmt.Println("init")
}
func main() {
	atomic.StoreInt32(&inited, 3)
	Init()
	Init()
}

```

**常见场景**

- once 初始化
- 无锁状态机
- 并发注册 / 竞争控制

**CAS 成功条件**

```
当前值 == old → 才会写入 new
```

### atomic.Value

```go
package main

import (
	"fmt"
	"sync/atomic"
)

type ConfigValue struct {
	Timeout int
	MaxConn int
	Debug   bool
}

var cfg atomic.Value // 存 *Config

func InitValue() {
	cfg.Store(&ConfigValue{Timeout: 100, MaxConn: 1000, Debug: false})
}

func Update(timeout, maxConn int, debug bool) {
	//old := cfg.Load().(*ConfigValue)

	// 复制
	newCfg := &ConfigValue{
		Timeout: timeout,
		MaxConn: maxConn,
		Debug:   debug,
	}

	// 原子替换
	cfg.Store(newCfg)
}
func main() {
	InitValue()
	old := cfg.Load().(*ConfigValue)
	fmt.Printf("old: %+v\n", old)
	Update(200, 2000, true)
	newCfg := cfg.Load().(*ConfigValue)
	fmt.Printf("new: %+v\n", newCfg)
}

/*
   严格按 “整体替换（Copy-On-Write）” 的模式来用
	结论：atomic.Value 能一次性发布一个包含多个字段的 struct，也能发布一个包含多个 key/value 的 map，并保证读者看到的是完整一致的快照。
但它不支持局部原子更新（不能只改某个字段或某个 key）。
*/

```

 特点

- **整体替换**
- **读无锁**
- 写入会有原子屏障

⚠ 限制：

- **首次 Store 决定类型**
- 后续 Store 类型必须一致
- 不能存 `nil`

### atomic.Pointer

`atomic.Pointer[T]` 提供：

对“指向 T 的指针”进行原子读/写/替换/条件更新

核心能力：

- `Load()`：原子读指针
- `Store(*T)`：原子写指针
- `Swap(*T)`：原子替换并返回旧指针
- `CompareAndSwap(old, new)`：CAS

⚠️ 它保证的是“指针本身”的原子性与可见性，不保证 `T` 内部字段的线程安全。

```go
type Config struct {
    Timeout int
    MaxConn int
    Debug   bool
}

var cfg atomic.Pointer[Config]

func Init() {
    cfg.Store(&Config{
        Timeout: 100,
        MaxConn: 1000,
        Debug:   false,
    })
}

func Update(timeout, maxConn int, debug bool) {
    // old := cfg.Load()

    newCfg := &Config{
        Timeout: timeout,
        MaxConn: maxConn,
        Debug:   debug,
    }

    cfg.Store(newCfg)
}
```

> 整体替换（Copy-On-Write），绝不原地修改

#### atomic.Pointer vs atomic.Value 的核心区别

类型安全

**atomic.Pointer**

```go
var p atomic.Pointer[Config]
```

- 泛型
- 编译期类型安全
- 不需要类型断言

**atomic.Value**

```go
var v atomic.Value
v.Store(&Config{})
c := v.Load().(*Config)
```

- 运行时类型检查
- 需要类型断言
- 类型不一致会 panic

Pointer 更安全。

#### 推荐场景

| 场景          | 推荐  |
| ------------- | ----- |
| 管理 *Config  | ⭐⭐⭐⭐⭐ |
| 路由表指针    | ⭐⭐⭐⭐⭐ |
| 状态机对象    | ⭐⭐⭐⭐  |
| 需要 nil 状态 | ⭐⭐⭐⭐  |

#### 总结

| 维度                | atomic.Pointer | atomic.Value |
| ------------------- | -------------- | ------------ |
| 类型安全            | 编译期         | 运行期       |
| 需断言              | ❌              | ✔            |
| 支持 nil            | ✔              | ❌            |
| 性能                | 更优           | 略慢         |
| 推荐程度（Go1.19+） | ⭐⭐⭐⭐⭐          | ⭐⭐           |

## atomic 的典型适用场景总结

| 场景            | 是否适合               |
| --------------- | ---------------------- |
| 计数器          | ✔ 非常适合             |
| 状态标志        | ✔                      |
| 配置热更新      | ✔（Value / Pointer）   |
| 多字段一致性    | ❌                      |
| map 并发修改    | ❌                      |
| 复杂业务逻辑    | ❌                      |
| 需要阻塞 / 唤醒 | ❌（用 Cond / channel） |

## atomic 的常见坑（一定要看）

### ❌ 1. 结构体字段分别 atomic ≠ 结构体安全

```
type S struct {
	a int64
	b int64
}
```

❌ 错误认知：

> a、b 都 atomic 就安全

✔ 正确结论： 

> **跨字段逻辑仍然不安全**

------

### ❌ 2. atomic 不是“轻量 mutex”

```
if atomic.LoadInt32(&x) == 0 {
	atomic.StoreInt32(&x, 1)
}
```

❌ **这不是原子操作！**

✔ 应该用：

```
atomic.CompareAndSwapInt32(&x, 0, 1)
```

------

### ❌ 3. 不能对临时变量 atomic

```
atomic.AddInt64(&getCounter(), 1) // 编译错误
```

atomic 操作的对象 **必须是可寻址变量**

------

### ❌ 4. 不要“atomic + mutex 混用控制同一变量”

这是**未定义行为的根源**。

## atomic vs mutex vs channel（定位对比）

| 维度     | atomic | mutex | channel |
| -------- | ------ | ----- | ------- |
| 是否阻塞 | ❌      | ✔     | ✔       |
| 临界区   | 单变量 | 任意  | 消息    |
| 复杂度   | 高     | 中    | 低      |
| 性能     | ⭐⭐⭐⭐⭐  | ⭐⭐⭐   | ⭐⭐      |
| 可读性   | ❌      | ✔     | ⭐⭐⭐⭐    |

**经验法则**

> 能 mutex 就 mutex
>  不能阻塞、极致性能 → atomic
>  业务通信 → channel

## 一句话总结

> **atomic 是“并发世界里的手术刀”**
>  精准、高效、危险
>  只适合解决“一个变量”的并发问题