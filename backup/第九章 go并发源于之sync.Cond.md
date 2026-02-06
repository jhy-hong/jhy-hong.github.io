## Cond的概念

Cond 原语的目的是，为等待 / 通知场景下的并发问题提供支持。Cond 通常应用于等待某个条件的一组 goroutine，等条件变为 true 的时候，其中一个 goroutine 或者所有的 goroutine 都会被唤醒执行。

> 从开发实践上，我们真正使用 Cond 的场景比较少，因为一旦遇到需要使用 Cond 的场景，我们更多地会使用 Channel 的方式去实现，因为那才是更地道的 Go 语言的写法

## Cond的方法

**Wait** 方法:会把调用者 Caller 放入 Cond 的等待队列中并阻塞，直到被 Signal 或者 Broadcast 的方法从等待队列中移除并唤醒。

调用 Wait 方法时必须要持有 c.L 的锁。

> 1. 必须先持有锁
> 2. 当前 goroutine 挂起（进入等待队列）
> 3. 自动释放锁
> 4. 被唤醒后 → 重新抢锁
> 5. 返回
> 6. **Wait = 解锁 + 休眠 + 被唤醒后加锁**

**Signal** 方法,允许调用者 Caller 唤醒一个等待此 Cond 的 goroutine。如果此时没有等待的 goroutine，显然无需通知 waiter；如果 Cond 等待队列中有一个或者多个等待的 goroutine，则需要从等待队列中移除第一个 goroutine 并把它唤醒。在其他编程语言中，比如 Java 语言中，Signal 方法也被叫做 notify 方法。 唤醒 **一个** 等待者

调用 Signal 方法时，不强求你一定要持有 c.L 的锁。

> 不会释放锁，只是发通知

**Broadcast** 方法,允许调用者 Caller 唤醒所有等待此 Cond 的 goroutine。如果此时没有等待的 goroutine，显然无需通知 waiter；如果 Cond 等待队列中有一个或者多个等待的 goroutine，则清空所有等待的 goroutine，并全部唤醒。在其他编程语言中，比如 Java 语言中，Broadcast 方法也被叫做 notifyAll 方法。

调用 Broadcast 方法时，也不强求你一定持有 c.L 的锁。

> 不会释放锁，只是发通知

## Cond的使用

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

type Queue struct {
	mu    sync.Mutex
	cond  *sync.Cond
	items []int
}

func NewQueue() *Queue {
	q := &Queue{}
	q.cond = sync.NewCond(&q.mu)
	return q
}

func (q *Queue) Push(v int) {
	q.mu.Lock()
	q.items = append(q.items, v)
	q.cond.Signal() // 唤醒一个等待的 Pop
	q.mu.Unlock()
}

func (q *Queue) Pop() int {
	q.mu.Lock()
	for len(q.items) == 0 {
		q.cond.Wait()
	}
	v := q.items[0]
	q.items = q.items[1:]
	q.mu.Unlock()
	return v
}

func main() {
	queue := NewQueue()
	var wg sync.WaitGroup

	// 生产者 goroutine
	wg.Add(1)
	go func() {
		defer wg.Done()
		for i := 1; i <= 5; i++ {
			fmt.Printf("生产者: 推送 %d\n", i)
			queue.Push(i)
			time.Sleep(100 * time.Millisecond) // 模拟生产间隔
		}
	}()

	// 消费者 goroutine
	wg.Add(1)
	go func() {
		defer wg.Done()
		for i := 1; i <= 5; i++ {
			value := queue.Pop()
			fmt.Printf("消费者: 弹出 %d\n", value)
			time.Sleep(150 * time.Millisecond) // 模拟消费时间
		}
	}()

	// 等待所有goroutine完成
	wg.Wait()
	fmt.Println("所有操作完成")

	// 更复杂的测试场景 - 多个消费者
	fmt.Println("\n=== 多消费者测试 ===")
	queue2 := NewQueue()

	// 启动多个消费者
	for i := 0; i < 3; i++ {
		wg.Add(1)
		go func(consumerID int) {
			defer wg.Done()
			// 每个消费者尝试获取2个元素
			for j := 0; j < 2; j++ {
				value := queue2.Pop()
				fmt.Printf("消费者 %d: 弹出 %d\n", consumerID, value)
				time.Sleep(50 * time.Millisecond)
			}
		}(i)
	}

	// 生产者
	wg.Add(1)
	go func() {
		defer wg.Done()
		// 生产6个元素
		for i := 1; i <= 6; i++ {
			time.Sleep(80 * time.Millisecond) // 控制生产节奏
			fmt.Printf("生产者: 推送 %d\n", i)
			queue2.Push(i)
		}
	}()

	wg.Wait()
	fmt.Println("多消费者测试完成")

	// 测试广播功能
	fmt.Println("\n=== Broadcast测试 ===")
	broadcastQueue := NewQueue()

	// 启动3个等待的消费者
	for i := 0; i < 3; i++ {
		wg.Add(1)
		go func(waiterID int) {
			defer wg.Done()
			fmt.Printf("等待者 %d: 开始等待\n", waiterID)
			value := broadcastQueue.Pop()
			fmt.Printf("等待者 %d: 收到值 %d\n", waiterID, value)
		}(i)
	}

	// 等待所有消费者进入等待状态
	time.Sleep(2 * time.Second)

	// 推送一个值，应该唤醒一个等待者
	fmt.Println("推送第一个值...")
	broadcastQueue.Push(999)

	// 再次等待
	time.Sleep(4 * time.Second)

	// 推送更多值以满足其他等待者
	for i := 1; i <= 2; i++ {
		broadcastQueue.Push(1000 + i)
		time.Sleep(100 * time.Millisecond)
	}

	wg.Wait()
	fmt.Println("Broadcast测试完成")
}

```

## 基本使用方法

1. 创建Cond

   ```go
   mu := sync.Mutex{}
   cond := sync.NewCond(&mu)
   ```

   > `Cond` 必须绑定一个锁（`Mutex` 或 `RWMutex`）

2. 等待条件（必须用 for）

   ```go
   cond.L.Lock()
   for !condition {
       cond.Wait()
   }
   doSomething()
   cond.L.Unlock()
   
   ```

   必须是 `for`，不能是 `if`

3. 修改条件 + 通知

   ```go
   cond.L.Lock()
   condition = true
   cond.Signal()     // 或 cond.Broadcast()
   cond.L.Unlock()
   ```

## 注意事项

1. 用 if 而不是 for（严重 bug）

   ```go
   //  错误
   if len(q.items) == 0 {
       cond.Wait()
   }
   ```

   原因：

   - 可能被“误唤醒”（spurious wakeup）
   - 多个 goroutine 被 `Broadcast` 同时唤醒
   - 条件可能再次不满足

   ```go
   // 正确
   for !condition {
       cond.Wait()
   }
   ```

2. Wait 前没加锁（直接 panic）❌

   `Wait` 内部会调用 `Unlock()`，没锁直接炸。

3. 修改条件不加锁（数据竞争）❌

   ```go
   condition = true
   cond.Signal()
   
   ```

   必须包在同一把锁里。

4. Signal 在 Unlock 之后调用（逻辑错误）❌

5. 期望 Signal 精确唤醒“某个” goroutine ❌

   Cond **不保证公平性、不保证顺序**。

## cond和channel该怎么选

| 场景                    | 推荐        |
| ----------------------- | ----------- |
| 简单通知、一次性事件    | channel     |
| 生产者 / 消费者（简单） | channel     |
| 多条件 + 状态反复变化   | `sync.Cond` |
| 广播唤醒                | `sync.Cond` |
| 实现并发数据结构        | `sync.Cond` |

## 总结

`sync.Cond` 是 Go 中用于 **基于共享条件的协程等待与唤醒机制**，
 适合 **多条件、广播通知、并发结构实现** 的场景，
 使用时必须 **锁 + for + Wait/Signal 原子配合**，
 一旦用错，**不是死锁就是竞态**。