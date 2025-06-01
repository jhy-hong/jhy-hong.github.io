`WaitGroup`很简单，就是`sync` 用来做任务编排的一个并发原语。它要解决的就是并发 - 等待的问题：现在有一个 goroutine A 在检查点（`checkpoint`）等待一组 `goroutine` 全部完成，如果在执行任务的这些 `goroutine` 还没全部完成，那么 goroutine A 就会阻塞在检查点，直到所有 `goroutine` 都完成后才能继续执行。

我们来看一个使用`WaitGroup`的场景。

比如，我们要完成一个大的任务，需要使用并行的 `goroutine` 执行三个小任务，只有这三个小任务都完成，我们才能去执行后面的任务。如果通过轮询的方式定时询问三个小任务是否完成，会存在两个问题：一是，性能比较低，因为三个小任务可能早就完成了，却要等很长时间才被轮询到；二是，会有很多无谓的轮询，空耗 CPU 资源。

## WaitGroup的基本用法

典型用法示例：

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

func main() {
    var wg sync.WaitGroup

    // 假设要并发启动 3 个任务
    taskCount := 3
    wg.Add(taskCount) // 将内部计数器 +3

    for i := 1; i <= taskCount; i++ {
        go func(id int) {
            defer wg.Done() // 在 goroutine 退出时将计数器 -1
            // 模拟实际工作
            time.Sleep(time.Duration(id) * 500 * time.Millisecond)
            fmt.Printf("任务 %d 执行完毕\n", id)
        }(i)
    }

    // 在 main goroutine 中等待所有子 goroutine 完成
    wg.Wait() // 阻塞，直到内部计数器归零
    fmt.Println("所有任务已完成")
}

```

- **`wg.Add(n int)`**：将内部计数器加上 `n`。一般在启动对应数量的 goroutine 前调用。
- **`wg.Done()`**：等价于 `wg.Add(-1)`。表示“当前 goroutine 完成一个任务”，将内部计数器减 1。通常放在 goroutine 的 `defer` 中，确保无论是否出错都会执行。
- **`wg.Wait()`**：阻塞调用方，直到内部计数器降为 0 后才解除阻塞，继续往下执行。

## WaitGroup 的实现

### WaitGroup的数据结构

在源码 `src/runtime/sema.go` 与 `src/sync/waitgroup.go` 中可以看到，`WaitGroup` 定义为：

```go
type WaitGroup struct {
	noCopy noCopy
	// state: 64 位无符号整数
    // 高 32 位 (state1) 用来记录等待的 goroutine（调用 Wait 并阻塞的 goroutine）数目
    // 低 32 位 (state0) 用来记录计数器（Add/Done 相加减的结果）
	state atomic.Uint64 // high 32 bits are counter, low 32 bits are waiter count.  内部字段，具体布局随版本而变化
	sema  uint32 // futex 信号量，供被阻塞的 Wait() & Done() 使用
}
```

- **`state0`（低 32 位）**：表示当前“未完成任务”的计数（等同 `Add() - Done()`）。
- **`state1`（高 32 位）**：表示有多少个 goroutine 正在执行 `Wait()` 并被阻塞，等待计数归零。
- **`sema`**：一个用于阻塞与唤醒的信号量，内部由 runtime 调度。

> 注意：实际源码中将 `state1` 和 `state0` 封装到一个 64 位整数组件里，通过原子操作（`atomic.AddUint64` 等）同时修改这两部分

### Add方法逻辑



```go
// Add 将 WaitGroup 的计数器增加或减少 delta。
// 当 delta 为正时，表示有新的任务加入；当 delta 为负时，表示有任务完成。
// 如果在有等待者时将计数器从正数减到 0，会释放所有阻塞在 Wait() 上的 goroutine。
func (wg *WaitGroup) Add(delta int) {
    // 下面是针对 Go race detector 的额外逻辑，用于捕捉竞态情况：
    if race.Enabled {
        if delta < 0 {
            // 当 delta < 0（即 Done 操作）时，需要与 Wait() 的释放进行同步。
            // race.ReleaseMerge 用于告诉检测器，“我在此处与 Wait 进行合并同步”。
            race.ReleaseMerge(unsafe.Pointer(wg))
        }
        // 在执行后续对 wg.state 的原子操作时，先禁用 race 检测
        race.Disable()
        defer race.Enable() // 在函数结束时重新启用 race 检测
    }

    // =============================== 核心原子操作 ===============================
    // state 是一个 64 位整数，由高 32 位和低 32 位组成。
    // - 高 32 位 (v) 表示 WaitGroup 的当前计数，即还有多少任务未完成。
    // - 低 32 位 (w) 表示当前有多少个 goroutine 正在 Wait() 中被阻塞，等待计数归零。
    //
    // 这里执行 Add(delta) 时，将 delta << 32，左移到“高 32 位”上，然后原子累加到 wg.state 上。
    // Add 返回加完之后的完整 64 位值。
    state := wg.state.Add(uint64(delta) << 32)

    // 从 state 中解析出：
    // v：高 32 位，表示“未完成任务”的数量
    // w：低 32 位，表示“等待者”的数量
    v := int32(state >> 32) // 高 32 位转换为 int32，即当前的计数值
    w := uint32(state)      // 低 32 位，即当前等待者的数量

    // race 检测：当 delta>0，并且 v==delta（即原来计数是 0，现在是 delta）
    // 说明这是第一次真正将计数从 0 推到正数。为了与可能已有的 Wait() 竞态，这里做一次“读”操作触发同步
    if race.Enabled && delta > 0 && v == int32(delta) {
        // 标记一下，表示“我读了一下 wg.sema”，让 race detector 能知道 Add 与 Wait 之间的依赖
        race.Read(unsafe.Pointer(&wg.sema))
    }

    // ============ 检查负数计数的 panic 情况 ============
    if v < 0 {
        // 如果 v<0，说明调用者做了多余的 Done（Add(-1) 超过了 Add() 的次数），导致计数器变为负数。
        panic("sync: negative WaitGroup counter")
    }

    // ============ 检查 Add 与 Wait 并发使用的 panic 情况 ============
    // 当 w != 0，说明当前已经有至少 1 个 goroutine 在 Wait() 阻塞（等待者 != 0）。
    // 如果又发生了 delta > 0 且 v == delta（即原来的计数为 0，现在正好被设为 delta），
    // 也就意味着“在有等待者挂起时，又从 0 重新开始 Add”，这属于 WaitGroup 不允许的误用，
    // 会导致竞态：因为 Wait() 假定一旦看见计数是0，就不再把自己计入等待者。
    // 既然此刻已有等待者，却又“首次 Add”，这种用法被认为是不安全的，故 panic。
    if w != 0 && delta > 0 && v == int32(delta) {
        panic("sync: WaitGroup misuse: Add called concurrently with Wait")
    }

    // ============ 如果当前计数仍然 > 0，或者当前没有等待者，直接返回 ============
    // v > 0 表示计数器不为 0，说明还有子任务尚未完成，此时无需触发任何等待者唤醒逻辑。
    // w == 0 则表示没有任何 goroutine 在 Wait() 上阻塞，也无需做后续的释放操作。
    if v > 0 || w == 0 {
        return
    }

    // ============ 下面的逻辑只在 “v == 0 && w > 0” 时执行 ============
    // 这种场景：刚好把计数从 1 减到了 0（v == 0），并且此时有多个 Wait() 正在阻塞（w > 0）。
    // 根据 WaitGroup 的设计，需要一次性唤醒所有的等待者 goroutine。

    // 先做一个小的 sanity check（健全性检查）：
    // 确保在我们读取到 state 之后，没有其它 concurrent 的 Add/Done 对 wg.state 产生变化。
    // 如果 wg.state.Load() 不等于刚刚计算得到的 state，就说明有人在此时刻同时修改了状态，
    // 那就说明 WaitGroup 又被误用（Add 与 Wait 并发），panic 提示用户。
    if wg.state.Load() != state {
        panic("sync: WaitGroup misuse: Add called concurrently with Wait")
    }

    // 因为 state 的高 32 位（计数器）已经是 0，而低 32 位（等待者计数）等于 w > 0。
    // 既然要唤醒完所有等待者，就要重置整个状态为 0（因为新的计数器 = 0，且等待者将被清空）。
    wg.state.Store(0)

    // 释放 semaphore，让所有在 Semacquire 阻塞的 Wait() goroutine 依次被唤醒
    // 注意：需要循环 w 次释放，因为低 32 位记录了有多少个 goroutine 阻塞在信号量上
    for ; w != 0; w-- {
        runtime_Semrelease(&wg.sema, false, 0)
    }
}

// Done 是 Add(-1) 的简写，用于表示当前一个任务完成，计数器减 1。
// 通常在每个 goroutine 中使用 defer wg.Done() 来确保在 goroutine 结束时自动调用 Done。
func (wg *WaitGroup) Done() {
    wg.Add(-1)
}


```

方法主要操作的是 state 的计数部分。你可以为计数值增加一个 delta 值，内部通过原子操作把这个值加到计数值上。需要注意的是，这个 delta 也可以是个负数，相当于为计数值减去一个值，Done 方法内部其实就是通过 Add(-1) 实现的。

1. **高性能与原子操作：**

- 利用 `uint64` 的高低 32 位分别存储任务计数器和等待者数量，使用一个原子变量实现并发安全，避免加锁。

2. **信号量唤醒：**

- 使用底层的 `runtime_Semrelease` 实现唤醒阻塞在 `Wait()` 的 goroutines，提高效率。

3. **误用检测：**

- 检测是否在 `Wait()` 调用后仍然执行了 `Add()`，这是一个严重的逻辑错误，Go 通过 panic 来提醒开发者。

Add的操作逻辑：

1. 调用者执行 `Add(delta)`，其中 `delta` 可能为正（增加任务数）或为负（表示 Done）。
2. 底层用原子加 `state += uint64(delta)`。
3. 如果 `delta` 为正，则只需要更新计数器，无须阻塞其它 goroutine。
4. 如果 `delta` 为负（即相当于调用 Done），则可能出现以下两种情况：

- 新的 `state0` 仍 > 0，说明还有未完成任务，不需要触发唤醒；
- 新的 `state0` 恰好变为 0，且此时有若干个 goroutine 正在 `Wait()` 阻塞（`state1 > 0`），这时就要唤醒它们：调用 `runtime_Semrelease(&wg.sema)`，让所有阻塞在 `sema` 上的 goroutine 解除阻塞。

### Wait方法的实现逻辑

```go
// Wait 会阻塞，直到 WaitGroup 的计数器变为 0。
// 即：所有调用过 Add(+1) 的任务都执行 Done()（Add(-1)）后，Wait 才会返回。
func (wg *WaitGroup) Wait() {
	if race.Enabled {
		// 如果启用了竞态检测器，暂时禁用它（避免在原子操作期间误判）
		race.Disable()
	}

	for {
		// 读取当前的 state：64位，包含“任务计数器”（高32位）和“等待者数量”（低32位）
		state := wg.state.Load()
		v := int32(state >> 32) // 高 32 位：表示当前还有多少个子任务未完成
		w := uint32(state)      // 低 32 位：表示当前已经有多少个 goroutine 在等待
		// ============ 如果 v == 0，说明已经没有未完成任务，无需阻塞 ============
		if v == 0 {
			// 如果计数器已经为 0，说明不需要等待，立即返回
			if race.Enabled {
				// 如果启用了 race 检测器，恢复，并声明 Acquire（获取了 wg 的同步锁）
				race.Enable()
				race.Acquire(unsafe.Pointer(wg))
			}
			return
		}

		// 尝试将等待者数量加 1（低 32 位加 1）
		// 比如原始 state = (v<<32 | w)，现在希望变成 (v<<32 | w+1)
		if wg.state.CompareAndSwap(state, state+1) {
			// 加 1 成功（这段是 CAS 原子操作，防止并发 Wait 出错）

			if race.Enabled && w == 0 {
				// 如果这是第一个等待者（w == 0），且开启了竞态检测
				// 需要与 Add 中的 race.Read 配对，模拟一次“写入”
				// 否则 Wait() 与后续 Add() 可能存在未捕捉的并发竞态

				// 注意：只有第一个 Wait() 会做 race 写操作，多个并发 Wait() 会互相竞态
				race.Write(unsafe.Pointer(&wg.sema))
			}

			// 真正阻塞当前 goroutine，等待被 runtime_Semrelease 唤醒（在 Add 中）
			runtime_SemacquireWaitGroup(&wg.sema)

			// 被唤醒后，如果发现状态不为 0，说明 WaitGroup 被重复使用了
			// 例如：一个 Wait() 返回后，还没等下一轮 Add() 开始，用户就又调用 Wait()，属于误用
			if wg.state.Load() != 0 {
				panic("sync: WaitGroup is reused before previous Wait has returned")
			}

			// 恢复 race 检测器状态
			if race.Enabled {
				race.Enable()
				race.Acquire(unsafe.Pointer(wg))
			}
			return
		}

		// 如果 CompareAndSwap 失败（说明有其它 goroutine 同时修改了 state），重新尝试
	}
}

```

Wait 方法的实现逻辑是：不断检查 state 的值。如果其中的计数值变为了 0，那么说明所有的任务已完成，调用者不必再等待，直接返回。如果计数值大于 0，说明此时还有任务没完成，那么调用者就变成了等待者，需要加入 waiter 队列，并且阻塞住自己。

1. **读取计数器**：从 `wg.state` 中获取当前任务数 `v` 和等待者数 `w`。
2. **计数器为 0**：直接返回，说明没有需要等待的任务。
3. **计数器不为 0**：调用 CAS 尝试将等待者数 +1。
4. **阻塞等待**：调用 `runtime_SemacquireWaitGroup` 阻塞，等待 `Add(-1)` 把计数器减到 0 时唤醒。
5. **唤醒时校验**：确保 WaitGroup 没被错误地重复使用（计数器应该清零）。
6. **race 检测同步**：通过 `race.Write` / `race.Acquire` 等让 race detector 能识别同步关系。

## 使用WaitGroup的常见错误

| 原则 | 说明 |
| ---- | ---- |
| ✅ `Add()` 必须在启动 goroutine **之前调用** | 否则与 `Wait()` 存在并发冲突 
| ✅ 每个 goroutine 必须确保调用 `Done()` | 可用 `defer wg.Done()` 保证 |
| ❌ 不要对同一个 `WaitGroup` **重复使用** | 每次使用请用新变量 |
| ❌ 不要 `Add(-1)` 或调用 `Done()` 多次 | 会导致 panic |
| ✅ `Wait()` 要等所有 goroutine 完成后再返回 | 不应提前退出主线程 |

1. **Add/Done 调用要匹配**

   - 每次 `wg.Add(1)` 必须对应一个 `wg.Done()`，否则计数永远不会归零，导致 `Wait()` 永远阻塞。
   - 反之，如果调用 `Done()` 过多，也会让计数变为负数并 panic。

2. **确保在启动 Goroutine 之前调用 `Add()`**
    如果你在某些并发场景下是先启动 goroutine, 然后再调用 `Add()`，会导致竞争条件。假设：

   ```go
   go func() {
       defer wg.Done() // 这里有 Done()
       // do something...
   }()
   
   wg.Add(1) // 可能 goroutine 内的 Done 会先执行，从而导致计数出现负数 panic
   ```

   - **正确示例**：先 `wg.Add(1)`，再启动 goroutine。
   - **可见**：`Add()` 与对应的 `Done()` 之间存在竞态，必须保证 `Add()` 始终“先行”或在同一个同步点之前执行。

3. **禁止复制 `WaitGroup`**
    `WaitGroup` 内部会包含一个 64 位整数作为计数器和信号量，若拷贝一个 `WaitGroup`（无论是值拷贝还是传值给函数），都会导致原本的内部状态与副本状态不一致，出现难以预料的死锁或 panic。

   - **示例（错误写法）**：

     ```go
     func f(wg sync.WaitGroup) {
         // 这里 wg 是拷贝，和调用方的 WaitGroup 不是同一个
         wg.Add(1)
         go func() {
             defer wg.Done()
             // ...
         }()
         // 这里返回后，调用方的 wg 无任何变化
     }
     ```

   - **正确做法**：通过指针传递 `*sync.WaitGroup`。

4. **不应重复使用同一个 `WaitGroup`，除非保证所有 goroutine 已经结束**
    如果多次调用 `Add()`/`Done()`/`Wait()`，务必保证上一次的 `Wait()` 已经返回后，计数器才会归零，才能用于下一轮。否则可能出现竞速、计数器负数等情况。

   - 建议：按“生命周期”新建一个 `WaitGroup`，用完即丢弃，不要混合多组任务到一个 `WaitGroup` 上。

5. **不要在 `Wait()` 返回后继续调用 `Done()`**
    一旦 `Wait()` 返回，说明计数器已归零。如果此时还有其他 goroutine 延迟调 `Done()`，就会导致计数器变为负数，从而 panic。

   - 如果确实需要 “延迟” 在某些异步操作后再 `Done()`，必须在调用 `Wait()` 前保证所有 `Done()` 都会被正确执行。

6. **避免让 `Wait()` 在锁持有期间阻塞**
    如果你在持有某把锁（如 `sync.Mutex`）的情况下调用 `Wait()`，可能会导致其他等待获取该锁的 goroutine 无法执行 `Done()`，从而死锁。

   - 最好将 `Wait()` 放在不持锁的上下文中，让子 goroutine 能够顺利执行 `Done()`。

7. **注意接口嵌入时的传递方式**
    如果你把一个包含 `WaitGroup` 的结构体嵌入到其他地方，然后进行拷贝，就会把 `WaitGroup` 一并拷贝，导致前面提到的“禁止复制”问题。

   - 理想做法：把 `WaitGroup` 作为指针 `*sync.WaitGroup` 放在结构体字段里。
