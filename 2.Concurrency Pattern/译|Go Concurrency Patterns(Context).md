# 译|Go Concurrency Patterns: Context



## Introduction

在Go语言实现的服务器中，每一个进来的请求都会在属于它自己的goroutine中运行(One request per goroutine)。请求处理程序通常会新开辟goroutine去访问后端服务，例如访问数据库或者rpc调用等。服务于请求的一组常用典型的goroutines访问特定的请求值，例如最终用户的身份，授权令牌和请求的截止日期等等。当请求被取消或触发超时的时候，在该请求上工作的所有goroutine会快速退出，以便系统可以回收所使用的任何资源。



在google内部，开发了一个 `context` 包，可以轻松地跨越api边界，传递请求范围值，取消信号和截止日期到 `request` 所涉及的所有 `goroutine` 。 该包就是开源的[context](https://golang.org/pkg/context/)。本文介绍了如何使用该包以及提供了一个完整的工作示例。



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

处理程序`handler`使用`ctx`和查询关键字`query`调用`google.Search`方法：

```go
    // Run the Google search and print the results.
    start := time.Now()
    results, err := google.Search(ctx, query)
    elapsed := time.Since(start)
```

如果搜索成功，处理程序将渲染返回结果：

```go
    if err := resultsTemplate.Execute(w, struct {
        Results          google.Results
        Timeout, Elapsed time.Duration
    }{
        Results: results,
        Timeout: timeout,
        Elapsed: elapsed,
    }); err != nil {
        log.Print(err)
        return
    }
```



## Package userip

`userip`包提供从请求中提取用户IP地址并将其与 `Context`相关联的函数。 `Context` 提供了 key-value 映射的 `map` ，其中 *key* 和 *value* 均为 `interface{}` 类型。 `key` 类型必须支持相等性， `value` 必须是多个 `goroutine` 安全的。 `userip` 这样的包会隐藏 map 的细节，并提供强类型访问特定的 `Context` 值。

为了避免关键字冲突， `userip` 定义了一个**不导出**的类型 `key` ，并使用此类型的值作为 `Context` 的关键字：

```go
// The key type is unexported to prevent collisions with context keys defined in
// other packages.
type key int

// userIPkey is the context key for the user IP address.  Its value of zero is
// arbitrary.  If this package defined other context keys, they would have
// different integer values.
const userIPKey key = 0
```

`FromRequests`负责从http请求中提取出`userIP`的值。

```go
func FromRequest(req *http.Request) (net.IP, error) {
    ip, _, err := net.SplitHostPort(req.RemoteAddr)
    if err != nil {
        return nil, fmt.Errorf("userip: %q is not IP:port", req.RemoteAddr)
    }
```

`NewContext`返回一个带有`userIP`的新`Context`：

```go
func NewContext(ctx context.Context, userIP net.IP) context.Context {
    return context.WithValue(ctx, userIPKey, userIP)
}
```

`FromContext` 从 `Context` 中提取 `userIP` ：

```go
func FromContext(ctx context.Context) (net.IP, bool) {
    // ctx.Value returns nil if ctx has no value for the key;
    // the net.IP type assertion returns ok=false for nil.
    userIP, ok := ctx.Value(userIPKey).(net.IP)
    return userIP, ok
}
```



## Package google

[google.Search](https://blog.golang.org/context/google/google.go) 函数向 [Google Web Search API](https://developers.google.com/web-search/docs/) 发出HTTP请求，并解析JSON编码结果。 它接受Context参数ctx，并且在ctx.Done关闭时立即返回。

Google Web Search API请求包括搜索查询和用户IP作为查询参数：

```go
func Search(ctx context.Context, query string) (Results, error) {
    // Prepare the Google Search API request.
    req, err := http.NewRequest("GET", "https://ajax.googleapis.com/ajax/services/search/web?v=1.0", nil)
    if err != nil {
        return nil, err
    }
    q := req.URL.Query()
    q.Set("q", query)

    // If ctx is carrying the user IP address, forward it to the server.
    // Google APIs use the user IP to distinguish server-initiated requests
    // from end-user requests.
    if userIP, ok := userip.FromContext(ctx); ok {
        q.Set("userip", userIP.String())
    }
    req.URL.RawQuery = q.Encode()
```

`Search` 使用一个辅助函数`httpDo` 来发出HTTP请求, 如果在处理请求或响应时关闭 `ctx.Done` ，取消 `httpDo` 。 `Search` 将传递闭包给 `httpDo` 来处理HTTP响应：

```go
    var results Results
    err = httpDo(ctx, req, func(resp *http.Response, err error) error {
        if err != nil {
            return err
        }
        defer resp.Body.Close()

        // Parse the JSON search result.
        // https://developers.google.com/web-search/docs/#fonje
        var data struct {
            ResponseData struct {
                Results []struct {
                    TitleNoFormatting string
                    URL               string
                }
            }
        }
        if err := json.NewDecoder(resp.Body).Decode(&data); err != nil {
            return err
        }
        for _, res := range data.ResponseData.Results {
            results = append(results, Result{Title: res.TitleNoFormatting, URL: res.URL})
        }
        return nil
    })
    // httpDo waits for the closure we provided to return, so it's safe to
    // read results here.
    return results, err
```

`httpDo` 函数发起HTTP请求，并在新的 `goroutine` 中处理其响应。 如果在 `goroutine` 退出之前关闭了`ctx.Done`，它将取消该请求：

```go
func httpDo(ctx context.Context, req *http.Request, f func(*http.Response, error) error) error {
    // Run the HTTP request in a goroutine and pass the response to f.
    c := make(chan error, 1)
    req = req.WithContext(ctx)
    go func() { c <- f(http.DefaultClient.Do(req)) }()
    select {
    case <-ctx.Done():
        <-c // Wait for f to return.
        return ctx.Err()
    case err := <-c:
        return err
    }
}
```



## Adapting code for Contexts

许多服务器框架提供用于承载请求范围值的包和类型。 可以定义 *Context* 接口的新实现，以便使得现有的框架和期望Context参数的代码进行适配。

例如，Gorilla的 [github.com/gorilla/context](http://www.gorillatoolkit.org/pkg/context) 包允许处理程序通过提供从HTTP请求到键值对的映射来将数据与传入的请求相关联。 在 [gorilla.go](https://blog.golang.org/context/gorilla/gorilla.go) 中，提供了一个 `Context` 实现，其 `Value` 方法返回与 Gorilla 包中的特定HTTP请求相关联的值。

其他软件包提供了类似于 *Context* 的取消支持。 例如，[Tomb](http://godoc.org/gopkg.in/tomb.v2) 提供了一种杀死方法，通过关闭死亡 `channel` 来发出取消信号。 [Tomb](https://godoc.org/gopkg.in/tomb.v2)还提供了等待 `goroutine` 退出的方法，类似于sync.WaitGroup。 在 [tomb.go](https://blog.golang.org/context/tomb/tomb.go) 中，提供一个 *Context* 实现，当其父 *Context* 被取消或提供的 *Tomb* 被杀死时，该 *Context* 被取消。如下：

```go
// +build OMIT

// Package tomb provides a Context implementation that is canceled when either
// its parent Context is canceled or a provided Tomb is killed.
package tomb

import (
	"golang.org/x/net/context"
	tomb "gopkg.in/tomb.v2"
)

// NewContext returns a Context that is canceled either when parent is canceled
// or when t is Killed.
func NewContext(parent context.Context, t *tomb.Tomb) context.Context {
	ctx, cancel := context.WithCancel(parent)
	go func() {
		select {
		case <-t.Dying():
			cancel()
		case <-ctx.Done():
		}
	}()
	return ctx
}
```



## Conclusion

在谷歌内部，我们要求Go语言的程序员传递`Context`参数作为在调用路径的输入输出请求的第一参数。这样有利于Go代码由不同的开发团队还可以良好协作。它可以轻松地跨越api边界，传递请求范围值，以及控制超时和取消调用。

基于Context构建的服务器框架需要实现Context接口中的方法，这样才能在服务器框架包和需要Context参数的包之间建立联系。Context通过建立一个请求作用域的数据访问以及请求终止功能的通用接口，使得开发可扩展服务变得更加容易。