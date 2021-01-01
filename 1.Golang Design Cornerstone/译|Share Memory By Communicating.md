# 译|Share Memory By Communicating



传统的线程模型（例如在使用Java，C++和Python编写程序中被广泛使用的）要求开发通过使用共享内存进行线程间的通信。最常见的实现方式是，共享数据的数据结构通过锁来保证一致性，线程之间想要访问数据需要抢锁。在某些情况下，使用线程安全的数据结构更容易达到目的，比如Python的Queue。



Go语言的并发原语——goroutine 和 channels——为构建并发应用提供了一种优雅直接的手段。（goroutine和channel依托于[CSP](http://www.usingcsp.com/)(顺序通信模型)的概念，有兴趣可以查阅[这里](https://swtch.com/~rsc/thread/)。对比起需要显式地获得锁去访问共享数据，Go语言更加鼓励使用channel在goroutine之间传递数据。这种方法确保在某一特定时候只有一个goroutine能够访问数据。这个概念在[Effective Go](https://golang.org/doc/effective_go.html)（一本对于go程序员必须要读的一本书）有所总结。



> 不要通过共享内存来通信，取而代之，要通过通信来共享内存。



假设现在有一个爬虫，去爬取一堆url回来。在传统的线程模型中，或许数据结构的一种常见的实现方式如下：

```go
type Resource struct {
    url        string
    polling    bool
    lastPolled int64
}

type Resources struct {
    data []*Resource
    lock *sync.Mutex
}
```



然后这个爬虫程序（多线程并发）它的实现或许如下：

```go
func Poller(res *Resources) {
    for {
        // get the least recently-polled Resource
        // and mark it as being polled
        res.lock.Lock()
        var r *Resource
        for _, v := range res.data {
            if v.polling {
                continue
            }
            if r == nil || v.lastPolled < r.lastPolled {
                r = v
            }
        }
        if r != nil {
            r.polling = true
        }
        res.lock.Unlock()
        if r == nil {
            continue
        }

        // poll the URL

        // update the Resource's polling and lastPolled
        res.lock.Lock()
        r.polling = false
        r.lastPolled = time.Nanoseconds()
        res.lock.Unlock()
    }
}
```



上面这个函数大概有一页纸长，而且还省略了爬虫爬取的主要逻辑。而且这样的实现方式并没有优雅的消耗资源池的资源。

让我们看下使用go原语来实现同样相等的逻辑。在下面这个例子当中，Poller是这样的一个函数，它接收从输入channel而来的一个带爬取的资源，以及当它爬取结束之后，会把资源放到输出channel的功能。

```go
type Resource string

func Poller(in, out chan *Resource) {
    for r := range in {
        // poll the URL

        // send the processed Resource to out
        out <- r
    }
}
```

这个函数对比起上面的实现方式显得更加的微妙，而且我们的`Resource`数据结构不再包含记录锁状态的数据。

上面的实现代码只是一个省略版，完整版可以看如下源码：

```go
// Copyright 2010 The Go Authors. All rights reserved.
// Use of this source code is governed by a BSD-style
// license that can be found in the LICENSE file.

package main

import (
	"log"
	"net/http"
	"time"
)

const (
	numPollers     = 2                // number of Poller goroutines to launch
	pollInterval   = 60 * time.Second // how often to poll each URL
	statusInterval = 10 * time.Second // how often to log status to stdout
	errTimeout     = 10 * time.Second // back-off timeout on error
)

var urls = []string{
	"http://www.google.com/",
	"http://golang.org/",
	"http://blog.golang.org/",
}

// State represents the last-known state of a URL.
type State struct {
	url    string
	status string
}

// StateMonitor maintains a map that stores the state of the URLs being
// polled, and prints the current state every updateInterval nanoseconds.
// It returns a chan State to which resource state should be sent.
func StateMonitor(updateInterval time.Duration) chan<- State {
	updates := make(chan State)
	urlStatus := make(map[string]string)
	ticker := time.NewTicker(updateInterval)
	go func() {
		for {
			select {
			case <-ticker.C:
				logState(urlStatus)
			case s := <-updates:
				urlStatus[s.url] = s.status
			}
		}
	}()
	return updates
}

// logState prints a state map.
func logState(s map[string]string) {
	log.Println("Current state:")
	for k, v := range s {
		log.Printf(" %s %s", k, v)
	}
}

// Resource represents an HTTP URL to be polled by this program.
type Resource struct {
	url      string
	errCount int
}

// Poll executes an HTTP HEAD request for url
// and returns the HTTP status string or an error string.
func (r *Resource) Poll() string {
	resp, err := http.Head(r.url)
	if err != nil {
		log.Println("Error", r.url, err)
		r.errCount++
		return err.Error()
	}
	r.errCount = 0
	return resp.Status
}

// Sleep sleeps for an appropriate interval (dependent on error state)
// before sending the Resource to done.
func (r *Resource) Sleep(done chan<- *Resource) {
	time.Sleep(pollInterval + errTimeout*time.Duration(r.errCount))
	done <- r
}

func Poller(in <-chan *Resource, out chan<- *Resource, status chan<- State) {
	for r := range in {
		s := r.Poll()
		status <- State{r.url, s}
		out <- r
	}
}

func main() {
	// Create our input and output channels.
	pending, complete := make(chan *Resource), make(chan *Resource)

	// Launch the StateMonitor.
	status := StateMonitor(statusInterval)

	// Launch some Poller goroutines.
	for i := 0; i < numPollers; i++ {
		go Poller(pending, complete, status)
	}

	// Send some Resources to the pending queue.
	go func() {
		for _, url := range urls {
			pending <- &Resource{url: url}
		}
	}()

	for r := range complete {
		go r.Sleep(pending)
	}
}
```

