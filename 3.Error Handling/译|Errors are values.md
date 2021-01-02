# 译|Errors are values



Go程序员经常讨论的一个点，尤其对于Go这门新的语言来说，就是怎么处理异常。每次讨论到下面这段代码的时候都会摇头叹息，众说纷纭：

```go
if err != nil {
    return err
}
```

在我们阅读的开源代码里面，上面代码片段出现频率相当高。



这对于个别程序员来说，是很容易被误导的，他们会对”如果进行异常处理“这个问题产生疑惑。在其他的语言，可能更多会使用try...catch块或者类似原理的思想去解决问题。他们会觉得go语言没有try...catch块来统一捕获异常，而是使用散落在代码各处的`if err != nil`，会让代码膨胀难以维护。



基于上面的分析，其实暴露出go程序员迷失了一个最基本的认识：**异常也是一个值**。

值是可以被构造控制的，既然异常也是一个值，所以它需要被构造控制。

当然把异常当作值类型的一个常见场景就是判断异常值类型是否为`nil`，我们还有数不尽的场景可以针对异常这个值类型进行处理，其中的一些例子还可以让你更好地编程，可以很大程度排除使用if语句检查错误的固定模式。

这是一个简单的示例,来自[bufio](https://golang.org/pkg/bufio/#pkg-overview) 包的[Scanner](http://golang.org/pkg/bufio/#Scanner)类型.它的 [Scan](http://golang.org/pkg/bufio/#Scanner.Scan)方法执行底层的I/O操作,显然它可能引起一个错误.然而[Scan](http://golang.org/pkg/bufio/#Scanner.Scan)方法并不会暴露错误.他返回一个布尔值,通过在[Scan](http://golang.org/pkg/bufio/#Scanner.Scan)运行之后执行的另一个方法来报告是否发生了错误.调用代码如下:

```go
scanner := bufio.NewScanner(input)
for scanner.Scan() {
    token := scanner.Text()
    // process token
}
if err := scanner.Err(); err != nil {
    // process the error
}
```

当然，也有一个对错误的`nil`检查,但是只出现和执行了一次。`Scan` 也可以这样定义：

```go
func (s *Scanner) Scan() (token []byte, error)
```

然后示例代码可能写成这样：

```go
scanner := bufio.NewScanner(input)
for {
    token, err := scanner.Scan()
    if err != nil {
        return err // or maybe break
    }
    // process token
}
```

代码没有很大不同，但是这里有一个重要的区别。在这段代码中，调用代码必须在每个迭代检查错误。但是在原始的`Scanner`API中.错误处理是从关键API抽象出来的。通过token迭代。使用原始的API客户端代码感觉更加自然：循环直到完成，然后再担心错误。错误处理不会干扰流程控制。

幕后发生了什么。 `Scan` 一旦发生I/O错误，他记录并返回`False`，另外一个方法 [Err](http://golang.org/pkg/bufio/#Scanner.Err) ，当调用代码请求时报告错误值。虽然这很普通但是和到处写 `if err != nil` 或者让调用代码在每个token后检查错误还是不同的.。这就是使用错误值编程，简单的编程。



在我出席2014年秋天东京的GoCon时，出现了错误检查代码的话题。Twitter上一个热情的gopher(@jaxk_)发出了同样的抱怨。他展示了一些类似下面的代码：

```go
_, err = fd.Write(p0[a:b])
if err != nil {
    return err
}
_, err = fd.Write(p1[c:d])
if err != nil {
    return err
}
_, err = fd.Write(p2[e:f])
if err != nil {
    return err
}
// and so on
```

非常重复，实际的代码更长,会有更多重复。所以不太容易只通过一个帮助函数进行重构。在理想情况下使用闭包对错误变量进行包装会有一些帮助。如下：

```go
var err error
write := func(buf []byte) {
    if err != nil {
        return
    }
    _, err = w.Write(buf)
}
write(p0[a:b])
write(p1[c:d])
write(p2[e:f])
// and so on
if err != nil {
    return err
}
```

这种方式有效，但是要求在进行写入操作的每个函数中都必须要有一个闭包函数,比起使用独立的帮助函数显得比较笨拙，因为`err`变量需要通过调用进行维护。

通过借鉴上面`Scan`方法的思路，我们可以让错误处理更清晰，更通用和可复用。在我们的讨论中提到这种方式，但是@jxck_ 不太明白怎么去应用它。因为有点语言障碍，经过长时间的交流，我问他是否可以借用他的笔记本来给他展示一些实际的代码。



我定义了一个叫做`errWriter`的类型，如下：

```go
 type errWriter struct {
    w   io.Writer
    err error
}
```

定义一个`write`方法。他不需要声明成标准的`Write`函数，另外一个明显的区别是他是小写的。`write`调用w的`Write`方法并记录第一个错误供后面使用。（值得学习）

```go
func (ew *errWriter) write(buf []byte) {
    if ew.err != nil {
        return
    }
    _, ew.err = ew.w.Write(buf)
}
```

一旦发生错误，`write`方法不会进行任何操作只会保存错误值。

基于`errWrite`类型和他的`write`方法。上面的代码可以重构为：

```go
ew := &errWriter{w: fd}
ew.write(p0[a:b])
ew.write(p1[c:d])
ew.write(p2[e:f])
// and so on
if ew.err != nil {
    return ew.err
}
```

和使用闭包相比，代码更整洁，并且更容易看到实际的写入代码段。没有杂乱的东西,通过对错误值和接口(interface)编程让代码更好。

在同一个包的一些其他代码片段也可以使用这种方式，甚至直接使用`errWriter`类型。

一旦存在`errWriter`，他就能做更多的事情,特别是用来减少人为的工作。他可以计算字节数，将多个写入内容收集到一个缓冲器中再一起发送。等等

实际上这种模式在标准库中经常出现[archive/zip](http://golang.org/pkg/archive/zip/)包 和[net/http](http://golang.org/pkg/net/http/)包使用了这种方式。特别是[bufio](http://golang.org/pkg/bufio/)包的，如下：

```go
b := bufio.NewWriter(fd)
b.Write(p0[a:b])
b.Write(p1[c:d])
b.Write(p2[e:f])
// and so on
if b.Flush() != nil {
    return b.Flush()
}
```



这种方法有一些明显的**缺点**，至少对某些应用场景是这样。我们无法知道错误发生时，我们的处理过程完成了多少。通常一个简单的检查已经足够，但是如果这个信息很重要，那么我们有必要进行一个细粒度的检查。（也就是无法定位失败在哪一步发生的，**一般业务代码需要打log**，这样的设计确实不太好，不过也能学到东西吧）



我们看到了避免重复的错误处理代码的一种方式，请记住`errWriter`或者`bufio.Writer`不是简化错误处理的唯一方法，而且也不是适用与所有情况。关键在于错误就是值，Go语言完全可以处理它们。

使用这个语言去简化你的错误处理。但是记住：无论怎么做，一定要检查你自己的错误！

