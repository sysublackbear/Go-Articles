# 译|Go Concurrency Patterns: Pipelines and cancellation



## Introduction

Go 的并发原语可以轻松的的通过建立数据流pipeline来有效的利用I/O和多CPU。本文介绍了这种pipeline的一些例子，强调了当操作失败时的细节处理，以及错误后如何清理。



## What is a pipeline?

在Go中pipeline并没有明确的定义；它只是多种并发程序中的一种。通俗的讲，一个pipeline是由一系列channel连接的*stages*，其中每个stage是一组运行相同函数的goroutine。在每个stage中，goroutine：

+ 通过输入channel从上游接收值；
+ 对这些数据执行某些函数，通常是生成一些新的值；
+ 通过输出channel发送值到下游。

每个阶段有任意数量的输入输出`channel`，除了第一个和最后阶段，分别只有输出或输入`channel`。第一个阶段有时被称为*source*或*producer*；最后阶段称为*sink*或*consumer*。

我们将开始通过一个简单的pipeline的例子来阐述这个思想和技巧。稍后，我们会介绍一个更贴近现实的例子。



## Squaring numbers

思考一个有三个stage的pipeline。

第一个stage，`gen`，是一个转换一个integer的list并将list中的integer发送到一个channel中的函数。这个`gen`函数启动一个goroutine将integer发送到channel并且当所有值发送完后关闭channel，如下：

```go
func gen(nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        for _, n := range nums {
            out <- n
        }
        close(out)
    }()
    return out
}
```

第二个stage，`sq`，从一个channel中接收integer，将每个integer平方后发送到一个channel中并返回这个channel。在输入channel关闭并将这个阶段的所有值发送到下游后，关闭输出channel：（注意这里for...range.. channel需要先把channel关闭掉）

```go
func sq(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        for n := range in {
            out <- n * n
        }
        close(out)
    }()
    return out
}
```

在`main`函数中建立pipeline并运行最后一个stage：它从第二个stage中接收值并显示出每一个，直到channel被关闭：

```go
func main() {
    // Set up the pipeline.
    c := gen(2, 3)
    out := sq(c)

    // Consume the output.
    fmt.Println(<-out) // 4
    fmt.Println(<-out) // 9
}
```

因为`sq`有相同类型的输入输出channel，我们可以任意次数的组合他们。我们还可以在`main`中重写一个范围循环，像这样的一组stage：

```go
func main() {
    // Set up the pipeline and consume the output.
    for n := range sq(sq(gen(2, 3))) {
        fmt.Println(n) // 16 then 81
    }
}
```



## Fan-out, fan-in

多个函数可以从相同的 channel 中读取，直到 channel 关闭，这称为：`fan-out`。这个提供了一个途径，分发工作给一组可以并行利用 CPU 和 I/O 的 workers 。 一个函数可以从多个输入读取，然后处理直到所有的都关闭，然后关闭单个 channel。这称为 `fan-in`。 我们可以改变我们的管道，运行两个 `sq` 的实例，都从相同的 input channel 读取。我们引入一个新的函数 `merge` 来 `fan-in` 这个结果：

```go
func main() {
    in := gen(2, 3)

    // Distribute the sq work across two goroutines that both read from in.
    c1 := sq(in)
    c2 := sq(in)

    // Consume the merged output from c1 and c2.
    for n := range merge(c1, c2) {
        fmt.Println(n) // 4 then 9, or 9 then 4
    }
}
```

这个`merge`函数通过一个goroutine来转换一个channel的list到一个channel来把每个输入channel中的值复制到唯一的输出channel中。一旦所有的`output`goroutine都启动了，`merge`启动另一个gorotine在在这个channel发送完成后关闭这个输出channel。

发送到一个已经关闭的channel会引起panic，所以重要的是要确保在关闭前都已经发送完。[`sync.WaitGroup`](http://golang.org/pkg/sync/#WaitGroup)类型提供了一种简单的方法来安排这种同步。

```go
// fan-in 示例
func merge(cs ...<-chan int) <-chan int {
    var wg sync.WaitGroup // HL
    out := make(chan int)

    // Start an output goroutine for each input channel in cs.  output
    // copies values from c to out until c is closed, then calls wg.Done.
    output := func(c <-chan int) {
        for n := range c {
            out <- n
        }
        wg.Done() // HL
    }
    wg.Add(len(cs)) // HL
    for _, c := range cs {
        go output(c)
    }

    // Start a goroutine to close out once all the output goroutines are
    // done.  This must start after the wg.Add call.
    go func() {
        wg.Wait() // HL
        close(out)
    }()
    return out
}
```



## Stopping short

这是我们pipeline功能的一个模式：

- 当所有发送操作结束后stages关闭他们的输出channels。
- stages持续的从输入channels中接收值直到这些channel都关闭。

该模式允许每个接收stage都写成`range`循环，并且一旦所有的值都顺利的发送到下游将确保所有的goroutine退出。

但在实际的pipeline中，stages并不总是获取所有的输入值。经常是因为这样的设计：接收着在进行下一步只需要这些值中的子集。更多的时候，当一个stage因为一个输入的值提早结束时意味着在前面的stages中出现了一个错误。在这些情况下接收者都不用等待剩下的值的到来，并且我们希望前面的stages停止值的生产但是后面的stages并不需要这样。

在我们的pipeline例子中，如果一个stage在处理所有的输入值时发生错误，这些goroutines试图发送这些值的时候会无限期的阻塞。

```go
    // Consume the first value from the output.
    out := merge(c1, c2)
    fmt.Println(<-out) // 4 or 9
    return
    // Since we didn't receive the second value from out,
    // one of the output goroutines is hung attempting to send it.
		// 比如上面的例子,加入某个goroutine发生异常，导致out这个channel无法close，会陷入无限阻塞。
		// 放入两个数，但是只接受了一个，会一直阻塞在这，发生协程泄露
}
```

这是个资源泄露：goroutines消耗内存和运行时资源，以及在 goroutine stacks 保存数据的 heap 引用，而不能被垃圾回收。 Goroutines 不会被垃圾回收，他们必须自己退出。（**goroutine不会自动退出**）

我们需要为管道的上游 stages安排退出，即使在下游 stages 出错未能接受所有的 `inbound values`。 一种方法是将输出channel改为带缓冲的。缓冲可以容纳固定数量的值；如果缓冲区中有足够的空间的话发送操作可以立即完成：

```go
c := make(chan int, 2) // buffer size 2
c <- 1  // succeeds immediately
c <- 2  // succeeds immediately
c <- 3  // blocks until another goroutine does <-c and receives 1
```

当要发送的值的数量在创建channel时是已知的话，缓冲可以简化代码。例如，我们可以重写`gen`来复制integer list到一个带缓冲的channel中并且避免产生新的goroutine：（带缓冲长度的管道）

```go
func gen(nums ...int) <-chan int {
    out := make(chan int, len(nums))
    for _, n := range nums {
        out <- n
    }
    close(out)
    return out
}
```

在我们的pipeline中返回一个阻塞的goroutine，我们或许可以考虑为`merge`中返回的输出channel增加一个缓冲：

```go
func merge(cs ...<-chan int) <-chan int {
    var wg sync.WaitGroup
    out := make(chan int, 1) // enough space for the unread inputs
    // ... the rest is unchanged ...
```

虽然这个处理了程序中被阻塞的 goroutine ，但这是 bad code。这里选择长度为1的 buffer，依赖于知道 merge 将要接收值的数量，和知道下游 stage 消耗多少。这个非常脆弱：如果我们多传递一个值给 gen ，或者下游读取更少的值，那么就又会阻塞 goroutine了。 我们得找到一个替换方法，使得下游 stage 告诉发送者，我们将要停止接收输入了。



## Explicit cancellation

当`main`在没有从`out`中接收完所有值时决定结束，它必须告诉上游的stage中的goroutines，让其放弃正在试图发送的值。通过向一个叫`done`的channel发送一个值来做这件事。它发送两个值因为可能有两个要阻止的发送者，如下：

```go
func main() {
    in := gen(2, 3)

    // Distribute the sq work across two goroutines that both read from in.
    c1 := sq(in)
    c2 := sq(in)

    // Consume the first value from output.
    done := make(chan struct{}, 2) // HL
    out := merge(done, c1, c2)
    fmt.Println(<-out) // 4 or 9

    // Tell the remaining senders we're leaving.
    done <- struct{}{} // HL
    done <- struct{}{} // HL
}
```

然后，我们在`merge`中引入对`done`的逻辑：

```go
func merge(done <-chan struct{}, cs ...<-chan int) <-chan int {
    var wg sync.WaitGroup
    out := make(chan int)

    // Start an output goroutine for each input channel in cs.  output
    // copies values from c to out until c is closed or it receives a value
    // from done, then output calls wg.Done.
    output := func(c <-chan int) {
        for n := range c {
            select {
            case out <- n:
            case <-done:  // 引入done处理
            }
        }
        wg.Done()
    }
    // ... the rest is unchanged ...
```

上面这种方法有个问题：*每个*下游的接收者需要知道上游可能阻塞的发送者的数量，然后安排这些发送至提前退出的信号。持续追踪这些数量既繁琐又容易出错。

我们需要一种方法来告诉未知的和位置数量的goroutine去停止向下游发送值。在Go中，我们可以通过关闭channel来做到这点，因为[在一个关闭的channel上做接收操作总会立刻执行，会产生一个零值的元素。](http://golang.org/ref/spec#Receive_operator)

这意味着`main`可以简单通过关闭`done`channel来为所有发送者解除阻塞。这种关闭是一种有效的向发送者的广播。我们扩展pipeline的*每个*函数为其增加一个`done`的参数，并在通过 defer 语句来处理关闭操作，所以所有从 main 出发的返回路径，都会通过给管道 stages 信号来退出。

**优化**：不主动定义带缓冲长度的channel，转移改成`close`管道进行通知。如下：

```go
func main() {
    // Set up a done channel that's shared by the whole pipeline,
    // and close that channel when this pipeline exits, as a signal
    // for all the goroutines we started to exit.
    done := make(chan struct{})  // 定义不带缓冲的管道
    defer close(done)   // 这里是关键

    in := gen(done, 2, 3)

    // Distribute the sq work across two goroutines that both read from in.
    c1 := sq(done, in)
    c2 := sq(done, in)

    // Consume the first value from output.
    out := merge(done, c1, c2)
    fmt.Println(<-out) // 4 or 9

    // done will be closed by the deferred call.
}
```

现在，我们所有的管道 stages 都可以在 done 关闭时随意返回。在 merge 中的 output 程序可在不 `draining` 他的 `inbound channel`的情况下返回，因为她知道上游的 sender `sq` 会在 done 关闭时停止发送。output 保证了 `wg.Done` 会在所有的返回路径中被 defer 语句调用。

`merge`函数处理如下，其中这里做出的优化点为主动`wg.Done()`转化为`defer wg.Done()`。

```go
func merge(done <-chan struct{}, cs ...<-chan int) <-chan int {
    var wg sync.WaitGroup
    out := make(chan int)

    // Start an output goroutine for each input channel in cs.  output
    // copies values from c to out until c or done is closed, then calls
    // wg.Done.
    output := func(c <-chan int) {
        defer wg.Done() // 关键
        for n := range c {
            select {
            case out <- n:
            case <-done:
              return // HL  // 直接返回了，wg.Done()放在defer中
            }
        }
    }
    // ... the rest is unchanged ...
```

类似的， `sq` 可以在 `done` 关闭时立刻返回。 `sq` 保证他的 out channel 在所有返回路径上通过 `defer` 语句关闭。如下：

```go
func sq(done <-chan struct{}, in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)  // 放在defer中
        for n := range in {
            select {
            case out <- n * n:
            case <-done:
                return
            }
        }
    }()
    return out
}
```

总结下来，下面是构建管道的几个建议：

+ 只有当所有的发送操作都结束时，stages 才关闭他们的 outbound channel。
+ 只要这些 inbound channels 不关闭，或者发送者未阻塞， stages 就会一直保持从 inbound channels 接收值。 管道通过保障所有的发送的值有足够的 buffer ，或者当接收者可能放弃这个 channel 时，从 inbound channels 显式地给发送者传递信号，来使得发送者不被阻塞。（比如`close`）



## Digesting a tree

让我们考虑一个更为实际的管道构建例子。

MD5是一个对文件校验优秀的信息摘要算法。该命令行实用程序`md5sum`可以打印出文件列表中的摘要值。

```bash
% md5sum *.go
d47c2bbc28298ca9befdfbc5d3aa4e65  bounded.go
ee869afd31f83cbb2d10ee81b2b831dc  parallel.go
b88175e65fdcbc01ac08aaf1fd9b5e96  serial.go
```

我们的例子就像是`md5sum`但是替换为一个单独的目录作为参数并打印出该目录下的每个文件的摘要值，按路径名排序。

```bash
% go run serial.go .
d47c2bbc28298ca9befdfbc5d3aa4e65  bounded.go
ee869afd31f83cbb2d10ee81b2b831dc  parallel.go
b88175e65fdcbc01ac08aaf1fd9b5e96  serial.go
```

我们程序中的主函数调用一个辅助函数`MD5All`，返回一个路径名和摘要值的map，然后排序输出结果：

```go
func main() {
    // Calculate the MD5 sum of all files under the specified directory,
    // then print the results sorted by path name.
    m, err := MD5All(os.Args[1])
    if err != nil {
        fmt.Println(err)
        return
    }
    var paths []string
    for path := range m {
        paths = append(paths, path)
    }
    sort.Strings(paths)
    for _, path := range paths {
        fmt.Printf("%x  %s\n", m[path], path)
    }
}
```

函数 `MD5All` 是我们主要讨论的，在 serial.go 中，这个实现没有使用并发，简单的对目录树中的每个文件读取并求合。

```go
// MD5All reads all the files in the file tree rooted at root and returns a map
// from file path to the MD5 sum of the file's contents.  If the directory walk
// fails or any read operation fails, MD5All returns an error.
func MD5All(root string) (map[string][md5.Size]byte, error) {
    m := make(map[string][md5.Size]byte)
    err := filepath.Walk(root, func(path string, info os.FileInfo, err error) error {
        if err != nil {
            return err
        }
        if !info.Mode().IsRegular() {
            return nil
        }
        data, err := ioutil.ReadFile(path)
        if err != nil {
            return err
        }
        m[path] = md5.Sum(data)
        return nil
    })
    if err != nil {
        return nil, err
    }
    return m, nil
}
```



## Parallel digestion

在[parallel.go](http://blog.golang.org/pipelines/parallel.go)中，我们把`MD5All`分割到一个两个stage的pipeline中。在第一个stage，`sumFiles`，walks the tree，起一个新的goroutine中摘要每个文件，然后发送类型为`result`的结果到channel中，`result`结构定义如下：

```go
type result struct {
    path string
    sum  [md5.Size]byte
    err  error
}
```

`sumFiles`返回两个channel：一个是`results`另一个是`filepath.Walk`返回的错误。这个`walk`函数启动一个新的goroutine来处理每个检查的文件，然后检查`done`。如果`done`关闭了，walk则立刻停止：

```go
func sumFiles(done <-chan struct{}, root string) (<-chan result, <-chan error) {
    // For each regular file, start a goroutine that sums the file and sends
    // the result on c.  Send the result of the walk on errc.
    c := make(chan result)
    errc := make(chan error, 1)
    go func() {
        var wg sync.WaitGroup
        err := filepath.Walk(root, func(path string, info os.FileInfo, err error) error {
            if err != nil {
                return err
            }
            if !info.Mode().IsRegular() {
                return nil
            }
            wg.Add(1)
            go func() {
                data, err := ioutil.ReadFile(path)
                select {
                case c <- result{path, md5.Sum(data), err}:
                case <-done:
                }
                wg.Done()
            }()
            // Abort the walk if done is closed.
            select {
            case <-done:
                return errors.New("walk canceled")
            default:
                return nil
            }
        })
        // Walk has returned, so all calls to wg.Add are done.  Start a
        // goroutine to close c once all the sends are done.
        go func() {
            wg.Wait()
            close(c)
        }()
        // No select needed here, since errc is buffered.
        errc <- err
    }()
    return c, errc
}
```

`MD5All`从`c`中接受摘要值。`MD5All`发生错误时则提前返回，在`defer`中关闭`done`：

```go
func MD5All(root string) (map[string][md5.Size]byte, error) {
    // MD5All closes the done channel when it returns; it may do so before
    // receiving all the values from c and errc.
    done := make(chan struct{})
    defer close(done)

    c, errc := sumFiles(done, root)

    m := make(map[string][md5.Size]byte)
    for r := range c {
        if r.err != nil {
            return nil, r.err
        }
        m[r.path] = r.sum
    }
    if err := <-errc; err != nil {
        return nil, err
    }
    return m, nil
}
```



## Bounded parallelism

MD5All 是在 [[parallel.go\]](https://link.zhihu.com/?target=https%3A//blog.golang.org/pipelines/parallel.go) 中实现，为每个文件开始一个 goroutine。在一个有很多大文件的目录中，这个会分配很多内存。

我们可以限制分配的并行读取文件的数量。在[bounded.go](http://blog.golang.org/pipelines/bounded.go)中，我们通过创建固定数量的goroutine来读取文件。现在我们的pipeline有着三个stage：walk the tree，读取文件并摘要，汇总摘要。第一个stage，`walkFiles`发送了目录树中普通文件的路径：

```go
func walkFiles(done <-chan struct{}, root string) (<-chan string, <-chan error) {
    paths := make(chan string)
    errc := make(chan error, 1)
    go func() {
        // Close the paths channel after Walk returns.
        defer close(paths)
        // No select needed for this send, since errc is buffered.
        errc <- filepath.Walk(root, func(path string, info os.FileInfo, err error) error {
            if err != nil {
                return err
            }
            if !info.Mode().IsRegular() {
                return nil
            }
            select {
            case paths <- path:
            case <-done:
                return errors.New("walk canceled")
            }
            return nil
        })
    }()
    return paths, errc
}
```

第二个 stage 开始了一个固定数量的 `digester goroutines` 来接受文件名，然后发送结果到`channel` c：

```go
func digester(done <-chan struct{}, paths <-chan string, c chan<- result) {
    for path := range paths {
        data, err := ioutil.ReadFile(path)
        select {
        case c <- result{path, md5.Sum(data), err}:
        case <-done:
            return
        }
    }
}
```

不像我们前面的例子， `digester`不会关闭他的输出 channel，因为多个 goroutines 发送给一个共享 channel。所以，MD5All 中会在所有的 digesters 完成之后关闭 channel。

```go
    // Start a fixed number of goroutines to read and digest files.
    c := make(chan result)
    var wg sync.WaitGroup
    const numDigesters = 20
    wg.Add(numDigesters)
    for i := 0; i < numDigesters; i++ {
        go func() {
            digester(done, paths, c)
            wg.Done()
        }()
    }
    go func() {
        wg.Wait()
        close(c)
    }()
```

我们可以把每个digester改成创建并返回自己的输出channel，但是我们需要添加工多的goroutine来fan-in结果。

在最后一个stage在从`errc`检查错误时从`c`中接收所有的`results`。这个检查不能过早进行，因为在这之前，`walkFiles`可能会阻塞发送到下游的值：

```go
    m := make(map[string][md5.Size]byte)
    for r := range c {
        if r.err != nil {
            return nil, r.err
        }
        m[r.path] = r.sum
    }
    // Check whether the Walk failed.
    if err := <-errc; err != nil {
        return nil, err
    }
    return m, nil
}
```

其实可以考虑使用b站封装的`errgroup`包去套用，使用起来也很方便。





## Conclusion

这篇文章展现了在 Go 中构造流数据管道的技术。处理在这种管道中的失败比较 tricky，因为管道中的每个 stage都可能在向下游发送数据时阻塞，而下游 stage 可能不在关心进来的数据。我们展现了如何在关闭一个 channel 时广播 “done” 信号给所有由管道开始的 goroutines ，也指导了正确的构造管道。











