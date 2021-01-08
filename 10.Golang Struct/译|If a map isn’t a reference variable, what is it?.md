# 译|If a map isn’t a reference variable, what is it?



在我之前的[文章](https://dave.cheney.net/2017/04/29/there-is-no-pass-by-reference-in-go)，我已经说过Go的maps不是引用变量，它无法进行引用传递。这遗留下了问题，如果maps不是引用变量，那它们是什么？

如果简单来说，答案是：

> map是一个指向runtime.hmap数据结构的指针。

如果你不满足这样的解释，可以继续阅读下去。



## What is the type of a map value?

当你写下如下代码：

```go
m := make(map[int]int)
```

编译器会在编译的时候把这个函数替换成`runtime.makemap`，它的签名内容如下：

```go
// makemap implements a Go map creation make(map[k]v, hint)
// If the compiler has determined that the map or the first bucket
// can be created on the stack, h and/or bucket may be non-nil.
// If h != nil, the map can be created directly in h.
// If bucket != nil, bucket can be used as the first bucket.
func makemap(t *maptype, hint int64, h *hmap, bucket unsafe.Pointer) *hmap
```

正如你所见的，函数`runtime.makemap`的返回数据类型是一个指向`runtime.hmap`数据结构的指针。我们不能够从普通的go代码去发现这个东西，但是我们能够确认一个map的值得长度和`uintptr`相等。

```go
package main

import (
	"fmt"
	"unsafe"
)

func main() {
	var m map[int]int
	var p uintptr
	fmt.Println(unsafe.Sizeof(m), unsafe.Sizeof(p)) // 8 8 (linux/amd64)
}
```



## If maps are pointers, shouldn’t they be *map[key]value?

有一个很好的问题是，如果maps是个指针类型的值，`make(map[int]int)`返回了类型`map[int]int`。为什么它不返回一个`*map[int]int`值呢？lan Taylor在[answered this recently in a golang-nuts](https://groups.google.com/d/msg/golang-nuts/SjuhSYDITm4/jnrp7rRxDQAJ)回答道：

> In the very early days what we call maps now were written as pointers, so you wrote *map[int]int. We moved away from that when we realized that no one ever wrote `map` without writing `*map`.

其实有证据可以说明为什么要把`*map[int]int`转变成`map[int]int`，因为这个类型看起来就不像是个指针，之前的用法会让人觉得疑惑。尤其是反向引用的时候。



## Conclusion

maps，跟channels类型，不像slices，他们属于指向runtime类型的指针。正如我们上面所见，map是指向`runtime.hmap`数据结构的指针。

maps拥有go程序中的其它指针类型的同样语义。但是它并不存在语法糖能够让你直接可以调用到`runtime/hmap.go`内部的功能函数。