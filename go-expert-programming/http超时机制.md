# **Go net/http 超时机制完全手册**

> 参考：https://colobu.com/2016/07/01/the-complete-guide-to-golang-net-http-timeouts/

- 当用 Go 写 HTTP 的服务器和客户端的时候，超时处理总是最易犯错和最微妙的地方之一。

- 错误可能来自很多地方，一个错误可能等待很长时间没有结果，直到网络故障或者进程挂起。

- HTTP 是一个复杂的、多阶段 (multi-stage) 协议，所以没有一个放之四海而皆准的超时解决方案，比如一个流服务、一个 JSON API 和一个 Comet 服务对超时的需求都不相同，往往默认值不是你想要的。

- 本文我将拆解需要超时设置的各个阶段，看看用什么不同的方式去处理它， 包括服务器端和客户端。

<br>

## **1. SetDeadline**
- 首先，**你需要了解 Go 实现超时的网络原语 (primitive): `Deadline` (最后期限)**。

- **net.Conn 为 Deadline 提供了多个方法 `Set[Read|Write]Deadline(time.Time)`**。

- **Deadline 是一个绝对时间值**，当到达这个时间的时候，所有的 I/O 操作都会失败，返回超时 (timeout) 错误。

- **Deadline 不是超时 (timeout)。一旦设置它们永久生效 (或者直到下一次调用 SetDeadline)**, 不管此时连接是否被使用和怎么用。所以如果想使用 SetDeadline 建立超时机制，你不得不每次在 Read/Write 操作之前调用它。

- 你可能不想自己调用 SetDeadline, 而是让 net/http 代替你调用，所以你可以调用更高级的 timeout 方法。但是请记住，所有的超时的实现都是基于 Deadline, 所以它们不会每次接收或者发送重新设置这个值 (so they do NOT reset every time data is sent or received)。

<br>

## **II. 服务器端超时设置**
- 对于暴露在网上的服务器来说，为客户端连接设置超时至关重要，否则巨慢的或者隐失的客户端可能导致文件句柄无法释放，最终导致服务器出现下面的错误:

  ```bash
  http: Accept error: accept tcp [::]:80: accept4: too many open files; retrying in 5ms  
  ```

- http.Server 有两个设置超时的方法: `ReadTimeout` 和 `andWriteTimeout`。

- 你可以显示地设置它们：

  ```go
  srv := &http.Server{  
      ReadTimeout: 5 * time.Second,
      WriteTimeout: 10 * time.Second,
  }

  log.Println(srv.ListenAndServe())
  ```

- **`ReadTimeout` 的时间计算是从`连接被接受` (accept) 到 `request body` 完全被读取** (如果你不读取 body，那么时间截止到读完 header 为止)。

- **它的内部实现是在 Accept 立即调用 SetReadDeadline 方法 (代码行)**。

  ```go
  ...
    if d := c.server.ReadTimeout; d != 0 {
    c.rwc.SetReadDeadline(time.Now().Add(d))
  }

  if d := c.server.WriteTimeout; d != 0 {
    c.rwc.SetWriteDeadline(time.Now().Add(d))
  }
  ...
  ```

- **`WriteTimeout` 的时间计算正常是从 `request header 的读取结束`开始，到 `response write` 结束为止** (也就是 ServeHTTP 方法的声明周期), 它是通过在 readRequest 方法结束的时候调用 SetWriteDeadline 实现的 (代码行)。

  ```go
  func (c *conn) readRequest(ctx context.Context) (w *response, err error) {
    if c.hijacked() {
      return nil, ErrHijacked
    }

    if d := c.server.ReadTimeout; d != 0 {
      c.rwc.SetReadDeadline(time.Now().Add(d))
    }

    if d := c.server.WriteTimeout; d != 0 {
      defer func() {
        c.rwc.SetWriteDeadline(time.Now().Add(d))
      }()
    }
    ...
  }
  ```

