# 译|Go Modules: v2 and Beyond

## Introduction

随着成功项目的成熟和新需求的增加，过去的特性和设计决策可能不再有意义。开发人员可能希望通过删除废弃的函数、重命名类型或将复杂的包拆分为可管理的部分，来将他们所学到的经验教训整合到项目中。这一类的更改需要下游用户努力将代码迁移到新的 API 中，因此在进行这些更改时，必须仔细考虑收益是否大于成本。

对于仍处于试验阶段的项目（大版本是 `v0` ），用户希望偶尔进行一些突破性的更改。对于声明为稳定的项目 - 在大版本 `v1` 或更高版本 - 突破性的更改必须放在新的大版本中。本文将探讨大版本语义、如何创建和发布新的大版本以及如何维护模块的多个大版本。

## Adding to a function

模块形式化了 Go 中的一个重要原则， [**导入兼容性规则**](https://research.swtch.com/vgo-import)：

> 如果旧包和新包具有相同的导入路径，
> 新包必须与旧包向后兼容。

根据定义，包的新大版本与以前的版本不向后兼容。这意味着模块的新大版本必须具有与以前版本不同的模块路径。从 `v2` 开始，大版本必须出现在模块路径的末尾（在 `go.mod` 文件的 `module` 语句中声明）。例如，当 `github.com/googleapis/gax-go` 模块的作者开发了 `v2` 时，他们使用了新的模块路径 `github.com/googleapis/gax-go/v2`。想要使用 `v2` 的用户必须将其包导入和模块需求更改为 `github.com/googleapis/gax-go/v2`。

对大版本后缀的需求是 Go 模块不同于大多数其他依赖性管理系统的方式之一。需要后缀来解决[菱形依赖问题](https://research.swtch.com/vgo-import#dependency_story)。在 Go 模块之前，[gopkg.in](http://gopkg.in/) 允许包维护人员遵循我们现在所说的导入兼容性规则。对于 gopkg.in，如果依赖于导入`gopkg.in/yaml.v1` 的包和另一个导入 `gopkg.in/yaml.v2` 的包，则不会发生冲突，因为这两个 `yaml` 包具有不同的导入路径，它们使用版本后缀，就像 Go 模块一样。由于 gopkg.in 与 Go 模块共享相同的版本后缀方法，Go 命令接受 `gopkg.in/yaml.v2` 中的 `.v2` 作为有效的大版本后缀。这是与 gopkg.in 兼容的特殊情况。托管在其他域中的模块需要像 `/v2` 这样的斜杠后缀。



## Major version strategies

建议的策略是在以大版本后缀命名的目录中开发 `v2+` 模块。

```go
github.com/googleapis/gax-go @ master branch
/go.mod    → module github.com/googleapis/gax-go
/v2/go.mod → module github.com/googleapis/gax-go/v2
```

此方法与不知道模块的工具兼容：大版本仓库中的文件路径与 `GOPATH` 模式下 `go get` 所期望的路径匹配。这个策略还允许所有大版本在不同的目录中一起开发。

其他策略可能会将大版本保存在不同的分支上。但是，如果 `v2+` 源代码位于仓库的默认分支（通常是 `master`）上，则不知道版本的工具（包括 `GOPATH` 模式下的 `go` 命令）可能无法区分大版本。

本文中的示例将遵循大版本子目录策略，因为它提供了最大的兼容性。我们建议模块作者遵循这个策略，只要他们有用户在 `GOPATH` 模式下开发。



## Publishing v2 and beyond

我们在这篇文章中使用 `github.com/googleapis/gax-go` 这个包作为例子：

```bash
$ pwd
/tmp/gax-go
$ ls
CODE_OF_CONDUCT.md  call_option.go  internal
CONTRIBUTING.md     gax.go          invoke.go
LICENSE             go.mod          tools.go
README.md           go.sum          RELEASING.md
header.go
$ cat go.mod
module github.com/googleapis/gax-go

go 1.9

require (
    github.com/golang/protobuf v1.3.1
    golang.org/x/exp v0.0.0-20190221220918-438050ddec5e
    golang.org/x/lint v0.0.0-20181026193005-c67002cb31c3
    golang.org/x/tools v0.0.0-20190114222345-bf090417da8b
    google.golang.org/grpc v1.19.0
    honnef.co/go/tools v0.0.0-20190102054323-c2f93a96b099
)
$
```

要开始我们对 `github.com/googleapis/gax-go` 下的 `v2` 开发，先要创建一个新的 `v2/` 文件目录，随后将我们的包复制到其中。

```go
$ mkdir v2
$ cp *.go v2/
building file list ... done
call_option.go
gax.go
header.go
invoke.go
tools.go

sent 10588 bytes  received 130 bytes  21436.00 bytes/sec
total size is 10208  speedup is 0.95
$
```

现在，我们就复制现有的 `go.mod` 然后给其中的模块路径加上 `v2/` 前缀，这样创建一个 v2 的 `go.mod` 文件。

```bash
$ cp go.mod v2/go.mod
$ go mod edit -module github.com/googleapis/gax-go/v2 v2/go.mod
$
```

要注意，`v2` 版本被当作一个和 `v0 / v1` 版本分离的模块来对待：两者可以在同一次生成之中共存。所以如果你有多个 `v2+` 的模块，你是需要修改他们其中的引入路径的，否则的话他们依赖的将会是 `v0 / v1` 包。详细来说，就比如应该把 所有对 `github.com/my/project` 的引用都改成 `github.com/my/project/v2`。

这个可以通过 `find` 和 `sed` 完成：

```bash
$ find . -type f \
    -name '*.go' \
    -exec sed -i -e 's,github.com/my/project,github.com/my/project/v2,g' {} \;
$
```

现在 `v2` 算是有了，但是在公开之前，我们还是想要做一些实验和改动。在我们发布 `v2.0.0` （或者其他不带有预发布前缀的版本号）之前，我们可以自由的对包内容大动刀子，或者思考如何开放新的 API。如果我们希望能有用户在开发进行到比较完善的阶段之前，就对新的 API 进行测试，也可以发布 `v2` 预发布版本：

```bash
$ git tag v2.0.0-alpha.1
$ git push origin v2.0.0-alpha.1
$
```

等到我们对  `v2` 的 API 满意了，比较确定不会再有什么重大改变了，就可以创建 `v2.0.0` 的标签：

```bash
$ git tag v2.0.0
$ git push origin v2.0.0
$
```

这个节点上，我们就有两个大版本号要维护了。向后兼容和 bug 的修复会带来次版本号和补丁的发布（比如 `v1.1.0`、`v2.0.1` 之类的）。



## Conclusion

大版本的修改既会给开发和维护带来一定的负担，也会给下游的用户对向上迁移带来投入的要求。项目越大，这样的负担也越大。大版本的变动，只应当发生在发现了不容拒绝的理由的时。当有这么一个理由促使作出重大变动的时候，我们建议在 master 分支下开发多个大版本，这样和更多的既有工具兼容。

`v1+` 模块的突破性的更改，应该要总是做在一个新的 `vN+1` 模块之中。一个新的包被发布，就意味着维护者和想要迁移到新版本的用户要有额外的工作。所以维护者在发布稳定版本之前必须证实 API 确实正常工作，并且同时思考 `v1` 之外的突破性的更改是否真的有必要。

