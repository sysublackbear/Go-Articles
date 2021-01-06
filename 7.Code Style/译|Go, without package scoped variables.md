# 译|Go, without package scoped variables

让我们进行一个思维实验，假如我们不能在包级别声明变量，go语言会变得怎么样？当我们删除包作用域的声明变量的时候，会产生什么影响？以及我们能够学习到go程序的设计吗？

我只探讨删除变量，和五个顶级声明。这样的情况仍然是被允许的，因为它们会在编译期间当做常量进行处理。当然，你还是可以在函数内部或者块内部声明变量。



## Why are package scoped variables bad?

首先，为什么包作用域的变量会这么糟糕呢？抛开这个问题，对于大量并发的语言属于可变的状态范围探讨，包作用域变量从根本上来说是个单例，有可能会导致在不相关的包之间交换了状态，鼓吹了紧密耦合，使得代码测试的时候相互依赖严重。

Peter Bourgon 最近写过：

> tl;dr: magic is bad; global state is magic → [therefore, you want] no package level vars; no func init.



## Removing package scoped variables, in practice

想要应用这个方法到测试里面，我调研了众多Go代码里面最受欢迎的代码例子；看看go的标准库，去了解包作用域变量是怎么被使用的，已经评估下对这个实验的影响。

### Errors

包作用域变量里面使用得最频繁的是`error`。比如：`io.EOF`，`sql.ErrNoRows`，`crypto/x509.ErrUnsupportedAlgorithm`，等等。删除包作用域的变量将会移除了使用了哨兵错误值的能力。但是，我们又能用什么方式去替代它们呢？