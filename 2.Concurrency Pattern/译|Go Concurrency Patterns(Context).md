# 译|Go Concurrency Patterns: Context



## Introduction

在Go语言实现的服务器中，每一个进来的请求都会在属于它自己的goroutine中运行(One request per goroutine)。请求处理程序通常会新开辟goroutine去访问后端服务，例如访问数据库或者rpc调用等。服务于请求的一组常用典型的goroutines访问特定的请求值，例如最终用户的身份，授权令牌和请求的截止日期等等。当请求被取消或触发超时的时候，在该请求上工作的所有goroutine会快速退出，以便系统可以回收所使用的任何资源。



在google内部，开发了一个 *context* 包，可以轻松地跨越api边界,传递请求范围值，取消信号和截止日期到 *request* 所涉及的所有 `goroutine` 。 该包就是开源的[context](https://golang.org/pkg/context/)。本文介绍了如何使用该包以及提供了一个完整的工作示例。



## Context

context包的核心就是`Context`类型。

```go
// A Context carries a deadline, cancelation signal, and request-scoped values
// across API boundaries. Its methods are safe for simultaneous use by multiple
// goroutines.
type Context interface {
    // Done returns a channel that is closed when this Context is canceled
    // or times out.
    Done() <-chan struct{}

    // Err indicates why this context was canceled, after the Done channel
    // is closed.
    Err() error

    // Deadline returns the time when this Context will be canceled, if any.
    Deadline() (deadline time.Time, ok bool)

    // Value returns the value associated with key or nil if none.
    Value(key interface{}) interface{}
}
```

上面这个是精简的描述，详细的可以看[godoc](https://golang.org/pkg/context/)

`Done` 方法返回一个 `channel` ，用于发送取消信号(代表 `Context` 已关闭)到运行时函数：当 `channel` 关闭时，函数应该放弃后续流程并返回。`Err` 方法返回一个错误，指出为什么`context` 被取消。 [Pipelines and Cancelation](https://blog.golang.org/pipelines)更详细地讨论了 `Done` 返回的channel的惯用手法。

由于`Done`方法返回的`channel`只接收的原因，`Context`并没有`Cancel`方法。接收取消信号的函数通常不应当具备发送信号的功能。特别是，当父操作启动子操作的 `goroutines` 时，这些子操作不应该能够取消父操作。 相反，`WithCancel` 函数（如下所述）提供了一种取消新的 `Context` 值的方法。

`Context` 可以安全地同时用于多个 `goroutines` 。 代码可以将单个 *Context* 传递给任意数量的 `goroutine` ，并能发送取消该`Context`的信号到所有的关联的 `goroutine` 。

`Deadline` 方法允许功能确定是否应该开始工作; 如果剩下的时间太少，可能不值得。 代码中也可能会使用截止时间来为I/O操作设置超时。

`Value` 允许 `Context` 传送请求数据。 该数据必须能安全的同时用于多个 `goroutine` 。

## Derived contexts

`context`包提供了从现有 `Context` 衍生出新的 `Context` 的函数。 这些 `Context` 形成一个树状的层级结构：当一个 `Context` 被取消时，从它衍生出的所有 `Context` 也被取消。

`Background` 是任何Context树的根；它永远不会被取消：

```go
// Background returns an empty Context. It is never canceled, has no deadline,
// and has no values. Background is typically used in main, init, and tests,
// and as the top-level Context for incoming requests.
func Background() Context
```

`WithCancel` 和 `WithTimeout` 返回衍生出的 `Context` ，衍生出的子 `Context` 可早于父`Context` 被取消。 与传入的 `request` 相关联的上下文通常在请求处理程序返回时被取消。 `WithCancel` 也可用于在使用多个副本时取消冗余请求。 `WithTimeout*`对设置后台服务器请求的最后期限很有用：

```go
// WithCancel returns a copy of parent whose Done channel is closed as soon as
// parent.Done is closed or cancel is called.
func WithCancel(parent Context) (ctx Context, cancel CancelFunc)

// A CancelFunc cancels a Context.
type CancelFunc func()

// WithTimeout returns a copy of parent whose Done channel is closed as soon as
// parent.Done is closed, cancel is called, or timeout elapses. The new
// Context's Deadline is the sooner of now+timeout and the parent's deadline, if
// any. If the timer is still running, the cancel function releases its
// resources.
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)
```

`WithValue` 提供了一种将请求范围内的值与 `Context` 相关联的方法：

```go
// WithValue returns a copy of parent whose Value method returns val for key.
func WithValue(parent Context, key interface{}, val interface{}) Context
```

掌握如何使用 `context` 包的最佳方法是通过一个真实完整的示例。

## Example: Google Web Search

我们的例子是一个http的服务器，主要用于处理诸如`/search?q=golang&timeout=1s`，就是转发"golang"的请求到[Google Web Search API](https://developers.google.com/web-search/docs/) 和传送结果。`timeout`参数为服务端取消请求的超时时间。

代码主要被分成三个包：

+ [server](https://blog.golang.org/context/server/server.go) 提供了`/search`的主函数逻辑；
+ [userip](https://blog.golang.org/context/userip/userip.go) 提供了从一个请求中提取出用户的IP并与之绑定到一个`Context`的功能；
+ [google](https://blog.golang.org/context/google/google.go) 提供了发送请求到Google的搜索函数功能。



## The server program

服务器通过为 golang 提供前几个 Google 搜索结果来处理像 `/search?q=golang` 之类的请求。 它注册 `handleSearch` 来处理`/search`。 处理函数创建一个名为`ctx`的`Context` ，并在处理程序返回时,一并被取消。 如果request包含超时URL参数，则超时时会自动取消上下文。

```go
func handleSearch(w http.ResponseWriter, req *http.Request) {
    // ctx is the Context for this handler. Calling cancel closes the
    // ctx.Done channel, which is the cancellation signal for requests
    // started by this handler.
    var (
        ctx    context.Context
        cancel context.CancelFunc
    )
    timeout, err := time.ParseDuration(req.FormValue("timeout"))
    if err == nil {
        // The request has a timeout, so create a context that is
        // canceled automatically when the timeout expires.
        ctx, cancel = context.WithTimeout(context.Background(), timeout)
    } else {
        ctx, cancel = context.WithCancel(context.Background())
    }
    defer cancel() // Cancel ctx as soon as handleSearch returns.
```

处理程序从 `request` 中提取查询关键字，并通过调用 `userip` 包来提取客户端的IP地址。 后端请求需要客户端的IP地址，因此`handleSearch`将其附加到`ctx`：

```go
// Check the search query.
query := req.FormValue("q")
if query == "" {
  http.Error(w, "no query", http.StatusBadRequest)
  return
}

// Store the user IP in ctx for use by code in other packages.
userIP, err := userip.FromRequest(req)
if err != nil {
  http.Error(w, err.Error(), http.StatusBadRequest)
  return
}
ctx = userip.NewContext(ctx, userIP)
```



