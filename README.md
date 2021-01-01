# Go-Articles

最近在组内 Code Review 发现很多同学学习时间虽然不短，但Golang的一些基本Pattern仍然有所疏漏。

> 譬如，RPC接口内随意go协程，不考虑调用量大之后，协程暴涨可能导致程序OOM。再譬如，协程不知道有哪些同步、退出机制。

如果时间不是问题，那么就是知识体系存在漏洞。那么一个体系化、结构化的指南，对他们应该会帮助，因此整理之前主体阅读过的经典文章，形成了本篇文章。

## 1.Golang Design Cornerstone

> Do not communicate by sharing memory; instead, share memory by communicating.

+ [《Share Memory By Communicating》](https://blog.golang.org/codelab-share)
+ [《Concurrency is not parallelism》](https://blog.golang.org/waza-talk)

## 2.Concurrency Pattern
+ [《Go Concurrency Patterns: Context》](https://blog.golang.org/context)
+ [《Go Concurrency Patterns: Pipelines and cancellation》](https://blog.golang.org/waza-talk)
+ [《Go Concurrency Patterns: Timing out, moving on》](https://blog.golang.org/concurrency-timeouts)
+ [《Advanced Go Concurrency Patterns》](https://blog.golang.org/io2013-talk-concurrency)
+ [《Context isn’t for cancellation》](https://dave.cheney.net/2017/08/20/context-isnt-for-cancellation)
+ [《Never start a goroutine without knowing how it will stop》](https://dave.cheney.net/2016/12/22/never-start-a-goroutine-without-knowing-how-it-will-stop)
+ [《Rethinking Classical Concurrency Patterns》](https://drive.google.com/file/d/1nPdvhB0PutEJzdCq5ms6UI58dp50fcAN/view)
+ [《Timeouts and Deadlines》](https://github.com/golang/go/wiki/Timeouts)

## 3.Error Handling
+ [《Errors are values》](https://blog.golang.org/errors-are-values)
+ [《Don’t just check errors, handle them gracefully》](https://dave.cheney.net/2016/04/27/dont-just-check-errors-handle-them-gracefully)

## 4.Naming
+ [《Package names》](https://blog.golang.org/package-names)
+ [《Avoid package names like base, util, or common》](https://dave.cheney.net/2019/01/08/avoid-package-names-like-base-util-or-common)

## 5.Package Management
+ [《Using Go Modules》](https://blog.golang.org/using-go-modules)

## 6.Logging
+ [《Let’s talk about logging》](https://dave.cheney.net/2015/11/05/lets-talk-about-logging)

## 7.Code Style
+ [《SOLID Go Design》](https://dave.cheney.net/2016/08/20/solid-go-design)
+ [《Do not fear first class functions》](https://dave.cheney.net/2016/11/13/do-not-fear-first-class-functions)
+ [《The Zen of Go》](https://dave.cheney.net/2020/02/23/the-zen-of-go)
+ [《Clear is better than clever》](https://dave.cheney.net/2019/07/09/clear-is-better-than-clever)
+ [《Go, without package scoped variables》](https://dave.cheney.net/2017/06/11/go-without-package-scoped-variables)
+ [《Should methods be declared on T or *T》](https://dave.cheney.net/2016/03/19/should-methods-be-declared-on-t-or-t)
+ [《Simplicity and collaboration》](https://dave.cheney.net/2015/03/08/simplicity-and-collaboration)

## 8.Debug&Profile&Diagnostics
+ [《Profiling Go Programs》](https://blog.golang.org/pprof)
+ [《Diagnostics》](https://golang.org/doc/diagnostics.html)
+ [《Stack Traces In Go》](https://www.ardanlabs.com/blog/2015/01/stack-traces-in-go.html)

## 9.TDD
+ [《Prefer table driven tests》](https://dave.cheney.net/2019/05/07/prefer-table-driven-tests)

## 10.Golang Struct
+ [《If a map isn’t a reference variable, what is it?》](https://dave.cheney.net/2017/04/30/if-a-map-isnt-a-reference-variable-what-is-it)
+ [《Slices from the ground up》](https://dave.cheney.net/2018/07/12/slices-from-the-ground-up)

## 11.Website
+ [Gopher Academy Blog](https://blog.gopheracademy.com/)
+ [The official Go Blog](https://blog.golang.org/)
+ [Go Wiki](https://github.com/golang/go/wiki)
+ [Dave Cheney](https://dave.cheney.net/)


以上内容转载自[cyningsun的博客文章:Golang进阶文章一览](https://www.cyningsun.com/10-15-2020/advanced-golang-article.html)，作者接下来会对上面的文章做一个翻译，持续更新中...



