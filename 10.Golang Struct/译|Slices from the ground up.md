# 译|Slices from the ground up

这篇文章是之前和一个同事讨论使用切片来实现栈而有感而发。这个对话变成了对go中slices的运作原理的讨论，因此我觉得这个东西非常实用，遍记录下来。



## Arrays

每个对go中slices的讨论便是，他们觉得go的slices不是切片类型，更像是一个array类型。go中的arrays拥有如下两个相关特性：

1. 它们有固定长度；`[5]int`是一个长度为5的数据，区别于`[3]int`；
2. 它们是值类型，看下如下代码：

```go
package main

import "fmt"

func main() {
        var a [5]int
        b := a
        b[2] = 7
        fmt.Println(a, b) // prints [0 0 0 0 0] [0 0 7 0 0]
}
```

`b := a`声明出了一个新的变量，`b`，具有`[5] int`类型，直接拷贝了`a`的内容。更改`b`的内容对`a`没有任何影响，因为`a`和`b`都有各自独立的空间。



## Slices

go的slices区别于它的array类型主要是如下两个方面：

1. Slices并没有固定的长度；一个slice的长度不会作为它的类型进行声明；
2. 声明一个slice不会做一个额外的空间拷贝，这是因为slice并不拥有它的内容空间。取而代之，slice拥有一个指向底层数组的指针，该底层数组记录真实内容；

**1.切割slices**

```go
package main

import "fmt"

func main() {
        var a = []int{1,2,3,4,5}
        b := a[2:]
        b[0] = 0
        fmt.Println(a, b) // prints [1 2 0 4 5] [0 4 5]
}
```

在这个例子之中，`a`和`b`共享同样的底层数组内容，只是`b`在一个不同的位移开始而已，两者具有不同的长度。修改`b`的内容会同时影响到`a`。



**2.往函数传递slice**

```go
package main

import "fmt"

func negate(s []int) {
        for i := range s {
                s[i] = -s[i]
        }
}

func main() {
        var a = []int{1, 2, 3, 4, 5}
        negate(a)
        fmt.Println(a) // prints [-1 -2 -3 -4 -5]
}
```

在这个例子当中，我们把`a`当做参数`s`传递给了函数`negate`。即使`negate`并没有返回值。但是通过在`main`函数定义`a`，`a`的内容也会被修改。

大多数程序员对go的slices的底层数组运作原理有一个直觉的认识因为它和其他语言的一些类似数组的概念，运作原理相近。例如，这里有个例子用python重写了：

```python
Python 2.7.10 (default, Feb  7 2017, 00:08:15) 
[GCC 4.2.1 Compatible Apple LLVM 8.0.0 (clang-800.0.34)] on darwin
Type "help", "copyright", "credits" or "license" for more information.
>>> a = [1,2,3,4,5]
>>> b = a
>>> b[2] = 0
>>> a
[1, 2, 0, 4, 5]
```

用ruby重写：

```ruby
irb(main):001:0> a = [1,2,3,4,5]
=> [1, 2, 3, 4, 5]
irb(main):002:0> b = a
=> [1, 2, 3, 4, 5]
irb(main):003:0> b[2] = 0
=> 0
irb(main):004:0> a
=> [1, 2, 0, 4, 5]
```

把数组当成对象或者引用类型同样适用于大多数的语言。



## The slice header value

定义一个slice的奇妙之处在于它既具备值的属性和指针的属性，但事实上它是一个`struct`类型。它通常被视为一个slice header，用于映射到reflect包的内容。slice header的定义看起来如下：

```go
package runtime

type slice struct {
        ptr   unsafe.Pointer
        len   int
        cap   int
}
```

![img](https://dave.cheney.net/wp-content/uploads/2018/07/slice.001-300x257.png)

值得注意的是，不像map和channel，切片是值类型，当它在声明的时候或者作为参数被传递到函数中，它的内容可以被拷贝。

为了说明这一特性，程序员会搬出如下的一个例子：

```go
package main

import "fmt"

func square(v int) {
        v = v * v
}

func main() {
        v := 3
        square(v)
        fmt.Println(v) // prints 3, not 9
}
```

可以看到，`square`中的`v`并不会影响`main`的`v`内容。然后我们假设把它换成切片：

```go
package main

import "fmt"

func double(s []int) {
        s = append(s, s...)  // 这里传递了个指针值，虽然触发了扩容，但是外部不可见，值得注意
}

func main() {
        s := []int{1, 2, 3}
        double(s)
        fmt.Println(s, len(s)) // prints [1 2 3] 3
}
```

这里可以清晰地看出了，go的slices变量是作为值去传递的，而不是作为一个指针进行传递。在你使用go的struct的时候，有90%的概率你会传递一个指针引用类型到函数当中。slices这种例子确实不常见，像这个的唯一反例可能只有`time.Time`了。

这样的一些特性确实会让程序员觉得疑惑，到底go的slices是怎么运作的。你只需要记住，当你声明，分片或者传递到函数或者从函数处返回一个slice的时候，你只是在拷贝slice header的三个要素的值(`ptr`, `len`, `cap`)。这个指针还是指向底层的数组，拥有当前的长度`len`和容量`cap`。



## Putting it all together

我将要用下面这个例子来总结这篇文章了：

```go
package main

import "fmt"

func f(s []string, level int) {
        if level > 5 {
               return
        }
        s = append(s, fmt.Sprint(level))
        f(s, level+1)
        fmt.Println("level:", level, "slice:", s)
}

func main() {
        f(nil, 0)
}
```

我们可以看到打印的内容如下：

```bash
level: 5 slice: [0 1 2 3 4 5]
level: 4 slice: [0 1 2 3 4]
level: 3 slice: [0 1 2 3]
level: 2 slice: [0 1 2]
level: 1 slice: [0 1]
level: 0 slice: [0]
```

你们可以看到每一层的值`s`都不会被其他`f`调用所影响，它会在每个函数里面都会重新构造新的slice。



或者看下面这一个例子，二叉树的中序遍历：

```go

func inOrder(root *TreeNode, array []int)  {
	if root == nil {
		return
	}
	inOrder(root.Left, array)
	array = append(array, root.Val)
	inOrder(root.Right, array)
}
```

我们知道，`array`会以值的方式传递给`inOrder`函数中。

> slice由指向底层数组的指针`ptr`(`*Elem`)，切片的长度`len`(`int`)，切片的容量`cap`(`int`)组成，在64位系统下，一个切片占24Byte的内存(`*Elem`占8Bytes，`len`占8Bytes，`cap`占8Bytes)，因此，传递参数时，如果传递slice，无论底层数组多大，都只会有24Bytes的拷贝，这点是和数组不同的。

因此我们传入的是指向数组的指针，能够共享底层的指针结构。

我们在`array = append(array, root.Val)`触发了slice的扩容了，扩容过程是：

+ 先分配一段新的内存
+ 然后把原来数组的内容拷贝过去

这里扩容后，把指针值赋值给了`array`。但是：**array里那个指向数组的指针，也是以值的形式传进来的，也就是说，函数内部修改了这个指针（注意是修改了指针本身，而不是修改了指针指向的内容），外面是看不到的**。

再看下这个例子来加深印象：

```go
package main

import "fmt"

func double(s []int) {
	s[0] += 100
	s = append(s, s...) // 这里传递了个指针值，虽然触发了扩容，但是外部不可见，值得注意
}

func main() {
	s := []int{1, 2, 3}
	double(s)
	double(s)
	fmt.Println(s, len(s)) // prints [201 2 3] 3
}
```

这个例子很好地体现了go的slices既具备值传递类型的特点（内部对其进行扩容，外部不受其影响），也具备引用传递类型的特点（对切片内部的成员进行修改，外部能够看到，因为它们共享相同的底层地址空间）。



这个例子确实值得注意，如果想要上面例子，扩容生效。**需要把函数改成了传递slice指针进来**，如下：

```go
func inOrder(root *TreeNode, array *[]int)  {
	if root == nil {
		return
	}
	inOrder(root.Left, array)
	*array = append(*array, root.Val)
	inOrder(root.Right, array)
}
```



##  Conclusion

slice作为参数，如果以值的形式传递，确实可以在函数内部修改数组，**但前提是**，**函数内部slice不会扩容**，如果函数内部slice会扩容，那还是乖乖地传递slice指针进去吧。