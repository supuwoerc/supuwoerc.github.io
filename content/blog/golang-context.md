---
title: Golang中的context
date: 2024-10-17
authors:
  - name: Idris
    link: https://github.com/supuwoerc
excludeSearch: true
---

本文介绍Golang中Context的实现和一些常见的使用场景
<!--more-->
## 关于上下文
Golang官方库的Context(上下文)是开发过程中经常用到的API，大多数时候我们会直接使用`context.Background()`来创建一个Context，这个上下文可以用来存储一些数据，
设置超时时间和主动取消，正是这些功能，我们可以用Context来控制Goroutine的生命周期，传递参数和信号。

## 接口定义
先看Context的接口定义：
```golang
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
```golang
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

```golang
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

```golang
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

```golang
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
```golang
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

<!--整理旧文档内容到新文档:cancelCtx/timerCtx/valueCtx-->
