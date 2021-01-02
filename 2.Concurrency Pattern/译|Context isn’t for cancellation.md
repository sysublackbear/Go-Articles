# 译|Context isn’t for cancellation



这是一篇有关Go语言中`context.Context`的使用，困难的经验之谈。

很多作者，包括我自己，都写了`context.Context`在未来的迭代中的使用，误用和它们怎么演进。针对`context.Context`的各个方面的争议很多，其中一个不约而同的点就是在`context.Context`接口中加入`Context.WithValue`方法（全局方法转化为接口的方法），作为上下文的资源的生命周期管理。

很多建议便是在`context.Context`自己排名和重载出一系列的`WithValue`方法。