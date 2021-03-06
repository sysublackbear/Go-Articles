# 译|Never start a goroutine without knowing how it will stop



在Go语言里面，goroutine是非常容易构造且有效率的运行的个体。Go runtime已经被写成了成千上万的各种例子。但是goroutine确实也会有占用内存的成本问题；你不能无限制的创建协程。



每次当你在代码里面使用`go`关键字去启动一个新的协程的时候，你必须非常清晰地知道，你创造的这个协程将会怎么样，在何时会退出。如果不确定的话·，这里将会是一个潜在的内存泄露的隐患点。



考虑如下的代码小片段：

```go
ch := somefunction()
go func() {
        for range ch { }
}()
```

这里新开辟了一个函数去for...range...管道。那么会有下面的疑问：

+ 这个协程将会在什么时候退出？它只有在`ch`被关闭的时候才会退出；
+ 怎么才会关闭？很难说，因为`ch`是通过函数`somefunction`返回的。也就是，这完全取决于`somefunction`的逻辑，`ch`也许根本没被关闭，导致协程泄露。



在你的代码里面，可能存在某些协程会一直运行直到你的程序退出，比如你可能会起一个后台协程去监听配置文件的变化，或者是在你的Server里面Accept协程。然而，尽管这些协程出现的足够稀有，以至于我并不认为他们属于一个例外，它们也得遵循上面的原则。

**因此**，每次当你在程序中使用`go`开辟新协程的时候，你应该考虑这些问题：你开辟的这个协程，将会在什么条件下，怎么触发下能够退出。

