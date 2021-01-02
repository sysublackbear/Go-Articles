# 译|Don’t just check errors, handle them gracefully

这篇文章是我在日本东京的[GoCon spring conference](http://gocon.connpass.com/event/27521/) 的分享摘要。



## Errors are just values

我花了很长的时间在思考在Go程序中处理异常的最好方式。我真的希望能够想出一种简单的方式来处理异常，使得能够像数学，字母的方式去叫Go程序员死记硬背。

然而，我得出的结论是：并没有单一的方式去处理异常。取而代之，Go语言的异常处理能够分为三种核心策略。



## Sentinel errors

异常处理的第一类情况我称之为*sentinel errors*。

```go
if err == ErrSomething { … }
```

这个名字来源于计算机程序的实践，用于使用特定的值去进一步标识某个值。因此对于Go，我们去使用特定的错误标识特定的错误。

例如包含诸如`io.EOF`或者`syscall`包的底层错误，比如`syscall.ENOENT`。

甚至还会有其他的*sentinel errors*去标识一个未曾出现的错误，比如 `go/build.NoGoError, and path/filepath.SkipDir from path/filepath.Walk`。

使用哨兵值是不太灵活的异常处理策略，因为作者需要对比返回值和特定的值做等号判断。这会产生一个问题：当你想要提供更细分的场景，返回不同类型的错误值会打破等号判断的策略。

甚至我们使用`fmt.Errorf`去加入不同的错误提示的时候，也会打破等号测试的平衡。会需要开发去比较输出的`error.Error`方法，是否匹配特定内容。

### Never inspect the output of error.Error

顺便一提，我认为你应该从未检查过`error.Error`方法的输出数据。`error`包的`Error`方法更多是以人化属性存在，而不是代码。

这个字符串的内容属于日志文件中的内容，或者打印在屏幕上。你不应该检查它的内容而改变了程序的行为。

我知道有时候上述的也会不成立，有人在twitter提出，这个建议并不适用于编写测试。尽管如此，需要比较错误文案的描述，但是在我的观点，应该尽可能避免这种情况。



### Sentinel errors become part of your public API

如果你的public函数或者方法返回一个具体的error，那么这个error必须是public，当然也加好注释说明。这样能够增加你的API的表面积。

如果你的API定义了一个返回特定错误类型的接口，这个接口的所有实现比如受限返回那种特定类型的错误，即使它们能够提供更具有描述性的错误。

我们可以看下`io.Reader`。诸如`io.Copy`的函数需要一个reader的实现返回特定的信号`io.EOF`信号，但它不算一个错误。



### Sentinel errors create a dependency between two packages

到目前为止，*sentinel error value*带来的最大问题是它们会创造出了两个包的源码依赖，造成耦合。举个例子，检查一个错误示范等于`io.EOF`，你的代码必须导入`io`包。

这个特定的例子听起来也不算特别糟糕，因为它真的很常见，但是想象下在你的项目工程会存在很多包之间的耦合，你的包在检查错误的特定类型的时候，会导入特定的包。

在一个大型工程中如果玩弄这种模式，这样的一个坏设计会一直在我们的脑海里。



### Conclusion: avoid sentinel errors

因此，我的建议是：避免在你写的代码中使用*sentinel error values*。在标准库是存在这样的一些例子，但是这不是你应该模仿的一种模式。

如果有人叫你在你的包导出一个错误值，你应该礼貌地婉拒和建议一种可以替代的方案，这就是我接下来要讲的。



## Error types

错误类型是我关于Go异常处理要讨论的第二方面。

```go
if err, ok := err.(SomeType); ok { … }
```

错误类型是指你创建了实现`error`接口的特定类型。例如，`MyError`类型标记了文件名，行数以及信息去标记发生的错误。

```go
type MyError struct {
        Msg string
        File string
        Line int
}

func (e *MyError) Error() string { 
        return fmt.Sprintf("%s:%d: %s”, e.File, e.Line, e.Msg)
}

return &MyError{"Something happened", “server.go", 42}
```

由于`MyError`是一个`error`类型，调用者可以使用断言从错误中提取额外的上下文信息。

```go
err := something()
switch err := err.(type) {
case nil:
        // call succeeded, nothing to do
case *MyError:
        fmt.Println(“error occurred on line:”, err.Line)
default:
// unknown error
}
```

对于`error`值，`error`类型的一大提升是它具备包装一个潜在的错误去提供更多上下文信息的能力。

一个很好的例子就是`os.PathError`类型，它包装了潜在的下游错误。

```go
// PathError records an error and the operation
// and file path that caused it.
type PathError struct {
        Op   string
        Path string
        Err  error // the cause
}

func (e *PathError) Error() string
```



### Problems with error types







