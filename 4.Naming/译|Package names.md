# 译|Package names

## Introduction

Go 代码被组织到包中。在包内，代码可以引用其中定义的任何标识符 (名称)，而包的客户端只能引用包的导出类型，函数，常量和变量。此类引用始终将软件包名称作为前缀包括: `foo.Bar` 引用导入的名为 `foo` 的程序包中的名称 `Bar`。

好的程序包名称会使代码更好。程序包的名称为其内容提供了上下文，从而使客户更容易了解程序包的用途以及如何使用它。该名称还可以帮助软件包维护者确定其在演化过程中属于和不属于的内容。具名的软件包使查找所需代码更加容易。

Effective Go 为命名包，类型，函数和变量提供了 [指导](https://golang.org/doc/effective_go.html#names)。本文在此讨论的基础上进行了扩展，并调查了标准库中的名称。它还讨论了错误的程序包名称以及如何解决它们。



## Package names

好的软件包名称简短明了。它们是小写字母，没有 `under_scores` 或 `mixedCaps`. 它们通常是简单的名词，例如：

- `time` (提供用于测量和显示时间的功能)
- `list` (实现双向链表)
- `http` (提供 HTTP 客户端和服务器实现)

在 Go 程序中，其他语言的典型名称样式可能不是惯用的。以下是两个名称示例，这些名称在其他语言中可能是不错的样式，但不适用于 Go：

- `computeServiceClient`
- `priority_queue`

Go 包可能会导出几种类型和功能。例如，`compute` 包可以导出 `Client` 类型，其中包含使用服务的方法以及用于在多个客户端之间划分计算任务的功能。

**请谨慎使用** 程序员熟悉时可以使用缩写名称。广泛使用的软件包通常具有压缩名称:

- `strconv` (字符串转换)
- `syscall` (系统调用)
- `fmt` (格式化的 I/O)

另一方面，如果缩写程序包名称使它含糊不清或不清楚，则不要这样做.

不要从用户那里窃取好名字 避免给软件包指定客户端代码中常用的名称。例如，缓冲的 I/O 包称为 bufio, 而不是 buf, 因为 buf 是缓冲区的好变量名。

## Naming package contents

包名及其内容的名称是耦合的，因为客户端代码将它们一起使用。设计包时，请站在客户的角度考虑下。

**避免卡顿** 由于客户端代码在引用软件包内容时使用软件包名称作为前缀，因此这些内容的名称无需重复软件包名称。`http` 软件包提供的 HTTP 服务器称为 `Server`， 而不是 `HTTPServer`。 客户端代码将此类型称为 `http.Server`， 因此没有歧义。

**简化函数名称** 当 pkg 软件包中的函数返回类型为 `pkg.Pkg` (或 `*pkg.Pkg`) 的值时，函数名称通常可以忽略该类型名称不混淆：

```go
start := time.Now()                                  // start is a time.Time
t, err := time.Parse(time.Kitchen, "6:06PM")         // t is a time.Time
ctx = context.WithTimeout(ctx, 10*time.Millisecond)  // ctx is a context.Context
ip, ok := userip.FromContext(ctx)                    // ip is a net.IP
```

程序包 `pkg` 中名为 `New` 的函数返回类型为 `pkg.Pkg` 的值。这是使用该类型的客户端代码的标准入口点：

```go
 q := list.New()  // q is a *list.List
```

当函数返回类型为 `pkg.T` 的值时，其中 `T` 不是 `Pkg`，函数名称可能包括要生成的 `T` 客户端代码更容易理解。一个常见的情况是一个包具有多个类似 New 的功能：

```go
d, err := time.ParseDuration("10s")  // d is a time.Duration
elapsed := time.Since(start)         // elapsed is a time.Duration
ticker := time.NewTicker(d)          // ticker is a *time.Ticker
timer := time.NewTimer(d)            // timer is a *time.Timer
```

不同软件包中的类型可以具有相同的名称，因为从客户端的角度来看，此类名称由软件包名称来区分。例如，标准库包含几种名为 `Reader` 的类型，包括 `jpeg.Reader`, `bufio.Reader` 和 `csv.Reader`。 每个软件包名称都适合 `Reader` 以产生一个好的类型名称。 

 如果您不能给出一个包名称，该名称是该包内容的有意义的前缀，则该包抽象边界可能是错误的。编写将您的程序包用作客户端的代码，如果结果不佳，请重新构造您的程序包。这种方法将产生易于客户理解和易于开发人员维护的软件包.



## Package paths

Go 软件包同时具有名称和路径。包名称在其源文件的 package 语句中指定；客户端代码将其用作软件包导出名称的前缀。导入程序包时，客户端代码使用程序包路径。按照惯例，包路径的最后一个元素是包名称：

```go
import (
    "context"                // package context
    "fmt"                    // package fmt
    "golang.org/x/time/rate" // package rate
    "os/exec"                // package exec
)
```

构建工具将包路径映射到目录. go 工具使用 GOPATH 环境变量在以下目录中找到路径 `"github.com/user/hello"` 的源文件目录 `$GOPATH/src/github.com/user/hello`。 (这种情况当然应该很熟悉，但是重要的是要清楚软件包的术语和结构。)

**目录** 标准库使用类似目录 `crypto`，`container`, `encoding` 和 `image` 的分组来打包相关协议的软件包和算法。这些目录之一中的软件包之间没有实际关系；目录仅提供一种整理文件的方法。只要导入不创建循环，任何软件包都可以导入任何其他软件包。

正如不同程序包中的类型可以具有相同的名称而没有歧义一样，不同目录中的程序包也可以具有相同的名称。例如 `runtime/pprof` 提供 `pprof` 分析工具期望的格式的数据分析，而 `net/http/pprof` 提供 HTTP 端点以这种格式显示性能分析数据。客户端代码使用包路径导入包，因此不会造成混淆。如果源文件需要同时导入两个 `pprof` 程序包，则可以在本地 重命名 一个或两个。重命名导入的软件包时，本地名称应遵循与软件包名称相同的准则 (小写，不是 `under_scores` 或 `mixedCaps`)。不要下划线或者驼峰。



## Bad package names

不太好的程序包名称使代码难以导航和维护。这里有一些识别和修复坏名的准则。

**避免使用无意义的程序包名称**。 名为 `util`,`common` 或 `misc` 的程序包为客户端提供了对程序包所含内容的了解。这使客户更难以使用该程序包，并使维护人员更难以保持程序包的重点。随着时间的推移，它们会累积依赖关系，从而使编译工作变得不必要地缓慢，特别是在大型程序中。而且由于此类软件包名称是通用的，因此它们更有可能与客户端代码导入的其他软件包发生冲突，从而迫使客户端发明名称来区分它们。

**破坏通用软件包**。要修复此类软件包，请查找具有通用名称元素的类型和函数，并将其放入自己的软件包中。例如，如果您有：

```go
package util
func NewStringSet(...string) map[string]bool {...}
func SortStringSet(map[string]bool) []string {...}
```

然后客户端代码看起来像：

```go
set := util.NewStringSet("c", "a", "b")
fmt.Println(util.SortStringSet(set))
```

将这些函数从 `util` 中拉出到新软件包中，选择适合内容的名称：

```go
package stringset
func New(...string) map[string]bool {...}
func Sort(map[string]bool) []string {...}
```

然后客户端代码变成：

```go
set := stringset.New("c", "a", "b")
fmt.Println(stringset.Sort(set))
```

进行此更改后，可以更轻松地了解如何改进新软件包：

```go
package stringset
type Set map[string]bool
func New(...string) Set {...}
func (s Set) Sort() []string {...}
```

生成更简单的客户端代码：

```go
set := stringset.New("c", "a", "b")
fmt.Println(set.Sort())
```

包名是设计的关键。努力从项目中消除无意义的程序包名称。

**不要为您的所有 API 使用单个程序包**。许多有心的程序员将其程序公开的所有接口放入一个名为 `api`, `types`, 或 `interfaces`，认为这样可以更轻松地找到其代码库的入口点。这是个错误。此类软件包与名为 `util` 或 `common` 的软件包存在相同的问题，无限制地增长，没有为用户提供指导，积累了依赖关系，并与其他导入冲突。分解它们，也许使用目录将公共包与实现分开。

**避免不必要的程序包名称冲突** 尽管不同目录中的程序包可能具有相同的名称，但经常一起使用的程序包应具有不同的名称。这样可以减少混乱，并减少客户端代码中本地重命名的需要。出于相同的原因，请避免使用与流行的标准软件包 (例如 `io` 或 `http`) 相同的名称。



## Conclusion

包名对于 Go 程序中的良好命名至关重要。花点时间选择好的包名称并组织好代码。这可以帮助客户理解和使用您的软件包，并帮助维护人员优雅地扩展它们。

