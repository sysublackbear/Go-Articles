# 译|Do not fear first class functions



## Functional options recap（函数式选项回顾）

一开始，我们先快速回顾函数式的选项的设计模式（这样的一个模板非常经典，值得收录）：

```go
type Config struct{ ... }

func WithReticulatedSplines(c *Config) { ... }

type Terrain struct {
        config Config
}

func NewTerrain(options ...func(*Config)) *Terrain {
        var t Terrain
        for _, option := range options {
                option(&t.config)
        }
        return &t 
}

func main() {
        t := NewTerrain(WithReticulatedSplines)
        // [ simulation intensifies ]
}
```

我们使用函数的方式(`WithXXX`)作为配置化的数据结构。我们把这些函数作为构造参数传给构造函数，作为构造对象的配置值。最后，我们在调用的地方`NewTerrain`传入我们想要设置的参数，进行对象构造。

OK，每个人对这种设计模式都非常熟悉了。当然我也认为这里面也会产生这样的一种疑惑，就是你需要把一个函数作为入参。例如，我们需要`WithCities`，给我们的`terrain`模型加入`cities`数量。

```go
// WithCities adds n cities to the Terrain model
func WithCities(n int) func(*Config) { ... }

func main() {        
        t := NewTerrain(WithCities(9))      
        // ...
}
```

因为`WithCities`需要传入一个参数，我们不能仅仅简单地传递`WithCities`给`NewTerrain`函数，它的签名不匹配（缺参数）。相反我们需要计算好`WithCities`，传入我们需要创造`cities`的数量。然后得出的值再传给`NewTerrain`。



## Functions as first class values

这是怎么一回事呢？让我们休息下。根本上来说，计算一个函数通常会返回一个值。我们有如下的函数，传入两个整型，返回一个整型。

```go
package math

func Min(a, b float64) float64
```

我们如下的函数，传入一个slice，然后返回一个数据结构的指针。

```go
package bytes

func NewReader(b []byte) *Reader
```

然后我们现在有一个函数去返回另外一个函数。

```go
func WithCities(n int) func(*Config)
```

直接说明了：函数也是一等公民。



## interface.Apply

解决这个问题的另外一种方法是通过使用接口重写这些参数选项：

```go
type Option interface {
        Apply(*Config)
}
```

取代我们之前的函数类型，我们定义出一个接口类型，命名为`Option`，然后给它包含一个方法，叫`Apply`，需要传入`*Config`参数。

```go
func NewTerrain(options ...Option) *Terrain {
        var config Config
        for _, option := range options {
                option.Apply(&config)
        }
        // ...
}
```

无论何时，当我们调用`NewTerrain`的时候，我们会传递一个或更多个实现了`Option`接口的值。在`NewTerrain`内部，我们循环地对每一个`Option`应用了`Apply`方法。

这个例子和上面的看起来区别不大。替代了之前遍历函数切片并轮流调用，我们遍历了接口切片并且轮流执行接口对应的方法。让我们看另外一个例子，声明`WithReticulatedSplines`选项。

```go
type splines struct{}

func (s *splines) Apply(c *Config) { ... }

func WithReticulatedSplines() Option {
        return new(splines)
}
```

因为我们通过接口的实现，我们需要声明能够实现`Apply`函数的类型。我们同样也需要声明一个构造函数返回我们的`splines`类型，你因此能看到更多的模式应用。

使用`Option`接口类型去重写我们的`WithCities`我们需要多做一点功夫，如下：

```go
type cities struct {
        cities int
}

func (c *cities) Apply(c *Config) { ... }

func WithCities(n int) Option {
        return &cities{
                cities: n,
        }
}
```

因为，我们的`main`函数逻辑调整成：

```go
func main() {
        t := NewTerrain(WithReticulatedSplines(), WithCities(9))
        // ...
}
```

在[GopherCon last year Tomás Senart](https://www.youtube.com/watch?v=xyDkyFjzFVc) 说过函数和一个带有方法的接口的二元性讨论。在我们的例子，你也看到这两者都可以作为一个变量。

但是你会发现，直接把函数作为一等公民会减少了很多代码。


### Encapsulating behaviour

让我们撇开接口，直接讨论函数作为一等公民的其它特性。

当我们调用一个函数或者方法的时候，我们本质做的就是传递数据。函数的功能就是解析某些数据和采取一些行动。事实上，传递一个函数值只是暂时声明了这段代码但是也许在另一个上下文延迟执行（类似闭包效果的函数延迟执行）。

为了说明这种情况，这里有一个简单的计算器例子：

```go
type Calculator struct {
        acc float64
}

const (
        OP_ADD = 1 << iota
        OP_SUB
        OP_MUL
)
```

它具有一个操作的集合。

```go
func (c *Calculator) Do(op int, v float64) float64 {
        switch op {
        case OP_ADD:
                c.acc += v
        case OP_SUB:
                c.acc -= v
        case OP_MUL:
                c.acc *= v
        default:
                panic("unhandled operation")
        }
        return c.acc
}
```

它具有一个`Do`方法，需要传入一个操作符`op`和操作数`v`。同时也返回计算出来的结果。

```go
func main() {
        var c Calculator
        fmt.Println(c.Do(OP_ADD, 100))     // 100
        fmt.Println(c.Do(OP_SUB, 50))      // 50
        fmt.Println(c.Do(OP_MUL, 2))       // 100
}
```

我们的计算器只知道怎么计算加法，减法和乘法。如果我们想要实现除法，我们还得专门分配一个除法的操作符和实现对应的操作逻辑。听起来非常合理，只需要添加几行代码，但是如果我们想要增加平方根运算和求幂运算呢？

每次我们添加操作，`Do`函数会膨胀得越来越大和难以维护。

让我们尝试重写下计算器。

```go
type Calculator struct {
        acc float64
}

type opfunc func(float64, float64) float64

func (c *Calculator) Do(op opfunc, v float64) float64 {
        c.acc = op(c.acc, v)
        return c.acc
}
```

我们看到不同点在于，`Do`方法支持传入一个`opfunc`类型的参数，来进行两个数的计算。

那么，要怎么使用这个新的计算器例子呢？

```go
func Add(a, b float64) float64 { return a + b }
func Sub(a, b float64) float64 { return a - b }
func Mul(a, b float64) float64 { return a * b }

func main() {
        var c Calculator
        fmt.Println(c.Do(Add, 5))       // 5
        fmt.Println(c.Do(Sub, 3))       // 2
        fmt.Println(c.Do(Mul, 8))       // 16
}
```

这样下来，代码不至于一直膨胀。



### Extending the calculator

现在，我们尝试扩展我们的计算器。

```go
func Sqrt(n, _ float64) float64 {
        return math.Sqrt(n)
}
```

但是这也会存在一个问题。`math.Sqrt`支持传入一个参数，而不是两个。但是，`opfunc`类型必须传入两个参数。

```go
func main() {
        var c Calculator
        c.Do(Add, 16)
        c.Do(Sqrt, 0) // operand ignored
}
```

我们当然可以第二个参数随便传一个进去来解决这个问题。（因为里面的逻辑不处理第二个参数）但是这样代码会有点糟糕，我认为我们能够做得更好。

让我们重新定义`Add`函数，做成类似偏函数的形式。

```go
func Add(n float64) func(float64) float64 {
        return func(acc float64) float64 {
                return acc + n
        }
}

func (c *Calculator) Do(op func(float64) float64) float64 {
        c.acc = op(c.acc)
        return c.acc
}
```

然后，我们的`main`方法改造成：

```go
func main() {
        var c Calculator
        c.Do(Add(10))   // 10
        c.Do(Add(20))   // 30
}
```

我们可以看到，另外一个强制参数已经放在`Add`里面。这样的话，相当于每个函数自己管理自身的参数，对于`Do`方法，它就只接收一个函数类型的参数`func(float64) float64`即可。

再看看减法和乘法：

```go
func Sub(n float64) func(float64) float64 {
        return func(acc float64) float64 {
                return acc - n
        }
}

func Mul(n float64) func(float64) float64 {
        return func(acc float64) float64 {
                return acc * n
        }
}
```

求平方根运算呢？

```go
func Sqrt() func(float64) float64 {
        return func(n float64) float64 {
                return math.Sqrt(n)
        }
}

func main() {
        var c Calculator
        c.Do(Add(2))
        c.Do(Sqrt())   // 1.41421356237
}
```

同时，作为一个装载器，它还支持传入多个包的不同方法，相当灵活。

```go
func main() {
        var c Calculator
        c.Do(Add(2))      // 2
        c.Do(math.Sqrt)   // 1.41421356237
        c.Do(math.Cos)    // 0.99969539804
}
```

一路下来，我们经历了最开始不灵活，光从逻辑层面去实现的挫计算器，到现在我们实现了非常灵活，易懂的计算器模型了。



## Let’s talk about actors

让我们改变一下轨迹，讨论下别的例子问题。

**经典的`actor`模式demo**。

假设我们在开发一个chat server，我们从最小粒度开始开发：

```go
type Mux struct {
        mu    sync.Mutex
        conns map[net.Addr]net.Conn
}

func (m *Mux) Add(conn net.Conn) {
        m.mu.Lock()
        defer m.mu.Unlock()
        m.conns[conn.RemoteAddr()] = conn
}
```

我们添加了一个注册新连接的方法`Add`。

```go
func (m *Mux) Remove(addr net.Addr) {
        m.mu.Lock()
        defer m.mu.Unlock()
        delete(m.conns, addr)
}
```

剔除旧连接的方法`Remove`。

```go
func (m *Mux) SendMsg(msg string) error {
        m.mu.Lock()
        defer m.mu.Unlock()
        for _, conn := range m.conns {
                err := io.WriteString(conn, msg)
                if err != nil {
                        return err
                }
        }
        return nil
}
```

但是，我们看到，这个是传统的并发模型的代码模型（依赖锁）（`actor`模式），各个请求会进行各个粒度的抢锁。我们能否使用go代码实现得更优雅呢？



### Don’t communicate by sharing memory, share memory by communicating.

这个其实算是老生常谈了。对于go语言来说，可以考虑用优雅的代码逻辑来实现。

**经典的csp模型demo**：

```go
type Mux struct {
        add     chan net.Conn
        remove  chan net.Addr
        sendMsg chan string
}

func (m *Mux) Add(conn net.Conn) {
        m.add <- conn
}

func (m *Mux) Remove(addr net.Addr) {
        m.remove <- addr
}

func (m *Mux) SendMsg(msg string) error {
        m.sendMsg <- msg
        return nil
}
```

然后服务端需要建立一个`loop`函数异步运行。

```go
func (m *Mux) loop() {
        conns := make(map[net.Addr]net.Conn)
        for {
                select {
                case conn := <-m.add:
                        m.conns[conn.RemoteAddr()] = conn
                case addr := <-m.remove:
                        delete(m.conns, addr)
                case msg := <-m.sendMsg:
                        for _, conn := range m.conns {
                                io.WriteString(conn, msg)
                        }
                }
        }
}
```

总的来说，代码改造主要分了三个步骤：

+ 创造一个管道；
+ 添加一个`helper`函数往管道塞入数据；
+ 在`loop`函数的`select`分片里面添加接收数据的处理逻辑。

类似我们上面的计算器例子，我们可以重写我们的`Mux`类，把函数作为一等公民作为我们想要执行的逻辑，进行传递，而不仅仅是传递数据。现在，所有的操作都放到`ops`管道中，函数作为参数进行传递。

```go
type Mux struct {
        ops chan func(map[net.Addr]net.Conn)  // 函数作为类型的channel
}

func (m *Mux) Add(conn net.Conn) {
        m.ops <- func(m map[net.Addr]net.Conn) {
                m[conn.RemoteAddr()] = conn  // 加入conn
        }
}
```

同理，`Remove`和`SendMsg`改写如下：

```go
func (m *Mux) Remove(addr net.Addr) {
        m.ops <- func(m map[net.Addr]net.Conn) {
                delete(m, addr)
        }
}

func (m *Mux) SendMsg(msg string) error {
        m.ops <- func(m map[net.Addr]net.Conn) {
                for _, conn := range m {
                        io.WriteString(conn, msg)
                }
        }
        return nil
}
```

然后我们的`loop`函数改写成：

```go
func (m *Mux) loop() { 
        conns := make(map[net.Addr]net.Conn)
        for op := range m.ops {
                op(conns)
        }
}
```

你看，现在我们的`loop`的功能就变得非常简洁了，它创造了一个map，然后直接`for ... range`管道来逐个处理逻辑。

然而，这还有一些问题需要修复。最紧迫的就是在`SendMsg`缺乏异常处理；在往连接buffer写数据的时候发生的错误并没有传递给调用方。因此我们作出下面的修改：

```go
func (m *Mux) SendMsg(msg string) error {
        result := make(chan error, 1)
        m.ops <- func(m map[net.Addr]net.Conn) {
                for _, conn := range m.conns {
                        err := io.WriteString(conn, msg)
                        if err != nil {
                                result <- err
                                return
                        }
                }
                result <- nil
        }
  			// 这里有点异步转同步内味了，值得学习。
        return <-result
}
```

`SendMsg`是个异步逻辑的函数，为了能够实时拿到发送的接口，我们自己加了一个`result`的逻辑，将`SendMsg`过程中产生的错误通过`result`管道进行同步。这个异步转同步的函数逻辑改造确实值得学习。

```go
func (m *Mux) loop() {
        conns := make(map[net.Addr]net.Conn)
        for op := range m.ops {
                op(conns)
        }
}
```

到目前为止，我们还没有改变`loop`的主体逻辑去合并异常处理。我们可以很方便地添加一个`PrivateMsg`方法来实现单发客户端。

```gi
func (m *Mux) PrivateMsg(addr net.Addr, msg string) error {
        result := make(chan net.Conn, 1)
        m.ops <- func(m map[net.Addr]net.Conn) {
                result <- m[addr]
        }
        conn := <-result
        if conn == nil {
                return errors.Errorf("client %v not registered", addr)
        }
        return io.WriteString(conn, msg)
}
```

我们可以达到不去改变`loop`函数的主题逻辑，去实现了异步转同步的函数逻辑转换。



## Conclusion

总结：

+ 函数作为一等公民你会带给你巨大的表达能力。他们能够让你去传递你将要做什么，而不是死沉沉的数据；
+ 函数作为一等公民并不是什么新兴事物东西。很多很老的语言，比如C，都已经支持了这些特性了。
+ Go提供的函数作为一等公民，应该尽可能地克制使用。我们不要过度使用channel来实现一些过于复杂，不通读的逻辑。要合理使用。
+ 函数作为一等公民应该是每个Go程序员的工具箱里面的一个技能。函数作为一等公民并不是go独有的，go程序员不应该惧怕它。
+ 如果你尝试学习使用接口去解决，那你应该转而使用函数作为一等公民的方式。后者不是更难，只是有点点区别。这点区别我们认为随着时间和努力，始终能够克服。



因此下一次你在定义一个只有一个方法的API（接口）的时候，问下你自己，为啥它不能直接设计成一个函数？



