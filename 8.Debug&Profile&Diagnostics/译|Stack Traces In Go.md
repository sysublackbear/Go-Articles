 

# 译|Stack Traces In Go



## Introduction

拥有基本的调试Go程序技能可以节省程序员很大的时间来发现问题。我当然相信你可以使用log信息来跟踪问题，但是有时候panic发生的时候log信息并没有提供充足的信息。如果你理解堆栈跟踪的信息，你可以即时的找出bug, 这和传统的利用日志追踪bug有很大的不同， 因为利用日志的话你需要增加更多的log然后再等待相同的错误发生。

自打我开始写Go程序的时候我就一直看堆栈跟踪信息。有些地方我们写了傻傻的代码导致运行时杀死了我们的程序并且抛出堆栈跟踪信息。我将演示堆栈跟踪信息能提供些什么信息，包括怎么找到我们传递给函数的参数的值。



## Functions

让我们从一个很小的代码片段里面看下它的堆栈树：

### Listing 1

```go
package main

func main() {
    slice := make([]string, 2, 4)
    Example(slice, "hello", 10)
}

func Example(slice []string, str string, i int) {
  panic("Want stack trace")
}
```

列表1是一个简单的程序， `main`函数在第5行调用`Example`函数。`Example`函数在第8行声明，它有三个参数，一个字符串`slice`,一个字符串和一个整数。它的方法体也很简单，只有一行，抛出一个`panic`，这会立即产生一个堆栈跟踪信息：



###  Listing 2

```bash
Panic: Want stack trace

goroutine 1 [running]:
main.Example(0x2080c3f50, 0x2, 0x4, 0x425c0, 0x5, 0xa)
        /Users/bill/Spaces/Go/Projects/src/github.com/goinaction/code/
        temp/main.go:9 +0x64
main.main()
        /Users/bill/Spaces/Go/Projects/src/github.com/goinaction/code/
        temp/main.go:5 +0x85

goroutine 2 [runnable]:
runtime.forcegchelper()
        /Users/bill/go/src/runtime/proc.go:90
runtime.goexit()
        /Users/bill/go/src/runtime/asm_amd64.s:2232 +0x1

goroutine 3 [runnable]:
runtime.bgsweep()
        /Users/bill/go/src/runtime/mgc0.go:82
runtime.goexit()
        /Users/bill/go/src/runtime/asm_amd64.s:2232 +0x1
```

列表2显示了panic发生时的所有的goroutine，每一个goroutine的状态，每一个goroutine的状态，以及相应的调用堆栈。导致panic的gotoutine在最上面，我们只看这它的堆栈信息。



### Listing 3

```go
goroutine 1 [running]:
main.Example(0x2080c3f50, 0x2, 0x4, 0x425c0, 0x5, 0xa)
        /Users/bill/Spaces/Go/Projects/src/github.com/goinaction/code/temp/main.go:9 +0x64
main.main()
        /Users/bill/Spaces/Go/Projects/src/github.com/goinaction/code/temp/main.go:5 +0x85
```

列表3的第一行显示panic发生前运行的goroutine是id为 1的goroutine。第二行是发生panic的代码位置，位于main package下的Example函数。它也显示了代码所在的文件和路径，以及panic发生的行数(第9行)。

Line 03也调用Example的函数的名字，它是main package的main函数。它也显示了文件名和路径，以及调用Example函数的行数。

堆栈跟踪信息显示了 panic发生时的这个goroutine的函数调用链。现在让我们看看传递给Example的参数的值。



### Listing 4

```go
// Declaration
main.Example(slice []string, str string, i int)

// Call to Example by main.
slice := make([]string, 2, 4)
Example(slice, "hello", 10)

// Stack trace
main.Example(0x2080c3f50, 0x2, 0x4, 0x425c0, 0x5, 0xa)
```

列表4列举了`Example`函数的声明，调用以及传递给它的值的信息。当你比较函数的声明以及传递的值时，发现它们并不一致。函数声明只接收三个参数，而堆栈中却显示6个16进制表示的值。理解这一点的关键是要知道每个参数类型的实现机制。

让我们看第一个[]string类型的参数。slice是引用类型，这意味着那个值是一个指针的头信息(header value)，它指向一个字符串。对于slice,它的头是三个word数，指向一个数组。因此前三个值代表这个slice。



###  Listing 5

```go
// Slice parameter value
slice := make([]string, 2, 4)

// Slice header values
Pointer:  0x2080c3f50
Length:   0x2
Capacity: 0x4

// Declaration
main.Example(slice []string, str string, i int)

// Stack trace
main.Example(0x2080c3f50, 0x2, 0x4, 0x425c0, 0x5, 0xa)
```

列表5显示了`0x2080c3f50`代表第一个参数[]string的指针，`0x2`代表slice长度，`0x4`代表容量。这三个值代表第一个参数。



### Figure 1

![Screen Shot](https://www.ardanlabs.com/images/goinggo/image02.png)







