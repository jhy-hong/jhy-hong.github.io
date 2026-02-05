## sync.Pool的概念

`sync.Pool`是Go语言提供的对象池机制，用于存储和复用临时对象，可以显著减少内存分配次数，降低垃圾回收压力

> `sync.Pool` 是一个面向“短生命周期临时对象”的、由 GC 参与管理的对象复用机制，用来显著降低分配次数与 GC 扫描成本。

- 临时对象
- 非可靠存储
- GC 可能清空
- 目标是 **减少 GC，而不是减少内存**

## 主要方法

- **Get()**：如果调用这个方法，就会从 Pool取走一个元素，这也就意味着，这个元素会从 Pool 中移除，返回给调用者。不过，除了返回值是正常实例化的元素，Get 方法的返回值还可能会是一个 nil（Pool.New 字段没有设置，又没有空闲元素可以返回），所以你在使用的时候，可能需要判断。

  > 1. 先从当前 P 的 private 取
  > 2. 再从 shared 取
  > 3. 再偷其他 P 的 shared
  > 4. 再从 victim pool 取
  > 5. 全部没有 → 调用 New()

- **Put()**：这个方法用于将一个元素返还给 Pool，Pool 会把这个元素保存到池中，并且可以复用。但如果 Put 一个 nil 值，Pool 就会忽略这个值。

  > 1. 放入当前 P 的 local.private
  >
  > 2. 若已占用 → 放入 local.shared（无锁队列）

- **New**：个字段的类型是函数 func() interface{}。当调用 Pool 的 Get 方法从池中获取元素，没有更多的空闲元素可返回时，就会调用这个 New 方法来创建新的元素。如果你没有设置 New 字段，没有更多的空闲元素可返回时，Get 方法将返回 nil，表明当前没有可用的元素。

## 基本使用示例

```go
var bufPool = sync.Pool{
    New: func() interface{} {   // 定义如何创建新对象的函数，当池中没有可用对象时被调用
        return new(bytes.Buffer)
    },
}

func getBuffer() *bytes.Buffer {
    return bufPool.Get().(*bytes.Buffer)
}

func putBuffer(buf *bytes.Buffer) {
    buf.Reset() // 重置缓冲区，避免脏数据
    bufPool.Put(buf)
}
```

```go
package main

import (
	"fmt"
	"sync"
)

type Student struct {
	Name  string
	Age   int
	Right bool
}

func (s *Student) Clear() {    // reset
	s.Name = ""
	s.Age = 0
	s.Right = false
}

var studentPool = sync.Pool{
	New: func() interface{} {
		return &Student{}
	},
}

func main() {
	student := studentPool.Get().(*Student)

	// 使用student
	student.Name = "张三"
	student.Age = 20
	student.Right = true

	fmt.Printf("学生信息: %+v\n", student)

	// 返回给studentPool之前必须清空
	student.Clear()
	studentPool.Put(student)
}

```



使用sync.Pool必须遵从

**Get 后必须 reset**

**Put 前对象必须可复用**

**不能假设下一次还能 Get 到**

## 底层实现

pool的strcut

```go
type Pool struct {
	noCopy noCopy

	local     unsafe.Pointer // local fixed-size per-P pool, actual type is [P]poolLocal
	localSize uintptr        // size of the local array

	victim     unsafe.Pointer // local from previous cycle
	victimSize uintptr        // size of victims array

	// New optionally specifies a function to generate
	// a value when Get would otherwise return nil.
	// It may not be changed concurrently with calls to Get.
	New func() any
}

```

Pool中最重要的两个字段victim和local，因为它们两个主要用来存储空闲的元素。

```go
func poolCleanup() {
	// This function is called with the world stopped, at the beginning of a garbage collection.
	// It must not allocate and probably should not call any runtime functions.

	// Because the world is stopped, no pool user can be in a
	// pinned section (in effect, this has all Ps pinned).

	// Drop victim caches from all pools.
	for _, p := range oldPools {
		p.victim = nil
		p.victimSize = 0
	}

	// Move primary cache to victim cache.
	for _, p := range allPools {
		p.victim = p.local
		p.victimSize = p.localSize
		p.local = nil
		p.localSize = 0
	}

	// The pools with non-empty primary caches now have non-empty
	// victim caches and no pools have primary caches.
	oldPools, allPools = allPools, nil
}
```



每次垃圾回收的时候，Pool 会把 victim 中的对象移除，然后把 local 的数据给 victim，这样的话，local 就会被清空，而 victim 就像一个垃圾分拣站，里面的东西可能会被当做垃圾丢弃了，但是里面有用的东西也可能被捡回来重新使用。

victim 中的元素如果被 Get 取走，那么这个元素就很幸运，因为它又“活”过来了。但是，如果这个时候 Get 的并发不是很大，元素没有被 Get 取走，那么就会被移除掉，因为没有别人引用它的话，就会被垃圾回收掉。

local 字段包含一个 poolLocalInternal 字段，并提供 CPU 缓存对齐，从而避免 false sharing。

```go
type poolLocalInternal struct {
	private any       // Can be used only by the respective P.
	shared  poolChain // Local P can pushHead/popHead; any P can popTail.
}
```

- private，代表一个缓存的元素，而且只能由相应的一个 P 存取。因为一个 P 同时只能执行一个 goroutine，所以不会有并发的问题。
- shared，可以由任意的 P 访问，但是只有本地的 P 才能 pushHead/popHead，其它 P 可以 **popTail**，相当于只有一个本地的 P 作为生产者（Producer），多个 P 作为消费者（Consumer），它是使用一个 local-free 的 queue 列表实现的。

`sync.Pool` 的价值在于“减少新对象进入 heap 的次数”，而不是保证对象不被回收。

## 适合使用sync.Pool的场景

| 场景                 | 原因       |
| -------------------- | ---------- |
| `[]byte` buffer      | 大、频繁   |
| `bytes.Buffer`       | 编解码     |
| HTTP/RPC 中间对象    | 生命周期短 |
| 日志拼接             | 高频       |
| protobuf / json 编码 | alloc 热点 |

不适合的场景

| 场景                  | 原因       |
| --------------------- | ---------- |
| DB 连接               | 资源不可丢 |
| 文件句柄              | 系统资源   |
| 长生命周期对象        | pool 会清  |
| 带状态对象            | 脏数据风险 |
| 跨 goroutine 共享状态 | 不安全     |

## 使用限制

1. 不保证命中
   - 不能用于逻辑依赖
2. 不保证顺序
   - 不要期望 FIFO / LIFO
3. 对象必须“干净”
   - Put 前必须 Reset
4. 不适合大对象滥用
5. 不适合低频场景

> “高频、短命、无状态、可丢失”
>
> 满足这 4 个条件，才值得用 `sync.Pool`。

对象池完全可以不使用New函数！

## 方式一：无New函数的对象池（Go标准库风格）

```go
// 不使用New函数的对象池
var (
    writerPool sync.Pool  // 空的池，没有New函数
)

func GetBufferedWriter(w io.Writer) *bufio.Writer {
    var bw *bufio.Writer
    
    // 从池中获取，可能返回nil
    if v := writerPool.Get(); v != nil {
        bw = v.(*bufio.Writer)
        bw.Reset(w)  // 重置已存在的对象
    } else {
        // 池为空时手动创建新对象
        bw = bufio.NewWriter(w)
    }
    
    return bw
}

func PutBufferedWriter(bw *bufio.Writer) {
    if bw != nil {
        bw.Reset(nil)  // 清理状态
        writerPool.Put(bw)  // 放回池中
    }
}

// 使用示例
func handleRequest(w io.Writer) {
    writer := GetBufferedWriter(w)
    defer PutBufferedWriter(writer)
    
    writer.WriteString("Hello World")
    writer.Flush()
}
```

和标准库的这个差不多

```go

func getCopyBuf() []byte {
	return copyBufPool.Get().(*[copyBufPoolSize]byte)[:]
}
func putCopyBuf(b []byte) {
	if len(b) != copyBufPoolSize {
		panic("trying to put back buffer of the wrong size in the copyBufPool")
	}
	copyBufPool.Put((*[copyBufPoolSize]byte)(b))
}

func bufioWriterPool(size int) *sync.Pool {
	switch size {
	case 2 << 10:
		return &bufioWriter2kPool
	case 4 << 10:
		return &bufioWriter4kPool
	}
	return nil
}



....
func newBufioWriterSize(w io.Writer, size int) *bufio.Writer {
	pool := bufioWriterPool(size)
	if pool != nil {
		if v := pool.Get(); v != nil {
			bw := v.(*bufio.Writer)
			bw.Reset(w)
			return bw
		}
	}
	return bufio.NewWriterSize(w, size)
}

```

比较类似。

## 方式二：预填充池对象

```go
// 预先创建对象的池
var preloadedPool = sync.Pool{
    // 不使用New，而是手动预填充
}

func init() {
    // 启动时预创建一些对象
    for i := 0; i < 10; i++ {
        preloadedPool.Put(bufio.NewWriter(nil))
    }
}

func GetFromPreloaded(w io.Writer) *bufio.Writer {
    v := preloadedPool.Get()
    if v == nil {
        // 如果预加载的对象用完了，临时创建新的
        return bufio.NewWriter(w)
    }
    
    bw := v.(*bufio.Writer)
    bw.Reset(w)
    return bw
}
```


## 方式三：条件性池化

```go
// 根据使用频率决定是否池化
type ConditionalPool struct {
    pool    sync.Pool
    counter int64
}

func (cp *ConditionalPool) Get(w io.Writer) *bufio.Writer {
    // 统计使用频率
    count := atomic.AddInt64(&cp.counter, 1)
    
    // 高频使用才启用池化
    if count > 100 {
        if v := cp.pool.Get(); v != nil {
            bw := v.(*bufio.Writer)
            bw.Reset(w)
            return bw
        }
    }
    
    // 低频直接创建
    return bufio.NewWriter(w)
}

func (cp *ConditionalPool) Put(bw *bufio.Writer) {
    if atomic.LoadInt64(&cp.counter) > 100 {
        bw.Reset(nil)
        cp.pool.Put(bw)
    }
    // 低频使用直接丢弃，让GC处理
}
```


## 方式四：混合策略池

```go
// 结合多种策略的对象池
type HybridPool struct {
    smallPool sync.Pool  // 小对象池
    largePool sync.Pool  // 大对象池
    useCount  int64      // 使用统计
}

func (hp *HybridPool) Get(size int, w io.Writer) *bufio.Writer {
    atomic.AddInt64(&hp.useCount, 1)
    
    // 根据对象大小选择不同的池
    var pool *sync.Pool
    if size <= 1024 {
        pool = &hp.smallPool
    } else {
        pool = &hp.largePool
    }
    
    if v := pool.Get(); v != nil {
        bw := v.(*bufio.Writer)
        bw.Reset(w)
        return bw
    }
    
    // 池为空时创建
    return bufio.NewWriterSize(w, size)
}

func (hp *HybridPool) Put(bw *bufio.Writer) {
    size := bw.Available()
    var pool *sync.Pool
    
    if size <= 1024 {
        pool = &hp.smallPool
    } else {
        pool = &hp.largePool
    }
    
    bw.Reset(nil)
    pool.Put(bw)
}
```


## 核心要点总结：

1. **New函数是可选的** - `sync.Pool`完全可以不设置New函数
2. **手动处理nil情况** - 需要自己检查Get()返回值是否为nil
3. **灵活的创建策略** - 可以根据业务需求决定何时创建新对象
4. **更好的控制权** - 不使用New可以获得更精细的资源控制

这种方式特别适合：
- 对象创建成本不高
- 需要精确控制对象生命周期
- 想要避免不必要的对象预创建
- 需要根据运行时条件决定是否池化

Go标准库HTTP服务器就是采用了这种无New函数的设计，体现了实用主义的设计哲学。