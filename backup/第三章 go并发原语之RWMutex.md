之前学习过Mutex，互斥锁，它可以保证读写安全的互斥性，无论是读还是写，Mutex都能保证**只有一个**goroutine访问共享资源，但这在某些情况下比较浪费资源，比如读多写少的情况下。在一段时间内没有写操作，但读还是加锁进行的，这在某些方面就不能合理的使用我们的cpu资源。大量并发的读访问也不得不在 Mutex 的保护下变成了串行访问，这个时候，使用 Mutex，对性能的影响就比较大。

那该怎么去解决呢？区分读写操作，就有了我们的RWMutex。

具体可以这样理解，当一个读操作的goroutine持有锁时，其他goroutine不必像之前那样傻傻的等待，而是也可以并发的访问共享变量，**串行读变成并行读**，很大程度提升了读操作的性能。当一个写操作的goroutine持有锁时，它就是一个排他锁，其他所有写操作和读操作的goroutine都会阻塞等待持有这个写操作的gortouine释放锁。

这一类并发读写问题叫作[readers-writers](https://en.wikipedia.org/wiki/Readers%E2%80%93writers_problem) 问题，意思就是，同时可能有多个读或者多个写，但是只要有一个线程在执行写操作，其它的线程都不能执行读写操作。

## RWMutex的相关定义
标准库中的 RWMutex 是一个 reader/writer 互斥锁。RWMutex 在某一时刻只能由**任意数量的 reader 持有**，或者是只被单个的 writer 持有。

readers-writers 问题一般有三类，基于对读和写操作的优先级，读写锁的设计和实现也分成三类。

- 第一类：**读者优先（Readers Preference）**
  只要有读协程 持有锁会一直进行读，阻塞写操作协程，写者可能“饥饿”，永远得不到写锁（如果读协程 持续到来）。
- 第二类：**写者优先（Writers Preference）**
  如果已经有一个 **写协程**在等待请求锁的话，它会阻止新来的请求锁的 **读协程** 获取到锁，所以优先保障 **写协程**。当然，如果有一些 **读协程** 已经**请求了锁**的话，新请求的 **写协程** 也会等待已经存在的 **读协程** 都释放锁之后才能获取。所以，**写优先级设计中的优先权是针对新来的请求而言的**。**这种设计主要避免了 写协程 的饥饿问题**。
- 第三类：**公平策略 / 无优先级（Fair or No Starvation）**
这种设计比较简单，不区分 读协程 和 写协程 优先级，某些场景下这种不指定优先级的设计反而更有效，因为第一类优先级会导致写饥饿，第二类优先级可能会导致读饥饿，这种不指定优先级的访问不再区分读写，大家都是同一个优先级，解决了饥饿的问题。

### RWMutex的方法只有五个
- Lock/Unlock：写操作时调用的方法。如果锁已经被 reader 或者 writer 持有，那么，Lock 方法会一直阻塞，直到能获取到锁；Unlock 则是配对的释放锁的方法。
- RLock/RUnlock：读操作时调用的方法。如果锁已经被 writer 持有的话，RLock 方法会一直阻塞，直到能获取到锁，否则就直接返回；而 RUnlock 是 reader 释放锁的方法。
- RLocker：这个方法的作用是为读操作返回一个 Locker 接口的对象。它的 Lock 方法会调用 RWMutex 的 RLock 方法，它的 Unlock 方法会调用 RWMutex 的 RUnlock 方法。
1. `func (rw *RWMutex) Lock()`

- **用途**：获取**写锁**。
- **行为**：会阻塞，直到没有其他读锁或写锁存在。
- **作用域**：写锁是**独占的**，期间任何读锁或写锁都不能获取。

```go
rw.Lock()
// 临界区：写操作
rw.Unlock()
```

2. `func (rw *RWMutex) Unlock()`

- **用途**：释放之前通过 `Lock()` 获取的写锁。
- **注意**：只能在调用过 `Lock()` 后调用，否则会导致 panic。

3. `func (rw *RWMutex) RLock()`

- **用途**：获取**读锁**。
- **行为**：多个 goroutine 可以同时持有读锁，只要没有写锁被持有。
- **阻塞条件**：如果当前有写锁（正在被写 goroutine 持有），读锁会阻塞等待。

```go
rw.RLock()
// 临界区：只读操作
rw.RUnlock()
```

4. `func (rw *RWMutex) RUnlock()`

- **用途**：释放通过 `RLock()` 获取的读锁。
- **注意**：
  - 每个 `RLock()` 必须对应一次 `RUnlock()`。
  - 多次 `RUnlock()` 会导致 panic。

------

## 🔁 使用示例：

```go
var rw sync.RWMutex
var shared int

// 写者
go func() {
    rw.Lock()
    shared += 1
    rw.Unlock()
}()

// 读者
go func() {
    rw.RLock()
    fmt.Println(shared)
    rw.RUnlock()
}()
```
5. RLocker：这个方法的作用是为读操作返回一个 Locker 接口的对象。它的 Lock 方法会调用 RWMutex 的 RLock 方法，它的 Unlock 方法会调用 RWMutex 的 RUnlock 方法。

另外一个比较完整的例子。
如何使用 RWMutex 保护共享资源。计数器的 count++操作是写操作，而获取 count 的值是读操作，这个场景非常适合读写锁，因为读操作可以并行执行，写操作时只允许一个线程执行，这正是 readers-writers 问题。

在这个例子中，使用 10 个 goroutine 进行读操作，每读取一次，sleep 1 毫秒，同时，还有一个 gorotine 进行写操作，每一秒写一次，这是一个 1 writer-n reader 的读写场景，而且写操作还不是很频繁（一秒一次）：

```go
func main() {
    var counter Counter
    for i := 0; i < 10; i++ { // 10个reader
        go func() {
            for {
                counter.Count() // 计数器读操作
                time.Sleep(time.Millisecond)
            }
        }()
    }

    for { // 一个writer
        counter.Incr() // 计数器写操作
        time.Sleep(time.Second)
    }
}
// 一个线程安全的计数器
type Counter struct {
    mu    sync.RWMutex
    count uint64
}

// 使用写锁保护
func (c *Counter) Incr() {
    c.mu.Lock()
    c.count++
    c.mu.Unlock()
}

// 使用读锁保护
func (c *Counter) Count() uint64 {
    c.mu.RLock()
    defer c.mu.RUnlock()
    return c.count
}
```



可以看到，Incr 方法会修改计数器的值，是一个写操作，我们使用 Lock/Unlock 进行保护。Count 方法会读取当前计数器的值，是一个读操作，我们使用 RLock/RUnlock 方法进行保护。Incr 方法每秒才调用一次，所以，writer 竞争锁的频次是比较低的，而 10 个 goroutine 每毫秒都要执行一次查询，通过读写锁，可以极大提升计数器的的性能，因为在读取的时候，可以并发进行。如果使用 Mutex，性能就不会像读写锁这么好。因为多个 reader 并发读的时候，使用互斥锁导致了 reader 要排队读的情况，没有 RWMutex 并发读的性能好。

**如果你遇到可以明确区分 reader 和 writer goroutine 的场景，且有大量的并发读、少量的并发写，并且有强烈的性能需求，你就可以考虑使用读写锁 RWMutex 替换 Mutex。**

### RWMutex的实现原理
Go 标准库中的 RWMutex 是基于 Mutex 实现的。

看一下RWMutex的相关定义。
```go
type RWMutex struct {
	w           Mutex        // held if there are pending writers // 互斥锁解决多个writer的竞争
	writerSem   uint32       // semaphore for writers to wait for completing readers // writer信号量
	readerSem   uint32       // semaphore for readers to wait for completing writers // reader信号量
	readerCount atomic.Int32 // number of pending readers  // reader的数量
	readerWait  atomic.Int32 // number of departing readers // writer等待完成的reader的数量
}

const rwmutexMaxReaders = 1 << 30
```
简单解释一下这几个字段。

- 字段 w：为 writer 的竞争锁而设计；
- 字段 readerCount：记录当前 reader 的数量（以及是否有 writer 竞争锁）；
- readerWait：记录 writer 请求锁时需要等待 read 完成的 reader 的数量；
- writerSem 和 readerSem：都是为了阻塞设计的信号量。

这里的常量 rwmutexMaxReaders，定义了最大的 reader 数量。

#### RLock/RUnlock 的实现

```go
func (rw *RWMutex) RLock() {
	if race.Enabled {
		race.Read(unsafe.Pointer(&rw.w))
		race.Disable()
	}
	if rw.readerCount.Add(1) < 0 {
        // rw.readerCount是负值的时候，意味着此时有writer等待请求锁，因为writer优先级高，所以把后来的reader阻塞休眠
		// A writer is pending, wait for it.
		runtime_SemacquireRWMutexR(&rw.readerSem, false, 0)
	}
	if race.Enabled {
		race.Enable()
		race.Acquire(unsafe.Pointer(&rw.readerSem))
	}
}
func (rw *RWMutex) RUnlock() {
	if race.Enabled {
		race.Read(unsafe.Pointer(&rw.w))
		race.ReleaseMerge(unsafe.Pointer(&rw.writerSem))
		race.Disable()
	}
	if r := rw.readerCount.Add(-1); r < 0 {
		// Outlined slow-path to allow the fast-path to be inlined
		rw.rUnlockSlow(r)  // 有等待的writer
	}
	if race.Enabled {
		race.Enable()
	}
}

func (rw *RWMutex) rUnlockSlow(r int32) {
	if r+1 == 0 || r+1 == -rwmutexMaxReaders {
		race.Enable()
		fatal("sync: RUnlock of unlocked RWMutex")
	}
	// A writer is pending.
	if rw.readerWait.Add(-1) == 0 {
        // 最后一个reader了，writer终于有机会获得锁了
		// The last reader unblocks the writer.
		runtime_Semrelease(&rw.writerSem, false, 1)
	}
}
```

`if rw.readerCount.Add(1) < 0` 对 reader 计数加 1。可能比较疑惑，readerCount怎么可能是负数？其实readerCount是有两层含义的。

1. 没有 writer 竞争或持有锁时，readerCount 和我们正常理解的 reader 的计数是一样的；
2. 但是，如果有 writer 竞争锁或者持有锁时，那么，readerCount 不仅仅承担着 reader 的计数功能，还能够标识当前是否有 writer 竞争或持有锁，在这种情况下，请求锁的 reader 的处理进入第 4 行，阻塞等待锁的释放。

调用 RUnlock 的时候，我们需要将 Reader 的计数减去 1 `if r := rw.readerCount.Add(-1); r < 0` ，因为 reader 的数量减少了一个。但是，rw.readerCount.Add 的返回值还有另外一个含义。如果它是负值，就表示当前有 writer 竞争锁，在这种情况下，还会调用 rUnlockSlow 方法，检查是不是 reader 都释放读锁了，如果读锁都释放了，那么可以唤醒请求写锁的 writer 了。

当一个或者多个 reader 持有锁的时候，竞争锁的 writer 会等待这些 reader 释放完，才可能持有这把锁。当 writer 请求锁的时候，是无法改变既有的 reader 持有锁的现实的，也不会强制这些 reader 释放锁，它的优先权只是限定后来的 reader 不要和它抢。

#### Lock/Unlock

```go
func (rw *RWMutex) Lock() {
	if race.Enabled {
		race.Read(unsafe.Pointer(&rw.w))
		race.Disable()
	}
	// First, resolve competition with other writers. 首先解决其他writer竞争问题
	rw.w.Lock()
	// Announce to readers there is a pending writer. 反转readerCount，告诉reader有writer竞争锁
	r := rw.readerCount.Add(-rwmutexMaxReaders) + rwmutexMaxReaders
	// Wait for active readers.如果当前有reader持有锁，那么需要等待
	if r != 0 && rw.readerWait.Add(r) != 0 {
		runtime_SemacquireRWMutex(&rw.writerSem, false, 0)
	}
	if race.Enabled {
		race.Enable()
		race.Acquire(unsafe.Pointer(&rw.readerSem))
		race.Acquire(unsafe.Pointer(&rw.writerSem))
	}
}
func (rw *RWMutex) Unlock() {
	if race.Enabled {
		race.Read(unsafe.Pointer(&rw.w))
		race.Release(unsafe.Pointer(&rw.readerSem))
		race.Disable()
	}

	// Announce to readers there is no active writer. 告诉reader没有活跃的writer了
	r := rw.readerCount.Add(rwmutexMaxReaders)
	if r >= rwmutexMaxReaders {
		race.Enable()
		fatal("sync: Unlock of unlocked RWMutex")
	}
	// Unblock blocked readers, if any. 唤醒阻塞的reader们
	for i := 0; i < int(r); i++ {
		runtime_Semrelease(&rw.readerSem, false, 0)
	}
	// Allow other writers to proceed. 释放内部的互斥锁
	rw.w.Unlock()
	if race.Enabled {
		race.Enable()
	}
}
```

如果 readerCount 不是 0，就说明当前有持有读锁的 reader，RWMutex 需要把这个当前 readerCount 赋值给 readerWait 字段保存下来 `if r != 0 && rw.readerWait.Add(r) != 0）`， 

同时，这个 writer 进入阻塞等待状`runtime_SemacquireRWMutex(&rw.writerSem, false, 0)`

每当一个 reader 释放读锁的时候（调用 RUnlock 方法时），readerWait 字段就减 1，直到所有的活跃的 reader 都释放了读锁，才会唤醒这个 writer。

当一个 writer 释放锁的时候，它会再次反转 readerCount 字段。可以肯定的是，因为当前锁由 writer 持有，所以，readerCount 字段是反转过的，并且减去了 rwmutexMaxReaders 这个常数，变成了负数。所以，这里的反转方法就是给它增加 rwmutexMaxReaders 这个常数值。

### RWMutex的三个踩坑点

1. 不可复制

和 Mutex 一样，RWMutex 也是不可复制。不能复制的原因和互斥锁一样。一旦读写锁被使用，它的字段就会记录它当前的一些状态。这个时候你去复制这把锁，就会把它的状态也给复制过来。但是，原来的锁在释放的时候，并不会修改你复制出来的这个读写锁，这就会导致复制出来的读写锁的状态不对，可能永远无法释放锁。

2. 不可重入

不可重入的原因是，获得锁之后，还没释放锁，又申请锁，这样有可能造成死锁。比如 reader A 获取到了读锁，writer B 等待 reader A 释放锁，reader 还没释放锁又申请了一把锁，但是这把锁申请不成功，他需要等待 writer B。这就形成了一个循环等待的死锁。

3. 加锁和释放锁一定要成对出现，不能忘记释放锁，也不能解锁一个未加锁的锁。