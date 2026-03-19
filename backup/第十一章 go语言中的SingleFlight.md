### SingleFlight是什么

在go语言当中是一个并发去重工具，当多个goroutine同一时刻请求”同一个key对应操作时“，保证只让**其中一个执行**，其他请求等待其中一个出结构后，复用改次请求的结果。这样可以减少并发处理的数量。

主要用于合并并发请求的场景中，比如缓存场景的缓存击穿/热点key等。

它是go语言开发组提供的一个扩展并发原语，其需要 go get golang.org/x/sync/singleflight

### SingleFlight对外api

Go 里最常用的是这个包：

```go
import "golang.org/x/sync/singleflight"
```

它最核心的类型是 `Group`，核心方法有 3 个：

- `Do(key, fn)`：同步执行，同 key 并发请求只执行一次
- `DoChan(key, fn)`：异步方式拿结果
- `Forget(key)`：忘掉某个 key，让后续请求不再复用当前正在进行中的调用

官方签名大致如下：

```go
func (g *Group) Do(key string, fn func() (any, error)) (v any, err error, shared bool)
func (g *Group) DoChan(key string, fn func() (any, error)) <-chan Result
func (g *Group) Forget(key string)
```

其中 `shared` 很关键：
 如果返回 `shared == true`，表示**这次结果被多个调用共享了**；如果是 `false`，通常表示这次是“真正执行者”或者没有发生并发复用。

### 使用举例

- ex1

  ```go
  package main
  
  import (
  	"fmt"
  	"golang.org/x/sync/singleflight"
  	"sync"
  	"time"
  )
  
  var g singleflight.Group
  
  func main() {
  	var wg sync.WaitGroup
  
  	for i := 0; i < 5; i++ {
  		wg.Add(1)
  		go func(i int) {
  			defer wg.Done()
  
  			v, _, shared := g.Do("user:1001", func() (interface{}, error) {
  				fmt.Println("query DB...")
  				time.Sleep(time.Second)
  				return "user-data", nil
  			})
  
  			fmt.Printf("goroutine %d got: %v, shared=%v\n", i, v, shared)
  		}(i)
  	}
  
  	wg.Wait()
  }
  
  ```

  

- ex2

  ```go
  package main
  
  import (
  	"fmt"
  	"golang.org/x/sync/singleflight"
  	"sync"
  	"time"
  )
  
  func main() {
  	var g singleflight.Group
  	var wg sync.WaitGroup
  
  	fn := func() (any, error) {
  		fmt.Println("真实执行一次")
  		time.Sleep(2 * time.Second)
  		return "hello", nil
  	}
  
  	for i := 0; i < 5; i++ {
  		wg.Add(1)
  		go func(idx int) {
  			defer wg.Done()
  
  			v, err, shared := g.Do("user:1001", fn)
  			if err != nil {
  				fmt.Printf("goroutine=%d err=%v\n", idx, err)
  				return
  			}
  
  			fmt.Printf("goroutine=%d val=%v shared=%v\n", idx, v, shared)
  		}(i)
  	}
  
  	wg.Wait()
  }
  ```

### 解决的问题

#### 缓存击穿 / 热点 Key 并发回源

例如某个热门商品详情缓存失效，瞬间 5000 个请求同时打到数据库：

- 没有 SingleFlight：5000 个请求都查 DB
- 有 SingleFlight：同一个商品 ID 只查 1 次 DB，其他请求等结果复用

这就是典型的 **thundering herd problem（惊群 / herd effect）** 场景，SingleFlight 很适合处理。第三方相关实现文档也明确把它描述为对该问题的处理方式。

#### 防止重复构建昂贵对象

例如：

- 初始化某个大配置
- 拉取某个远端接口数据
- 动态生成报表
- 加载某个模型/词典/规则文件
- 刷新某个租户配置

同一时刻如果很多请求都在做同样的事，SingleFlight 可以只做一次。

#### 降低下游压力

它常被用于保护：

- MySQL / PostgreSQL
- Redis miss 后的 DB 回源
- 第三方 API
- 配置中心
- 文件系统 / 对象存储
- 微服务间的热点查询接口

### `Do` 的语义怎么理解

`Do` 的核心流程可以理解为：

1. 先根据 `key` 查当前是否已有相同任务在执行
2. 如果有，当前请求不再执行 `fn`，而是等待已有任务完成
3. 如果没有，当前请求成为“首个执行者”，真正执行 `fn`
4. 执行结束后，把结果广播给所有等待者
5. 当前 key 的这次 in-flight 状态被移除

也就是说，SingleFlight 只对**“同一时刻正在飞行中的相同请求”**去重。
 它**不会缓存历史结果**。一次执行结束后，下一波请求还会重新执行。这个特性从官方文档的“duplicate suppression”而不是“cache”也能看出来。

#### 不是缓存

| 对比项               | SingleFlight               | 缓存                       |
| -------------------- | -------------------------- | -------------------------- |
| 作用                 | 合并并发中的重复请求       | 复用历史结果               |
| 生命周期             | 只在请求执行期间有效       | 直到过期/淘汰              |
| 是否减少后续请求执行 | 只能减少同一时刻的重复执行 | 可以减少后续很长时间的执行 |
| 是否适合单独使用     | 可用，但效果有限           | 常与 SingleFlight 配合更强 |

### 典型正确组合

最常见的生产写法是：

**本地缓存 / Redis 缓存 + SingleFlight + DB/远端服务**

流程通常是：

```
请求到来
   ↓
先查缓存
   ↓
缓存命中 → 直接返回
   ↓
缓存未命中
   ↓
进入 singleflight.Do(key, fn)
   ↓
只有一个请求真正回源
   ↓
查库/查远端
   ↓
写入缓存
   ↓
所有并发请求共享结果
```

### 最常见的用法

#### 场景一：缓存击穿保护

**场景描述**

商品详情 `product:123` 是热点数据，Redis 过期的一瞬间有大量请求同时到来。

实现

```go
type Product struct {
	ID   int64
	Name string
}

type ProductService struct {
	sf singleflight.Group
	// cache redis/local cache
	// repo  db access
}

func (s *ProductService) GetProduct(productID int64) (*Product, error) {
	cacheKey := fmt.Sprintf("product:%d", productID)

	// 1. 先查缓存
	if p, ok := s.getFromCache(cacheKey); ok {
		return p, nil
	}

	// 2. 缓存 miss，进入 singleflight
	v, err, shared := s.sf.Do(cacheKey, func() (any, error) {
		// 双检，防止等待期间别的协程已写入缓存
		if p, ok := s.getFromCache(cacheKey); ok {
			return p, nil
		}

		p, err := s.queryProductFromDB(productID)
		if err != nil {
			return nil, err
		}

		s.setCache(cacheKey, p)
		return p, nil
	})

	_ = shared // 可以做监控
	if err != nil {
		return nil, err
	}
	return v.(*Product), nil
}
```

缓存 miss 后，同一个 `product:123` 的并发请求会被合并成一次 DB 查询。

价值：

- 避免 Redis 过期瞬间把 DB 打垮

- 降低 MySQL QPS 尖峰

- 降低 P99 延迟抖动

#### 场景二：第三方接口去重调用

**场景描述**

比如支付、物流、风控、画像服务，有些查询接口昂贵且限频：

- 查询用户风控画像
- 查询快递轨迹
- 查询支付订单状态

同一时刻很多业务线程都在查同一个订单状态，其实只需要查一次。

实现示例

```go
type OrderStatus struct {
	OrderID string
	Status  string
}

type OrderService struct {
	sf singleflight.Group
}

func (s *OrderService) QueryOrderStatus(orderID string) (*OrderStatus, error) {
	key := "order_status:" + orderID

	v, err, _ := s.sf.Do(key, func() (any, error) {
		// 这里调用第三方接口
		return s.queryFromRemote(orderID)
	})
	if err != nil {
		return nil, err
	}
	return v.(*OrderStatus), nil
}
```

使用价值：

- 降低第三方 API 调用成本
- 防止触发对方限流
- 避免自己线程池/连接池被重复调用拖满