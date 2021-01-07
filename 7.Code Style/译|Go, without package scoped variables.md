# 译|Go, without package scoped variables

让我们进行一个思维实验，假如我们不能在包级别声明变量，go语言会变得怎么样？当我们删除包作用域的声明变量的时候，会产生什么影响？以及我们能够学习到go程序的设计吗？

我只探讨删除变量，和五个顶级声明。这样的情况仍然是被允许的，因为它们会在编译期间当做常量进行处理。当然，你还是可以在函数内部或者块内部声明变量。



## Why are package scoped variables bad?

首先，为什么包作用域的变量会这么糟糕呢？抛开这个问题，对于大量并发的语言属于可变的状态范围探讨，包作用域变量从根本上来说是个单例，有可能会导致在不相关的包之间交换了状态，鼓吹了紧密耦合，使得代码测试的时候相互依赖严重。

Peter Bourgon 最近写过：

> tl;dr: magic is bad; global state is magic → [therefore, you want] no package level vars; no func init.



## Removing package scoped variables, in practice

想要应用这个方法到测试里面，我调研了众多Go代码里面最受欢迎的代码例子；看看go的标准库，去了解包作用域变量是怎么被使用的，已经评估下对这个实验的影响。

### Errors

包作用域变量里面使用得最频繁的是`error`。比如：`io.EOF`，`sql.ErrNoRows`，`crypto/x509.ErrUnsupportedAlgorithm`，等等。删除包作用域的变量将会移除了使用了哨兵错误值的能力。但是，我们又能用什么方式去替代它们呢？

当上游检查错误的时候，我更倾向于用我之前文章的做法：[prefer behaviour over type or identity](https://dave.cheney.net/2016/04/27/dont-just-check-errors-handle-them-gracefully)。如果无法修改，则声明[error constants](https://dave.cheney.net/2016/04/07/constant-errors) 来去除对原来它们身份语义的潜在修改。

剩下的错误变量声明只会在传递一个错误信息的私有声明。这些错误值是非导出到包里面的，因此外部的调用者无法进行值的比较。在包级别声明错误，而不是在代码里面进行值比较。我推荐使用类似[`pkg/errors`](https://godoc.org/github.com/pkg/errors) 在错误出现的时候去捕获错误。

### Interface satisfaction assertions

接口的转换满足原语：

```go
var _ SomeInterface = new(SomeType)
```

至少在标准库出现19次。在我眼里，这样的断言只是测试。每次在你编译你的包的时候，它们不需要能够被编译，只会被淘汰。这样的逻辑应该放到`_test.go`文件当中。当我们进制包作用域的遍历，同样也禁止了这样的单元测试，那我们怎么能够保证测试正常执行呢？

一种方案是将变量的声明从包级别移动到函数级别，但是如果`SomeType`停止实现`SomeInterface`的话，还是会编译失败。

```go
func TestSomeTypeImplementsSomeInterface(t *testing.T) {
       // won't compile if SomeType does not implement SomeInterface
       var _ SomeInterface = new(SomeType)
}
```

但是，它事实上是一个单元测试，并不难把它重写成一个Go的标准测试。

```go
func TestSomeTypeImplementsSomeInterface(t *testing.T) {
       var i interface{} = new(SomeType)
       if _, ok := i.(SomeInterface); !ok {
               t.Fatalf("expected %t to implement SomeInterface", i)
       }
}
```



## It’s not all beer and skittles

上面的篇章说明的是避免包级别的变量声明是可行的，但其实在标准库的一些地方已经证明了某些地方不好去掉包作用域的变量。



### Real singletons

虽然我认为单例模式的作用是被夸大了，尤其是在它的注册代码上面看。但是事实上还有一些在每个程序都有单例变量。一个很好的例子就是`os.Stdout`和它的友元变量。

```go
package os 

var (
        Stdin  = NewFile(uintptr(syscall.Stdin), "/dev/stdin")
        Stdout = NewFile(uintptr(syscall.Stdout), "/dev/stdout")
        Stderr = NewFile(uintptr(syscall.Stderr), "/dev/stderr")
)
```

这样的声明还是会存在一些问题。首先，`Stdin`，`Stdout`和`Stderr`是`*os.File`类型，而不是它们属于它们各自的`io.Reader`或者`io.Writer`接口。有人在 [alternatives problematic](https://github.com/golang/go/issues/13473)讨论说应该把这几个变量调整成接口类型。然而，这并不好实施。

正像之前的错误码常量所说的，我们能够保持这些标准IO操作描述符的现状，比如像`log`包和`fmt`包能够直接寻址找到它们，但是避免像下面那样把它们作为包作用域去声明它们。

```go
package main

import (
        "fmt"
        "syscall"
)

type readfd int

func (r readfd) Read(buf []byte) (int, error) {
        return syscall.Read(int(r), buf)
}

type writefd int

func (w writefd) Write(buf []byte) (int, error) {
        return syscall.Write(int(w), buf)
}

const (
        Stdin  = readfd(0)
        Stdout = writefd(1)
        Stderr = writefd(2)
)

func main() {
        fmt.Fprintf(Stdout, "Hello world")
}
```



### Cache

第二个最常使用的，非导出包作用域的变量是`caches`。这里会包含两块尼尔：一个是由`map`分配的真实的缓存，另一块是像`sync.Pool`那样的改善编译开销的常量。

举个例子，`crypto/ecsda`包有一个`zr`类型支持传入一个buffer来支持`Read`方法。这个包引入了一个单例变量`zeroReader`因为它嵌入了`io.Reader`，每次在它实例化的时候都会发生内存逃逸。

```go
package ecdsa 

type zr struct {
        io.Reader
}

// Read replaces the contents of dst with zeros.
func (z *zr) Read(dst []byte) (n int, err error) {
        for i := range dst {
                dst[i] = 0
        }
        return len(dst), nil
}

var zeroReader = &zr{}
```

因为`zr`嵌入了`io.Reader`，它是个`io.Reader`类型。因此，尽管不使用`zr.Reader`，它还是占用了空间。如下例子：

```go
csprng := cipher.StreamReader{
                R: zr{},
                S: cipher.NewCTR(block, []byte(aesIV)),
        }
```





### Tables

最后经常看到的包作用域的变量是`tables`，比如我们在`unicode`，`crypto/*`和`math`看到的。这些tables的形式一般都是以整型数组的方式存在，或者少量的使用struct和map。

替代包作用域的变量需要语言级别的支持，在讨论[#20443](https://github.com/golang/go/issues/20443)有所体现。因此，从根本上来说，他们属于一种例外了。



## A bridge too far

尽管这篇文章是一个想法实验而已，但也很清晰说明了禁止所有的包作用域变量作为语言的准则来说，也是属于太残酷了。

然而，我还是相信这里还有一些值得参考的建议的，这些建议会让我们的代码更好。

+ 第一，包作用域公有的变量声明应该避免。这在其他语言里面也不算是什么争议性的讨论了。单例模式就是为了避免这个东西的产生，一个简单的公有变量被改变，它影响的范围可能是不可预料。
+ 第二，一旦公有变量的声明使用，这些变量的类型和它影响的作用域需要尽可能少。

私有变量声明和公有的变量声明只有细微的差别，但是如下模式需要被关注：

+ 私有变量带有公共的`setter`函数，我称之为注册函数。与公有变量相比会有同样的影响范围。他们应该引入的构造，比如使用[option function](https://dave.cheney.net/2014/10/17/functional-options-for-friendly-apis)。尽可能外部更少量的暴露。

最后，当你需要在你的程序中添加包作用域的变量时，慎重考虑。它将会是一个暴露给外部的总有接口。

