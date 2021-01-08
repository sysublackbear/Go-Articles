# 译|Prefer table driven tests



我是测试的狂热粉丝，尤其是[单元测试](https://dave.cheney.net/2019/04/03/absolute-unit-test)和[TDD](https://www.youtube.com/watch?v=EZ05e7EMOLM)(*Test-driven development*，测试驱动开发)。一个在go项目中广泛使用的实践是表格驱动测试。这篇文章探索怎么和为什么要写表格测试。

让我们看一个切割函数`Split`的例子：

```go
// Split slices s into all substrings separated by sep and
// returns a slice of the substrings between those separators.
func Split(s, sep string) []string {
    var result []string
    i := strings.Index(s, sep)
    for i > -1 {
        result = append(result, s[:i])
        s = s[i+len(sep):]
        i = strings.Index(s, sep)
    }
    return append(result, s)
}
```

在golang中，单元测试的实现非常常规，就是在同一个文件里面新定义一个`xxx_test.go`文件，它们共享同样的包名字和函数。如下：

```go
package split

import (
    "reflect"
    "testing"
)

func TestSplit(t *testing.T) {
    got := Split("a/b/c", "/")
    want := []string{"a", "b", "c"}
    if !reflect.DeepEqual(want, got) {
         t.Fatalf("expected: %v, got: %v", want, got)
    }
}
```

go的单元测试只需要满足如下规则：

1. 单元测试的函数必须要以`Test`开头；
2. 单元测试的函数必须要有类型为``*testing.T`作为参赛。`*testing.T`是一个内置的注入到`testing`包的变量，提供了打印，跳过以及抛出失败的方式。

在我们的测试例子里面，我们输入一些参数到`Split`，然后比较下我们期望的结果。



## Code coverage

下一个问题就是，就是这个包的代码覆盖率是多少？幸运的是go tool有一个内置的代码覆盖率统计功能。我们能够如下操作：

```bash
% go test -coverprofile=c.out
PASS
coverage: 100.0% of statements
ok      split   0.010s
```

我们看到代码覆盖率高达100%，不过真的不需要惊讶，因为你这个代码只有一个分支。

如果我们想要深入去写一份有关代码覆盖率的报告，`go tool`有若干个选项去打印代码覆盖率的信息。我们能够使用`go tool cover -func`去分别分析每个函数的代码覆盖率：

```bash
% go tool cover -func=c.out
split/split.go:8:       Split          100.0%
total:                  (statements)   100.0%
```

这个结果不会让你大吃一惊，因为在这个包里面只有一个函数，但是我相信你将会找到更多让你兴奋的包进行测试。



### Spray some .bashrc on that

命令对与我而言是多么的实用，我已经在我的`.bashrc`里面声明了一些`alias`让我可以用一条命令就可以跑代码测试的覆盖率以及输出报告。

```bash
cover () {
    local t=$(mktemp -t cover)
    go test $COVERFLAGS -coverprofile=$t $@ \
        && go tool cover -func=$t \
        && unlink $t
}
```



## Going beyond 100% coverage

因此，当我们写一个测试案例，得到100%的覆盖率，但这并不是真正的告一段落了。我们有很好的代码分支覆盖但是我们可能还是需要测试其中的一些边界条件。例如，如果我们想要尝试以逗号作为分隔将会发生什么？

```go
func TestSplitWrongSep(t *testing.T) {
    got := Split("a/b/c", ",")
    want := []string{"a/b/c"}
    if !reflect.DeepEqual(want, got) {
        t.Fatalf("expected: %v, got: %v", want, got)
    }
}
```

或者如果原字符串不包含分隔符呢？

```go
func TestSplitNoSep(t *testing.T) {
    got := Split("abc", "/")
    want := []string{"abc"}
    if !reflect.DeepEqual(want, got) {
        t.Fatalf("expected: %v, got: %v", want, got)
    }
}
```

我们如果有一个表格式的用例，里面收录了各种边界的条件的话，这样去构建单元测试是比较好的。



## Introducing table driven tests

在我们的单元测试案例里面，存在很多的重复。对于每个测试用例存在它的输入，和期望得到的输出和每个测试用例的名称。其他的东西都只是模板，可以被重复利用。我们需要建立起一个工具媒介把所有的输入和期望的输出汇集起来。这是一个介绍表格驱动测试的绝佳机会。

```go
func TestSplit(t *testing.T) {
    type test struct {
        input string
        sep   string
        want  []string
    }

    tests := []test{
        {input: "a/b/c", sep: "/", want: []string{"a", "b", "c"}},
        {input: "a/b/c", sep: ",", want: []string{"a/b/c"}},
        {input: "abc", sep: "/", want: []string{"abc"}},
    }

    for _, tc := range tests {
        got := Split(tc.input, tc.sep)
        if !reflect.DeepEqual(tc.want, got) {
            t.Fatalf("expected: %v, got: %v", tc.want, got)
        }
    }
}
```

我们声明了一个数据结构，来记录我们的输入和期望得到的输出。这就是我们的表格。测试结构是一个本地的声明因为我们想要在这个包的其它测试例子去复用这个表格。

事实上，我们不需要给一个名字的类型，我们能够使用一个匿名结构去声明即可：

```go
func TestSplit(t *testing.T) {
    tests := []struct {
        input string
        sep   string
        want  []string
    }{
        {input: "a/b/c", sep: "/", want: []string{"a", "b", "c"}},
        {input: "a/b/c", sep: ",", want: []string{"a/b/c"}},
        {input: "abc", sep: "/", want: []string{"abc"}},
    } 

    for _, tc := range tests {
        got := Split(tc.input, tc.sep)
        if !reflect.DeepEqual(tc.want, got) {
            t.Fatalf("expected: %v, got: %v", tc.want, got)
        }
    }
}
```

现在，添加一个新的例子成为一个问题；我们只需要在代码里面添加一行：

```go
{input: "a/b/c", sep: "/", want: []string{"a", "b", "c"}},
{input: "a/b/c", sep: ",", want: []string{"a/b/c"}},
{input: "abc", sep: "/", want: []string{"abc"}},
{input: "a/b/c/", sep: "/", want: []string{"a", "b", "c"}}, // trailing sep
```

但是，我们现在在跑例子，我们能够得到：

```bash
% go test
--- FAIL: TestSplit (0.00s)
    split_test.go:24: expected: [a b c], got: [a b c ]
```

撇开测试错误，这里还是有一些问题需要讨论下。

第一个就是通过这种方式重写错误的时候，当我们测试案例失败的时候，我们并不知道在哪个测试用例失败的。我们虽然在代码添加了一些注释内容，但是在跑测试用例的输出里面，我们找不到这些内容。

这里还是有好几种方法去解决这个问题的。



## Enumerating test cases

我们把代码做如下修改：

```go
func TestSplit(t *testing.T) {
    tests := []struct {
        input string
        sep . string
        want  []string
    }{
        {input: "a/b/c", sep: "/", want: []string{"a", "b", "c"}},
        {input: "a/b/c", sep: ",", want: []string{"a/b/c"}},
        {input: "abc", sep: "/", want: []string{"abc"}},
        {input: "a/b/c/", sep: "/", want: []string{"a", "b", "c"}},
    }

    for i, tc := range tests {
        got := Split(tc.input, tc.sep)
        if !reflect.DeepEqual(tc.want, got) {
            t.Fatalf("test %d: expected: %v, got: %v", i+1, tc.want, got)  // 注意这里把i也打印出来
        }
    }
}
```

现在我们运行`go test`我们能够得到如下信息：

```bash
% go test
--- FAIL: TestSplit (0.00s)
    split_test.go:24: test 4: expected: [a b c], got: [a b c ]
```

这样得到的效果比较好，我们能够知道在哪个例子里面发生了错误。但是，通过打印出切片下标来表示的这种方式还是比较不够清晰的。



## Give your test cases names

另外一种方法就是在你的`struct`结构中添加`name`字段属性。

```go
func TestSplit(t *testing.T) {
    tests := []struct {
        name  string
        input string
        sep   string
        want  []string
    }{
        {name: "simple", input: "a/b/c", sep: "/", want: []string{"a", "b", "c"}},
        {name: "wrong sep", input: "a/b/c", sep: ",", want: []string{"a/b/c"}},
        {name: "no sep", input: "abc", sep: "/", want: []string{"abc"}},
        {name: "trailing sep", input: "a/b/c/", sep: "/", want: []string{"a", "b", "c"}},
    }

    for _, tc := range tests {
        got := Split(tc.input, tc.sep)
        if !reflect.DeepEqual(tc.want, got) {
            t.Fatalf("%s: expected: %v, got: %v", tc.name, tc.want, got)
        }
    }
}
```

这样，当我们在某个用例出现了错误的时候，我们能够清晰地知道是哪个用例出现了问题。

```bash
% go test
--- FAIL: TestSplit (0.00s)
    split_test.go:25: trailing sep: expected: [a b c], got: [a b c ]
```

我们可以直接把切片改造成`map`进行尝试：

```go
func TestSplit(t *testing.T) {
    tests := map[string]struct {
        input string
        sep   string
        want  []string
    }{ 
        "simple":       {input: "a/b/c", sep: "/", want: []string{"a", "b", "c"}}, 
        "wrong sep":    {input: "a/b/c", sep: ",", want: []string{"a/b/c"}},
        "no sep":       {input: "abc", sep: "/", want: []string{"abc"}},
        "trailing sep": {input: "a/b/c/", sep: "/", want: []string{"a", "b", "c"}},
    }

    for name, tc := range tests {
        got := Split(tc.input, tc.sep)
        if !reflect.DeepEqual(tc.want, got) {
            t.Fatalf("%s: expected: %v, got: %v", name, tc.want, got)
        }
    }
}
```

这样就比较清晰明了。



## Introducing sub tests

在我们修复错误的单元测试的例子之前，我们还有其他的一些问题需要去解决。

第一个就是，当我们其中一个测试用例失败了，我们会调用`t.Fatalf`。这意味着后面的测试用例都会被停止执行了。由于测试用例执行的顺序是不确定的（尽管golang的map是个哈希表，但是golang官方说了，防止某些人利用顺序输出来做啥事情，所以会把输出顺序打乱），如果有一个测试用例跑失败了，它将会导致输出结果不确定。

Go1.7版本之后很好地解决了这个问题，让我们能够更容易实现表格驱动测试。它们被称为 [sub tests](https://blog.golang.org/subtests)。

```go
func TestSplit(t *testing.T) {
    tests := map[string]struct {
        input string
        sep   string
        want  []string
    }{
        "simple":       {input: "a/b/c", sep: "/", want: []string{"a", "b", "c"}},
        "wrong sep":    {input: "a/b/c", sep: ",", want: []string{"a/b/c"}},
        "no sep":       {input: "abc", sep: "/", want: []string{"abc"}},
        "trailing sep": {input: "a/b/c/", sep: "/", want: []string{"a", "b", "c"}},
    }

    for name, tc := range tests {
        t.Run(name, func(t *testing.T) {
            got := Split(tc.input, tc.sep)
            if !reflect.DeepEqual(tc.want, got) {
                t.Fatalf("expected: %v, got: %v", tc.want, got)
            }
        })
    }
}
```

然后每个测试用例都会在我们的测试循环里面自动地单独打印内容。

```bash
% go test
--- FAIL: TestSplit (0.00s)
    --- FAIL: TestSplit/trailing_sep (0.00s)
        split_test.go:25: expected: [a b c], got: [a b c ]
```

每个子测试都有自己的异步函数，因此我们可以使用`t.Fatalf`，`t.Skipf`和其他`testing.T`的工具函数，同时也保证了整个表格驱动测试的紧凑性。

### Individual sub test cases can be executed directly

因为子测试有自己的名字，你能够通过使用`go test -run`参数去直接跑子测试的某个片段。

```bash
% go test -run=.*/trailing -v
=== RUN   TestSplit
=== RUN   TestSplit/trailing_sep
--- FAIL: TestSplit (0.00s)
    --- FAIL: TestSplit/trailing_sep (0.00s)
        split_test.go:25: expected: [a b c], got: [a b c ]
```



## Comparing what we got with what we wanted

现在我们准备修复这个测试用例了。让我们看以下错误：

```bash
--- FAIL: TestSplit (0.00s)
    --- FAIL: TestSplit/trailing_sep (0.00s)
        split_test.go:25: expected: [a b c], got: [a b c ]
```

你能点出上面的问题所在了吗？很显然切片是不能直接比较的。这是`reflect.DeepEqual`有所让人沮丧的。但是有时候指出这种区别是不容易的，你必须清晰地指出变量`c`之后存在额外的空余空间。这貌似在这个简单的例子里面，你还能很容易就看出来区别，然而当我们比较两个深入嵌套的gRPC结构的时候，会变得非常困难。

我们能够改善输出，通过用golang的`%#v`的输出符，如下：

```go
got := Split(tc.input, tc.sep)
if !reflect.DeepEqual(tc.want, got) {
    t.Fatalf("expected: %#v, got: %#v", tc.want, got)
}
```

现在我们可以看到我们的单元测试输出，会明显看出了一个额外的空格。

```bash
% go test
--- FAIL: TestSplit (0.00s)
    --- FAIL: TestSplit/trailing_sep (0.00s)
        split_test.go:25: expected: []string{"a", "b", "c"}, got: []string{"a", "b", "c", ""}
```

然而，这还是有一些例子会说明`%#v`不会很好地生效：

```go
func main() {
    type T struct {
        I int
    }
    x := []*T{{1}, {2}, {3}}
    y := []*T{{1}, {2}, {4}}
    fmt.Printf("%v %v\n", x, y)
    fmt.Printf("%#v %#v\n", x, y)
}
```

首先`fmt.Printf`打印无效的，切片内容的地址：`[0xc000096000 0xc000096008 0xc000096010] [0xc000096018 0xc000096020 0xc000096028]`。然而我们的`%#v`不能做得更好了，打印了一系列的地址信息，易读性差。`[]*main.T{(*main.T)(0xc000096000), (*main.T)(0xc000096008), (*main.T)(0xc000096010)} []*main.T{(*main.T)(0xc000096018), (*main.T)(0xc000096020), (*main.T)(0xc000096028)}`。

由于`fmt.Prinf`具有这样的局限性，我想要介绍你使用Google的[go-cmp](https://github.com/google/go-cmp)库。

cmp库的目标在于它能够很好地比较两个值。与`reflect.DeepEqual`比较相似，但是它具有更多能力。使用cmp包，你的代码如下：

```go
func main() {
    type T struct {
        I int
    }
    x := []*T{{1}, {2}, {3}}
    y := []*T{{1}, {2}, {4}}
    fmt.Println(cmp.Equal(x, y)) // false
}
```

但是更实用的是，使用`cmp.Diff`函数，我们能够递归打印出两个值的差异性。

```go
func main() {
    type T struct {
        I int
    }
    x := []*T{{1}, {2}, {3}}
    y := []*T{{1}, {2}, {4}}
    diff := cmp.Diff(x, y)
    fmt.Printf(diff)
}
```

会有如下输出：

```bash
% go run
{[]*main.T}[2].I:
         -: 3
         +: 4
```

最后，重写我们的表格驱动测试代码，如下：

```go
func TestSplit(t *testing.T) {
    tests := map[string]struct {
        input string
        sep   string
        want  []string
    }{
        "simple":       {input: "a/b/c", sep: "/", want: []string{"a", "b", "c"}},
        "wrong sep":    {input: "a/b/c", sep: ",", want: []string{"a/b/c"}},
        "no sep":       {input: "abc", sep: "/", want: []string{"abc"}},
        "trailing sep": {input: "a/b/c/", sep: "/", want: []string{"a", "b", "c"}},
    }

    for name, tc := range tests {
        t.Run(name, func(t *testing.T) {
            got := Split(tc.input, tc.sep)
            diff := cmp.Diff(tc.want, got)
            if diff != "" {
                t.Fatalf(diff)
            }
        })
    }
}
```

运行的时候，我们得到：

```bash
% go test
--- FAIL: TestSplit (0.00s)
    --- FAIL: TestSplit/trailing_sep (0.00s)
        split_test.go:27: {[]string}[?->3]:
                -: <non-existent>
                +: ""
FAIL
exit status 1
FAIL    split   0.006s
```

通过使用上面的一系列改造，我们能够得到输出格式良好的内容。值得借鉴和学习。



