# 译|Publishing Go Modules

## Introduction

这篇文章讨论了如何编写和发布模块，以便其他模块可以依赖它们。

请注意：这篇文章涵盖了 `v1` 之前的开发。如果您使用 `v2`，请参阅 [Go 模块: v2 及更高版本](https://learnku.com/docs/go-blog/v2-go-modules)。

这篇文章在示例中使用 [Git](https://git-scm.com/) 。 [Mercurial](https://www.mercurial-scm.org/)， [Bazaar](http://wiki.bazaar.canonical.com/) 等也受支持。

## Project setup

对于本文，您将需要一个现有项目作为示例。因此，请从 [使用 Go 模块](https://learnku.com/docs/go-blog/using-go-modules) 文章末尾的文件开始：

```bash
$ cat go.mod
module example.com/hello

go 1.12

require rsc.io/quote/v3 v3.1.0

$ cat go.sum
golang.org/x/text v0.0.0-20170915032832-14c0d48ead0c h1:qgOY6WgZOaTkIIMiVjBQcw93ERBE4m30iBm00nkL0i8=
golang.org/x/text v0.0.0-20170915032832-14c0d48ead0c/go.mod h1:NqM8EUOU14njkJ3fqMW+pc6Ldnwhi/IjpwHt7yyuwOQ=
rsc.io/quote/v3 v3.1.0 h1:9JKUTTIUgS6kzR9mK1YuGKv6Nl+DijDNIc0ghT58FaY=
rsc.io/quote/v3 v3.1.0/go.mod h1:yEA65RcK8LyAZtP9Kv3t0HmxON59tX3rD+tICJqUlj0=
rsc.io/sampler v1.3.0 h1:7uVkIFmeBqHfdjD+gZwtXXI+RODJ2Wc4O7MPEh/QiW4=
rsc.io/sampler v1.3.0/go.mod h1:T1hPZKmBbMNahiBKFy5HrXp6adAjACjK9JXDnKaTXpA=

$ cat hello.go
package hello

import "rsc.io/quote/v3"

func Hello() string {
    return quote.HelloV3()
}

func Proverb() string {
    return quote.Concurrency()
}

$ cat hello_test.go
package hello

import (
    "testing"
)

func TestHello(t *testing.T) {
    want := "Hello, world."
    if got := Hello(); got != want {
        t.Errorf("Hello() = %q, want %q", got, want)
    }
}

func TestProverb(t *testing.T) {
    want := "Concurrency is not parallelism."
    if got := Proverb(); got != want {
        t.Errorf("Proverb() = %q, want %q", got, want)
    }
}

$
```

接下来，创建一个新的 `git` 存储库并添加初始提交。如果要发布自己的项目，请确保包含一个 `LICENSE` 文件.。转到包含 `go.mod` 的目录，然后创建存储库：

```bash
$ git init
$ git add LICENSE go.mod go.sum hello.go hello_test.go
$ git commit -m "hello: initial commit"
$
```



## Semantic versions and modules

 `go.mod` 中的每个必需模块都有一个 [语义版本](https://semver.org/)，这是用于构建模块的依赖项的最低版本。

语义版本的形式为 `vMAJOR.MINOR.PATCH`。

- 对模块的公共 API 进行[向后不兼容](https://golang.org/doc/go1compat)更改时，请增加 `MAJOR` 版本（大版本）。仅应在绝对必要时执行此操作。
- 对 API 进行向后兼容的更改时，请增加 `MINOR` 版本（中版本），例如更改依赖关系或添加新的函数，方法，结构字段或类型。
- 在进行不影响模块公共 API 或依赖项的细微更改后，增加 `PATCH` 版本（小版本），例如修复 bug。

您可以通过添加连字符和点分隔的标识符来指定预发行版本 (例如，`v1.0.1-alpha` 或 `v2.2.2-beta.2`)。 `go` 命令优先于普通版本而不是预发行版本，因此用户必须明确要求预发行版本 (例如，`go get example.com/hello@v1.0.1-alpha`)（如果您的模块具有任何的正常版本）。

`v0` 主要版本和预发行版本不保证向后兼容。它们使您可以在对用户做出稳定性承诺之前优化 API。但是，`v1` 主要版本及更高版本要求在该主要版本内向后兼容。

`go.mod` 中引用的版本可能是存储库中标记的显式发行版 (例如 `v1.5.2`)，也可能是 [pseudo-version](https://golang.org/cmd/go/#hdr-Pseudo_versions)(例如，`v0.0.0-20170915032832-14c0d48ead0c`)。伪版本是一种特殊的预发行版本。当用户需要依赖尚未发布任何语义版本标签的项目或针对尚未被标记的提交进行开发时，伪版本很有用，但用户不应假定伪版本可以提供稳定或良好的性能经过测试的 API。用显式版本标记模块会向您的用户发出信号，表明特定的版本已经过全面测试并可以使用。

一旦开始使用版本标记您的存储库，在开发模块时就必须继续标记新版本。当用户请求模块的新版本时 (`go get -u` 或 `go get example.com/hello`)，`go` 命令将选择最大版本可以使用语义发布版本，即使该版本已经使用了几年并且在主分支机构后面进行了许多更改。继续标记新版本将使您的用户可以进行持续的改进。

不要从您的仓库中删除版本标签。如果发现某个版本的错误或安全问题，请发布新版本。如果人们依赖于您已删除的版本，则其构建可能会失败。同样，发布版本后，请勿更改或覆盖版本。 [模块镜像和校验和数据库](https://learnku.com/docs/go-blog/module-mirror-launch)存储模块，它们的版本以及带符号的加密哈希值，以确保给定版本的构建在一段时间内仍可重现。



## v0: the initial, unstable version

让我们用 `v0` 语义版本标记模块。 `v0` 版本没有任何稳定性保证，因此在完善公共 API 时几乎所有项目都应以 `v0` 开头。

标记新版本有几个步骤：

1. 运行 `go mod tidy`，它将删除模块可能已加载后不再需要的任何依赖项。
2. 最后运行 `go test ./...` 以确保一切正常。
3. 使用 [`git tag`](https://git-scm.com/docs/git-tag)用新版本标记项目。
4. 将新标签推到原始存储库。

```bash
$ go mod tidy
$ go test ./...
ok      example.com/hello       0.015s
$ git add go.mod go.sum hello.go hello_test.go
$ git commit -m "hello: changes for v0.1.0"
$ git tag v0.1.0
$ git push origin v0.1.0
$
```

现在其他项目可以依赖 `example.com/hello` 的 `v0.1.0` 版本了。对于你自己的模块，你可以运行 `go list -m example.com/hello@v0.1.0` 确认可用的最新版本 (该示例模块不存在，因此没有可用的版本)。如果没有立即看到最新版本，并且使用的是 Go 模块代理 (Go 1.13 以来的默认设置)，请在几分钟后重试，以使代理有时间加载新版本。

如果添加到公共 API，请对 `v0` 模块进行重大更改，或者升级其中一个依赖项的次要版本或版本，为下一个发行版增加 `MINOR` 版本。例如，`v0.1.0` 之后的下一个发行版将是 `v0.2.0`。

如果您修复了现有版本中的错误，请增加 `PATCH` 版本。例如，`v0.1.0` 之后的下一个发行版将是 `v0.1.1`。



## v1: the first stable version

一旦确定模块的 API 稳定，就可以发布 `v1.0.0`。`v1` 主要版本向用户传达了不会对模块的 API 进行不兼容更改的消息。他们可以升级到新的 `v1` 次要版本和修补程序版本，并且其代码不应中断。函数和方法签名不会更改，不会删除导出的类型，依此类推。如果对 API 进行了更改，则它们将向后兼容 (例如，向结构添加新字段)，并将包含在新的次要版本中。如果存在错误修复程序 (例如，安全修复程序)，它们将包含在补丁程序发行版中 (或作为次要发行版的一部分)。

有时，保持向后兼容性可能会导致 API 笨拙。没关系。不完善的 API 比破坏用户的现有代码更好。.

标准库的 `strings` 软件包是以 API 一致性为代价维护向后兼容性的主要示例。

+ [`Split`](https://godoc.org/strings#Split)将一个字符串切成由分隔符分隔的所有子字符串，并返回这些分隔符之间的子字符串的一部分。
+ [`SplitN`](https://godoc.org/strings#SplitN)可用于控制要返回的子字符串的数量。

但是，[`Replace`](https://godoc.org/strings#Replace)从一开始就计算了要替换的字符串实例数 (与 `Split` 不同)。

给定 `Split` 和 `SplitN`，您会期望像 `Replace` 和 `ReplaceN` 这样的函数。但是，我们无法在不中断调用方的情况下更改现有的 `Replace`，我们承诺不会这样做。因此，在 Go 1.12 中，我们添加了一个新函数 [`ReplaceAll`](https://godoc.org/strings#ReplaceAll) 。产生的 API 有点奇怪，因为 `Split` 和 `Replace` 的行为有所不同，但是不一致比突破性更改要好。

假设您对 `example.com/hello` 的 API 感到满意，并且想要发布 `v1` 作为第一个稳定版本。

标记 `v1` 的过程与标记 `v0` 版本的过程相同：运行 `go mod tidy` 和 `go test ./...`，标记版本，然后将标记推送到原始存储库：

```bash
$ go mod tidy
$ go test ./...
ok      example.com/hello       0.015s
$ git add go.mod go.sum hello.go hello_test.go
$ git commit -m "hello: changes for v1.0.0"
$ git tag v1.0.0
$ git push origin v1.0.0
$
```

此时，`example.com/hello` 的 `v1` API 已固化。这向每个人传达了我们的 API 稳定的信息，他们应该放心使用它。



## Conclusion

这篇文章介绍了使用语义版本标记模块的过程以及何时发布 `v1` 的过程。以后的文章将介绍如何在 `v2` 及更高版本上维护和发布模块。

为了提供反馈并帮助塑造 Go 中依赖管理的未来，请向我们发送[错误报告](https://golang.org/issue/new)或[经验报告](https://golang.org/wiki/ExperienceReports)。

感谢您的所有反馈，并帮助改善 Go 模块。