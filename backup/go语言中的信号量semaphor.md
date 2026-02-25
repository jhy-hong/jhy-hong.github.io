## 信号量是什么

在系统中，会给每一个进程一个信号量，代表每个进程目前的状态。未得到控制权的进程，会在特定的地方被迫停下来，等待可以继续进行的信号到来。

| 类型         | 取值范围   | 典型用途      | 说明                                           |
| ------------ | ---------- | ------------- | ---------------------------------------------- |
| 二进制信号量 | 0 或 1     | 互斥（Mutex） | 等价于互斥锁，但无“持有者”概念（任意线程可 V） |
| 计数信号量   | ≥ 0 的整数 | 资源池管理    | 值 = 当前可用资源数量（如数据库连接池大小=10） |

## **核心作用**

1. **互斥（Mutual Exclusion）**
   保护临界区（如：`P(mutex); 临界区; V(mutex)`）。
2. **进程/线程同步**
   协调执行顺序（如生产者-消费者问题中控制缓冲区空/满状态）。
3. **资源计数管理**
   限制并发访问数量（如：最多 5 个线程同时处理任务）。

## **核心实现方式**

### **① 使用带缓冲 Channel（最常用、简洁）**

适用于简单并发控制（如限流、工作池）。

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

func main() {
    sem := make(chan struct{}, 3) // 容量=3，即最多3个并发
    var wg sync.WaitGroup

    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            
            sem <- struct{}{}       //  获取信号量（阻塞直到有空位）
            defer func() { <-sem }() //  确保释放（即使 panic）

            fmt.Printf("Task %d started\n", id)
            time.Sleep(time.Second)
            fmt.Printf("Task %d finished\n", id)
        }(i)
    }
    wg.Wait()
    fmt.Println(" All tasks completed")
}
```

**优点**：代码简洁、无需额外依赖
 **注意**：

- 用 `struct{}{}` 节省内存
- **必须用 `defer` 保证释放**，避免 panic 导致泄漏
- **切勿关闭该 channel**（会导致 panic 或逻辑错误）
- 仅支持固定权重（每个任务占1单位）

### ②**使用** `golang.org/x/sync/semaphore`**（生产推荐 ）**

```go
package main

import (
	"context"
	"fmt"
	"golang.org/x/sync/semaphore"
	"log"
	"runtime"
	"time"
)

var (
	maxWorkers = runtime.GOMAXPROCS(0)                    // worker数量
	sema       = semaphore.NewWeighted(int64(maxWorkers)) //信号量
	task       = make([]int, maxWorkers*4)                // 任务数，是worker的四倍
)

func main() {
	ctx := context.Background()

	for i := range task {
		// 如果没有worker可用，会阻塞在这里，直到某个worker被释放
		if err := sema.Acquire(ctx, 1); err != nil {
			break
		}

		// 启动worker goroutine
		go func(i int) {
			defer sema.Release(1)
			time.Sleep(100 * time.Millisecond) // 模拟一个耗时操作
			fmt.Println("当前协程数:", runtime.NumGoroutine())
			task[i] = i + 1
		}(i)
	}

	// 请求所有的worker,这样能确保前面的worker都执行完
	if err := sema.Acquire(ctx, int64(maxWorkers)); err != nil {
		log.Printf("获取所有的worker失败: %v", err)
	}

	fmt.Println("结束后协程数:", runtime.NumGoroutine())
	fmt.Println(task)
}

```

或

```go
package main

import (
	"context"
	"fmt"
	"golang.org/x/sync/semaphore"
	"runtime"
	"sync"
	"time"
)

var (
	maxWorkers1 = runtime.GOMAXPROCS(0)                     // worker数量
	sema1       = semaphore.NewWeighted(int64(maxWorkers1)) //信号量
	task1       = make([]int, maxWorkers1*4)                // 任务数，是worker的四倍
	wg          sync.WaitGroup
)

func main() {
	ctx := context.Background()

	for i := range task1 {
		wg.Add(1)
		// 如果没有worker可用，会阻塞在这里，直到某个worker被释放
		if err := sema1.Acquire(ctx, 1); err != nil {
			break
		}

		// 启动worker goroutine
		go func(i int) {
			defer wg.Done()
			defer sema1.Release(1)
			time.Sleep(100 * time.Millisecond) // 模拟一个耗时操作
			fmt.Println("当前协程数:", runtime.NumGoroutine())
			task1[i] = i + 1
		}(i)
	}

	// 请求所有的worker,这样能确保前面的worker都执行完
	//if err := sema1.Acquire(ctx, int64(maxWorkers)); err != nil {
	//	log.Printf("获取所有的worker失败: %v", err)
	//}
	wg.Wait()

	fmt.Println("结束后协程数:", runtime.NumGoroutine())
	fmt.Println(task1)
}

```

不同之处在于

```go
if err := sema.Acquire(ctx, int64(maxWorkers)); err != nil {
    log.Printf("获取所有的worker失败: %v", err)
}
```

这一步非常关键。

它的语义是：

> 一次性申请全部 worker 资源

当它能成功 Acquire maxWorkers 时：

说明：

> 当前没有任何 goroutine **持有资源**

也就是说：

> 所有 worker 都执行完了

这相当于：

WaitGroup.Wait() 的效果

#### 这个包对外的api

1. **Acquire 方法：**可以一次获取多个资源，如果没有足够多的资源，调用者就会被阻塞。它的第一个参数是 Context，这就意味着，你可以通过 Context 增加超时或者 cancel 的机制。如果是正常获取了资源，就返回 nil；否则，就返回 ctx.Err()，信号量不改变。
2. **Release 方法：**可以将 n 个资源释放，返还给信号量。
3. **TryAcquire 方法：**尝试获取 n 个资源，但是它不会阻塞，要么成功获取 n 个资源，返回 true，要么一个也不获取，返回 false。

```go
package main

import (
    "context"
    "fmt"
    "sync"
    "time"
    "golang.org/x/sync/semaphore"
)

func main() {
    sem := semaphore.NewWeighted(3) // 总资源量=3
    var wg sync.WaitGroup

    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            
            ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
            defer cancel()
            
            if err := sem.Acquire(ctx, 1); err != nil { // 可指定权重（如2）
                fmt.Printf("Task %d timeout: %v\n", id, err)
                return
            }
            defer sem.Release(1) //  自动释放

            fmt.Printf("Task %d running\n", id)
            time.Sleep(time.Second)
        }(i)
    }
    wg.Wait()
}
```

**优势**：

- 支持动态权重（`Acquire(ctx, weight)`）
- 与 Context 集成，支持超时/取消
- 内部实现健壮，避免常见 channel 陷阱
- 适合连接池、资源配额等场景

## channel vs semaphor

| 场景                                | 推荐方案                      | 理由               |
| ----------------------------------- | ----------------------------- | ------------------ |
| 简单并发控制（如限流10个goroutine） | 带缓冲 Channel                | 代码极简，无依赖   |
| 需要超时/取消/权重控制              | `golang.org/x/sync/semaphore` | 功能完备，安全可靠 |
| 仅需互斥（二进制信号量）            | `sync.Mutex`                  | 语义清晰，性能最优 |
| 教学/原理理解                       | 手动实现                      | 不推荐生产使用     |

## **重要注意事项**

1. **资源泄漏防护**：所有获取操作必须配对释放，**务必用 `defer`**。
2. **Channel 陷阱**：勿关闭信号量 channel；避免在循环中误用循环变量。
3. **panic 安全**：Channel 方案中，若获取后 panic，defer 仍会释放；但若卡在“获取”阶段（channel 满），需外部保障。
4. 与 Mutex 区别：
   - Mutex：仅1个持有者（二进制信号量），用于临界区保护
   - Semaphore：允许多个持有者（计数信号量），用于资源池/并发数控制

在 Go 的 `golang.org/x/sync/semaphore` 包中，`sem.Acquire(ctx, weight)` 的 **`weight`（权重）参数表示本次操作需占用的信号量单位数**。设置为 `1` 与 `2` 的区别、动态权重的含义与作用如下：

------

##  核心区别：`Acquire(ctx, 1)` vs `Acquire(ctx, 2)`

假设信号量总容量为 `5`（`sem := semaphore.NewWeighted(5)`）：

| 操作              | 效果        | 剩余容量 | 并发能力                                                  |
| ----------------- | ----------- | -------- | --------------------------------------------------------- |
| `Acquire(ctx, 1)` | 占用 1 单位 | 4        | 最多 5 个任务并发                                         |
| `Acquire(ctx, 2)` | 占用 2 单位 | 3        | 最多 2 个“大任务”并发（2×2=4），剩余 1 单位可被小任务使用 |

 **关键**：  

- 权重是**资源消耗的度量单位**，非“优先级”  
- 释放时必须 `Release(2)`（与获取权重严格匹配），否则导致计数错误（资源泄漏/溢出）

------

## 动态权重：含义与作用

###  含义

- **运行时动态决定**每次 `Acquire` 所需资源量（非固定为 1）  

- 权重由业务逻辑计算得出，例如：

  ```go
  weight := func(task Task) int64 {
      if task.Size > 100*MB { return 3 } // 大文件需更多资源
      if task.Size > 10*MB  { return 2 }
      return 1
  }(task)
  sem.Acquire(ctx, weight)
  ```

###  核心作用

| 场景               | 价值                                                         |
| ------------------ | ------------------------------------------------------------ |
| **异构任务调度**   | 大任务（视频转码）占 3 单位，小任务（文本处理）占 1 单位，避免“小任务被大任务饿死”或“大任务无法启动” |
| **资源精准配额**   | 数据库连接池：普通查询占 1 连接，分布式事务占 3 连接         |
| **防资源碎片优化** | 合理设计权重策略（如总容量设为权重公倍数），提升资源利用率   |
| **弹性限流**       | 高峰期动态降低单任务权重，提升系统吞吐                       |

------

##  关键注意事项

1. **严格匹配释放**
   `Acquire(ctx, 3)` → 必须 `Release(3)`，否则信号量计数永久错误（严重 Bug！）
2. **权重 ≤ 总容量**
   若 `weight > sem.total`，`Acquire` 会永久阻塞（除非 Context 超时），需业务层校验
3. **避免死锁风险**
   多任务以不同顺序申请不同权重资源时（如 A 占 2 等 1，B 占 1 等 2），可能死锁。需设计获取顺序或使用超时
4. **公平性保障**
   该包实现为 **FIFO 等待队列**（文档明确说明 *fair*），避免小权重任务长期饥饿
5. **资源碎片问题**
   例：总容量 5，已占 3+1=4，剩余 1 单位 → 无法满足需 2 单位的新任务。需结合业务设计权重策略（如容量设为常用权重的公倍数）

------

## 实际场景示例

```go
// 视频处理服务：根据分辨率动态分配资源
func processVideo(ctx context.Context, sem *semaphore.Weighted, resolution string) error {
    weight := map[string]int64{"4K": 3, "1080p": 2, "720p": 1}[resolution]
    if err := sem.Acquire(ctx, weight); err != nil {
        return fmt.Errorf("资源不足: %w", err)
    }
    defer sem.Release(weight) // 严格匹配
    
    // 处理视频...
    return nil
}
// 总容量=5 时：可同时处理 1个4K + 1个1080p，或 5个720p
```

------

##  总结

- **`weight=1` vs `2`**：本质是“单次操作消耗的资源单位数不同”，直接影响并发组合与系统吞吐  
- **动态权重价值**：将信号量从“固定槽位模型”升级为 **“弹性资源配额模型”**，使并发控制更贴合真实业务需求  
- **使用铁律**：
  -  获取与释放权重严格一致
  -  权重值需业务校验（≤总容量）
  -  结合 Context 超时避免永久阻塞

合理运用动态权重，可显著提升资源利用率与系统鲁棒性，是高阶并发设计的关键技巧 