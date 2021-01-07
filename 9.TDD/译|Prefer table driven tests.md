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





