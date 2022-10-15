# **上下文 Context**

> 参考: https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-context/

- **context.Context 是用来设置`截止日期`、`同步信号`，`传递请求相关值`的结构体**。

- 上下文与 Goroutine 有比较密切的关系。context.Context 是 Go 语言中独特的设计，在其他编程语言中我们很少见到类似的概念。

- context.Context 是 Go 语言在 1.7 版本中引入标准库的接口1，该接口定义了四个需要实现的方法，其中包括：

    ```go
    type Context interface {
        Deadline() (deadline time.Time, ok bool)
        Done() <-chan struct{}
        Err() error
        Value(key interface{}) interface{}
    }
    ```

    1. Deadline：
        
        - **返回 context.Context `被取消的时间`，也就是完成工作的截止日期**

    2. Done：
        
        - **返回一个 Channel，这个 Channel 会在当前工作完成或者上下文被取消之后关闭**
        
        - **多次调用 Done 方法会返回同一个 Channel**；

    3. Err：
        
        - **返回 context.Context 结束的原因，它只会在 Done 返回的 Channel 被关闭时才会返回非空的值**；

            - **如果 context.Context `被取消`，会返回 `Canceled` 错误**；
            
            - **如果 context.Context `超时`，会返回 `DeadlineExceeded` 错误**；

    4. Value：

        - **从 context.Context 中获取键对应的值，对于同一个上下文来说，多次调用 Value 并传入`相同的 Key` 会返回`相同的结果`**
        
        - **该方法可以用来传递请求特定的数据**

- **context 包中提供的 `context.Background`、`context.TODO`、`context.WithDeadline` 和 `context.WithValue` 函数会返回实现该接口的私有结构体**，我们会在后面详细介绍它们的工作原理。

<br>

## **I. 使用 Context 同步信号**
### **1. exp:**
- **创建一个过期时间为 1s 的上下文，并向上下文传入 handle 函数，该方法会使用 500ms 的时间处理传入的『请求』**：

    ```go
    func main() {
        // 创建一个过期时间为 1s 的上下文
        ctx, cancel := context.WithTimeout(context.Background(), 1*time.Second)
        defer cancel()

        // 向上下文传入 handle 函数
        go handle(ctx, 500*time.Millisecond)
        select {
        case <-ctx.Done():
            fmt.Println("main", ctx.Err())
        }
    }

    func handle(ctx context.Context, duration time.Duration) {
        select {
        case <-ctx.Done():
            fmt.Println("handle", ctx.Err())

        case <-time.After(duration):
            fmt.Println("process request with", duration)
        }
    }
    ```

- 因为过期时间大于处理时间，所以我们有足够的时间处理该『请求』，运行上述代码会打印出如下所示的内容：

    ```bash
    process request with 500ms
    main context deadline exceeded
    ```

- **handle 函数没有进入`超时的 select 分支`，但是 main 函数的 select 却会`等待 context.Context 的超时`并打印出 main context deadline exceeded。**

- 如果我们将处理『请求』时间增加至 1500ms，整个程序都会因为上下文的过期而被中止：

    ```bash
    main context deadline exceeded
    handle context deadline exceeded
    ```

- 相信这两个例子能够帮助各位读者理解 context.Context 的使用方法和设计原理：

    - **多个 Goroutine 同时订阅 `ctx.Done()` 管道中的消息，一旦接收到`取消信号`就立刻停止当前正在执行的工作**。

<br>

### **2. 应用**
- 控制 sql 执行超时

    ```go
    func (db *conn) query(c context.Context) {
        _, c, cancel := db.conf.QueryTimeout.Shrink(c)
        rs, err := db.DB.QueryContext(c, query, args...)
        ...
    }

    // Shrink will decrease the duration by comparing with context's timeout duration
    // and return new timeout\context\CancelFunc.
    func (d Duration) Shrink(c context.Context) (Duration, context.Context, context.CancelFunc) {
        if deadline, ok := c.Deadline(); ok {
            // 比较 context 的超时剩余时间和 conf 中的超时时间
            if ctimeout := xtime.Until(deadline); ctimeout < xtime.Duration(d) {
                // deliver small timeout 返回较小的 timeout
                return Duration(ctimeout), c, func() {}
            }
        }
        ctx, cancel := context.WithTimeout(c, xtime.Duration(d))
        return d, ctx, cancel
    } 
    ```
