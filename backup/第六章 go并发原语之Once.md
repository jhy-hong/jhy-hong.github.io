`sync.Once` 是 Go 标准库 `sync` 包中的一个非常实用的并发原语，用于确保某个操作只会被执行一次，无论有多少个 goroutine 同时调用。

## Once 的使用场景

sync.Once 只暴露了一个方法 Do，你可以多次调用 Do 方法，但是只有第一次调用 Do 方法时 f 参数才会执行，这里的 f 是一个无参数无返回值的函数。

```go
func (o *Once) Do(f func())
```

因为当且仅当第一次调用 Do 方法的时候参数 f 才会执行，即使第二次、第三次、第 n 次调用时 f 参数的值不一样，也不会被执行，比如下面的例子，虽然 f1 和 f2 是不同的函数，但是第二个函数 f2 就不会执行。

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	var once sync.Once

	// 第一个初始化函数
	f1 := func() {
		fmt.Println("in f1")
	}
	once.Do(f1) // 打印出 in f1

	// 第二个初始化函数
	f2 := func() {
		fmt.Println("in f2")
	}
	once.Do(f2) // 无输出
}

```

因为这里的 f 参数是一个无参数无返回的函数，所以你可能会通过闭包的方式引用外面的参数，比如：

```go
    var addr = "baidu.com"

    var conn net.Conn
    var err error

    once.Do(func() {
        conn, err = net.Dial("tcp", addr)
    })
```

比如以下的示例：

```go
package main

import (
    "fmt"
    "sync"
)

var once sync.Once
var config *Config

type Config struct {
    Name string
}

func loadConfig() {
    fmt.Println("Loading config...")
    config = &Config{Name: "AppConfig"}
}

func GetConfig() *Config {
    once.Do(loadConfig)
    return config
}

func main() {
    var wg sync.WaitGroup
    for i := 0; i < 5; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            cfg := GetConfig()
            fmt.Printf("Goroutine %d got config: %+v\n", id, cfg)
        }(i)
    }
    wg.Wait()
}

```

介绍一下很值得我们学习的 math/big/sqrt.go 中实现的一个数据结构，它通过 Once 封装了一个只初始化一次的值：

```go
   // 值是3.0或者0.0的一个数据结构
   var threeOnce struct {
    sync.Once
    v *Float
  }
  
    // 返回此数据结构的值，如果还没有初始化为3.0，则初始化
  func three() *Float {
    threeOnce.Do(func() { // 使用Once初始化
      threeOnce.v = NewFloat(3.0)
    })
    return threeOnce.v
  }
```

将 sync.Once 和 *Float 封装成一个对象，提供了只初始化一次的值 v。 你看它的 three 方法的实现，虽然每次都调用 threeOnce.Do 方法，但是参数只会被调用一次。

当你使用 Once 的时候，你也可以尝试采用这种结构，将值和 Once 封装成一个新的数据结构，提供只初始化一次的值。

总结一下 Once 并发原语解决的问题和使用场景：Once 常常用来初始化单例资源，或者并发访问只需初始化一次的共享资源，或者在测试的时候初始化一次测试资源。

## Once的实现

一个正确的 Once 实现要使用一个互斥锁，这样初始化的时候如果有并发的 goroutine，就会进入doSlow 方法。互斥锁的机制保证只有一个 goroutine 进行初始化，同时利用双检查的机制（double-checking），再次判断 o.done 是否为 0，如果为 0，则是第一次执行，执行完毕后，就将 o.done 设置为 1，然后释放锁。

即使此时有多个 goroutine 同时进入了 doSlow 方法，因为双检查的机制，后续的 goroutine 会看到 o.done 的值为 1，也不会再次执行 f。

源码如下：(go1.24.2)

```go
type Once struct {
	_ noCopy // 防止结构体被复制（嵌入 noCopy 会在编译器做静态检查）

	// done 表示 f 是否已经执行过。
	// 它放在结构体的最前面是为了优化性能：
	// - 对于 x86 架构（amd64/386），前置字段能生成更紧凑的汇编指令
	// - 在其他架构上，可以减少偏移地址计算指令
	done atomic.Uint32

	// 互斥锁，用于控制慢路径中的并发
	m Mutex
}


// Do calls the function f if and only if Do is being called for the
// first time for this instance of [Once]. In other words, given
//
//	var once Once
//
// if once.Do(f) is called multiple times, only the first call will invoke f,
// even if f has a different value in each invocation. A new instance of
// Once is required for each function to execute.
//
// Do is intended for initialization that must be run exactly once. Since f
// is niladic, it may be necessary to use a function literal to capture the
// arguments to a function to be invoked by Do:
//
//	config.once.Do(func() { config.init(filename) })
//
// Because no call to Do returns until the one call to f returns, if f causes
// Do to be called, it will deadlock.
//
// If f panics, Do considers it to have returned; future calls of Do return
// without calling f.
func (o *Once) Do(f func()) {
	// 快路径：先使用原子变量判断是否执行过
	if o.done.Load() == 0 {
		// 如果还未执行，则进入慢路径处理
		o.doSlow(f)
	}
}

func (o *Once) doSlow(f func()) {
	o.m.Lock()              // 加锁防止多个 goroutine 并发进入执行 f
	defer o.m.Unlock()      // 退出时释放锁

	if o.done.Load() == 0 { // 再次检查，防止其他 goroutine 抢先执行了 f
		defer o.done.Store(1) // 延迟设置 done=1，确保 f 完全执行完成后标记已执行
		f()                   // 执行用户提供的初始化函数
	}
}
```

## 常见错误

| ⚠️ 错误用法 | 说明 |
| ---------- | ---- |
| `Do(nil)` | 会 panic，参数不能为空函数 |
| `Do()` 中递归调用 `Do()` | 会死锁，不能在 `f()` 中再次调用 `Do()` |
| 想复用 `Once` 实现“多次执行” | `Once` 只设计为执行一次，不支持 reset |
| 假设 panic 会标记为 `done=1` | 实际上 **不会**，后续仍会再执行一次 |

如

```go
func main() {
    var once sync.Once
    once.Do(func() {
        once.Do(func() {
            fmt.Println("初始化")
        })
    })
}
```

### **执行失败导致未初始化，也可能导致相关后续错误**：

如：

```go
func main() {
    var once sync.Once
    var googleConn net.Conn // 到Google网站的一个连接

    once.Do(func() {
        // 建立到google.com的连接，有可能因为网络的原因，googleConn并没有建立成功，此时它的值为nil
        googleConn, _ = net.Dial("tcp", "google.com:80")
    })
    // 发送http请求
    googleConn.Write([]byte("GET / HTTP/1.1\r\nHost: google.com\r\n Accept: */*\r\n\r\n"))
    io.Copy(os.Stdout, googleConn)
}
```

由于一些防火墙的原因，`googleConn` 并没有被正确的初始化，后面如果想当然认为既然执行了 Do 方法 googleConn 就已经初始化的话，会抛出空指针的错误

我们可以自己实现一个类似 Once 的并发原语，既可以返回当前调用 Do 方法是否正确完成，还可以在初始化失败后调用 Do 方法再次尝试初始化，直到初始化成功才不再初始化了。

```go
// 一个功能更加强大的Once
type Once struct {
    m    sync.Mutex
    done uint32
}
// 传入的函数f有返回值error，如果初始化失败，需要返回失败的error
// Do方法会把这个error返回给调用者
func (o *Once) Do(f func() error) error {
    if atomic.LoadUint32(&o.done) == 1 { //fast path
        return nil
    }
    return o.slowDo(f)
}
// 如果还没有初始化
func (o *Once) slowDo(f func() error) error {
    o.m.Lock()
    defer o.m.Unlock()
    var err error
    if o.done == 0 { // 双检查，还没有初始化
        err = f()
        if err == nil { // 初始化成功才将标记置为已初始化
            atomic.StoreUint32(&o.done, 1)
        }
    }
    return err
}
```

所做的改变就是 Do 方法和参数 f 函数都会返回 error，如果 f 执行失败，会把这个错误信息返回。

对 slowDo 方法也做了调整，如果 f 调用失败，我们不会更改 done 字段的值，这样后续的 goroutine 还会继续调用 f。如果 f 执行成功，才会修改 done 的值为 1。

### 如何判断是否初始化过了

我可以自己去扩展实现

```go
// Once 是一个扩展的sync.Once类型，提供了一个Done方法
type Once struct {
    sync.Once
}

// Done 返回此Once是否执行过
// 如果执行过则返回true
// 如果没有执行过或者正在执行，返回false
func (o *Once) Done() bool {
    return atomic.LoadUint32((*uint32)(unsafe.Pointer(&o.Once))) == 1
}

func main() {
    var flag Once
    fmt.Println(flag.Done()) //false

    flag.Do(func() {
        time.Sleep(time.Second)
    })

    fmt.Println(flag.Done()) //true
}
```