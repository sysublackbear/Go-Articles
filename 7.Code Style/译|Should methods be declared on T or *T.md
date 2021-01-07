# 译|Should methods be declared on T or *T

这篇文章是我前些天在Twitter的[a suggestion](https://twitter.com/davecheney/status/710604764640256000)的内容延续。

在Go中，对于任何的类型`T`，都会存在一个类型`*T`代表对于`T`类型变量的地址。例如：

```go
type T struct { a int; b bool }
var t T    // t's type is T
var p = &t // p's type is *T
```

这两个类型`T`和`*T`是不同的，但是`*T`不可以替代`T`。

你能够定义了一个方法在你拥有的任意类型；这就是，在你的包里面所声明的类型。因此这让你的方法建立在你声明的类型`T`，或者对应它的派生指针类型`*T`。另外一个考虑的点是，建立在一个类型的方法到底是进行值传递还是指针传递。因此问题变成：我们使用哪种接收者（`T`和`*T`）更为合适呢？

显而易见，如果你的方法会修改它的接收者，它应该定义在`*T`类型上。然而，如果你的方法不需要修改它的接收者，那么是不是使用`T`类型比较安全呢？

下面说明了从安全角度去这么做的原因其实非常有限。举个例子，众所周知你不应该拷贝一个`sync.Mutex`值类型因为它打破了锁的不变性。因此对锁控制的访问，它们会用一个struct进行包装，使用了`*T`类型接收者：

```go
package counter

type Val struct {
        mu  sync.Mutex
        val int
}

func (v *Val) Get() int {
        v.mu.Lock()
        defer v.mu.Unlock()
        return v.val
}

func (v *Val) Add(n int) {
        v.mu.Lock()
        defer v.mu.Unlock()
        v.val += n
}
```

大多数go程序员知道忘记定义`Get`或者`Add`方法在类型接收者`*Val`是一个错误。然而，任何嵌入`Val`会被利用它的零值。这样也会产生问题：

```go
type Stats struct {
        a, b, c counter.Val
}

func (s Stats) Sum() int {
        return s.a.Get() + s.b.Get() + s.c.Get() // whoops
}
```

一个出现的陷阱诸如：[unintended data race](http://dave.cheney.net/2015/11/18/wednesday-pop-quiz-spot-the-race)。



总结下，我认为你在下面这些场合使用`*T`作为接收者比较好，除非你有另外很强烈的原因不这么做。

1. 我们认为`T`只是你所声明的类型的一个占位符。
2. 1的规则是递归的，取一个`*T`类型的变量地址返回`**T`的类型结果。
3. 这就是为什么没有人在原始类型，像`int`上面定义方法的原因。
4. go里面的方法只是这样的一个语法糖，它的本质还是一个函数，传递它的接收者作为它的第一个固定参数。
5. 如果方法并不需要改变它的接收者，思考下，**它有必要是一个方法吗**？为啥不能是函数？