# 译|Context isn’t for cancellation



这是一篇有关Go语言中`context.Context`的使用，困难的经验之谈。

很多作者，包括我自己，都写了`context.Context`在未来的迭代中的使用，误用和它们怎么演进。针对`context.Context`的各个方面的争议很多，其中一个不约而同的点就是在`context.Context`接口中加入`Context.WithValue`方法（全局方法转化为接口的方法），作为上下文的资源的生命周期管理。很多建议便是在`context.Context`自己排名和重载出一系列的`WithValue`方法，去控制与请求相关的资源的生命周期。

很多类似的提案就因此涌现了，去让`context.Context`去重载一系列的数据和值。近似于tls(Thread Local Storage)的提案却不被接受。

这篇文章去探索`context.Context`和生命周期管理之间的关系，以及提出这样的问题：修复`Context.WithValue`是在错误的道路越走越远吗？



## Context是一个与请求作用域的范式

`context`包的文档说明强烈地推荐`context.Context`只能作为一个请求作用域的变量。

> 不要在一个struct类型里面存储Context类型的值；取而代之，显式地传递Context到每一个需要的函数里面去。Context应该是第一个函数参数，通常命名为ctx：

```go
func DoSomething(ctx context.Context, arg Arg) error {
        // ... use ctx ...
}
```

特别地，`context.Context`值应该位于函数参数的存活周期，而不是存放在一个struct的field中或者全局之中。这让`context.Context`只适用于请求作用域的相关生命周期。考虑go出自于服务端开发的血统，这是一个引人注目的用户案例。然而，还会存在其他的案例用于取消在一个请求之外的资源生命周期。例如，作为agent或者pipeline一部分的，在后台运行的goroutine。



## Context作为取消资源的一个钩子

`context`包规定的目标在于：

> context包定义出Context类型，携带上了deadlines，cancelation信号和其他横跨进程之间API边界请求相关的作用域值。

听起来是挺不错的，但它却掩盖了它笼统的本质。`Context`被用于三个独立，虽然有时候会相关的情景，如下：

+ Cancellation 来自于 `context.WithCancel`。
+ Timeout 来自于 `context.WithDeadline`。
+ A bag of values 来自于 `context.WithValue`。

在任何时候，一个`context.Context`值能够反映这三个情景中的一个或者多个问题。然而，`context.Context`最重要的功能，是广播一个取消的信号，是不完全的功能，因为没有一个方式能够等待到信号被确认接收。



## 回顾过去

作为一篇有经验的分享报告，这是有密切关系的总结出一些实际的经验。在2012年，Gustavo Niemeyer 写过一个生命周期管理的包叫`tomb`，被`juju`所使用来管理来自`juju`系统中，不同agent的worker协程。

`tomb.Tombs`只与生命周期的管理有关。更重要的是，这是一个生命周期的通用概念，而并不唯一地和一个请求，一个goroutine绑定在一起。资源的生命周期范围仅仅简单地和一个tomb值绑定在一起了。

一个`tomb.Tomb`值有三个这样的属性：

1. 标识tomb的拥有者去关闭资源的能力；
2. 等待，直到关闭信号被收到的能力；
3. 捕获最终`error`值的一种方式。

然而，`tomb.Tombs`有一个缺点，就是它们不能跨多个协程去进行分享。考虑到典型的网络服务器，`tomb.Tomb`并不能替代`sync.WaitGroup`的用途。

```go
func serve(l net.Listener) error {
        var wg sync.WaitGroup
        var conn net.Conn
        var err error
        for {
                conn, err = l.Accept()
                if err != nil {
                        break
                }
                wg.Add(1)
                go func(c net.Conn) {
                        defer wg.Done()
                        handle(c)
                }(conn)
        }
        wg.Wait()
        return err
}
```

为了公平起见，`context.Context`不能做这个因为它没有提供内建的接收取消资源的方式。我们所需要的是这样的一种形式，能够让`sync.WaitGroup`去支持资源的取消，相当于所有的并发处理中的`wg.Done`逻辑。



## Context应该，并且最好，就仅仅是一个上下文含义

`context.Context`类型的意图就在于它的名字：

> context这个名词，是用来形容特定的事件，状态或者想法，为了让别人充分理解而存在的一个东西。俗称上下文

我推崇`context.Context`仅仅变成一个请求相关的相关的列表上下文环境。

在`context.Context`中解耦出生命周期管理，作为一些列与请求相关的值存储将会是日后的关心课题。

最好，我们不需要等到Go 2.0才去探索类似`tomb`包的设计思路去解决上面这个问题。

