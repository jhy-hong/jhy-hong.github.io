# 为什么需要互斥锁？
1. 防止数据竞争​：当多个goroutine并发访问共享资源时，可能导致数据不一致或逻辑错误。互斥锁确保同一时刻只有一个goroutine操作共享资源。
2. 保证原子性​：某些操作需要不可分割的原子性（如计数器递增），互斥锁可以保证这些操作在并发环境下的正确性。
3. 假设多个线程同时修改一个共享变量，会发生什么？
例如：两个线程同时执行 counter++（实际分为三步：读值、加1、写回）。

- 无锁时​：两个线程可能同时读取到旧值（如 counter=1），各自加1后写回 2，最终结果是 2，而不是预期的 3。
- 有锁时​：一个线程先获取锁，完成读、加、写操作后释放锁，另一个线程再执行。最终结果是正确的 3。
  ​互斥锁的缺失会导致数据不一致、程序崩溃或不可预测的行为。


## 互斥锁的核心概念
1. 互斥性​
    互斥锁在任何时刻只能被一个线程持有，其他线程必须等待锁释放后才能获取锁。这确保了共享资源的操作是串行化的（同一时间只有一个操作被执行）。
2. 临界区（Critical Section）​​  被互斥锁保护的代码段称为临界区。只有持有锁的线程才能执行临界区的代码。
3. 原子性（Atomicity）​​
    互斥锁确保对共享资源的操作是“不可分割”的，即其他线程无法看到中间状态的操作结果。
## 互斥锁的实现机制
在并发编程中，如果程序中的一部分会被并发访问或修改，那么，为了避免并发访问导致的意想不到的结果，这部分程序需要被保护起来，这部分被保护起来的程序，就叫做**临界区**。

可以说，临界区就是一个被共享的资源，或者说是一个整体的一组共享资源，比如对数据库的访问、对某一个共享数据结构的操作、对一个 I/O 设备的使用、对一个连接池中的连接的调用，等等。

如果很多线程同步访问临界区，就会造成访问或操作错误，这当然不是我们希望看到的结果。所以，我们可以使用互斥锁，限定临界区只能同时由一个线程持有。

当临界区由一个线程持有的时候，其它线程如果想进入这个临界区，就会返回失败，或者是等待。直到持有的线程退出临界区，这些等待线程中的某一个才有机会接着持有这个临界区。

![Image](https://github.com/user-attachments/assets/0c44c057-ea30-44c5-b883-94b87e8271a2)

## 两种模式​
- **​正常模式​**：
-- 新到的goroutine会先尝试自旋（循环检查锁状态），若**多次失败则进入FIFO队列**。
-- 释放锁时，优先唤醒队列中的**等待者**；若**队列为空，直接释放锁**。
- **饥饿模式​**（解决长时间等待问题）：
 -- 当某个等待的goroutine超过1ms未获取锁，触发饥饿模式。
  -- 锁直接交给队列中的第一个等待者，新到的goroutine不竞争，直接入队。

## 自旋的优化
- 目的​：减少上下文切换，适用于锁持有时间短的场景。
- 限制​：自旋次数有限（通常4次），避免浪费CPU。

## 关键操作流程
- 加锁（Lock()）​​：
  1. 快速路径：通过原子操作尝试获取锁（若未被持有）。
  2. 若失败，进入慢路径：
          自旋尝试（正常模式）。
          自旋失败后加入等待队列，可能切换为饥饿模式。
- 解锁（Unlock()）​​：
  1. 原子操作释放锁。
  2. 根据当前模式唤醒等待队列中的goroutine（正常模式可能直接释放，饥饿模式必须传递锁）。

### 示例代码
```go
package main

import (
    "fmt"
    "sync"
)

var counter int
var mu sync.Mutex

func main() {
    var wg sync.WaitGroup
    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            mu.Lock()
            counter++ // 临界区操作
            mu.Unlock()
        }()
    }
    wg.Wait()
    fmt.Println(counter) // 正确输出1000
}
```
当一个 goroutine 通过调用 Lock 方法获得了这个锁的拥有权后， 其它请求锁的 goroutine 就会阻塞在 Lock 方法的调用上，直到锁被释放并且自己获取到了这个锁的拥有权。

如果不加锁：
```go
package main

import (
    "fmt"
    "sync"
)

var counter int
var mu sync.Mutex

func main() {
    var wg sync.WaitGroup
    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            
            counter++ // 临界区操作
     
        }()
    }
    wg.Wait()
    fmt.Println(counter) // 正确输出1000
}
```
可以尝试运行发现结果每次可能都不一样

![Image](https://github.com/user-attachments/assets/af42f84c-b3fa-4808-8c9e-52f706dc4515)

我们使用 sync.WaitGroup 来等待所有的 goroutine 执行完毕后，再输出最终的结果。sync.WaitGroup 这个同步原语我会在后面的课程中具体介绍，现在你只需要知道，我们使用它来控制等待一组 goroutine 全部做完任务。

但是，每次运行，你都可能得到不同的结果，基本上不会得到理想中的1000的结果。

这是因为，count++ 不是一个原子操作，它至少包含几个步骤，比如读取变量 count 的当前值，对这个值加 1，把结果再保存到 count 中。因为不是原子操作，就可能有并发的问题。

比如，10 个 goroutine 同时读取到 count 的值为 952，接着各自按照自己的逻辑加 1，值变成了 953，然后把这个结果再写回到 count 变量。但是，实际上，此时我们增加的总数应该是 10 才对，这里却只增加了 1，好多计数都被“吞”掉了。这是并发访问共享数据的常见错误。

这个问题，有经验的开发人员比较容易发现，但是，很多时候，并发问题隐藏得非常深，即使是有经验的人，也不太容易发现或者 Debug 出来。

针对这个问题，Go 提供了一个检测并发访问共享资源是否有问题的工具： [race detector](https://go.dev/blog/race-detector)，它可以帮助我们自动发现程序有没有 data race 的问题。

在编译（compile）、测试（test）、运行（run）Go 代码的时候，加上 race 参数，就有可能发现并发问题。比如在上面的例子中，我们可以加上 race 参数运行，检测一下是不是有并发问题。如果你 go run -race counter.go，就会输出警告信息。

![Image](https://github.com/user-attachments/assets/097ae0c3-9aaa-461c-bbf7-adb481f8c516)

这个警告不但会告诉你有并发问题，而且还会告诉你哪个 goroutine 在哪一行对哪个变量有写操作，同时，哪个 goroutine 在哪一行对哪个变量有读操作，就是这些并发的读写访问，引起了 data race。

可以把获取锁、释放锁、计数加一的逻辑封装成一个方法，对外不需要暴露锁等逻辑：
```go
func main() {
    // 封装好的计数器
    var counter Counter

    var wg sync.WaitGroup
    wg.Add(10)

    // 启动10个goroutine
    for i := 0; i < 10; i++ {
        go func() {
            defer wg.Done()
            // 执行10万次累加
            for j := 0; j < 100000; j++ {
                counter.Incr() // 受到锁保护的方法
            }
        }()
    }
    wg.Wait()
    fmt.Println(counter.Count())
}

// 线程安全的计数器类型
type Counter struct {
    CounterType int
    Name        string

    mu    sync.Mutex
    count uint64
}

// 加1的方法，内部使用互斥锁保护
func (c *Counter) Incr() {
    c.mu.Lock()
    c.count++
    c.mu.Unlock()
}

// 得到计数器的值，也需要锁保护
func (c *Counter) Count() uint64 {
    c.mu.Lock()
    defer c.mu.Unlock()
    return c.count
}
```

## 总结
介绍了并发问题的背景知识、标准库中 Mutex 的使用和最佳实践、通过 race detector 工具发现计数器程序的问题以及修复方法。相信你已经大致了解了 Mutex 这个同步原语。

在项目开发的初始阶段，我们可能并没有仔细地考虑资源的并发问题，因为在初始阶段，我们还不确定这个资源是否被共享。经过更加深入的设计，或者新功能的增加、代码的完善，这个时候，我们就需要考虑共享资源的并发问题了。当然，如果你能在初始阶段预见到资源会被共享并发访问就更好了。

意识到共享资源的并发访问的早晚不重要，重要的是，一旦你意识到这个问题，你就要及时通过互斥锁等手段去解决。

查看一些开源项目 比如 Docker issue [37583](https://github.com/moby/moby/pull/37583)、[35517](https://github.com/moby/moby/pull/35517)、[30696](https://github.com/moby/moby/pull/30696)等、kubernetes issue [72361](https://github.com/kubernetes/kubernetes/pull/72361)、[71617](https://github.com/kubernetes/kubernetes/pull/71617)等，都是后来发现的 data race 而采用互斥锁解决的。
