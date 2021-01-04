

## Introduction

当您添加新功能、更改行为和重新考虑模块公共表面的部分时，您的模块将随着时间的推移而进化。正如 [Go Modules: v2 and Beyond](https://blog.golang.org/v2-go-modules) 中所讨论的，对 v1+ 模块的中断性更改必须作为主要版本的一部分(或者采用一个新的模块路径)。

然而，发布一个新的主要版本对您的用户来说是很困难的。他们必须找到新版本，学习新 API，并修改代码。有些用户可能永远不会更新，这意味着您必须永远维护代码的两个版本。因此，通常最好以兼容的方式更改现有的包。

在这篇文章中，我们将探讨一些引入非破坏性变更的技巧。常见的主题是：添加、不更改或删除。我们还将从一开始就讨论如何设计您的 API 以实现兼容性。



## Adding to a function

通常情况下，对函数的破坏性变更以函数的新参数的形式出现。我们将描述一些处理这种变化的方法，但首先让我们看看一种不起作用的技术。

当添加带有合理默认值的新参数时，很容易将它们添加为可变参数。如扩展如下函数：

```go
func Run(name string)
```

如果使用`size`默认为零的额外参数，则可能会建议：

```go
func Run(name string, size ...int)
```

理由是所有现有的调用都将继续工作。虽然这是真的，但 `Run` 的其他用途可能会中断，例如：

```go
package mypkg
var runner func(string) = yourpkg.Run
```

原始的 `Run` 函数在这里能正常工作，因为它的类型是 `func(String)`，但是新的 `Run` 函数的类型是`func(string, ...int)`，所以赋值在编译时失败。

此示例说明，对于向后兼容性而言，只满足调用兼容性是不够的。事实上，您不能对函数的签名进行向后兼容的更改。

与其更改函数的签名，不如添加一个新函数。例如，在引入 `context` 包之后，将 `context.Context` 作为第一个参数传递给函数已成为一种常见的做法。但是，稳定的 API 不能将导出的函数更改为接受`context.Context`，因为它会破坏该函数的所有使用。

相反，可以增加新的函数。例如，`database/sql` 包的 `Query` 方法的签名是(现在仍然是)：

```go
func (db *DB) Query(query string, args ...interface{}) (*Rows, error)
```

在创建了 `context` 包后，Go 团队向 `database/sql` 包添加了一个新方法：

```go
func (db *DB) QueryContext(ctx context.Context, query string, args ...interface{}) (*Rows, error)
```

为了避免复制代码，旧方法调用新代码：

```go
func (db *DB) Query(query string, args ...interface{}) (*Rows, error) {
    return db.QueryContext(context.Background(), query, args...)
}
```

添加一个方法允许用户以自己的节奏迁移到新的 API。由于这些方法以类似的方式读取和排序，并且 `Context` 位于新方法的名称中，所以 `database/sql` API 的这个扩展并没有降低包的可读性或理解性。

如果您预期一个函数将来可能需要更多的参数，您可以提前计划将可选参数作为函数签名的一部分。最简单的方法是添加一个 struct 参数，就像 `crypto/tls.Dial` 函数所做的那样：

```go
func Dial(network, addr string, config *Config) (*Conn, error)
```

由 Dial 进行的 TLS 握手需要一个网络和地址，但它有许多其他参数，具有合理的缺省值。传递 `nil` 对于 `config` 将使用这些默认值；通过设置了一些字段的构造结构将覆盖这些字段的默认值。将来，添加一个新的 TLS 配置参数只需要在 `Config` 结构上添加一个新字段，这是一个向后兼容的更改(几乎总是–请参阅下面的“维护结构兼容性”)。

有时，添加新函数和添加选项的技术可以通过将选项构造为方法接收器来组合。考虑 `net` 包侦听网络地址的能力的演变。在 Go 1.11 之前，`net` 包只提供了一个带有签名的侦听函数。

```go
func Listen(network, address string) (Listener, error)
```

对于 Go 1.11，在 net 侦听中添加了两个特性：传递上下文，并允许调用方提供一个“控制函数”，以便在创建后但在绑定之前调整原始连接。结果可能是一个新的函数，它具有上下文、网络、地址和控制功能。相反，包的作者添加了 [ListenConfig](https://pkg.go.dev/net@go1.11?tab=doc#ListenConfig) 结构体，因为有一天可能需要更多的选项。他们并没有用一个累赘的名称定义一个新的顶级函数，而是在 `ListenConfig` 结构中添加了一个 `Listen` 方法：

```go
type ListenConfig struct {
    Control func(network, address string, c syscall.RawConn) error
}

func (*ListenConfig) Listen(ctx context.Context, network, address string) (Listener, error)
```

为将来提供新选项的另一种方法是“Option types”模式，其中选项作为变量参数传递，并且每个选项都是一个函数，可以更改正在构造的值的状态。他们被描述在更详细的 Rob Pike 的帖子 [Self-referential functions and the design of options](https://commandcenter.blogspot.com/2014/01/self-referential-functions-and-design.html)。一个被广泛使用的例子是 [google.golang.org/grpc](https://pkg.go.dev/google.golang.org/grpc?tab=doc) 的 [DialOption](https://pkg.go.dev/google.golang.org/grpc?tab=doc#DialOption)。

在函数参数中，选项类型履行与 struct 相同的角色：它们是一种可扩展的传递行为修改配置的方法。决定选择哪一个在很大程度上取决于风格。考虑 gRPC 的拨号选项类型的简单用法：

```go
grpc.Dial("some-target",
  grpc.WithAuthority("some-authority"),
  grpc.WithMaxDelay(time.Second),
  grpc.WithBlock())
```

这也可以作为一种结构选项来实现：

```go
notgrpc.Dial("some-target", &notgrpc.Options{
  Authority: "some-authority",
  MaxDelay:  time.Minute,
  Block:     true,
})
```

函数选项有一些缺点：它们需要在每次调用的选项之前写入包名；它们增加了包命名空间的大小；如果提供相同的选项两次，则不清楚行为应该是什么。另一方面，采用选项结构的函数需要一个几乎总是为零的参数，有些人认为这个参数没有吸引力。当一个类型的零值有一个有效的含义时，指定该选项应该有它的默认值，这种设计欠佳，通常需要一个指针或一个额外的布尔字段。

这两种方法都是确保模块公共 API 未来可扩展性的合理选择。



## Working with interfaces

有时，新特性需要对公开的接口进行更改：例如，需要用新方法扩展接口。直接添加到接口是一个破坏性的变化，但是，我们如何在公开的接口上支持新方法呢？

基本思想是**用新方法定义一个新接口**，**然后在使用旧接口的地方**，**动态检查所提供的类型是旧类型还是新类型**。

让我们用 archive/tar 包中的一个示例来说明这一点。 [tar.NewReader](https://pkg.go.dev/archive/tar?tab=doc#NewReader) 接收 [io.Reader](https://golang.org/pkg/io/#Reader)，但随着时间的推移，Go 团队意识到，如果可以调用 `Seek`，从一个文件头跳到下一个文件头会更有效。但是，他们无法将 `Seek` 方法添加到 `io.Reader`：这将破坏 `io.Reader`。

另一个被排除的选择是改变 `tar.NewReader` 接收 io.ReadSeeker 而不是 io.Reader，因为它支持两者io.Reader 的方法和 Seek（通过 io.Seeker)。但是，如上所述，更改函数签名也是一个破坏性的更改。

所以，他们决定 `tar.NewReader` 签名未更改，但在 `tar.Reader` 的方法中对 `io.Seeker` 进行类型检查 ：

```go
package tar

type Reader struct {
  r io.Reader
}

func NewReader(r io.Reader) *Reader {
  return &Reader{r: r}
}

func (r *Reader) Read(b []byte) (int, error) {
  if rs, ok := r.r.(io.Seeker); ok {
    // Use more efficient rs.Seek.
  }
  // Use less efficient r.r.Read.
}
```

有关实际代码，请参见 [reader.go](https://github.com/golang/go/blob/60f78765022a59725121d3b800268adffe78bde3/src/archive/tar/reader.go#L837)。

当您遇到要向现有接口添加方法的情况时，您可以遵循此策略。首先用新方法创建一个新接口，或者用新方法标识现有接口。接下来，确定需要支持它的相关函数，为第二个接口键入check，并添加使用它的代码。

这种策略只在不使用新方法的旧接口仍然受支持的情况下有效，这限制了模块未来的可扩展性。

在可能的情况下，最好完全避免这类问题。例如，在设计构造函数时，更喜欢返回具体类型。与接口不同，使用具体类型可以在将来添加方法而不会破坏用户。该属性允许您的模块在将来更容易扩展。

提示：如果您确实需要使用一个接口，但不想让用户实现它，您可以添加一个未导出的方法。这可以防止在包外定义的类型在不嵌入的情况下满足接口要求，从而使您可以在以后添加方法而不会破坏用户实现。例如，请参见 [testing.TB’s private()](https://github.com/golang/go/blob/83b181c68bf332ac7948f145f33d128377a09c42/src/testing/testing.go#L564-L567) 函数。

```go
type TB interface {
    Error(args ...interface{})
    Errorf(format string, args ...interface{})
    // ...

    // A private method to prevent users implementing the
    // interface and so future additions to it will not
    // violate Go 1 compatibility.
  	// 定义出一个私有方法，不希望用户去实现这个接口。
    private()
}
```

这个主题在 Jonathan Amsterdam 的“Detecting Incompatible API Changes” 演讲中也有更详细的探讨([视频](https://www.youtube.com/watch?v=JhdL5AkH-AQ)、[幻灯片](https://github.com/gophercon/2019-talks/blob/master/JonathanAmsterdam-DetectingIncompatibleAPIChanges/slides.pdf))。

## Add configuration methods

到目前为止，我们已经讨论了公开的中断更改，在这种情况下，更改类型或函数会导致用户的代码停止编译。但是，行为更改也会破坏用户，即使用户代码继续编译。例如，许多用户期望 [`json.decoder`](https://pkg.go.dev/encoding/json#Decoder) 忽略 JSON 中不存在于参数结构中的字段。当 Go 团队想在这种情况下返回一个错误时，他们必须小心。如果没有选择机制，就意味着许多依赖这些方法的用户可能会收到以前没有的错误。

因此，他们没有更改所有用户的行为，而是向 `Decoder` 结构体：[`Decoder.DisallowUnknownFields`](https://pkg.go.dev/encoding/json#Decoder.DisallowUnknownFields) 中添加了一个配置方法。调用此方法会选择用户加入新行为，但不这样做会保留现有用户的旧行为。

## Maintaining struct compatibility

我们在上面看到，对函数签名的任何更改都是破坏性的更改。使用 structs 的情况要好得多。如果您有一个导出的结构类型，您几乎总是可以添加一个字段或删除一个未导出的字段，而不会破坏兼容性。添加字段时，请确保其零值有意义并保留旧的行为，以便不设置新字段的现有代码能够继续工作。

回想一下，net 包的作者在 Go 1.11 中添加了 ListenConfig，因为他们认为可能会有更多的选择。结果证明他们是对的。在 Go1.13 中，添加了 [KeepAlive](https://pkg.go.dev/net@go1.13#ListenConfig) 字段以允许禁用 keep-alive 或更改其周期。默认值为零将保留启用 keep-alive 的原始行为，并使用默认时间段。

新字段有一种微妙的方式可以意外地破坏用户代码。如果一个结构中的所有字段类型都是可比较的，那么这些类型的值可以用 == 和 != 并用作映射键，则整个结构类型也具有可比性。在这种情况下，添加一个不可比较类型的新字段将使整个struct类型不可比较，从而破坏任何比较该结构类型值的代码。

若要保持结构的可比性，请不要向其添加不可比较的字段。您可以为此编写一个测试，或者依赖即将到来的 [gorelease](https://pkg.go.dev/golang.org/x/exp/cmd/gorelease?tab=doc) 工具来捕捉它。

首先要防止比较，请确保结构具有不可比较的字段。没有切片，映射或函数类型则的 struct 是可比较的，如果没有可以这样添加一个：

```go
type Point struct {
        _ [0]func()
        X int
        Y int
}
```

`func()` 类型不可比较的，零长度数组不占用空间。我们可以定义一种类型来澄清我们的意图：

```go
type doNotCompare [0]func()

type Point struct {
        doNotCompare
        X int
        Y int
}
```

应该在结构中使用 `doNotCompare` 吗？
+ 如果您已经定义了要作为指针使用的结构，也就是说，它有指针方法，可能还有一个返回指针的 `NewXXX` 构造函数，那么添加 `doNotCompare` 字段可能有点过头了。指针类型的用户理解类型的每个值是不同的：如果他们想比较两个值，他们应该比较指针。
+ 如果您正在定义一个打算直接用作值的结构，比如我们的 `Point` 示例，那么您通常希望它是可比较的。在不常见的情况下，您有一个不希望比较的值结构，然后添加一个 `doNotCompare` 字段，您以后可以自由地更改结构，而不必担心破坏比较。缺点是，该类型不能作为映射键使用。



## Conclusion

在从头开始规划 API 时，请仔细考虑 API 对未来新变化的可扩展性。当您确实需要添加新特性时，请记住以下规则：添加、不更改、不删除。记住异常–接口、函数参数和返回值不能以向后兼容的方式添加。

如果您需要较大程度地更改 API，或者随着更多特性的添加，API 开始失去重点，那么可能是时候推出一个新的主要版本了。但是在大多数情况下，进行向后兼容的更改很容易，并且避免给用户带来痛苦。
