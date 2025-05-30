## map的基本使用方法

1. **声明与初始化**

```go
// 声明一个未初始化的map（nil）
var m1 map[string]int

// 使用make初始化
m2 := make(map[string]int) // 空map
m3 := make(map[string]int, 10) // 指定初始容量

// 字面量初始化
m4 := map[string]int{
    "apple":  5,
    "banana": 3,
}
```

2. **操作键值对**

```go
// 添加/修改
m := make(map[string]int)
m["apple"] = 10

// 获取值
value := m["apple"]   // 10
v, exists := m["pear"] // v=0, exists=false

// 删除键
delete(m, "apple")
```

3. **遍历map**

```go
for key, value := range m {
    fmt.Println(key, value) // 顺序随机
}
```

4. **获取长度**

```go
n := len(m) // 键值对数量
```

------

### **注意事项**

1. **未初始化map为nil**

- nil map无法写入：对`nil`map赋值会导致`panic`

  ```go
  var m map[string]int
  m["key"] = 1 // panic: assignment to nil map
  ```

- **解决方案**：用`make`或字面量初始化。

2. **检测键是否存在**

- 使用双返回值模式避免零值混淆：

  ```go
  if v, ok := m["key"]; ok {
      // 键存在，v是对应值
  } else {
      // 键不存在
  }
  ```

3. **map是引用类型**

- 赋值或传参时传递引用（底层同一数据结构）：

  ```go
  m1 := map[string]int{"a": 1}
  m2 := m1
  m2["a"] = 2
  fmt.Println(m1["a"]) // 2 （同步修改）
  ```

4. **键的限制**

- 键必须支持`==`操作（可比较类型）：
  - **可用**：基本类型（如`int`、`string`）、指针、结构体（字段均为可比较类型）、数组。
  - **不可用**：切片、函数、包含切片的其他类型（编译错误）。

5. **并发访问不安全**

- 并发读写会panic：

  ```go
  // 错误示例！
  go write(m) // 写操作
  go read(m)   // 读操作
  ```

- 解决方案：用`sync.RWMutex`或`sync.Map`（Go 1.9+）：

  ```go
  var mu sync.RWMutex
  mu.Lock()
  m["key"] = 1 // 写加锁
  mu.Unlock()
  
  mu.RLock()
  v := m["key"] // 读加锁
  mu.RUnlock()
  ```

6. **遍历顺序随机**

- 每次`range`遍历顺序可能不同（设计如此，避免依赖顺序）。

7. **删除键不会缩容**

- `delete`删除键后，内存占用可能不会立即释放。若需释放大map，需重建或替换为新map。

8. **值为nil时操作**

- 若值为`nil`（如`map[int][]int`），需先初始化切片再操作：

  ```go
  m := make(map[int][]int)
  // m[1][0] = 1 // panic: 索引nil切片
  
  // 正确写法
  if m[1] == nil {
      m[1] = make([]int, 0)
  }
  m[1] = append(m[1], 100)
  ```

**代码示例**

```go
package main

import (
    "fmt"
    "sync"
)

func main() {
    // 初始化
    m := map[string]int{
        "a": 1,
        "b": 2,
    }

    // 安全读写
    var mu sync.RWMutex
    mu.Lock()
    m["c"] = 3
    mu.Unlock()

    // 检测键
    if v, ok := m["d"]; !ok {
        fmt.Println("d not exist") // 输出
    }

    // 并发安全遍历
    mu.RLock()
    for k, v := range m {
        fmt.Printf("%s:%d ", k, v) // 输出顺序随机
    }
    mu.RUnlock()
}
```

**总结**

- **初始化**：避免nil map。

- **键存在判断**：用双返回值。

- **并发**：用互斥锁或`sync.Map`。

- **键类型**：必须可比较。

- **内存管理**：大map删除键后可能需重建释放内存。
### 分片加锁：更高效的并发 map

虽然使用读写锁可以提供线程安全的 map，但是在大量并发读写的情况下，锁的竞争会非常激烈。锁是性能下降的万恶之源之一。

在并发编程中，我们的一条原则就是尽量减少锁的使用。一些单线程单进程的应用（比如 Redis 等），基本上不需要使用锁去解决并发线程访问的问题，所以可以取得很高的性能。但是对于 Go 开发的应用程序来说，并发是常用的一个特性，在这种情况下，我们能做的就是，**尽量减少锁的粒度和锁的持有时间**。

减少锁的粒度常用的方法就是分片（Shard），将一把锁分成几把锁，每个锁控制一个分片。Go 比较知名的分片并发 map 的实现是[orcaman/concurrent-map: a thread-safe concurrent map for go](https://github.com/orcaman/concurrent-map)。

下面是一些源码的阅读：

```go
var SHARD_COUNT = 32

type Stringer interface {
	fmt.Stringer
	comparable
}

// A "thread" safe map of type string:Anything. 分成SHARD_COUNT个分片的map
// To avoid lock bottlenecks this map is dived to several (SHARD_COUNT) map shards.
type ConcurrentMap[K comparable, V any] struct {
	shards   []*ConcurrentMapShared[K, V]
	sharding func(key K) uint32
}

// A "thread" safe string to anything map. 通过RWMutex保护的线程安全的分片，包含一个map
type ConcurrentMapShared[K comparable, V any] struct {
	items        map[K]V
	sync.RWMutex // Read Write mutex, guards access to internal map.
}

func create[K comparable, V any](sharding func(key K) uint32) ConcurrentMap[K, V] {
	m := ConcurrentMap[K, V]{
		sharding: sharding,
		shards:   make([]*ConcurrentMapShared[K, V], SHARD_COUNT),
	}
	for i := 0; i < SHARD_COUNT; i++ {
		m.shards[i] = &ConcurrentMapShared[K, V]{items: make(map[K]V)}
	}
	return m
}

// Creates a new concurrent map. 创建并发map
func New[V any]() ConcurrentMap[string, V] {
	return create[string, V](fnv32)
}

// Creates a new concurrent map.
func NewStringer[K Stringer, V any]() ConcurrentMap[K, V] {
	return create[K, V](strfnv32[K])
}

// Creates a new concurrent map.
func NewWithCustomShardingFunction[K comparable, V any](sharding func(key K) uint32) ConcurrentMap[K, V] {
	return create[K, V](sharding)
}

// GetShard returns shard under given key  根据key计算分片索引
func (m ConcurrentMap[K, V]) GetShard(key K) *ConcurrentMapShared[K, V] {
	return m.shards[uint(m.sharding(key))%uint(SHARD_COUNT)]
}
```

增加或者查询时，首先根据分片索引得到分片对象，然后对分片对象加锁进行操作：

```go
// Sets the given value under the specified key.
func (m ConcurrentMap[K, V]) Set(key K, value V) {
	// Get map shard. 根据key计算出对应的分片
	shard := m.GetShard(key)
	shard.Lock()
	shard.items[key] = value
	shard.Unlock()
}

// Callback to return new element to be inserted into the map
// It is called while lock is held, therefore it MUST NOT
// try to access other keys in same map, as it can lead to deadlock since
// Go sync.RWLock is not reentrant
type UpsertCb[V any] func(exist bool, valueInMap V, newValue V) V

// Insert or Update - updates existing element or inserts a new one using UpsertCb
func (m ConcurrentMap[K, V]) Upsert(key K, value V, cb UpsertCb[V]) (res V) {
	shard := m.GetShard(key)
	shard.Lock()
	v, ok := shard.items[key]
	res = cb(ok, v, value)
	shard.items[key] = res
	shard.Unlock()
	return res
}

// Sets the given value under the specified key if no value was associated with it.
func (m ConcurrentMap[K, V]) SetIfAbsent(key K, value V) bool {
	// Get map shard.
	shard := m.GetShard(key)
	shard.Lock()
	_, ok := shard.items[key]
	if !ok {
		shard.items[key] = value
	}
	shard.Unlock()
	return !ok
}

// Get retrieves an element from map under given key.
func (m ConcurrentMap[K, V]) Get(key K) (V, bool) {
	// Get shard  根据key计算出对应的分片
	shard := m.GetShard(key)
	shard.RLock()
	// Get item from shard. 从这个分片读取key的值
	val, ok := shard.items[key]
	shard.RUnlock()
	return val, ok
}

```

## 特殊场景使用sync.Map

在以下两个场景中使用 sync.Map，会比使用 map+RWMutex 的方式，性能要好得多：

1. 只会增长的缓存系统中，一个 key 只写入一次而被读很多次；
2. 多个 goroutine 为不相交的键集读、写和重写键值对。

这两个场景说得都比较笼统，而且，这些场景中还包含了一些特殊的情况。所以，官方建议你针对自己的场景做性能评测，如果确实能够显著提高性能，再使用 sync.Map。

能用到 sync.Map 的场景确实不多。即使是 sync.Map 的作者 Bryan C. Mills，也很少使用 sync.Map，即便是在使用 sync.Map 的时候，也是需要临时查询它的 API，才能清楚记住它的功能。所以，我们可以把 sync.Map 看成一个生产环境中很少使用的同步原语。

### sync.Map的实现

其实 sync.Map 的实现有几个优化点，这里先列出来，我们后面慢慢分析。

- 空间换时间。通过冗余的两个数据结构（只读的 read 字段、可写的 dirty），来减少加锁对性能的影响。对只读字段（read）的操作不需要加锁。
- 优先从 read 字段读取、更新、删除，因为对 read 字段的读取不需要锁。
- 动态调整。miss 次数多了之后，将 dirty 数据提升为 read，避免总是从 dirty 中加锁读取。
- double-checking。加锁之后先还要再检查 read 字段，确定真的不存在才操作 dirty 字段。
- 延迟删除。删除一个键值只是打标记，只有在提升 dirty 字段为 read 字段的时候才清理删除的数据。

来看一下sync.Map的设计和实现，学习其处理的方式。

相关源码：

```go
type Map struct {
	_ noCopy

	mu Mutex

	// read contains the portion of the map's contents that are safe for
	// concurrent access (with or without mu held).
	//
	// The read field itself is always safe to load, but must only be stored with
	// mu held.
	//
	// Entries stored in read may be updated concurrently without mu, but updating
	// a previously-expunged entry requires that the entry be copied to the dirty
	// map and unexpunged with mu held.
    // 基本上你可以把它看成一个安全的只读的map 
    // 它包含的元素其实也是通过原子操作更新的，但是已删除的entry就需要加锁操作了
	read atomic.Pointer[readOnly]

	// dirty contains the portion of the map's contents that require mu to be
	// held. To ensure that the dirty map can be promoted to the read map quickly,
	// it also includes all of the non-expunged entries in the read map.
	//
	// Expunged entries are not stored in the dirty map. An expunged entry in the
	// clean map must be unexpunged and added to the dirty map before a new value
	// can be stored to it.
	//
	// If the dirty map is nil, the next write to the map will initialize it by
	// making a shallow copy of the clean map, omitting stale entries.
    // 包含需要加锁才能访问的元素 
    // 包括所有在read字段中但未被删除 的元素以及新加的元素
	dirty map[any]*entry

	// misses counts the number of loads since the read map was last updated that
	// needed to lock mu to determine whether the key was present.
	//
	// Once enough misses have occurred to cover the cost of copying the dirty
	// map, the dirty map will be promoted to the read map (in the unamended
	// state) and the next store to the map will make a new dirty copy.
    //// 记录从read中读取miss的次数，一旦miss数和dirty长度一样了，就会把dirty提升为read，并把dirty置空
	misses int
}

// readOnly is an immutable struct stored atomically in the Map.read field.
type readOnly struct {
	m       map[any]*entry
	amended bool // true if the dirty map contains some key not in m. 当dirty中包含read没有的数据时为true，比如新增一条数据
}

// expunged is an arbitrary pointer that marks entries which have been deleted
// from the dirty map.
// expunged是用来标识此项已经删掉的指针
// 当map中的一个项目被删除了，只是把它的值标记为expunged，以后才有机会真正删除此项
var expunged = new(any)

// An entry is a slot in the map corresponding to a particular key. entry代表一个值
type entry struct {
	// p points to the interface{} value stored for the entry.
	//
	// If p == nil, the entry has been deleted, and either m.dirty == nil or
	// m.dirty[key] is e.
	//
	// If p == expunged, the entry has been deleted, m.dirty != nil, and the entry
	// is missing from m.dirty.
	//
	// Otherwise, the entry is valid and recorded in m.read.m[key] and, if m.dirty
	// != nil, in m.dirty[key].
	//
	// An entry can be deleted by atomic replacement with nil: when m.dirty is
	// next created, it will atomically replace nil with expunged and leave
	// m.dirty[key] unset.
	//
	// An entry's associated value can be updated by atomic replacement, provided
	// p != expunged. If p == expunged, an entry's associated value can be updated
	// only after first setting m.dirty[key] = e so that lookups using the dirty
	// map find the entry.
	p atomic.Pointer[any]
}
```

如果 dirty 字段非 nil 的话，map 的 read 字段和 dirty 字段会包含相同的非 expunged 的项，所以如果通过 read 字段更改了这个项的值，从 dirty 字段中也会读取到这个项的新值，因为本来它们指向的就是同一个地址。

dirty 包含重复项目的好处就是，一旦 miss 数达到阈值需要将 dirty 提升为 read 的话，只需简单地把 dirty 设置为 read 对象即可。不好的一点就是，当创建新的 dirty 对象的时候，需要逐条遍历 read，把非 expunged 的项复制到 dirty 对象中。

### Store方法

```go
// Store sets the value for a key.
func (m *Map) Store(key, value any) {
	_, _ = m.Swap(key, value)
}
// Swap swaps the value for a key and returns the previous value if any.
// The loaded result reports whether the key was present.
func (m *Map) Swap(key, value any) (previous any, loaded bool) {
	read := m.loadReadOnly()
    // 如果read字段包含这个项，说明是更新，cas更新项目的值即可
	if e, ok := read.m[key]; ok {
		if v, ok := e.trySwap(&value); ok {
			if v == nil {
				return nil, false
			}
			return *v, true
		}
	}
	// read中不存在，或者cas更新失败，就需要加锁访问dirty了
	m.mu.Lock()
	read = m.loadReadOnly()
	if e, ok := read.m[key]; ok { // 双检查，看看read是否已经存在了
		if e.unexpungeLocked() {
			// The entry was previously expunged, which implies that there is a
			// non-nil dirty map and this entry is not in it.
            // 此项目先前已经被删除了，通过将它的值设置为nil，标记为unexpunged
			m.dirty[key] = e // 更新
		}
		if v := e.swapLocked(&value); v != nil {
			loaded = true
			previous = *v
		}
	} else if e, ok := m.dirty[key]; ok { // 如果dirty中有此项
		if v := e.swapLocked(&value); v != nil {
			loaded = true
			previous = *v
		}
	} else {
		if !read.amended {  //如果dirty为nil
			// We're adding the first new key to the dirty map.
			// Make sure it is allocated and mark the read-only map as incomplete.
            // 需要创建dirty对象，并且标记read的amended为true
            // 说明有元素它不包含而dirty包含
			m.dirtyLocked()
			m.read.Store(&readOnly{m: read.m, amended: true})
		}
		m.dirty[key] = newEntry(value) //将新值增加到dirty对象中
	}
	m.mu.Unlock()
	return previous, loaded
}
```

Store 既可以是新增元素，也可以是更新元素。如果运气好的话，更新的是已存在的未被删除的元素，直接更新即可，不会用到锁。如果运气不好，需要更新（重用）删除的对象、更新还未提升的 dirty 中的对象，或者新增加元素的时候就会使用到了锁，这个时候，性能就会下降。

所以从这一点来看，sync.Map 适合那些只会增长的缓存系统，可以进行更新，但是不要删除，并且不要频繁地增加新元素。

### Load方法

```go
func (m *Map) Load(key any) (value any, ok bool) {
    // 首先从read处理
	read := m.loadReadOnly()
	e, ok := read.m[key]
	if !ok && read.amended {  // 如果不存在并且dirty不为nil(有新的元素)
		m.mu.Lock()
		// Avoid reporting a spurious miss if m.dirty got promoted while we were
		// blocked on m.mu. (If further loads of the same key will not miss, it's
		// not worth copying the dirty map for this key.)
        // 双检查，看看read中现在是否存在此key
		read = m.loadReadOnly()
		e, ok = read.m[key]
		if !ok && read.amended { //依然不存在，并且dirty不为nil
			e, ok = m.dirty[key] // 从dirty中读取
			// Regardless of whether the entry was present, record a miss: this key
			// will take the slow path until the dirty map is promoted to the read
			// map. // 不管dirty中存不存在，miss数都加1
			m.missLocked()
		}
		m.mu.Unlock()
	}
	if !ok {
		return nil, false
	}
	return e.load() //返回读取的对象，e既可能是从read中获得的，也可能是从dirty中获得的
}
```

如果幸运的话，我们从 read 中读取到了这个 key 对应的值，那么就不需要加锁了，性能会非常好。但是，如果请求的 key 不存在或者是新加的，就需要加锁从 dirty 中读取。所以，读取不存在的 key 会因为加锁而导致性能下降，读取还没有提升的新值的情况下也会因为加锁性能下降。



其中，missLocked 增加 miss 的时候，如果 miss 数等于 dirty 长度，会将 dirty 提升为 read，并将 dirty 置空。

```go
func (m *Map) missLocked() {
	m.misses++ // misses计数加一
	if m.misses < len(m.dirty) { // 如果没达到阈值(dirty字段的长度),返回
		return
	}
	m.read.Store(&readOnly{m: m.dirty}) /把dirty字段的内存提升为read字段
	m.dirty = nil // 清空dirty
	m.misses = 0 // misses数重置为0
}
```

### Delete方法

```go
// Delete deletes the value for a key.
func (m *Map) Delete(key any) {
	m.LoadAndDelete(key)
}


func (m *Map) LoadAndDelete(key any) (value any, loaded bool) {
	read := m.loadReadOnly()
	e, ok := read.m[key]
	if !ok && read.amended {
		m.mu.Lock()
        // 双检查
		read = m.loadReadOnly()
		e, ok = read.m[key]
		if !ok && read.amended {
			e, ok = m.dirty[key]
			delete(m.dirty, key)
			// Regardless of whether the entry was present, record a miss: this key
			// will take the slow path until the dirty map is promoted to the read
			// map.
            // miss数加1
			m.missLocked()
		}
		m.mu.Unlock()
	}
	if ok {
		return e.delete()
	}
	return nil, false
}

func (e *entry) delete() (value any, ok bool) {
	for {
		p := e.p.Load()
		if p == nil || p == expunged {
			return nil, false
		}
		if e.p.CompareAndSwap(p, nil) {
			return *p, true
		}
	}
}
```

如果 read 中不存在，那么就需要从 dirty 中寻找这个项目。最终，如果项目存在就删除（将它的值标记为 nil）。如果项目不为 nil 或者没有被标记为 expunged，那么还可以把它的值返回。

sync.map 还有一些 LoadAndDelete、LoadOrStore、Range 等辅助方法，但是没有 Len 这样查询 sync.Map 的包含项目数量的方法，并且官方也不准备提供。如果你想得到 sync.Map 的项目数量的话，你可能不得不通过 Range 逐个计数。