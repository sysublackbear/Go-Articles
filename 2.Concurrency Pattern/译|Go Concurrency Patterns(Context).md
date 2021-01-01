# 译|Go Concurrency Patterns: Context



## Introduction

在Go语言实现的服务器中，每一个进来的请求都会在属于它自己的goroutine中运行(One request per goroutine)。请求处理程序通常会新开辟goroutine去访问后端服务，例如访问数据库或者rpc调用等。服务于请求的一组常用典型的goroutines访问特定的请求值，例如最终用户的身份，授权令牌和请求的截止日期等等。当请求被取消或触发超时的时候，在该请求上工作的所有goroutine会快速退出，以便系统可以回收所使用的任何资源。



在google





