# 译|Migrating to Go Modules

## Introduction

Go 的项目使用了一种宽泛的依赖管理策略。比如 [Vendoring](https://golang.org/cmd/go/#hdr-Vendor_Directories)， [dep](https://github.com/golang/dep) 和 [glide](https://github.com/Masterminds/glide) 都是比较流行的，但是它们都有一些差异，并且不能同时使用。一些项目仓库的文件夹都在 `GOPATH` 下 一块放到 git 仓库，其他的则简单的依赖于 `go get` 并寄希望于它能将正确的版本安装到 `GOPATH` 下。

Go module 在 Go 的 1.11 版本后引进，提供一个官方的依赖管理策略来解决 go 的构建版本依赖问题。这篇文章将介绍该工具，并教你如果将 go 项目转变为一个 go module 项目。

请注意：如果你的项目已经使用了 v2.0.0 或更高的版本，你需要更新你的 `go.mod` 文件路径。我们将在后面的文章中告诉你如何迁移。



## Migrating to Go modules in your project

一个 Go 项目想转变为 Go modules 项目需要满足下列任一一个条件：

- 一个全新的 Go 项目。
- 此项目使用了一个不是 go module 的依赖管理工具。
- 此项目没有任何依赖管理工具。

第一种情况可以在此系列文章中的第一部分查看 [Using Go Modules](https://learnku.com/docs/go-blog/using-go-modules); 在这篇文章中，我们着重讨论后面两种情况：



## With a dependency manager

因为要转换的项目已经有一个依赖管理工具了，为了能顺利转换，先运行下面命令：

```go
$ git clone https://github.com/my/project
[...]
$ cd project
$ cat Godeps/Godeps.json
{
    "ImportPath": "github.com/my/project",
    "GoVersion": "go1.12",
    "GodepVersion": "v80",
    "Deps": [
        {
            "ImportPath": "rsc.io/binaryregexp",
            "Comment": "v0.2.0-1-g545cabd",
            "Rev": "545cabda89ca36b48b8e681a30d9d769a30b3074"
        },
        {
            "ImportPath": "rsc.io/binaryregexp/syntax",
            "Comment": "v0.2.0-1-g545cabd",
            "Rev": "545cabda89ca36b48b8e681a30d9d769a30b3074"
        }
    ]
}
$ go mod init github.com/my/project
go: creating new go.mod: module github.com/my/project
go: copying requirements from Godeps/Godeps.json
$ cat go.mod
module github.com/my/project

go 1.12

require rsc.io/binaryregexp v0.2.1-0.20190524193500-545cabda89ca
$
```

`go mod init` 命令会创建一个 `go.mod` 文件，并且自动导入依赖从 `Godeps.json`, `Gopkg.lock` 中， 还有其他支持的格式[点击查看](https://go.googlesource.com/go/+/362625209b6cd2bc059b6b0a67712ddebab312d9/src/cmd/go/internal/modconv/modconv.go#9)。

在继续之前我们最好先运行下 `go build ./...` 和 `go test ./...`。下一步就需要修改你的 `go.mod` 文件了，因此如果你更喜欢迭代的更新 `go.mod`，这个方式是最合适的，这个文件也将成为你的依赖规范。

```bash
$ go mod tidy
go: downloading rsc.io/binaryregexp v0.2.1-0.20190524193500-545cabda89ca
go: extracting rsc.io/binaryregexp v0.2.1-0.20190524193500-545cabda89ca
$ cat go.sum
rsc.io/binaryregexp v0.2.1-0.20190524193500-545cabda89ca h1:FKXXXJ6G2bFoVe7hX3kEX6Izxw5ZKRH57DFBJmHCbkU=
rsc.io/binaryregexp v0.2.1-0.20190524193500-545cabda89ca/go.mod h1:qTv7/COck+e2FymRvadv62gMdZztPaShugOCi3I+8D8=
$
```

`go mod tidy` 会检查你需要依赖的包，如果你代码中使用到的第三方包没有在 `go.mod` 文件中，它将会把此包放入到你的 `go.mod` 文件中，并且它也会移除 `go.mod` 中你没有使用过的第三方包，如果你导入的第三方包还没有迁移为 go module 项目，那么将会在此包后面加入 `// indirect` 的备注，提交 `go.mod` 文件到版本控制中心前最好先运行一下 `go mod tidy`，这是一个比较好的习惯。

让我们确保代码可以正常构建和测试吧，运行下列命令：

```bash
$ go build ./...
$ go test ./...
[...]
$
```

注意，其他依赖项管理器可能会指定的一个特殊的依赖版本 (不是 modules), 并且通常不会明确的在 `go.mod` 文件中指出。因此，你可能会得到两个不同的版本，所以会有一些迁移的风险。因此，这是非常重要的去运行下上面的命令，观察代码是否还可以重新构建和测试通过。

```go
$ go list -m all
go: finding rsc.io/binaryregexp v0.2.1-0.20190524193500-545cabda89ca
github.com/my/project
rsc.io/binaryregexp v0.2.1-0.20190524193500-545cabda89ca
$
```

比较一下最新的依赖版本和你之前的依赖版本是否一致。如果你发现某个包的版本不是你想要的，你可以通过这两个命令 `go mod why -m` 和 `go mod graph` 来找出原因，并且升级或降级为你想要的版本通过 `go get`. (如果你获取的版本低于现在的版本，那么将会被优先选择， `go get` 同时会下调其他第三方库的依赖版本，来保证其兼容性。) 举个例子：

```bash
$ go mod why -m rsc.io/binaryregexp
[...]
$ go mod graph | grep rsc.io/binaryregexp
[...]
$ go get rsc.io/binaryregexp@v0.2.0
$
```



## Without a dependency manager

对于没有依赖项管理系统的 Go 项目，请先创建一个 `go.mod` 文件：

```go
$ git clone https://go.googlesource.com/blog
[...]
$ cd blog
$ go mod init golang.org/x/blog
go: creating new go.mod: module golang.org/x/blog
$ cat go.mod
module golang.org/x/blog

go 1.12
$
```

如果没有以前的依赖项管理器提供的配置文件，`go mod init` 将创建一个仅包含`模块` 和 `go` 的 `go.mod` 文件指令。在此示例中，我们将模块路径设置为 `golang.org/x/blog` ，因为这是其 [自定义导入路径](https://golang.org/cmd/go/#hdr-Remote_import_paths) 。用户可以使用此路径导入软件包，我们必须小心不要更改它。

`module` 指令声明模块路径，而 `go` 指令声明用于编译模块内代码的 Go 语言的预期版本。

接下来，运行 `go mod tidy` 添加模块的依赖项：

```bash
$ go mod tidy
go: finding golang.org/x/website latest
go: finding gopkg.in/tomb.v2 latest
go: finding golang.org/x/net latest
go: finding golang.org/x/tools latest
go: downloading github.com/gorilla/context v1.1.1
go: downloading golang.org/x/tools v0.0.0-20190813214729-9dba7caff850
go: downloading golang.org/x/net v0.0.0-20190813141303-74dc4d7220e7
go: extracting github.com/gorilla/context v1.1.1
go: extracting golang.org/x/net v0.0.0-20190813141303-74dc4d7220e7
go: downloading gopkg.in/tomb.v2 v2.0.0-20161208151619-d5d1b5820637
go: extracting gopkg.in/tomb.v2 v2.0.0-20161208151619-d5d1b5820637
go: extracting golang.org/x/tools v0.0.0-20190813214729-9dba7caff850
go: downloading golang.org/x/website v0.0.0-20190809153340-86a7442ada7c
go: extracting golang.org/x/website v0.0.0-20190809153340-86a7442ada7c
$ cat go.mod
module golang.org/x/blog

go 1.12

require (
    github.com/gorilla/context v1.1.1
    golang.org/x/net v0.0.0-20190813141303-74dc4d7220e7
    golang.org/x/text v0.3.2
    golang.org/x/tools v0.0.0-20190813214729-9dba7caff850
    golang.org/x/website v0.0.0-20190809153340-86a7442ada7c
    gopkg.in/tomb.v2 v2.0.0-20161208151619-d5d1b5820637
)
$ cat go.sum
cloud.google.com/go v0.26.0/go.mod h1:aQUYkXzVsufM+DwF1aE+0xfcU+56JwCaLick0ClmMTw=
cloud.google.com/go v0.34.0/go.mod h1:aQUYkXzVsufM+DwF1aE+0xfcU+56JwCaLick0ClmMTw=
git.apache.org/thrift.git v0.0.0-20180902110319-2566ecd5d999/go.mod h1:fPE2ZNJGynbRyZ4dJvy6G277gSllfV2HJqblrnkyeyg=
git.apache.org/thrift.git v0.0.0-20181218151757-9b75e4fe745a/go.mod h1:fPE2ZNJGynbRyZ4dJvy6G277gSllfV2HJqblrnkyeyg=
github.com/beorn7/perks v0.0.0-20180321164747-3a771d992973/go.mod h1:Dwedo/Wpr24TaqPxmxbtue+5NUziq4I4S80YR8gNf3Q=
[...]
$
```

`go mod tidy` 为模块中的软件包以可迁移方式导入的所有软件包添加了模块要求，并针对特定版本的每个库构建了 `go.sum` 和校验。最后，确保代码仍在构建并且测试仍通过：

```bash
$ go build ./...
$ go test ./...
ok  	golang.org/x/blog	0.335s
?   	golang.org/x/blog/content/appengine	[no test files]
ok  	golang.org/x/blog/content/cover	0.040s
?   	golang.org/x/blog/content/h2push/server	[no test files]
?   	golang.org/x/blog/content/survey2016	[no test files]
?   	golang.org/x/blog/content/survey2017	[no test files]
?   	golang.org/x/blog/support/racy	[no test files]
$
```

请注意，当 `go mod tidy` 添加需求时，它将添加模块的最新版本。如果你的 `GOPATH` 包含较旧版本的依赖关系，该依赖关系随后发布了重大更改，则你可能会在 `go mod tidy` ， `go build` 或 `go test` 中看到错误。如果发生这种情况，请尝试使用 `go get` 降级到较旧的版本 (例如， `go get github.com/broken/module@v1.1.0`)，或者花一些时间制作你的模块与每个依赖项的最新版本兼容。



## Tests in module mode

迁移到 Go 模块后，某些测试可能需要进行调整。

如果测试需要在程序包目录中写入文件，则当程序包目录位于模块缓存 (只读) 中时，它可能会失败。特别是，这可能会导致 `全部测试` 失败。测试应将需要写入的文件复制到临时目录中。

如果测试依赖于相对路径 ( `../ package-in-another-module` ) 来定位和读取另一个包中的文件，则如果该包位于另一个模块中，则该测试将失败。模块缓存的版本化子目录或 `replace` 指令中指定的路径。在这种情况下，你可能需要将测试输入复制到模块中，或将测试输入从原始文件转换为嵌入在 `.go` 源文件中的数据。

如果测试期望测试中的 `go` 命令以 GOPATH 模式运行，则可能会失败。在这种情况下，你可能需要向要测试的树型源中添加一个 `go.mod` 文件，或明确设置 `GO111MODULE = off`。



## Publishing a release

最后，你应该标记并发布新模块的发行版本。如果你尚未发布任何版本，但是此版本是可选的，如果没有正式版本，则下游用户将依赖于使用 [pseudo-versions](https://golang.org/cmd/go/#hdr-Pseudo_versions) 的特定提交，这可能会更难支持。

```bash
$ git tag v1.2.0
$ git push origin v1.2.0
```

新的 `go.mod` 文件为模块定义了规范的导入路径，并添加了新的最低版本要求。如果你的用户已经在使用正确的导入路径，并且你的依赖项尚未进行重大更改，则添加 `go.mod` 文件是向后兼容的 - 但是这是一个重大更改，可能会暴露现有问题。如果已有版本标记，则应增加 [次要版本](https://semver.org/#spec-item-7) 。请参阅 [发布 Go 模块](https://learnku.com/docs/go-blog/publishing-go-modules) 以了解如何增加和发布版本。



## Imports and canonical module paths

每个模块在其 `go.mod` 文件中声明其模块路径。引用模块内包的每个 `import` 语句必须具有模块路径作为包路径的前缀。但是，`go` 命令可能会通过许多不同的 [远程导入路径](https://golang.org/cmd/go/#hdr-Remote_import_paths) 遇到包含模块的存储库。例如，`golang.org/x/lint` 和 `github.com/golang/lint` 都解析为包含托管在 [go.googlesource.com/lint](https://go.googlesource.com/lint) 。该存储库中包含的 [`go.mod` file](https://go.googlesource.com/lint/+/refs/heads/master/go.mod) 声明其路径为 `golang.org/x/lint`，因此仅该路径对应于有效模块。

Go 1.4 提供了一种使用`// import` [ comments](https://golang.org/cmd/go/#hdr-Import_path_checking) 声明规范导入路径的机制，但是包作者并不总是提供它们。结果，在模块之前编写的代码可能已经为模块使用了非规范的导入路径，而没有出现不匹配的错误。使用模块时，导入路径必须与规范的模块路径匹配，因此你可能需要更新 `import` 语句：例如，你可能需要更改 `import "github.com/golang/lint"` 到 `导入 " golang.org/x/lint"`。

在主要版本 2 或更高版本的 Go 模块中，发生了另一种情况，其中模块的规范路径可能不同于其存储库路径。主版本高于 1 的 Go 模块的模块路径中必须包含主版本后缀：例如，版本 `v2.0.0` 必须具有后缀 `/ v2`。但是，`import` 语句可能已经引用了模块中的后缀。例如，`v2.0.1` 中 `github.com/russross/blackfriday/v2` 的非模块用户可能已将其导入为 `github.com/russross/blackfriday` 代替，并且将需要更新导入路径以包含 `/ v2` 后缀。



## Conclusion

对于大多数用户而言，转换为 Go 模块应该是一个简单的过程。有时会由于非规范的导入路径或在依赖项中破坏更改而引起偶尔的问题。未来的文章将探讨 [发布新版本](https://learnku.com/docs/go-blog/publishing-go-modules)，v2 及更高版本，以及调试异常情况的方法。

为了提供反馈并帮助塑造 Go 中依赖管理的未来，请向我们发送 [错误报告](https://golang.org/issue/new) 或 [经验报告](https://golang.org/wiki/ExperienceReports) 。

感谢您的所有反馈，并帮助改善模块。



