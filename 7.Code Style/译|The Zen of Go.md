# 译|The Zen of Go

编写简单、可读、可维护Go代码的十个工程经验, Dave Cheney于2020/02/03在 [GopherCon Israel 2020](https://www.gophercon.org.il/schedule.html)的演讲。



## Each package fulfils a single purpose（每个package实现单一的目的）

一个设计良好的包提供单一目的，和相关行为的集合。一个设计良好的包从一个好的包名开始。考虑你的包名是否和你所描述的功能相关，而不单单是一个单词。



## Handle errors explicitly（显式处理错误）

健壮的代码应该是由处理异常的小片段去组成。`if err != nil { return err }`的冗长性被故意在发生故障的每个点处理故障的值所抵消。`panic`和`recover`也不例外，它们并非打算以这种方式使用。



## Return early rather than nesting deeply（尽早返回，而不是使用深嵌套）

每次缩进时，您都会在程序员的堆栈中添加另一个先决条件，这会占用他们短期内存中的7±2个插槽之一。避免需要深缩进的控制流。与其深入嵌套，不如使用保护子句将成功路径保持在左侧。



## Leave concurrency to the caller（让调用者选择并发）

让调用者选择是否要异步运行您的库或函数，不要强加于他们。如果您的库使用并发，则应透明地进行。



## Before you launch a goroutine, know when it will stop（在启动一个goroutine时，需要知道何时它会停止）

Goroutines拥有资源；锁，变量，内存等。释放这些资源的可靠方法是停止拥有的goroutine。



## Avoid package level state（避免package级别的状态）

通过提供类型需要的依赖项作为该类型上的字段，而不是使用包变量，来寻求明确的，减少耦合和诡异的动作。



## Simplicity matters（简单很重要）

简单性不是老练的代名词。简单并不意味着粗糙，而是可读性和可维护性。如果可以选择，请遵循较简单的解决方案。



## Write tests to lock in the behaviour of your package’s API（编写测试以锁定 package API的行为）

请确保测试用户可以观察和依赖的行为。



## If you think it’s slow, first prove it with a benchmark（如果觉得慢，首先编写benchmark来证明）

以表现为名犯下了许多危害可维护性的罪行。优化会破坏抽象，暴露内部和紧密耦合。如果您选择承担这笔费用，请确保有充分理由这样做。



## Moderation is a virtue（节制是一种美德）

适度使用goroutine，通道，锁，接口，嵌入。



## Maintainability counts（可维护性计数）

清晰，易读，简单是可维护性的所有方面。离开后，您可以努力维护的东西可以保留吗？您今天该如何做，才能使以后的人们变得更轻松？

 

