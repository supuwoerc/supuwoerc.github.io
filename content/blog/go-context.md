---
title: Golang中的context
date: 2024-10-17
authors:
  - name: Idris
    link: https://github.com/supuwoerc
excludeSearch: true
tags:
  - Golang
  - 基础
---

本文介绍Golang中Context的实现和一些常见的使用场景
<!--more-->
{{< callout type="warning" >}}
请注意 go 版本，不同版本的 go context 存在差异，例如本文使用的 go 版本中 cancelCtx 支持自定义 cause 。

本文使用的go版本信息： go version go1.21.1 darwin/arm64
{{< /callout >}}
## 关于上下文
Golang官方库的Context(上下文)是开发过程中经常用到的API，大多数时候我们会直接使用`context.Background()`来创建一个Context。

我们可以借助不同的上下文来存储数据，设置超时时间和主动取消它来控制Goroutine的生命周期，传递参数和信号或是实现其他业务场景。

## 接口定义
先看Context的接口定义：
```go
type Context interface {
	Deadline() (deadline time.Time, ok bool)
	Done() <-chan struct{}
	Err() error
	Value(key any) any
}
```
是的，Go中的Context定义十分的简单，只需要按照定义实现`Deadline`、`Done`、`Err`、`Value`就能快速的定义一个Context，下面将简单的介绍一下这四个方法作用和含义。

### Deadline
方法返回两个值：一个是当前上下文的截止时间（time.Time 类型）；另一个是一个布尔值，表示是否设置了这个截止时间（用于区分零值）。

如果截止时间未设置，返回的布尔值为 false ，此时截止时间的值没有实际意义。

在实际应用中，通过检查 Deadline 方法返回的布尔值和截止时间，可以决定是否需要在操作达到截止时间时进行相应的处理，
常用语取消操作、释放资源等。

例如 GORM 中的[Context-Timeout](https://gorm.io/zh_CN/docs/context.html#Context-Timeout)可以传入一个带超时时间的上下文，当上下文超时依旧未完成对应的Sql任务的话就会取消这次行为。
```go
func(u *User)Find(){
	// 创建一个2秒后截止的上下文
    ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
    defer cancel()
	// 在传递给 db 的上下文上设置超时。WithContext 可以控制长时间运行查询的持续时间。这对于维护性能和避免数据库交互中的资源锁定至关重要。
    db.WithContext(ctx).Find(&users)
}
```

### Done
该方法返回一个只读的 Channel，当上下文被取消（例如主动调用`cancel`方法，或者到达了设置的超时时间而自动取消），该方法返回的 Channel 将被关闭。

通过监听这个通道，您可以在上下文被取消时执行相应的清理或结束操作。

例如 Gin 官方的示例中提供了关于优雅关闭服务([graceful-shutdown](https://github.com/gin-gonic/examples/blob/master/graceful-shutdown/graceful-shutdown/notify-with-context/server.go))的案例中就使用了这个方式来监听信号量。

```go
func main() {
	// 创建一个用于监听系统信号量的上下文
	ctx, stop := signal.NotifyContext(context.Background(), syscall.SIGINT, syscall.SIGTERM)
	defer stop()
	router := gin.Default()
	router.GET("/", func(c *gin.Context) {
		time.Sleep(10 * time.Second)
		c.String(http.StatusOK, "Welcome Gin Server")
	})
	srv := &http.Server{
		Addr:    ":8080",
		Handler: router,
	}
	// 为了避免程序堵塞，使用协程创建http服务
	go func() {
		if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
			log.Fatalf("listen: %s\n", err)
		}
	}()
	// 堵塞在此处，监听上下文的channel有无写入内容，直到通道被写入或被关闭
	<-ctx.Done()
	stop()
	log.Println("shutting down gracefully, press Ctrl+C again to force")
	// 为了防止有正在处理的请求尚未返回数据，再创建一个带有截止时间（5s）的上下文来关闭http服务，如果5s服务依旧未执行完成，服务将被强制关闭
	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()
	if err := srv.Shutdown(ctx); err != nil {
		log.Fatal("Server forced to shutdown: ", err)
	}
	log.Println("Server exiting")
}
```

### Err
Context 的 Err 方法将返回上下文取消相关的错误，包含下面几种情况：
* `nil`:当前上下文未被取消

* `context.Canceled`:表示是主动调用了`cancel`方法取消了上下文

* `context.DeadlineExceeded`:表示到达了截止时间，上下文自动取消

```go
func doPrint(ctx context.Context) {
    for {
        select {
        case <-ctx.Done():
            fmt.Println("Context cancelled. Error:", ctx.Err())
            return
        default:
            fmt.Println("default is running...")
            time.Sleep(1 * time.Second)
        }
    }
}
func main() {
    ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
    defer cancel()
    go doPrint(ctx)
    time.Sleep(5 * time.Second)
}
// default is running...
// default is running...
// default is running...
// Context cancelled. Error: context deadline exceeded
```

### Value
Context 的 Value 用户在上下文中存储和获取键值对数据。

可以通过 context.WithValue(parentContext, key, value) 可以创建一个携带指定值的子上下文。

其中 key 需要是可比较的，这里我的理解应该类似于在上下文中创建一个 map 。

```go
func main() {
    ctx := context.Background()
    ctx = context.WithValue(ctx, "uid", 123)
	uid, ok := ctx.Value("uid").(int)
    if ok {
        fmt.Println("User ID:", uid)
    } else {
        fmt.Println("uid not found")
    }
}
```

## emptyContext
在 Context 包中存在两个方法：`Background`和`TODO`，方法实现如下：
```go
// 一个不会被取消，不存在Value和截止时间的上下文。
type emptyCtx struct{}
type backgroundCtx struct{ emptyCtx }
type todoCtx struct{ emptyCtx }
func Background() Context {
	return backgroundCtx{}
}
func TODO() Context {
	return todoCtx{}
}
```
可以发现，这两个方法都是返回了一个嵌套了emptyCtx的结构体，在注释里我们可以知道它是一个不会被取消、没有截止时间、也不携带任何值的上下文。

实际这两个方法的使用场景也略有差异：
* `context.Background`：通常用于在顶层（例如在主函数、初始化或在一个没有其他合适上下文的起始位置）创建一个根上下文。

* `context.TODO`：当您不确定应该使用哪种具体的上下文，或者当前的函数稍后会被修改以接收一个合适的上下文时使用。它类似于一个占位符，表示这个上下文可能需要在未来被完善或替换。

## cancelCtx
cancelCtx 结构体定义如下:
```go
type cancelCtx struct {
	Context
	mu       sync.Mutex            // protects following fields
	done     atomic.Value          // of chan struct{}, created lazily, closed by first cancel call
	children map[canceler]struct{} // set to nil by the first cancel call
	err      error                 // set to non-nil by the first cancel call
	cause    error                 // set to non-nil by the first cancel call
}
type canceler interface {
    cancel(removeFromParent bool, err, cause error)
    Done() <-chan struct{}
}
```
* `Context`：这是创建当前 `cancelCtx` 所基于的父上下文。

* `mu sync.Mutex`：互斥锁，用于保护对内部状态的并发访问，确保在多协程环境下数据的一致性和安全性。

* `done atomic.Value`：这是一个通道，当上下文被取消时会被关闭，用于通知相关的操作该上下文已被取消。

* `children map[canceler]struct{}`：用于存储基于当前上下文创建的子上下文，以便在当前上下文取消时，能够通知子上下文也进行取消操作。

* `err error`：存储取消操作的错误信息。当上下文被取消时，通过 `Err` 方法可以获取这个错误。

* `cause error`：存储取消操作的自定义错误信息。当上下文被取消时，通过 `Cause` 方法可以获取这个自定义错误。

### Deadline
cancelCtx 并未重写该方法，调用 cancelCtx 的 Deadline 方法实际上就是在调用嵌入的 Context 的 Deadline 方法。

### Done
方法定义：
```go
func (c *cancelCtx) Done() <-chan struct{} {
	d := c.done.Load()
	if d != nil {
		return d.(chan struct{})
	}
	c.mu.Lock()
	defer c.mu.Unlock()
	d = c.done.Load()
	if d == nil {
		d = make(chan struct{})
		c.done.Store(d)
	}
	return d.(chan struct{})
}
```
当创建 cancelCtx 的时候，done 并未被初始化，只有在调用时才去懒初始化。

其中 `d = c.done.Load()` 调用了两次，一次是开始的检查，目的是判断当前的done有没有初始化，此时并未加锁，因为 done 本身的类型是 atomic.Value ，对他的读写的是原子的，如果提前加锁会降低性能。

但是，既然 done 的类型是 atomic.Value 为何还需要借助 c.mu.Lock 来为初始化行为加锁呢？原因很简单：atomic.Value 的 Store 方法虽然是原子的，但是多个协程同时调用的话依旧会导致 done 被错误的初始化多次，所以在初始化 done 时需要加锁后再次检查。

也就是说，第一次的读取检查实际上是利用了 atomic.Value 的读锁，而为了避免 done 被多个协程初始化，需要借助 mutex 来手动锁定写的行为。

### Err
```go
func (c *cancelCtx) Err() error {
    c.mu.Lock()
    err := c.err
    c.mu.Unlock()
    return err
}
```
返回上下文中的 err ,为了避免读写竞争，加上了互斥锁。

### Value
```go
func (c *cancelCtx) Value(key any) any {
	if key == &cancelCtxKey {
		return c
	}
	return value(c.Context, key)
}
```
首先判断 key 是否是 cancelCtxKey ，如果是则返回自身的指针，否则返回 value 函数的结果， value 函数会在下面的 valueContext 中介绍。

### 创建 cancelCtx
一般会使用 `context.WithCancel`方法来创建 cancelCxt：
```go
ctx, cancelFunc := context.WithCancel(context.Background())
```
下面看WithCancel的实现：
```go
func withCancel(parent Context) *cancelCtx {
    if parent == nil {
    panic("cannot create context from nil parent")
    }
    c := newCancelCtx(parent)
    propagateCancel(parent, c)
    return c
}
func newCancelCtx(parent Context) *cancelCtx {
    return &cancelCtx{Context: parent}
}
func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {
    c := withCancel(parent)
    return c, func() { c.cancel(true, Canceled, nil) }
}
```
参数检查，父 Context 时候为空时引发 panic，创建 cancelCtx 注入父 Context。

借助 propagateCancel 方法用户创建一个守护协程，作用是在传入的父 Context 终止时取消自身（传播取消行为）：

为了避免代码太多，查阅注释和代码不方便，我会在关键代码上加上注释来解释方法的逻辑。

```go
// cancelCtx 实现的 Value 方法定义了一个规则，当 key 为cancelCtxKey的时候返回自身！
func (c *cancelCtx) Value(key any) any {
    if key == &cancelCtxKey {
		return c
    }
    return value(c.Context, key)
}
// 返回父 Context (参数parent)是不是一个可以取消的上下文，如果 parent 是可取消的则会返回 parent & true,否则返回 nil & false
func parentCancelCtx(parent Context) (*cancelCtx, bool) {
    done := parent.Done()
    if done == closedchan || done == nil {
        return nil, false
    }
	// 如果一个 Context 是 cancelCtx，Value(&cancelCtxKey) 方法会返回自身
    p, ok := parent.Value(&cancelCtxKey).(*cancelCtx)
    if !ok {
        return nil, false
    }
    pdone, _ := p.done.Load().(chan struct{})
    // 确保 cancelCtx 的 done 通道确实对应于传递进来的 parent 上下文的 Done 
    if pdone != done {
        return nil, false
    }
    return p, true
}

func propagateCancel(parent Context, child canceler) {
	done := parent.Done()
	// 判断父context是不是emptyContext之类的上下文，这类上下文永远不会被取消
	// 如果是这类上下文，直接返回，取消不会发生，自然不需要传播
	if done == nil {
		return  
	}
	select {
	// 当上下文已经取消，取消child上下文
	case <-done:
		child.cancel(false, parent.Err(), Cause(parent))
		return
	default:
	}
	// 判断 parent 是不是 cancelCtx
	if p, ok := parentCancelCtx(parent); ok {
		// 加锁，p.err 读取 和 map 操作都不是并发安全的
		p.mu.Lock()
		if p.err != nil {
            // 当上下文已经取消，取消child上下文
			child.cancel(false, p.err, p.cause)
		} else {
            // 设置内层cancelCtx与child的父子关系
			if p.children == nil {
				p.children = make(map[canceler]struct{})
			}
			p.children[child] = struct{}{}
		}
		p.mu.Unlock()
	} else {
		goroutines.Add(1)
		// parent 不是一个 cancelCtx，创建一个协程来进行监听，当 parent 关闭了则调用 child 的 cancel
		go func() {
			select {
			case <-parent.Done():
				child.cancel(false, parent.Err(), Cause(parent))
			case <-child.Done():
			}
		}()
	}
}
```
propagateCancel 和 parentCancelCtx 较为重要，此处再次总结 propagateCancel 是如何将取消行为向下传播给 child 上下文：

* 调用 parent 的 Done 方法来获取标记上下文本取消的 chanel，如果返回值是 nil 则代表 parent 上下文不可取消，不用向下传播取消行为。

* Done 返回值不是 nil，借助 select 来检查一下 parent 是不是已经关闭，是的话调用 child 的 cancel 来将取消行为传播下去

* parent 未关闭，因为 select 存在 default 分支，继续向下执行

* 调用 parentCancelCtx 来判断 parent 是不是 cancelCtx ，是的话先判断 parent 的 err，如果 err 不是 nil 则取消 child，否则将 child 添加到 parent 的 children map 中，整个判断过程需要加锁，因为 err 和 map 操作都不是并发安全的。

* 当 parentCancelCtx 返回值表明 parent 不是 cancelCtx ，那就创建一个协程来监听 parent.Done() 的通道，如果 parent 取消，则调用 child 的 cancel。

### cancel 方法
cancel 方法用于将一个 cancelCtx 上下文取消：
```go
func (c *cancelCtx) cancel(removeFromParent bool, err, cause error) {
	// 如果未指定 cancel 的错误，引发 panic
	if err == nil {
		panic("context: internal error: missing cancel error")
	}
	// 如果未指定 cause ，以 err 作为 cause
	if cause == nil {
		cause = err
	}
	// 如果当前上下文已经 cancel，则直接返回
	c.mu.Lock()
	if c.err != nil {
		c.mu.Unlock()
		return  
	}
	c.err = err
	c.cause = cause
	d, _ := c.done.Load().(chan struct{})
	// 如果 done 存储的通道为 nil(因为done是懒初始化的，只有首次调用 Done 才开始初始化)，那么意味着未调用Done 而是直接调用了cancel，
	// 直接将 done 初始化为一个已经 close 的通道
	if d == nil {
		c.done.Store(closedchan)
	} else {
		// close 掉 done 存储的 chanel
		close(d)
	}
	// 调用每一个 child 的 cancel 方法 
	for child := range c.children {
		child.cancel(false, err, cause)
	}
	// 清理 children
	c.children = nil
	c.mu.Unlock()
    // 如果 removeFromParent 为 true, 将把自身从 parent 上下文的 children 中移除。
	if removeFromParent {
		removeChild(c.Context, c)
	}
}

func removeChild(parent Context, child canceler) {
    p, ok := parentCancelCtx(parent)
    if !ok {
        return
    }
    p.mu.Lock()
    if p.children != nil {
        delete(p.children, child)
    }
    p.mu.Unlock()
}
```

## valueCtx
valueCtx 的结构体定义十分简单：
```go
type valueCtx struct {
	Context
	key, val any
}
```
* Context：这是创建当前 valueCtx 所基于的父上下文。

* key/val：当前 valueCtx 所存储的键值对。

> 从结构体定义来看 valueCtx 只能存储一对键值，同时 Value 方法的实现（下文介绍了查找原理）是依次向上查找 key 对应的 val 的。
> 所以 valueCtx 的 Value 结果和查找起点有关系，一般实际场景用它存储全局的标识类数据。

### Value
```go
func (c *valueCtx) Value(key any) any {
    if c.key == key {
        return c.val
    }
    return value(c.Context, key)
}
func value(c Context, key any) any {
	for {
		switch ctx := c.(type) {
		case *valueCtx:
			if key == ctx.key {
				return ctx.val
			}
			c = ctx.Context
		case *cancelCtx:
			if key == &cancelCtxKey {
				return c
			}
			c = ctx.Context
		case *timerCtx:
			if key == &cancelCtxKey {
				return ctx.cancelCtx
			}
			c = ctx.Context
		case *emptyCtx:
			return nil
		default:
			return c.Value(key)
		}
	}
}
```
* 如果 valueCtx 存储的 key 就是要查询的 key ，那么直接返回 val 。

* 如果不是，则调用 value 方法去做循环，依次向上级上下文中查找，因为 golang 不存在 while 循环，直接用了 for 去实现了 while 查找。

### 创建valueCtx
一般会使用 context.WithValue 创建 valueCxt：
```go
ctx := context.WithValue(context.Background(), "key", "value")
```
WitValue源码：
```go
func WithValue(parent Context, key, val any) Context {
	if parent == nil {
		panic("cannot create context from nil parent")
	}
	if key == nil {
		panic("nil key")
	}
	// 重点：key 需要是可比较的
	if !reflectlite.TypeOf(key).Comparable() {
		panic("key is not comparable")
	}
	return &valueCtx{parent, key, val}
}
```

## timerCtx
timerCtx 的结构体定义如下：
```go
type timerCtx struct {
	*cancelCtx
	timer *time.Timer // Under cancelCtx.mu.
	deadline time.Time
}
```
* cancelCtx：嵌入的 cancelCtx 指针，也就是说 timerCtx 实际上是 cancelCtx 的扩展。
* timer：定时器，用于触发超时行为。
* deadline：记录上下文的截止时间。

### Deadline
```go
func (c *timerCtx) Deadline() (deadline time.Time, ok bool) {
	return c.deadline, true
}
```
结合前面介绍的几种context，只有 timerCtx 具体实现了 Deadline 方法，返回当前上下文的截止时间。

### cancel
timerCtx 重写了 cancelCtx 的 cancel 方法，源码如下：
```go
func (c *timerCtx) cancel(removeFromParent bool, err, cause error) {
	c.cancelCtx.cancel(false, err, cause)
	if removeFromParent {
		// Remove this timerCtx from its parent cancelCtx's children.
		removeChild(c.cancelCtx.Context, c)
	}
	c.mu.Lock()
	if c.timer != nil {
		c.timer.Stop()
		c.timer = nil
	}
	c.mu.Unlock()
}
```
从逻辑来看，是在调用内部的 cancelCtx 的 cancel 方法后将自身的 timer 停止并清理掉，仅此而已。

### 创建timerCtx
golang 提供了两种创建 timerCtx 的方法：
```go
// 指定具体的截止时间
deadline, cancel := context.WithDeadline(context.Background(), time.Now().Add(10*time.Second))
// 指定持续时间
timeout, cancelFunc := context.WithTimeout(context.Background(), 5*time.Second)
```
* `context.WithDeadline`：创建一个在具体时间截止的上下文。

* `context.WithTimeout`：创建一个在多长时间后截止的上下文。

下面是`WithDeadline`的源码实现，我将关键位置添加了注释：

```go
func WithDeadline(parent Context, d time.Time) (Context, CancelFunc) {
	if parent == nil {
		panic("cannot create context from nil parent")
	}
	// 如果 parent 的截止时间在自己的截止时间之前，直接调用 WithCancel 返回 cancelCtx
	if cur, ok := parent.Deadline(); ok && cur.Before(d) {
		return WithCancel(parent)
	}
	// 构建 timerCtx
	c := &timerCtx{
		cancelCtx: newCancelCtx(parent),
		deadline:  d,
	}
    // 构建取消行为传播
	propagateCancel(parent, c)
	dur := time.Until(d)
	// 如果截止时间已经到了或者过了，直接取消
	if dur <= 0 {
		c.cancel(true, DeadlineExceeded, nil) // deadline has already passed
		return c, func() { c.cancel(false, Canceled, nil) }
	}
	c.mu.Lock()
	defer c.mu.Unlock()
	// 创建定时器来定时触发 cancel 方法
	if c.err == nil {
		c.timer = time.AfterFunc(dur, func() {
			c.cancel(true, DeadlineExceeded, nil)
		})
	}
	return c, func() { c.cancel(true, Canceled, nil) }
}
```
WithTimeout 方法只是指定了一个时间，底层依旧调用的 WithDeadline 方法：
```go
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc) {
	return WithDeadline(parent, time.Now().Add(timeout))
}
```

以上就是关于官方库的context的一些个人理解，如果存在错误，请在 github 提交 issue 给我，感谢～