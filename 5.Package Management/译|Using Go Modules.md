# 译|Using Go Modules

Go 1.11 和 1.12 初步包含了 [对模块的支持](https://golang.org/doc/go1.11#modules)，Go 的 [新依赖管理系统](https://learnku.com/docs/go-blog/versioning-proposal) 使依赖版本信息明确且易于管理。本文介绍了开始使用模块所需要的基本操作。

模块是  [Go packages](https://golang.org/ref/spec#Packages) 的集合，以 `go.mod` 文件的形式存储在文件树的根目录。`go.mod` 定义了模块的 *模块路径* （根目录的导入路径）和 *依赖项需求* （成功构建所需要的其他模块）。每一个依赖项需求都以模块路径和特定 [语义版本](http://semver.org/) 的形式给出。

从 Go 1.11 开始，当前目录或者任何父目录包含 `go.mod` 文件时，Go 命令就可以使用模块，前提是该目录要在 `$GOPATH/src` *之外*。（在 `$GOPATH/src` 内的目录，出于兼容性考虑，Go 命令仍然以旧的 GOPATH 模式运行，即使存在 `go.mod` 文件。详细内容请参阅 [Go 命令文档](https://golang.org/cmd/go/#hdr-Preliminary_module_support) ）。从 Go 1.13 开始，模块模式将是所有开发的默认模式。

本文介绍了使用模块进行 Go 开发的一系列常见操作：

- 创建新模块。
- 添加依赖。
- 升级依赖。
- 添加对新的主版本的依赖。
- 将依赖升级到新的主版本。
- 删除未使用的依赖。



## Creating a new module

让我们创建一个新模块。

在 `$GOPATH/src` 之外的某个地方创建一个新的空目录，在该目录中的 `cd` 内，然后创建一个新的源文件 `hello.go`：

```go
package hello

func Hello() string {
    return "Hello, world."
}
```

让我们也在 `hello_test.go` 中编写一个测试：

```go
package hello

import "testing"

func TestHello(t *testing.T) {
    want := "Hello, world."
    if got := Hello(); got != want {
        t.Errorf("Hello() = %q, want %q", got, want)
    }
}
```

此时，该目录包含一个软件包，但不包含模块，因为没有 `go.mod` 文件。如果我们在 `/home/gopher/hello` 中工作并且现在运行 `go test`，我们会看到：

```bash
$ go test
PASS
ok  	_/home/gopher/hello	0.020s
$
```

后一行总结了整体包装测试。因为我们在`$GOPATH`之外以及任何模块之外工作，所以`go`命令不知道当前目录的导入路径，而是根据目录名称组成一个伪目录：` _/home/gopher/hello` 。

让我们使用 `go mod init` 将当前目录设为模块的根目录，然后再次尝试 `go test`：

```bash
$ go mod init example.com/hello
go: creating new go.mod: module example.com/hello
$ go test
PASS
ok  	example.com/hello	0.020s
$
```

恭喜你！您已经编写并测试了第一个模块。

`go mod init` 命令编写了一个 `go.mod` 文件：

```go
$ cat go.mod
module example.com/hello

go 1.12
$
```

`go.mod` 文件仅出现在模块的根目录中。子目录中的程序包具有由模块路径以及子目录路径组成的导入路径。例如，如果我们创建了一个子目录 `world`，则无需 (也不希望) 运行 `go mod init`。该软件包将自动被识别为 `example.com/hello` 模块的一部分，并带有导入路径 `example.com/hello/world`。



## Adding a dependency

Go 模块的首要目的是提高其他开发者使用代码（也就是添加依赖）的效率。

让我们更新下 `hello.go` 代码，导入 `rsc.io/quote` 包，使用这个包来实现 `Hello` ：

```go
package hello

import "rsc.io/quote"

func Hello() string {
    return quote.Hello()
}
```

现在我们再次运行这个测试代码：

```bash
$ go test
go: finding rsc.io/quote v1.5.2
go: downloading rsc.io/quote v1.5.2
go: extracting rsc.io/quote v1.5.2
go: finding rsc.io/sampler v1.3.0
go: finding golang.org/x/text v0.0.0-20170915032832-14c0d48ead0c
go: downloading rsc.io/sampler v1.3.0
go: extracting rsc.io/sampler v1.3.0
go: downloading golang.org/x/text v0.0.0-20170915032832-14c0d48ead0c
go: extracting golang.org/x/text v0.0.0-20170915032832-14c0d48ead0c
PASS
ok  	example.com/hello	0.023s
$
```

`go` 命令通过使用 `go.mod` 中列出的指定模块版本来解析依赖。当遇到 `import` 了一个没有在 `go.mod` 中提供的模块中包含的包时，`go` 命令会自动搜索包含这个包的模块，并将其添加到 `go.mod` 中，并且使用这个包的最新版本（「最新」指的是包的最新的带标签的稳定版（非[预发布版](https://semver.org/#spec-item-9)），如果没有稳定版，这使用预发布版，如果都没有，再使用不打标签的最新版本）。在这个例子中，`go test` 命令将新添加的导入包 `rsc.io/quote` 解析为模块 `rsc.io/quote v1.5.2` 。同时，命令自动下载了 `rsc.io/quote` 使用的名为 `rsc.io/sampler` 和 `golang.org/x/text` 的两个依赖，但是，只有**源码直接依赖**的包会被记录到 `go.mod` 文件中，如下所示：

```go
$ cat go.mod
module example.com/hello

go 1.12

require rsc.io/quote v1.5.2
$
```

第二次运行 `go test` 命令，不会重复下载依赖这些工作，因为 `go.mod` 文件现在已经是最新，且下载的模块已经缓存在本地了（在目录 `$GOPATH/pkg/mod` 中），此命令运行输出如下所示：

```bash
$ go test
PASS
ok  	example.com/hello	0.020s
$
```

注意，虽然 `go` 命令添加新的依赖时，很快很方便，但它也有代价。例如，你的模块*真正地*在关键的地方，依赖新的库的正确性、安全性和许可证要求。更多需要考虑的地方，请参考 Russ Cox 的博客 「[Our Software Dependency Problem](https://research.swtch.com/deps)」。

如上所述，添加一个直接依赖，经常会引入其他间接依赖。命令 `go list -m all` 会列出当前模块的所有依赖，如下所示：

```go
$ go list -m all
example.com/hello
golang.org/x/text v0.0.0-20170915032832-14c0d48ead0c
rsc.io/quote v1.5.2
rsc.io/sampler v1.3.0
$
```

在 `go list` 命令的输出中，当前模块，也就是*主模块*，总是出现在第一行，后面按模块路径的顺序列出所有依赖。

`golang.org/x/text` 包的版本 `v0.0.0-20170915032832-14c0d48ead0c` 就是一个 [伪版本](https://golang.org/cmd/go/#hdr-Pseudo_versions) 的例子，这个版本号是 `go` 命令为不打标签的提交所定义的版本。

除了 `go.mod` 文件， `go` 命令也维护了一个名为 `go.sum` 的文件，文件内容为指定版本的模块内容的 [加密哈希值](https://golang.org/cmd/go/#hdr-Module_downloading_and_verification) ，如下所示：

```bash
$ cat go.sum
golang.org/x/text v0.0.0-20170915032832-14c0d48ead0c h1:qgOY6WgZO...
golang.org/x/text v0.0.0-20170915032832-14c0d48ead0c/go.mod h1:Nq...
rsc.io/quote v1.5.2 h1:w5fcysjrx7yqtD/aO+QwRjYZOKnaM9Uh2b40tElTs3...
rsc.io/quote v1.5.2/go.mod h1:LzX7hefJvL54yjefDEDHNONDjII0t9xZLPX...
rsc.io/sampler v1.3.0 h1:7uVkIFmeBqHfdjD+gZwtXXI+RODJ2Wc4O7MPEh/Q...
rsc.io/sampler v1.3.0/go.mod h1:T1hPZKmBbMNahiBKFy5HrXp6adAjACjK9...
$
```

`go` 命令通过 `go.sum` 文件来保证未来下载模块依赖时，依然使用的相同的内容，保证你的项目依赖的模块不会遇到恶意代码、随机异常等原因导致的异常变化。`go.mod` 和 `go.sum` 两个文件，应该都需要被版本控制系统管理起来。



## Upgrading dependencies

通过使用 Go 模块，模块版本通过语义化的标签来引用。带有语义性质的版本包含三个部分：主版本号，次版本号和补丁版本号。例如，版本 `v0.1.2` 的主版本号为 0 ，次版本号为 1 ，补丁版本号为 2 。我们先略过次版本号的更新，下一章节介绍组版本号的更新。

在 `go list -m all` 命令的输出中，我们可以看到使用了 `golang.org/x/text` 的无标签版本。我们可以将其更新为最新的带标签版本，并且测试下我们的代码还可以继续正常工作，如下所示：

```bash
$ go get golang.org/x/text
go: finding golang.org/x/text v0.3.0
go: downloading golang.org/x/text v0.3.0
go: extracting golang.org/x/text v0.3.0
$ go test
PASS
ok  	example.com/hello	0.013s
$
```

哇哦！一切正常。我们再来看看 `go list -m all` 命令的输出和 `go.mod` 文件的内容：

```go
$ go list -m all
example.com/hello
golang.org/x/text v0.3.0
rsc.io/quote v1.5.2
rsc.io/sampler v1.3.0
$ cat go.mod
module example.com/hello

go 1.12

require (
    golang.org/x/text v0.3.0 // indirect
    rsc.io/quote v1.5.2
)
$
```

`golang.org/x/text` 包已经被升级到了最新的带标签版本（`v0.3.0`）。`go.mod` 文件也自动更新为指定的 `v0.3.0` 版本。`go.mod` 文件也自动更新为指定的 `v0.3.0` 版本。注释 `indirect` 说明这个包不是当前模块的直接依赖，而是其他模块的间接依赖。更多细节可以参考 `go help modules` 命令的输出。

现在，我们可以试着更新 `rsc.io/sampler` 的次版本号。同样的，我们通过 `go get` 命令更新依赖，并且运行测试，其输出如下所示：

```bash
$ go get rsc.io/sampler
go: finding rsc.io/sampler v1.99.99
go: downloading rsc.io/sampler v1.99.99
go: extracting rsc.io/sampler v1.99.99
$ go test
--- FAIL: TestHello (0.00s)
    hello_test.go:8: Hello() = "99 bottles of beer on the wall, 99 bottles of beer, ...", want "Hello, world."
FAIL
exit status 1
FAIL	example.com/hello	0.014s
$
```

啊哦！测试失败了，说明 `rsc.io/sampler` 的最新版本不兼容我们的使用方式。让我们看看这个模块可以使用的所有版本号，具体操作如下所示：

```bash
$ go list -m -versions rsc.io/sampler
rsc.io/sampler v1.0.0 v1.2.0 v1.2.1 v1.3.0 v1.3.1 v1.99.99
$
```

我们已经使用了 v1.3.0 版本，而 v1.99.99 明显是不可用的，让我们试试 v1.3.1 版本吧：

```go
$ go get rsc.io/sampler@v1.3.1
go: finding rsc.io/sampler v1.3.1
go: downloading rsc.io/sampler v1.3.1
go: extracting rsc.io/sampler v1.3.1
$ go test
PASS
ok  	example.com/hello	0.022s
$
```

注意 `go get` 命令中，显式指定了版本号 `@v1.3.1`。通常，`go get` 命令的参数需要使用显式指定版本，默认为 `@latest`，对应就是前面所述的「最新」版本。



## Adding a dependency on a new major version

让我们在包中添加一个新的函数: `func Proverb` 通过调用 `quote.Concurrency` 返回 Go 并发谚语，这是由模块 `rsc.io/quote/v3` 提供的。首先我们修改 `hello.go` 文件来添加新函数：

```go
package hello

import (
    "rsc.io/quote"
    quoteV3 "rsc.io/quote/v3"
)

func Hello() string {
    return quote.Hello()
}

func Proverb() string {
    return quoteV3.Concurrency()
}
```

然后我们添加测试文件 `hello_test.go`：

```go
func TestProverb(t *testing.T) {
    want := "Concurrency is not parallelism."
    if got := Proverb(); got != want {
        t.Errorf("Proverb() = %q, want %q", got, want)
    }
}
```

然后测试我们的代码：

```bash
$ go test
go: finding rsc.io/quote/v3 v3.1.0
go: downloading rsc.io/quote/v3 v3.1.0
go: extracting rsc.io/quote/v3 v3.1.0
PASS
ok  	example.com/hello	0.024s
$
```

注意我们的模块现在同时依赖 `rsc.io/quote` 和 `rsc.io/quote/v3`：

```bash
$ go list -m rsc.io/q...
rsc.io/quote v1.5.2
rsc.io/quote/v3 v3.1.0
$
```

Go 模块的每个不同的主版本 (`v1`, `v2` 等) 使用不同的模块路径：从 `v2` 开始，该路径必须是主版本号。在示例中，`rsc.io/quote` 的 `v3` 不再是 `rsc.io/quote`: 而是由模块路径 `rsc.io/quote/v3` 标识并代替。此约定称为 [语义导入版本控制](https://research.swtch.com/vgo-import) , 它为不兼容的软件包 (具有不同主版本的软件包) 提供了不同的名称。相反，`rsc.io/quote` 的 `v1.6.0` 应该与 `v1.5.2` 向后兼容，因此它重用了模块名称 `rsc.io/quote`。(在上一节中，`rsc.io/sampler` `v1.99.99` *应该* 与 `rsc.io/sampler` `v1.3.0` 向后兼容，但是关于模块行为的错误或错误的客户端假设都可能发生。)

`go` 命令允许构建最多包含任何特定模块路径的一个版本，即每个主版本有且仅有一个：一个 `rsc.io/quote`，一个 `rsc.io/quote/v2`，一个 `rsc.io/quote/v3`，依此类推。这为模块作者提供了关于单个模块路径可能重复的明确规则：程序无法同时使用 `rsc.io/quote v1.5.2` 和 `rsc.io/quote v1.6.0`。同时，允许模块的不同主要版本 (因为它们具有不同的路径) 使模块使用者可以逐步升级到新的主版本。在此示例中，我们想使用 `rsc/quote/v3 v3.1.0` 中的 `quote.Concurrency`，但我们尚未准备好对 `rsc.io/quote v1.5.2` 的升级迁移。在大型程序或代码库中，增量迁移的能力尤为重要。



## Upgrading a dependency to a new major version

让我们完成从使用 `rsc.io/quote` 到仅使用 `rsc.io/quote/v3` 的转换。由于版本的重大更改，我们应该懂得某些 APIs 可能已不兼容的方式被删除了，重命名了，或是其他方式被改变。查看文档后，我们会看到 `Hello` 变成了 `HelloV3`：

```bash
$ go doc rsc.io/quote/v3
package quote // import "rsc.io/quote"

Package quote collects pithy sayings.

func Concurrency() string
func GlassV3() string
func GoV3() string
func HelloV3() string
func OptV3() string
$
```

(输出中还存在一个[已知错误](https://golang.org/issue/30778)；显示导入路径被错误地删除了 `/v3`。)

我们可以修改 `hello.go` 文件中使用的 `quote.Hello()` 来使用 `quoteV3.HelloV3()`：

```go
package hello

import quoteV3 "rsc.io/quote/v3"

func Hello() string {
    return quoteV3.HelloV3()
}

func Proverb() string {
    return quoteV3.Concurrency()
}
```

就这点而言，不需要任何重命名的导入，所以我们应该撤消该操作：

```go
package hello

import "rsc.io/quote/v3"

func Hello() string {
    return quote.HelloV3()
}

func Proverb() string {
    return quote.Concurrency()
}
```

让我们重新运行测试测试来确保代码一如既往地正常：

```bash
$ go test
PASS
ok      example.com/hello       0.014s
```



## Removing unused dependencies

我们已经删除了 `rsc.io/quote` 的所有使用，但它仍然显示在 `go list -m all` 和我们的 `go.mod` 文件中：

```go
$ go list -m all
example.com/hello
golang.org/x/text v0.3.0
rsc.io/quote v1.5.2
rsc.io/quote/v3 v3.1.0
rsc.io/sampler v1.3.1
$ cat go.mod
module example.com/hello

go 1.12

require (
    golang.org/x/text v0.3.0 // indirect
    rsc.io/quote v1.5.2
    rsc.io/quote/v3 v3.0.0
    rsc.io/sampler v1.3.1 // indirect
)
$
```

为什么？因为像 `go build` 或 `go test` 一样，构建一个包可以很容易地判断什么时候缺少了什么，什么时候需要添加什么，而不是什么时候可以安全地删除什么。只有在检查模块中的所有包以及这些包的所有可能的生成标记组合之后，才能删除依赖项。普通的生成命令不会加载此信息，因此无法安全地删除依赖项。

用 `go mod tidy` 命令来清除这些未使用的依赖：

```bash
$ go mod tidy
$ go list -m all
example.com/hello
golang.org/x/text v0.3.0
rsc.io/quote/v3 v3.1.0
rsc.io/sampler v1.3.1
$ cat go.mod
module example.com/hello

go 1.12

require (
    golang.org/x/text v0.3.0 // indirect
    rsc.io/quote/v3 v3.1.0
    rsc.io/sampler v1.3.1 // indirect
)

$ go test
PASS
ok  	example.com/hello	0.020s
$
```



## Conclusion

Go 模块是 Go 中依赖管理的未来。模块功能现在在所有支持的 Go 版本 (这里的所有是指，Go 1.11 和 Go 1.12) 中都可用。

下面介绍了这些使用 Go 模块的工作流：

- `go mod init` 创建一个新模块，会初始化一个 `go.mod` 并有其描述。
- `go build`， `go test`, 以及其他包生成命令根据需要向 `go.mod` 添加新的依赖项。
- `go list -m all` 打印当前模块的依赖。
- `go get` 更改依赖项的所需版本 (或添加一个新的依赖项)。
- `go mod tidy` 移除未使用的依赖。

我们鼓励你在本地开发中开始使用模块，并将 `go.mod` 和 `go.sum` 文件添加到你的项目中。为了提供反馈并帮助塑造 Go 中依赖关系管理的未来，请向我们发送 [错误报告](https://golang.org/issue/new) 或 [体验报告](https://golang.org/wiki/ExperienceReports)。

感谢您对改进模块的所有反馈和帮助。





