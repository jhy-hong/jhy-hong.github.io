## Channel 的应用场景

go 语言中流传很广的谚语：

> Don’t communicate by sharing memory, share memory by communicating.
> 
> Go Proverbs by Rob Pike

“执行业务处理的 goroutine 不要通过共享内存的方式通信，而是要通过 Channel 通信的方式分享数据。”

“communicate by sharing memory”和“share memory by communicating”是两种不同的并发处理模式。“communicate by sharing memory”是传统的并发编程处理方式，就是指，共享的数据需要用锁进行保护，goroutine 需要获取到锁，才能并发访问数据。

“share memory by communicating”则是类似于 CSP 模型的方式，通过通信的方式，一个 goroutine 可以把数据的“所有权”交给另外一个 goroutine（虽然 Go 中没有“所有权”的概念，但是从逻辑上说，你可以把它理解为是所有权的转移）。

掌握这几种类型。

1. `数据交流`：当作并发的 buffer 或者 queue，解决生产者 - 消费者问题。多个 goroutine 可以并发当作生产者（Producer）和消费者（Consumer）。

2. `数据传递`：一个 goroutine 将数据交给另一个 goroutine，相当于把数据的拥有权 (引用) 托付出去。

3. `信号通知`：一个 goroutine 可以将信号 (closing、closed、data ready 等) 传递给另一个或者另一组 goroutine 。

4. `任务编排`：可以让一组 goroutine 按照一定的顺序并发或者串行的执行，这就是编排的功能。

5. `锁`：利用 Channel 也可以实现互斥锁的机制。

## Channel 基本用法

可以往 Channel 中发送数据，也可以从 Channel 中接收数据，所以，Channel 类型分为**只能接收**、**只能发送**、**既可以接收又可以发送**三种类型。下面是它的语法定义：

```go
ChannelType = ( "chan" | "chan" "<-" | "<-" "chan" ) ElementType .
```

相应地，Channel 的正确语法如下：

```go
chan string          // 可以发送接收string
chan<- struct{}      // 只能发送struct{}
<-chan int           // 只能从chan接收int
```

把既能接收又能发送的 chan 叫做双向的 chan，把只能发送和只能接收的 chan 叫做单向的 chan。其中，“<-”表示单向的 chan，如果你记不住，我告诉你一个简便的方法：**这个箭头总是射向左边的，元素类型总在最右边。如果箭头指向 chan，就表示可以往 chan 中塞数据；如果箭头远离 chan，就表示 chan 会往外吐数据。**

chan 中的元素是任意的类型，所以也可能是 chan 类型，比如下面的 chan 类型也是合法的：

```go
chan<- chan int   
chan<- <-chan int  
<-chan <-chan int
chan (<-chan int)
```

可是，怎么判定箭头符号属于哪个 chan 呢？其实，“<-”有个规则，**总是尽量和左边的 chan 结合**（The <- operator associates with the leftmost chan possible:），因此，上面的定义和下面的使用括号的划分是一样的：

```go
chan<- （chan int） // <- 和第一个chan结合
chan<- （<-chan int） // 第一个<-和最左边的chan结合，第二个<-和左边第二个chan结合
<-chan （<-chan int） // 第一个<-和最左边的chan结合，第二个<-和左边第二个chan结合 
chan (<-chan int) // 因为括号的原因，<-和括号内第一个chan结合
```

通过 make，我们可以初始化一个 chan，**未初始化的 chan 的零值是 nil**。你可以设置它的**容量**，比如下面的 chan 的容量是 9527，我们把这样的 chan 叫做 **buffered chan**；如果没有设置，它的容量是 0，我们把这样的 chan 叫做 **unbuffered chan**。

```go
make(chan int, 9527)
```

如果 chan 中还有数据，那么，从这个 chan 接收数据的时候就不会阻塞，如果 chan 还未满（**“满”指达到其容量**），给它发送数据也不会阻塞，否则就会阻塞。**unbuffered chan** 只有**读写都准备好之后才不会阻塞**，这也是很多使用 unbuffered chan 时的常见 Bug。

### 发送数据

往 chan 中发送一个数据使用“ch<-”，发送数据是一条语句:

```go
ch <- 2000
```

这里的 ch 是 chan int 类型或者是 chan <-int

### 接收数据

从 chan 中接收一条数据使用“<-ch”，接收数据也是一条语句：

```go
  x := <-ch // 把接收的一条数据赋值给变量x
  foo(<-ch) // 把接收的一个的数据作为参数传给函数
  <-ch // 丢弃接收的一条数据
```

这里的 ch 类型是 chan T 或者 <-chan T。

接收数据时，还可以返回两个值。**第一个值是返回的 chan 中的元素**，**第二个值是 bool 类型**，**代表是否成功地从 chan 中读取到一个值**，如果第二个参数是 false，chan 已经被 close 而且 chan 中没有缓存的数据，这个时候，第一个值是零值。所以，如果从 chan 读取到一个零值，可能是 sender 真正发送的零值，也可能是 closed 的并且没有缓存元素产生的零值。

### 其它操作

Go 内建的函数 **close、cap、len** 都可以操作 chan 类型：**close** 会把 chan 关闭掉，**cap** 返回 chan 的容量，**len** 返回 chan 中缓存的还未被取走的元素数量。

send 和 recv 都可以作为 select 语句的 case clause，如下面的例子：

```go
func main() {
    var ch = make(chan int, 10)
    for i := 0; i < 10; i++ {
        select {
        case ch <- i:
        case v := <-ch:
            fmt.Println(v)
        }
    }
}
```

chan 还可以应用于 for-range 语句中，比如：

```go
 for v := range ch {
        fmt.Println(v)
    }
```

或者是忽略读取的值，只是清空 chan：

```go
for range ch {
    }
```

## 什么时候使用channel？

> 1. 共享资源的并发访问使用传统并发原语；
> 2. 复杂的任务编排和消息传递使用 Channel；
> 3. 消息通知机制使用 Channel，除非只想 signal 一个 goroutine，才使用 Cond；
> 4. 简单等待所有任务的完成用 WaitGroup，也有 Channel 的推崇者用 Channel，都可以；
> 5. 需要和 Select 语句结合，使用 Channel；
> 6. 需要和超时配合时，使用 Channel 和 Context。

## 使用channel出现的一些常见错误

chan 的值和状态有多种情况，而不同的操作（send、recv、close）又可能得到不同的结果

还有一个值得注意的点是，**只要一个 chan 还有未读的数据，即使把它 close 掉，你还是可以继续把这些未读的数据消费完，之后才是读取零值数据。**

![Image](https://github.com/user-attachments/assets/a9c26377-2179-48a0-905c-d17c1ef8d8a2)

下表列出了一些与通道（channel）操作相关的常见错误以及对应的错误类型和错误情况详情：

| 错误情况                                           | 错误类型                            | 说明                                                                                      |
| ---------------------------------------------- | ------------------------------- | --------------------------------------------------------------------------------------- |
| 向已关闭的通道发送数据                                    | `panic: send on closed channel` | 当通道已经被关闭时，再向通道发送数据会导致运行时错误。通道关闭后，不能再向其发送数据，否则会引发 panic。                                 |
| 从已关闭的通道接收数据                                    | `value, ok := <-closedChannel`  | 当通道已经被关闭时，再从通道接收数据会返回类型的零值，并且 `ok` 值为 `false`。尝试从已关闭的通道接收数据会导致接收到的数据为零值。                |
| 向未初始化的通道发送数据                                   | `nil` channel                   | 当通道未初始化（为 `nil`）时，向其发送数据会导致运行时错误。在使用通道之前，应该通过 `make` 函数初始化通道。                           |
| 从未初始化的通道接收数据                                   | `nil` channel                   | 当通道未初始化（为 `nil`）时，尝试从其接收数据会导致运行时错误。在使用通道之前，应该通过 `make` 函数初始化通道。                         |
| 在没有接收方的情况下关闭通道                                 | `panic: close of nil channel`   | 当通道未初始化（为 `nil`）时，尝试关闭通道会导致运行时错误。在关闭通道之前，应该通过 `make` 函数初始化通道。                           |
| 在有缓冲通道已满时向通道发送数据                               | 阻塞等待                            | 当有缓冲通道已满时，继续向通道发送数据会导致发送操作阻塞，直到有缓冲区有足够的空间可用。如果没有其他 goroutine 接收数据，发送操作将一直阻塞。            |
| 在无缓冲通道中没有接收方的情况下向通道发送数据                        | 阻塞等待                            | 在无缓冲通道中，发送操作和接收操作必须同时准备好，否则它们会相互阻塞。当没有其他 goroutine 准备接收数据时，发送操作将一直阻塞，直到有接收方准备好。         |
| 在没有发送方的情况下从通道接收数据                              | 阻塞等待                            | 在没有发送方的情况下，从通道接收数据会导致接收操作阻塞，直到有发送方向通道发送数据。如果没有其他 goroutine 发送数据，接收操作将一直阻塞。              |
| 使用 `range` 迭代一个未关闭的通道，并且没有其他 goroutine 向通道发送数据 | 阻塞等待                            | 当使用 `range` 迭代一个未关闭的通道时，如果没有其他 goroutine 向通道发送数据，迭代操作会一直阻塞，直到有数据可接收。如果通道不被关闭，迭代操作将永远阻塞。 |

## reflect.SelectCase 包处理多项反射

`reflect.SelectCase` 是 Go 语言中反射包（reflect package）中的一个类型，用于在 `select` 语句中进行通道选择操作。它提供了一种将通道和对应的操作（发送或接收）进行绑定的方式。

下面是对 `reflect.SelectCase` 的用法进行详细解释，并附带一个代码示例：

`reflect.SelectCase` 类型的定义如下：

```go
type SelectCase struct {
    Dir  SelectDir
    Chan Value
    Send Value
    Recv Value
}
```

其中，`SelectDir` 是一个枚举类型，表示通道操作的方向。它可以是以下两个值之一：

- `SelectSend`：表示通道发送操作。
- `SelectRecv`：表示通道接收操作。

`Value` 是一个 `reflect.Value` 类型的值，用于存储通道或通道操作的信息。

`reflect.SelectCase` 结构体的字段如下：

- `Dir`：表示通道操作的方向，是一个 `SelectDir` 类型的值。
- `Chan`：存储通道的 `reflect.Value` 类型的值。
- `Send`：存储发送操作的 `reflect.Value` 类型的值。
- `Recv`：存储接收操作的 `reflect.Value` 类型的值。

`reflect.SelectCase` 类型的对象可以用于构建 `select` 语句的 case。通过设置 `Dir` 字段来指定操作方向，然后将通道或操作相关的 `reflect.Value` 分配给相应的字段。

以下是一个使用 `reflect.SelectCase` 的简单示例：

```go
package main

import (
    "fmt"
    "reflect"
    "time"
)

func main() {
    ch1 := make(chan int)
    ch2 := make(chan string)

    go func() {
        time.Sleep(2 * time.Second)
        ch1 <- 42
    }()

    go func() {
        time.Sleep(3 * time.Second)
        ch2 <- "Hello, world!"
    }()

    cases := []reflect.SelectCase{
        {Dir: reflect.SelectRecv, Chan: reflect.ValueOf(ch1)},
        {Dir: reflect.SelectRecv, Chan: reflect.ValueOf(ch2)},
    }

    for i := 0; i < 2; i++ {
        chosen, value, _ := reflect.Select(cases)
        switch chosen {
        case 0:
            fmt.Println("Received from ch1:", value.Interface())
        case 1:
            fmt.Println("Received from ch2:", value.Interface())
        }
    }
}
```

在上述示例中，我们创建了两个通道 `ch1` 和 `ch2`。然后，我们分别在不同的 goroutine 中向这两个通道发送数据。接下来，我们使用 `reflect.SelectCase` 创建了一个包含两个 case 的切片 `cases`，每个 case 对应一个通道的接收操作。最后，我们使用 `reflect.Select` 在 `select` 语句中进行通道选择操作，根据选择的结果打印接收到的数据。

运行上述代码，你会看到输出类似以下内容（具体的输出顺序可能有所不同）：

```
Received from ch1: 42
Received from ch2: Hello, world!
```

该示例演示了如何使用 `reflect.SelectCase` 构建 `select` 语句的 case，以便在运行时根据不同的通道接收操作选择不同的 case 进行处理。这种方法可以在动态场景下灵活地进行通道选择操作。

## Channel的一些应用场景

### 信号通知

chan 类型有这样一个特点：chan 如果为空，那么，receiver 接收数据的时候就会阻塞等待，直到 chan 被关闭或者有新的数据到来。利用这个机制，可以实现 wait/notify 的设计模式。

代码示例如下：

```go
package main

import (
    "fmt"
    "os"
    "os/signal"
    "syscall"
    "time"
)

func main() {
    var closing = make(chan struct{})
    var closed = make(chan struct{})

    go func() {
        // 模拟业务处理
        for {
            select {
            case <-closing:
                return
            default:
                // ....... 业务计算
                time.Sleep(100 * time.Millisecond)
            }
        }
    }()

    // 处理CTRL+C等中断信号
    termChan := make(chan os.Signal)
    signal.Notify(termChan, syscall.SIGINT, syscall.SIGTERM)
    <-termChan

    close(closing)
    // 执行退出之前的清理动作
    go doCleanup(closed)

    select {
    case <-closed:
    case <-time.After(time.Second):
        fmt.Println("清理超时，不等了")
    }
    fmt.Println("优雅退出")
}

func doCleanup(closed chan struct{}) {
    time.Sleep((time.Minute))
    close(closed)
}
```

### 锁

使用 chan 也可以实现互斥锁。在 chan 的内部实现中，就有一把互斥锁保护着它的所有字段。从外在表现上，chan 的发送和接收之间也存在着 happens-before 的关系，保证元素放进去之后，receiver 才能读取到（关于 happends-before 的关系，是指事件发生的先后顺序关系）

要想使用 chan 实现互斥锁，至少有两种方式。一种方式是先初始化一个 capacity 等于 1 的 Channel，然后再放入一个元素。这个元素就代表锁，谁取得了这个元素，就相当于获取了这把锁。另一种方式是，先初始化一个 capacity 等于 1 的 Channel，它的“空槽”代表锁，谁能成功地把元素发送到这个 Channel，谁就获取了这把锁。

代码如下：

```go
package main

import (
    "fmt"
    "time"
)

// 使用chan实现互斥锁
type Mutex struct {
    ch chan struct{}
}

// 使用锁需要初始化
func NewMutex() *Mutex {
    mu := &Mutex{make(chan struct{}, 1)}
    mu.ch <- struct{}{}
    return mu
}

// 请求锁，直到获取到
func (m *Mutex) Lock() {
    <-m.ch
}

// 解锁
func (m *Mutex) Unlock() {
    select {
    case m.ch <- struct{}{}:
    default:
        panic("unlock of unlocked mutex")
    }
}

// 尝试获取锁
func (m *Mutex) TryLock() bool {
    select {
    case <-m.ch:
        return true
    default:
    }
    return false
}

// 加入一个超时的设置
func (m *Mutex) LockTimeout(timeout time.Duration) bool {
    timer := time.NewTimer(timeout)
    select {
    case <-m.ch:
        timer.Stop()
        return true
    case <-timer.C:
    }
    return false
}

// 锁是否已被持有
func (m *Mutex) IsLocked() bool {
    return len(m.ch) == 0
}

func main() {
    m := NewMutex()
    ok := m.TryLock()
    fmt.Printf("locked v %v\n", ok)
    ok = m.TryLock()
    fmt.Printf("locked %v\n", ok)
}
```

可以用 buffer 等于 1 的 chan 实现互斥锁，在初始化这个锁的时候往 Channel 中先塞入一个元素，谁把这个元素取走，谁就获取了这把锁，把元素放回去，就是释放了锁。元素在放回到 chan 之前，不会有 goroutine 能从 chan 中取出元素的，这就保证了互斥性。

### Or-Done 模式

Or-Done 模式是信号通知模式中更宽泛的一种模式。

我们会使用“信号通知”实现某个任务执行完成后的通知机制，在实现时，我们为这个任务定义一个类型为 chan struct{}类型的 done 变量，等任务结束后，我们就可以 close 这个变量，然后，其它 receiver 就会收到这个通知。

如果有多个任务，只要有任意一个任务执行完，我们就想获得这个信号，这就是 Or-Done 模式。

```go
package main

import (
    "fmt"
    "time"
)

// refer to https://github.com/kat-co/concurrency-in-go-src/blob/master/concurrency-patterns-in-go/the-or-channel/fig-or-channel.go
func or(channels ...<-chan any) <-chan any {
    switch len(channels) {
    case 0:
        return nil
    case 1:
        return channels[0]
    }

    orDone := make(chan any)
    go func() {
        defer close(orDone)

        switch len(channels) {
        case 2:
            select {
            case <-channels[0]:
            case <-channels[1]:
            }
        default:
            select {
            case <-channels[0]:
            case <-channels[1]:
            case <-channels[2]:
            case <-or(append(channels[3:], orDone)...):
            }
        }
    }()
    return orDone
}

func sig(after time.Duration) <-chan any {
    c := make(chan any)
    go func() {
        defer close(c)
        time.Sleep(after)
    }()
    return c
}

func main() {

    start := time.Now()

    <-or(
        sig(5*time.Second),
        sig(20*time.Second),
        sig(30*time.Second),
        sig(40*time.Second),
        sig(50*time.Second),
        sig(01*time.Minute),
    )

    fmt.Printf("done after %v", time.Since(start))
}
```

### 扇入模式

扇入借鉴了数字电路的概念，它定义了单个逻辑门能够接受的数字信号输入最大量的术语。一个逻辑门可以有多个输入，一个输出。

模块的扇入是指有多少个上级模块调用它。而对于我们这里的 Channel 扇入模式来说，就是指有多个源 Channel 输入、一个目的 Channel 输出的情况。扇入比就是源 Channel 数量比 1。

每个源 Channel 的元素都会发送给目标 Channel，相当于目标 Channel 的 receiver 只需要监听目标 Channel，就可以接收所有发送给源 Channel 的数据。

```go
package main

import (
    "fmt"
    "reflect"
    "sync"
)

// https://github.com/campoy/justforfunc/blob/master/27-merging-chans/main.go

func fanIn(chans ...<-chan any) <-chan any {
    out := make(chan any)
    go func() {
        var wg sync.WaitGroup
        wg.Add(len(chans))

        for _, c := range chans {
            go func(c <-chan any) {
                for v := range c {
                    out <- v
                }
                wg.Done()
            }(c)
        }

        wg.Wait()
        close(out)
    }()
    return out
}

func fanInReflect(chans ...<-chan any) <-chan any {
    out := make(chan any)
    go func() {
        defer close(out)
        var cases []reflect.SelectCase
        for _, c := range chans {
            cases = append(cases, reflect.SelectCase{
                Dir:  reflect.SelectRecv,
                Chan: reflect.ValueOf(c),
            })
        }

        for len(cases) > 0 {
            i, v, ok := reflect.Select(cases)
            if !ok { //remove this case
                cases = append(cases[:i], cases[i+1:]...)
                continue
            }
            out <- v.Interface()
        }
    }()
    return out

}

func fanInRec(chans ...<-chan any) <-chan any {
    switch len(chans) {
    case 0:
        c := make(chan any)
        close(c)
        return c
    case 1:
        return chans[0]
    case 2:
        return mergeTwo(chans[0], chans[1])
    default:
        m := len(chans) / 2
        return mergeTwo(
            fanInRec(chans[:m]...),
            fanInRec(chans[m:]...))
    }
}

func mergeTwo(a, b <-chan any) <-chan any {
    c := make(chan any)

    go func() {
        defer close(c)
        for a != nil || b != nil {    //只要还有可读的chan
            select {
            case v, ok := <-a:
                if !ok {
                    a = nil    // a 已关闭，设置为nil
                    continue
                }
                c <- v
            case v, ok := <-b:
                if !ok {
                    b = nil    // b 已关闭，设置为nil
                    continue
                }
                c <- v
            }
        }
    }()
    return c
}

func asStream(done <-chan struct{}) <-chan any {
    s := make(chan any)
    values := []int{1, 2, 3, 4, 5}
    go func() {
        defer close(s)

        for _, v := range values {
            select {
            case <-done:
                return
            case s <- v:
            }
        }

    }()
    return s
}

func main() {
    fmt.Println("fanIn by goroutine:")
    done := make(chan struct{})
    // 直接模式
    ch := fanIn(asStream(done), asStream(done), asStream(done))
    for v := range ch {
        fmt.Println(v)
    }

    fmt.Println("fanIn by reflect:")
    // 超过两个chan时使用的反射形式
    ch = fanInReflect(asStream(done), asStream(done), asStream(done))
    for v := range ch {
        fmt.Println(v)
    }

    fmt.Println("fanIn by recursion:")
    // 递归模式
    ch = fanInRec(asStream(done), asStream(done), asStream(done))
    for v := range ch {
        fmt.Println(v)
    }
}
```

### 扇出模式

扇出模式只有一个输入源 Channel，有多个目标 Channel，扇出比就是 1 比目标 Channel 数的值，经常用在设计模式中的**观察者模式**中（观察者设计模式定义了对象间的一种一对多的组合关系。这样一来，一个对象的状态发生变化时，所有依赖于它的对象都会得到通知并自动刷新）。在观察者模式中，数据变动后，多个观察者都会收到这个变更信号

代码如下

```go
package main

import (
    "fmt"
    "reflect"
)


func fanOut(ch <-chan interface{}, out []chan interface{}, async bool) {
    go func() {
        defer func() { //退出时关闭所有的输出chan
            for i := 0; i < len(out); i++ {
                close(out[i])
            }
        }()

        for v := range ch { // 从输入chan中读取数据
            v := v
            for i := 0; i < len(out); i++ {
                i := i
                if async { //异步
                    go func() {
                        out[i] <- v // 放入到输出chan中,异步方式
                    }()
                } else {
                    out[i] <- v // 放入到输出chan中，同步方式
                }
            }
        }
    }()
}

func fanOutReflect(ch <-chan any, out []chan any) {
    go func() {
        defer func() {
            for i := 0; i < len(out); i++ {
                close(out[i])
            }
        }()

        cases := make([]reflect.SelectCase, len(out))
        for i := range cases {
            cases[i].Dir = reflect.SelectSend
        }

        for v := range ch {
            v := v
            for i := range cases {
                cases[i].Chan = reflect.ValueOf(out[i])
                cases[i].Send = reflect.ValueOf(v)
            }

            for _ = range cases { // for each channel
                chosen, _, _ := reflect.Select(cases)
                cases[chosen].Chan = reflect.ValueOf(nil)
            }
        }
    }()
}

func asStream(done <-chan struct{}) <-chan any {
    s := make(chan any)
    values := []int{1, 2, 3, 4, 5}
    go func() {
        defer close(s)

        for _, v := range values {
            select {
            case <-done:
                return
            case s <- v:
            }
        }

    }()
    return s
}

func main() {
    source := asStream(nil)
    channels := make([]chan any, 5)

    fmt.Println("fanOut")
    for i := 0; i < 5; i++ {
        channels[i] = make(chan any)
    }
    fanOut(source, channels, false)
    for i := 0; i < 5; i++ {
        for j := 0; j < 5; j++ {
            fmt.Printf("channel#%d: %v\n", j, <-channels[j])
        }
    }

    fmt.Println("fanOut By Reflect")
    source = asStream(nil)
    for i := 0; i < 5; i++ {
        channels[i] = make(chan any)
    }
    fanOutReflect(source, channels)
    for i := 0; i < 5; i++ {
        for j := 0; j < 5; j++ {
            fmt.Printf("channel#%d: %v\n", j, <-channels[j])
        }
    }
}
```

### stream

一种把 Channel 当作流式管道使用的方式，也就是把 Channel 看作流（Stream），提供跳过几个元素，或者是只取其中的几个元素等方法。

```go
package main

import (
    "fmt"
)

func asStreamSte[T any](done <-chan struct{}, values ...T) <-chan T {
    s := make(chan T) //创建一个unbuffered的channel
    go func() {       // 启动一个goroutine，往s中塞数据
        defer close(s) // 退出时关闭chan

        for _, v := range values { // 遍历数组
            select {
            case <-done:
                return
            case s <- v: // 将数组元素塞入到chan中
            }
        }

    }()
    return s
}
func asRepeatStream[T any](done <-chan struct{}, values ...T) <-chan T {
    s := make(chan T)
    go func() {
        defer close(s)
        for {
            for _, v := range values {
                select {
                case <-done:
                    return
                case s <- v:
                }
            }
        }
    }()
    return s
}

func takeN[T any](done <-chan struct{}, valueStream <-chan T, num int) <-chan T {
    takeStream := make(chan T) // 创建输出流
    go func() {
        defer close(takeStream)
        for i := 0; i < num; i++ { // 只读取前num个元素
            select {
            case <-done:
                return
            case takeStream <- <-valueStream: //从输入流中读取元素
            }
        }
    }()
    return takeStream
}

func takeFn[T any](done <-chan struct{}, valueStream <-chan T, fn func(any) bool) <-chan T {
    takeStream := make(chan T)
    go func() {
        defer close(takeStream)
        for {
            select {
            case <-done:
                return
            case v := <-valueStream:
                if fn(v) {
                    takeStream <- v
                }
            }
        }
    }()
    return takeStream
}

func takeWhile[T any](done <-chan struct{}, valueStream <-chan T, fn func(any) bool) <-chan T {
    takeStream := make(chan T)
    go func() {
        defer close(takeStream)
        for {
            select {
            case <-done:
                return
            case v := <-valueStream:
                if !fn(v) {
                    return
                }
                takeStream <- v
            }
        }
    }()
    return takeStream
}

func skipN[T any](done <-chan struct{}, valueStream <-chan T, num int) <-chan T {
    takeStream := make(chan T)
    go func() {
        defer close(takeStream)
        for i := 0; i < num; i++ {
            select {
            case <-done:
                return
            case <-valueStream:
            }
        }
        for {
            select {
            case <-done:
                return
            case takeStream <- <-valueStream:
            }
        }
    }()

    return takeStream
}

func skipFn[T any](done <-chan struct{}, valueStream <-chan T, fn func(any) bool) <-chan T {
    takeStream := make(chan T)
    go func() {
        defer close(takeStream)
        for {
            select {
            case <-done:
                return
            case v := <-valueStream:
                if !fn(v) {
                    takeStream <- v
                }
            }
        }
    }()
    return takeStream
}

func skipWhile[T any](done <-chan struct{}, valueStream <-chan T, fn func(any) bool) <-chan T {
    takeStream := make(chan T)
    go func() {
        defer close(takeStream)
        take := false
        for {
            select {
            case <-done:
                return
            case v := <-valueStream:
                if !take {
                    take = !fn(v)
                    if !take {
                        continue
                    }
                }
                takeStream <- v
            }
        }
    }()
    return takeStream
}

func main() {
    fmt.Println("asStream:")
    done := make(chan struct{})
    ch := asStreamSte(done, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
    for v := range ch {
        fmt.Println(v)
    }

    fmt.Println("asRepeatStream:")
    done = make(chan struct{})
    ch = asRepeatStream(done, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
    i := 0
    for v := range ch {
        fmt.Println(v)
        i++
        if i == 20 {
            break
        }
    }

    fmt.Println("takeN:")
    done = make(chan struct{})
    for v := range takeN(done, asRepeatStream(done, 1, 2, 3), 6) {
        fmt.Println(v)
    }

    evenFn := func(v any) bool {
        return v.(int)%2 == 0
    }
    lessFn := func(v any) bool {
        return v.(int) < 3
    }

    fmt.Println("takeFn:")
    done = make(chan struct{})
    i = 0
    for v := range takeFn(done, asRepeatStream(done, 1, 2, 3), evenFn) {
        fmt.Println(v)
        i++
        if i == 20 {
            break
        }
    }

    fmt.Println("takeWhile:")
    done = make(chan struct{})
    for v := range takeWhile(done, asRepeatStream(done, 1, 2, 3), lessFn) {
        fmt.Println(v)
    }

    fmt.Println("skipN:")
    done = make(chan struct{})
    for v := range takeN(done, skipN(done, asRepeatStream(done, 1, 2, 3), 2), 4) {
        fmt.Println(v)
    }

    fmt.Println("skipFn:")
    done = make(chan struct{})
    for v := range takeN(done, skipFn(done, asRepeatStream(done, 1, 2, 3), evenFn), 4) {
        fmt.Println(v)
    }

    fmt.Println("skipWhile:")
    done = make(chan struct{})
    for v := range takeN(done, skipWhile(done, asRepeatStream(done, 1, 2, 3), lessFn), 4) {
        fmt.Println(v)
    }
}
```

1. takeN：只取流中的前 n 个数据；
2. takeFn：筛选流中的数据，只保留满足条件的数据；
3. takeWhile：只取前面满足条件的数据，一旦不满足条件，就不再取；
4. skipN：跳过流中前几个数据；
5. skipFn：跳过满足条件的数据；
6. skipWhile：跳过前面满足条件的数据，一旦不满足条件，当前这个元素和以后的元素都会输出给 Channel 的 receiver。

### map-reduce

map-reduce 是一种处理数据的方式，最早是由 Google 公司研究提出的一种面向大规模数据处理的并行计算模型和方法，开源的版本是 hadoop，前几年比较火。

要讲的并不是分布式的 map-reduce，而是单机单进程的 map-reduce 方法。

map-reduce 分为两个步骤，第一步是映射（map），处理队列中的数据，第二步是规约（reduce），把列表中的每一个元素按照一定的处理方式处理成结果，放入到结果队列中。

就像做汉堡一样，map 就是单独处理每一种食材，reduce 就是从每一份食材中取一部分，做成一个汉堡。

```go
package main

import "fmt"

func mapChan[T, K any](in <-chan T, fn func(T) K) <-chan K {
    out := make(chan K) //创建一个输出chan
    if in == nil {      // 异常检查
        close(out)
        return out
    }

    go func() { // 启动一个goroutine,实现map的主要逻辑
        defer close(out)

        for v := range in { // 从输入chan读取数据，执行业务操作，也就是map操作
            out <- fn(v)
        }
    }()

    return out
}

func reduce[T, K any](in <-chan T, fn func(r K, v T) K) K {
    var out K

    if in == nil {
        return out
    }

    for v := range in {  // 实现reduce的主要逻辑
        out = fn(out, v)
    }

    return out
}

func asStreamMap(done <-chan struct{}) <-chan int {
    s := make(chan int)
    values := []int{1, 2, 3, 4, 5}
    go func() {
        defer close(s)

        for _, v := range values {
            select {
            case <-done:
                return
            case s <- v:
            }
        }

    }()
    return s
}

func main() {
    in := asStreamMap(nil)

    // map op: time 10
    mapFn := func(v int) int {
        return v * 10
    }

    // reduce op: sum
    reduceFn := func(r, v int) int {
        return r + v
    }

    sum := reduce(mapChan(in, mapFn), reduceFn)
    fmt.Println(sum)
}
```

![Image](https://github.com/user-attachments/assets/1ffb7560-ed6e-41fc-a09d-3bb364b82955)
