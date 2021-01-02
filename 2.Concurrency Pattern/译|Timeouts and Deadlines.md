# 译|Timeouts and Deadlines



抛弃长时间运行的同步调用吧，使用`select`语句和`time.After`函数：

```go
import "time"

c := make(chan error, 1)  // 重要,这里记住是带缓冲非阻塞管道
go func() { c <- client.Call("Service.Method", args, &reply) } ()
select {
  case err := <-c:
    // use err and reply
  case <-time.After(timeoutNanoseconds):
    // call timed out
}
```

注意到管道`c`的缓冲长度为1。这里值得注意一个问题，如果`c`定义为阻塞管道(`c := make(chan error)`)，当`client.Call`调用花费超过了`timeoutNanoseconds`的时候，管道的发送端（即：`go func() { c <- client.Call("Service.Method", args, &reply) } ()`将会永久阻塞（因为主协程退出，没有人消费）而导致协程泄漏。

