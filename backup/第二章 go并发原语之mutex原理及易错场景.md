mutex互斥锁经历过很多版本，我们只讨论最新的版本。当前1.24.2的版本。
## 相关源码
之前的版本比较简单实现，当前的的版本代码几乎不可读了，相对来说复杂了很多，但其更加高效，<span style = "color:#2291b3">新来的 goroutine 也参与竞争，有可能每次都会被新来的 goroutine 抢到获取锁的机会，在极端情况下，等待中的 goroutine 可能会一直获取不到锁，这就是饥饿问题</span>。当前版本已经解决了这个最大的问题。
2016 年 Go 1.9 中 Mutex 增加了饥饿模式，让锁变得更公平，不公平的等待时间限制在 1 毫秒，并且修复了一个大 Bug：总是把唤醒的 goroutine 放在等待队列的尾部，会导致更加不公平的等待时间。其实现在的mutex源码和之前变化不是很大了。只是增加了一些特性，但基本还是那样。
**Mutex 绝不容忍一个 goroutine 被落下，永远没有机会获取锁。不抛弃不放弃是它的宗旨，而且它也尽可能地让等待较长的 goroutine 更有机会获取到锁。**
相关代码
```go
type Mutex struct {
	state int32
	sema  uint32
}

const (
	mutexLocked = 1 << iota // mutex is locked
	mutexWoken
	mutexStarving // 从state字段中分出一个饥饿标记
	mutexWaiterShift = iota

	starvationThresholdNs = 1e6
)

// Lock locks m.
//
// See package [sync.Mutex] documentation.
func (m *Mutex) Lock() {
        // Fast path: 幸运之路，一下就获取到了锁
	// Fast path: grab unlocked mutex.
	if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
		if race.Enabled {
			race.Acquire(unsafe.Pointer(m))
		}
		return
	}
        // Slow path：缓慢之路，尝试自旋竞争或饥饿状态下饥饿goroutine竞争
	// Slow path (outlined so that the fast path can be inlined)
	m.lockSlow()
}
func (m *Mutex) lockSlow() {
	var waitStartTime int64
	starving := false  // 此goroutine的饥饿标记
	awoke := false // 唤醒标记
	iter := 0 // 自旋次数
	old := m.state  // 当前的锁的状态
	for {
                //  锁是非饥饿状态，锁还没被释放，尝试自旋
		// Don't spin in starvation mode, ownership is handed off to waiters
		// so we won't be able to acquire the mutex anyway.
		if old&(mutexLocked|mutexStarving) == mutexLocked && runtime_canSpin(iter) {
			// Active spinning makes sense.
			// Try to set mutexWoken flag to inform Unlock
			// to not wake other blocked goroutines.
			if !awoke && old&mutexWoken == 0 && old>>mutexWaiterShift != 0 &&
				atomic.CompareAndSwapInt32(&m.state, old, old|mutexWoken) {
				awoke = true
			}
			runtime_doSpin()
			iter++
			old = m.state // 再次获取锁的状态，之后会检查是否锁被释放了
			continue
		}
		new := old
		// Don't try to acquire starving mutex, new arriving goroutines must queue.
		if old&mutexStarving == 0 {
			new |= mutexLocked  // 非饥饿状态，加锁
		}
		if old&(mutexLocked|mutexStarving) != 0 {
			new += 1 << mutexWaiterShift // waiter数量加1
		}
		// The current goroutine switches mutex to starvation mode.
		// But if the mutex is currently unlocked, don't do the switch.
		// Unlock expects that starving mutex has waiters, which will not
		// be true in this case.
		if starving && old&mutexLocked != 0 {
			new |= mutexStarving  // 设置饥饿状态
		}
		if awoke {
			// The goroutine has been woken from sleep,
			// so we need to reset the flag in either case.
			if new&mutexWoken == 0 {
				throw("sync: inconsistent mutex state")
			}
			new &^= mutexWoken  // 新状态清除唤醒标记
		}
               // 成功设置新状态
		if atomic.CompareAndSwapInt32(&m.state, old, new) {
                        // // 原来锁的状态已释放，并且不是饥饿状态，正常请求到了锁，返回
			if old&(mutexLocked|mutexStarving) == 0 {
				break // locked the mutex with CAS
			}
                        // 处理饥饿状态 
                        // 如果以前就在队列里面，加入到队列头
			// If we were already waiting before, queue at the front of the queue.
			queueLifo := waitStartTime != 0
			if waitStartTime == 0 {
				waitStartTime = runtime_nanotime()
			}
                       // 阻塞等待
			runtime_SemacquireMutex(&m.sema, queueLifo, 2)
                        // 唤醒之后检查锁是否应该处于饥饿状态
			starving = starving || runtime_nanotime()-waitStartTime > starvationThresholdNs
			old = m.state
                        // 如果锁已经处于饥饿状态，直接抢到锁，返回
			if old&mutexStarving != 0 {
				// If this goroutine was woken and mutex is in starvation mode,
				// ownership was handed off to us but mutex is in somewhat
				// inconsistent state: mutexLocked is not set and we are still
				// accounted as waiter. Fix that.
				if old&(mutexLocked|mutexWoken) != 0 || old>>mutexWaiterShift == 0 {
					throw("sync: inconsistent mutex state")
				}
                               // 加锁并且将waiter数减1
				delta := int32(mutexLocked - 1<<mutexWaiterShift)
				if !starving || old>>mutexWaiterShift == 1 {
					// Exit starvation mode.
					// Critical to do it here and consider wait time.
					// Starvation mode is so inefficient, that two goroutines
					// can go lock-step infinitely once they switch mutex
					// to starvation mode.
					delta -= mutexStarving  // 最后一个waiter或者已经不饥饿了，清除饥饿标记
				}
				atomic.AddInt32(&m.state, delta)
				break
			}
			awoke = true
			iter = 0
		} else {
			old = m.state
		}
	}

	if race.Enabled {
		race.Acquire(unsafe.Pointer(m))
	}
}

// Unlock unlocks m.
//
// See package [sync.Mutex] documentation.
func (m *Mutex) Unlock() {
	if race.Enabled {
		_ = m.state
		race.Release(unsafe.Pointer(m))
	}

	// Fast path: drop lock bit.
	new := atomic.AddInt32(&m.state, -mutexLocked)
	if new != 0 {
		// Outlined slow path to allow inlining the fast path.
		// To hide unlockSlow during tracing we skip one extra frame when tracing GoUnblock.
		m.unlockSlow(new)
	}
}

func (m *Mutex) unlockSlow(new int32) {
	if (new+mutexLocked)&mutexLocked == 0 {
		fatal("sync: unlock of unlocked mutex")
	}
	if new&mutexStarving == 0 {
		old := new
		for {
			// If there are no waiters or a goroutine has already
			// been woken or grabbed the lock, no need to wake anyone.
			// In starvation mode ownership is directly handed off from unlocking
			// goroutine to the next waiter. We are not part of this chain,
			// since we did not observe mutexStarving when we unlocked the mutex above.
			// So get off the way.
			if old>>mutexWaiterShift == 0 || old&(mutexLocked|mutexWoken|mutexStarving) != 0 {
				return
			}
			// Grab the right to wake someone.
			new = (old - 1<<mutexWaiterShift) | mutexWoken
			if atomic.CompareAndSwapInt32(&m.state, old, new) {
				runtime_Semrelease(&m.sema, false, 2)
				return
			}
			old = m.state
		}
	} else {
		// Starving mode: handoff mutex ownership to the next waiter, and yield
		// our time slice so that the next waiter can start to run immediately.
		// Note: mutexLocked is not set, the waiter will set it after wakeup.
		// But mutex is still considered locked if mutexStarving is set,
		// so new coming goroutines won't acquire it.
		runtime_Semrelease(&m.sema, true, 2)
	}
}
```
通过加入饥饿模式，可以避免把机会全都留给新来的 goroutine，保证了请求锁的 goroutine 获取锁的公平性，对于我们使用锁的业务代码来说，不会有业务一直等待锁不被处理。仔细阅读源码可以得知其设计的巧妙，但这个源码不可读性太强。

<img width="768" alt="Image" src="https://github.com/user-attachments/assets/ac6075fc-230e-49d9-a08f-2dba65b4408c" />

## 使用互斥锁的易错场景
### Lock/Unlock 没有成对出现
Lock/Unlock 没有成对出现，就意味着会出现死锁的情况，或者是因为 Unlock 一个未加锁的 Mutex 而导致 panic。
缺少 Unlock 的场景，常见的有三种情况：
1. 代码中有太多的 if-else 分支，可能在某个分支中漏写了 Unlock
2. 在重构的时候把 Unlock 给删除了
3. Unlock 误写成了 Lock
### 重入
当一个线程获取锁时，如果没有其它线程拥有这个锁，那么，这个线程就成功获取到这个锁。之后，如果其它线程再请求这个锁，就会处于阻塞等待的状态。但是，如果拥有这把锁的线程再请求这把锁的话，不会阻塞，而是成功返回，所以叫可重入锁（有时候也叫做递归锁）。只要你拥有这把锁，你可以可着劲儿地调用，比如通过递归实现一些算法，调用者不会阻塞或者死锁。

go的mutex**不是可重入锁**。mutex当中并没有那些字段记录哪个goroutine 拥有这把锁，所有无法做到重入锁的。
如下面的代码：
```go
func foo(l sync.Locker) {
    fmt.Println("in foo")
    l.Lock()
    bar(l)
    l.Unlock()
}


func bar(l sync.Locker) {
    l.Lock()
    fmt.Println("in bar")
    l.Unlock()
}


func main() {
    l := &sync.Mutex{}
    foo(l)
}
```
实现一个可重入锁并：
代码如下，其原理就是标识持有锁的goroutine ，以达到可重入的目的：
```go
package main

import (
	"fmt"
	"sync"
	"sync/atomic"
)

type TokenRecursiveMutex struct {
	sync.Mutex
	token     int64 // 当前持有锁的token
	recursion int32 // 重入的次数
}

// Lock 请求锁需要传入token
func (m *TokenRecursiveMutex) Lock(token int64) {
	// 如果传入的token和持有锁的token一致，此时递归加锁，重入次数+1
	if atomic.LoadInt64(&m.token) == token {
		m.recursion++
		return
	}

	m.Mutex.Lock()
	// 上面的条件不满足，说明不是递归调用，抢到新锁后记录此锁拥有者token值
	atomic.StoreInt64(&m.token, token)
	m.recursion = 1
}

// Unlock 释放锁
func (m *TokenRecursiveMutex) Unlock(token int64) {
	// 如果传入token 和持有锁的token不一致，不能解锁，直接运行恐慌
	if atomic.LoadInt64(&m.token) != token {
		panic(fmt.Sprintf("wrong the owner(%d):%d", m.token, token))
	}

	// 否则的话把当前此有锁的token重入次数减去1
	m.recursion--

	// 如果发现当前持有锁的token重入次数不等于0，说明还有其他重入锁未解锁
	if m.recursion != 0 {
		return
	}
	// 没有其他重入锁了，释放锁 之前清楚当前锁的持有者token
	atomic.StoreInt64(&m.token, 0)
	// 解锁
	m.Mutex.Unlock()
}

func main() {
	r := &TokenRecursiveMutex{}
	StartTokenLayer(r, 1024)
	recursiveFunc(r, 10, 1025)
}

func recursiveFunc(lock *TokenRecursiveMutex, depth int, token int64) {
	lock.Lock(token)
	defer lock.Unlock(token)
	if depth > 0 {
		fmt.Printf("depth %d\n", depth)
		recursiveFunc(lock, depth-1, token) // 递归调用需要再次获取同一把锁
	}
}
func StartTokenLayer(r *TokenRecursiveMutex, token int64) {
	r.Lock(token)
	fmt.Println("开始")
	TwoTokenLayer(r, token)
	r.Unlock(token)
}

func TwoTokenLayer(r *TokenRecursiveMutex, token int64) {
	r.Lock(token)
	fmt.Println("进入第二层")
	ThreeTokenLayer(r, token)
	r.Unlock(token)
}

func ThreeTokenLayer(r *TokenRecursiveMutex, token int64) {
	r.Lock(token)
	fmt.Println("最后一层")
	r.Unlock(token)
}
```
### 死锁
死锁（Deadlock）​​ 是指两个或多个 goroutine 因互相等待对方释放资源而永久阻塞的状态。Go 的运行时（runtime）会自动检测到这种情况并抛出 fatal error: all goroutines are asleep - deadlock! 错误。
1. **资源互斥**：
   - 共享资源（如锁、通道）被独占，其他 goroutine 无法访问。
2. **持有并等待**：
   - Goroutine 持有资源的同时等待其他资源。
3. **不可抢占**：
   - 资源只能由持有者释放，无法强制剥夺。
4. **循环等待**：
   - Goroutine 之间形成环形依赖链。 