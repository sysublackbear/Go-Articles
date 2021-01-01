# Go-Articles

最近在组内 Code Review 发现很多同学学习时间虽然不短，但Golang的一些基本Pattern仍然有所疏漏。

> 譬如，RPC接口内随意go协程，不考虑调用量大之后，协程暴涨可能导致程序OOM。再譬如，协程不知道有哪些同步、退出机制。

如果时间不是问题，那么就是知识体系存在漏洞。那么一个体系化、结构化的指南，对他们应该会帮助，因此整理之前主体阅读过的经典文章，形成了本篇文章。

## 1.Golang Design Cornerstone

> Do not communicate by sharing memory; instead, share memory by communicating.
