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

由于调用者会进行断言或者`switch`类型去描述错误，因此`error`类型必须是`public`属性。

如果你实现了契约的接口实现需要特定的`error`类型，所有接口的实现需要依赖定义`error`类型的包。对代码来说是个强耦合，和制作一个脆弱的API。



### Conclusion: avoid error types

虽然`error`类型比特定的哨兵`error`值要好，因为它们能够捕获更多的出现问题的上下文信息，但是`error`类型也会有`error`的很多问题。

因此我再一次的建议是避免`error`类型，或者说，至少避免让它们变成你的公共API。



## Opaque errors

现在我们到了异常处理的第三部分。在我看来是最方便的异常处理的方式，因为它在你的代码和调用者之间产生了最小化的耦合。

我称之为不透明的异常处理，因为当你知道有异常出现，你并不能看到内部的错误信息。对于调用方而言，能知道的操作的结果。

不透明的异常处理会返回不带有额外信息的错误类型。如下：

```go
import “github.com/quux/bar”

func fn() error {
        x, err := bar.Foo()
        if err != nil {
                return err
        }
        // use x
}
```

如上，`Foo`的契约并不保证会携带`error`的上下文类型进行返回。`Foo`的作者可以非常自由地给异常提供注释内容，而并不会打破调用方和被调用方的契约。



### Assert errors for behaviour, not type

在少数情况下，这样的一种处理异常的二元方法并不足够的。

例如，你的程序在对外部世界的交互，比如网络调用，需要调用者去判断这样的一个错误是否需要重试。（比如某些网络错误可以通过重试能够扭转成功，调用方需要更多的信息）

在这样的一个案例里面，与其定义出特定的错误类型或者错误值去标识，我们可以通过断言去实现一个特定的逻辑。考虑下面的例子：

```go
type temporary interface {
        Temporary() bool
}
 
// IsTemporary returns true if err is temporary.
func IsTemporary(err error) bool {
        te, ok := err.(temporary)
        return ok && te.Temporary()
}
```

我们能够传递任何的错误到`IsTemporary`去判断错误是否能够重试。

如果错误并没有实现`temporary`接口，那么，它就没有`Temporary`方法，说明错误是不可重试的。

如果错误实现了`temporary`接口，那么调用者如果发现`IsTemporary`返回true，那么说明错误是可重试的。

关键是这个逻辑可以不需要导入外部包能够被实现出来，我们无须感知其底层的错误，我们可以感兴趣去定义自己的行为。



## Don’t just check errors, handle them gracefully

这是我想讲的第二个关于go语言的谚语；不单单检查错误，优雅地处理他们。你能够针对下面的代码提出一些问题吗？

```go
func AuthenticateRequest(r *Request) error {
        err := authenticate(r.User)
        if err != nil {
                return err
        }
        return nil
}
```

其中一个显而易见的建议是上面5行的代码可以用下面一行来表示：

```go
return authenticate(r.User)
```

这应该是是在代码review的时候，每个人都很容易发现了。但是会有个问题，我没办法知道这个原始的错误来自哪里。

如果`authenticate`返回一个错误，那么`AuthenticateRequest`也会原封不动返回同样的错误给调用方。在程序的顶部，`main`函数会打印出这个错误到日志文件，将会被显示：`No  such file or directory`。你根本没办法知道这个错误来自哪里。

我们不能发现这个错误在哪个文件和哪一行产生的。没有调用堆栈去跟踪这个错误。需要花费很大功夫去检查这个错到底是从哪里抛出来的。

*Donovan and Kernighan’s The Go Programming Language* 建议你使用`fmt.Errorf`去为你的错误路径增加上下文，如下：

```go
func AuthenticateRequest(r *Request) error {
        err := authenticate(r.User)
        if err != nil {
                return fmt.Errorf("authenticate failed: %v", err)
        }
        return nil
}
```

但是如我们上面所看，这种模式与特定错误值和错误类型是不相容的，因为对于将错误转变为字符串，把它和其他字符串进行合并，以及转变回一个新的error，使用`fmt.Errorf`会打破原来的error上下文。



### Annotating errors

我建议对错误添加上下文信息的方法是，跟我接下来介绍的简单的包有关。这个代码在[`github.com/pkg/errors`](https://godoc.org/github.com/pkg/errors)。这个`errors`包有两个主要的函数：

```go
// Wrap annotates cause with a message.
func Wrap(cause error, message string) error
```

第一个函数是`Wrap`，可以携带现在已有的错误，和附加的信息`message`，会产生一个新的`error`。

```go
// Cause unwraps an annotated error.
func Cause(err error) error
```

第二个函数是`Cause`，会反过来，将一个现有的错误，去解包装，获取原始的错误。

因此，我们使用上面两个函数，重写的例子如下：

```go
func ReadFile(path string) ([]byte, error) {
        f, err := os.Open(path)
        if err != nil {
                return nil, errors.Wrap(err, "open failed")   // 包装出新的错误
        } 
        defer f.Close()
 
        buf, err := ioutil.ReadAll(f)
        if err != nil {
                return nil, errors.Wrap(err, "read failed")  //  包装出新的错误
        }
        return buf, nil
}
```

又比如，我们将实现一个函数，实现的功能是读取一个配置文件，然后`main`会调用的。

```go
func ReadConfig() ([]byte, error) {
        home := os.Getenv("HOME")
        config, err := ReadFile(filepath.Join(home, ".settings.xml"))
        return config, errors.Wrap(err, "could not read config")
}
 
func main() {
        _, err := ReadConfig()
        if err != nil {
                fmt.Println(err)
                os.Exit(1)
        }
}
```

如果读取配置文件发生错误，那么我们会得到一个注释性良好的错误：

```bash
could not read config: open failed: open /Users/dfc/.settings.xml: no such file or directory
```



同理，反过来，对于上面的`IsTemporary`，应该用`errors.Cause`重新包装。

```go
// IsTemporary returns true if err is temporary.
func IsTemporary(err error) bool {
        te, ok := errors.Cause(err).(temporary) // 解error包装
        return ok && te.Temporary()
}
```



## Only handle errors once

最后，我建议你应该只处理一次异常。处理它的异常包含检查它的值，以及做出处理。

```go
func Write(w io.Writer, buf []byte) {
        w.Write(buf)
}
```

如果你无法做出一个决定，你将会忽略掉这个错误。比如上面的例子，`w.Write(buf)`的异常会被忽略。

有时候面对单一错误做出不同的决定，也是有问题的。

```go
func Write(w io.Writer, buf []byte) error {
        _, err := w.Write(buf)
        if err != nil {
                // annotated error goes to log file
                log.Println("unable to write:", err)
 
                // unannotated error returned to caller
                return err
        }
        return nil
}
```

在这个例子，如果`Write`出现错误，会打印一条日志，打印出错误的文件数和行数，以及这个错误也会返回给调用者，调用者也极有可能也会打印日志，再返回，一直到最上游，这样的代码其实比较臃肿。

因此在你的日志文件会打印一系列的堆栈内容（日志行数），一次报错打印了好多次行。相当于你的报错堆栈在日志中，然后`main`顶层无法获取原始的`error`。

这样的方式最好是使用`errors`包，它给予你往错误值添加额外上下文信息的能力，以一种友好可读的方式去演示。

```go
func Write(w io.Write, buf []byte) error {
        _, err := w.Write(buf)
        return errors.Wrap(err, "write failed")
}
```



## Conclusion



总而言之，`errors`是你包的public API的一部分，尽可能关注他们，他们极有可能是你的包导出的一部分。

为了最大的灵活性，你应该处理你的异常视作不透明。以及，尽可能避免处理你的错误视作特定的类型或者值。

在你的程序中，尽可能简化你的特定的哨兵错误值的数量以及转化你的错误用一种不透明的方式（不侵入细节），通过使用`errors.Wrap`方法。

最后，使用`errors.Cause`当你在检查的时候，去恢复底层的错误。











