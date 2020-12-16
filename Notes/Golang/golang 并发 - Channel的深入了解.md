# 1.Channels是什么？

这里又可以搬出知乎名言了，在认识一件事物之前，先问问是什么，再回答为什么！直接来说，一个Channel 是一个通信机制，它**可以让一个Goroutine 通过它给另一个Goroutine 发送值信息**。每个Channel 都有一个特殊的类型，也就是Channels可发送数据的类型（例如：一个可以发送int类型数据的Channel 一般写为chan int）。

在我们常见的一些语言中，多个线程传递数据的方式一般都是共享内存，<u>为了解决线程冲突的问题，我们需要限制同一时间能够读写这些变量的线程数量</u>。而Golang 语言提供了一种不同与使用使用共享内存加互斥锁也能进行通信的并发模型，也就是**通信顺序进程（Communicating sequential processes，CSP）**。其中Goroutine 和 Channel 分别对应 CSP 中的实体和传递信息的媒介，Go 语言中的 Goroutine 会通过 Channel 传递数据。这也是Golang一直提倡的**不要通过共享内存的方式进行通信，而是应该通过通信的方式共享内存**。

目前的 Channel 收发操作均遵循了先入先出（FIFO）的设计，而且**带缓存区和不带缓存区的 Channel 都会遵循先入先出对数据进行接收和发送**（关于带缓存区与不带缓存区在下面会提及），具体规则如下：

- 先从 Channel 读取数据的 Goroutine 会先接收到数据； 
- 先向 Channel 发送数据的 Goroutine 会得到先发送数据的权利；

通过源码查看我们可知，Channel 在运行时使用 [runtime.hchan](https://github.com/golang/go/blob/e35876ec6591768edace6c6f3b12646899fd1b11/src/runtime/chan.go#L32) 结构体进行表示，而这玩意最后包含这一个互斥锁用于保护成员变量，所以从某种程度上说，Channel 是一个<u>用于同步和通信的有锁队列</u>。具体数据结构如下所示：

```go
type hchan struct {
    qcount   uint           // total data in the queue
    dataqsiz uint           // size of the circular queue
    buf      unsafe.Pointer // points to an array of dataqsiz elements
    elemsize uint16
    closed   uint32
    elemtype *_type // element type
    sendx    uint   // send index
    recvx    uint   // receive index
    recvq    waitq  // list of recv waiters
    sendq    waitq  // list of send waiters

    // lock protects all fields in hchan, as well as several
    // fields in sudogs blocked on this channel.
    //
    // Do not change another G's status while holding this lock
    // (in particular, do not ready a G), as this can deadlock
    // with stack shrinking.
    lock mutex
}
```

上面提到了CSP，这里进行一些补充：CSP 是 **Communicating Sequential Process** 的简称，中文可以叫做**通信顺序进程**，是一种并发编程模型，由 [Tony Hoare](https://en.wikipedia.org/wiki/Tony_Hoare) 于 1977 年提出。简单来说，**CSP 模型由并发执行的实体（线程或者进程）所组成，实体之间通过发送消息进行通信**，这里发送消息时使用的就是通道，或者叫 channel。CSP 模型的关键是关注 channel，而不关注发送消息的实体。Go 语言实现了 CSP 部分理论，goroutine 对应 CSP 中并发执行的实体，channel 也就对应着 CSP 中的 channel。



---

# 2.如何实现Channels？

## 2.1 基本实现及操作

详细来说，Channel 类型的格式如下：`var 变量 chan 元素类型`，具体举例如下：

```go
var ch1 chan int   // 声明一个传递整型的通道
var ch2 chan bool  // 声明一个传递布尔型的通道
var ch3 chan []int // 声明一个传递int切片的通道
```
如果仅仅进行Channel 创建：`var ch chan int`，而没有进行使用make函数初始化，打印输出会发现为nil：`fmt.Println(ch) // <nil>`，而且！**在一个`nil`的channel上发送和接收操作会被永久阻塞**，可以看看下面的代码结果会怎样？

```go
package main
import (  
    "fmt"
    "time"
)
func main() {  
    // 没有进行make函数初始化，该ch为nil
    var ch chan int
    for i := 0; i < 3; i++ {
        go func(idx int) {
            ch <- (idx + 1) * 2
        }(i)
    }
    // 得到第一个结果
    fmt.Println("result:",<-ch)
    // 休眠
    time.Sleep(2 * time.Second)
}
```

面这段代码能够通过编译，但是执行的时候会发现runtime错误：

```go
fatal error: all goroutines are asleep - deadlock!
```

我们可以通过使用内置的make函数创建一个Channel ：`ch := make(chan int)`，创建Channel 的格式：`make(chan 元素类型, [缓存大小])`，具体举例如下：

```go
ch4 := make(chan int)
ch5 := make(chan bool)
ch6 := make(chan []int)
```

很简单就可以实现一个Channel，需要注意的是经过上面的make创建了之后的Channel对应的<u>是make底层数据结构的一个引用</u>，意思就是当复制一个Channel 或者 将Channel 用于参数传递的时候，仅仅只是复制了一个Channel 的引用而已！此时调用者何被调用者将引用同一个Channel 对象。

既然是对象，那么Channel 也是可以使用`==`运算符进行比较了，但是注意的是<u>相同类型才可以进行比较</u>，简单验证如下：

```go
package main

import "fmt"

func main() {

    ch1 := make(chan int)
    ch2 := make(chan int)
    //ch3 := make(chan string)

    fmt.Println(ch1 == ch2)	// 可以比较，但是比较结果为false
    //fmt.Println(ch1 == ch3) // 不可以比较，错误为：invalid operation: ch1 == ch3 (mismatched types chan int and chan string)
}
```

一个Channel 有发送、接收及关闭等简单操作，如果浏览源码，时不时的会发现这么一个符号：`<-`，简直不要太有代表性，这玩意就是Channel 的使用！而且发送和接收两个操作都是用`<-`运算符。具体如下：

<u>在发送语句中，`<-`运算符分割channel和要发送的值</u>。比如：`ch <- x `，我们将x发送到通道ch之中。

<u>在接收语句中，`<-`运算符写在channel对象之前</u>。比如：`x = <-ch` 

- 当然一个不使用接收结果的接收操作也是合法的。比如：`<-ch`

<u>当然我们可以关闭一个通道，使用close就可以了</u>：`close(ch)`

- 需要注意的是，我们要对close的使用进行比较好的处理，如果还有数据没发送完或者没有数据没有接收完，我们直接close，是不是不是很妥？而且Golang并没有办法直接测试一个Channels是否被关闭，但是接收操作有一个**变体形式**：它多接收一个结果，多接收的第二个结果是一个布尔值ok，<u>ture表示成功从channels接收到值，false表示channels已经被关闭并且里面没有值可接收</u>。

-  示例如下：

  - ```go
    ch := make(chan int, 10)
    // ok-idiom 
    val, ok := <-ch
    if ok == false {
        // channel closed
        close(ch)
    }
    ```

- 这里简单回答两个关于**Channels关闭后接收发送会怎么样？**的问题

  - Q：**Channels被关闭还可以读吗？**
    - A：对一个已经被close过的Channels进行接收操作依然可以接受到之前已经成功发送的数据；如果Channels中已经没有数据的话将产生一个零值的数据。
  - Q：**向一个已经关闭了的Channels写入数据会怎样？**
    - A：随后对基于该channel的任何发送操作都将导致panic异常

- 简单来说：

  - 重复关闭 channel 会导致 panic。
  - 向关闭的 channel 发送数据会 panic。
  - 从关闭的 channel 读数据不会 panic，读出 channel 中已有的数据之后再读就是 channel 类似的默认值，比如 chan int 类型的 channel 关闭之后读取到的值为 0。

- 注：其实close最后会调用方法[runtime.closechan](https://github.com/golang/go/blob/e35876ec6591768edace6c6f3b12646899fd1b11/src/runtime/chan.go#L335)，有兴趣的可以了解一下源码。

这里给出对已经关闭了的Channels 再次进行数据发送的案例，看是否会引起Panic！

```go
package main

import (
    "fmt"
    "time"
)
func main() {
    ch := make(chan int)
    for i := 0; i < 3; i++ {
        go func(idx int) {
            ch <- (idx + 1) * 2
        }(i)
    }

    // 得到第一个元素值
    fmt.Println(<-ch)
    close(ch) // 还有一些元素值未发送
    // 进行其他的工作
    time.Sleep(2 * time.Second)
}
```

结果如下：

![向已经关闭的Channel继续发送数据](https://garmen-imgsubmit.oss-cn-shenzhen.aliyuncs.com/img/20200822142805.png)

这里给出一个简单的解决方案，我们需要确保应用不会向关闭的channel中发送数据：通过使用一个特殊的废弃的channel来向剩余的worker发送不再需要它们的结果的信号来修复。

```go
package main
import (
    "fmt"
    "time"
)
func main() {
    ch := make(chan int)
    done := make(chan struct{})
    for i := 0; i < 3; i++ {
        go func(idx int) {
            select {
                case ch <- (idx + 1) * 2:
                fmt.Println(idx,"sent result")
                case <- done:
                fmt.Println(idx,"exiting")
            }
        }(i)
    }
    // 取得通道Channels的首个结果
    fmt.Println("result:",<-ch)
    // 关闭特殊Channel通道done
    close(done)
    // 等待三秒
    time.Sleep(3 * time.Second)
}
```

于是我们便掌握了简单的发送、接受及关闭，难道Channel 就这？这么简单就都搞定了？其实不然，还挺复杂，继续往下。

## 2.2 不带缓存的Channels

无缓存的通道又称为**阻塞的通道**，通常我们使用不带缓存的Channels来做同步操作，所以无缓存Channels有时候也被称为**同步Channels。**

其中同步Channel 不需要携带额外的信息，它仅仅是用作两个goroutine之间的同步，我们还可以使用`struct{}`空结构体作为channels元素的类型（这里也是为什么之前看到很多`struct{}`懵逼的原因了），虽然也可以使用bool或int类型实现同样的功能，`done <- 1`语句也比`done <- struct{}{}`更短。

具体如何进行阻塞呢？

- 一个基于无缓存Channels的发送操作将导致发送者goroutine阻塞，直到另一个goroutine在相同的Channels上执行接收操作，当发送的值通过Channels成功传输之后，两个goroutine可以继续执行后面的语句。
- 如果接收操作先发生，那么接收者goroutine也将阻塞，直到有另一个goroutine在相同的Channels上执行发送操作。

这里的同步操作则引出了一个Golang并发内存模型的关键术语：Happens Before！具体如何体现？

比如**无缓存的Channel上的发送操作总在对应的接收操作完成前 发生。**

具体案例如下：

```go
package main

var done = make(chan bool)
var msg string

func send() {
    // 对msg进行写入
    msg = "Hello World！"
    // 发送同步信号
    done <- true
}

func main() {
    go send()
    // 接收同步信号
    <-done
    println(msg)
}
```

怎么体现？体现在：<u>可保证打印出"Hello World！"</u>，原因记载上面了！结合例子思考一下。

具体流程为：该程序首先对`msg`进行写入，然后在`done`管道上发送同步信号，随后从`done`接收对应的同步信号，最后执行`println`函数。

还有**对于从无缓存Channel进行的接收，发生在对该Channel进行的发送<u>完成</u> 之前。**

具体案例如下：

```go
package main

var done = make(chan bool)
var msg string

func recv() {
    // 对msg进行写入
    msg = "Hello World！"
    // 接收同步信号
    <-done
}

func main() {
    go recv()
    // 发送同步信号
    done <- true
    println(msg)
}
```

怎么体现？体现在：<u>也可保证打印出"Hello World！"</u>。因为`main`线程中`done <- true`发送完成前，后台线程`<-done`接收已经开始，这保证`msg = "Hello World！`被执行了，所以之后`println(msg)`的msg已经被赋值过了。具体流程为：后台线程首先对`msg`进行写入，然后从`done`中接收信号，随后`main`线程向`done`发送对应的信号，最后执行`println`函数完成。

对于带缓冲的Channel也自然有Happens Before：**对于Channel的第`K`个接收完成操作发生在第`K+C`个发送操作完成之前，其中`C`是Channel的缓存大小。** 如果将`C`设置为0自然就对应无缓存的Channel，也即使第K个接收完成在第K个发送完成之前。因为无缓存的Channel只能同步发1个，也就简化为前面无缓存Channel的规则：**对于从无缓冲Channel进行的接收，发生在对该Channel进行的发送完成之前。**

如果用反向思维来辩证一下的话，要是没有这条Happens Before的话，是不是`done <- true`发送了之后就瞬间直接打印"Hello World！了，而运行的结果并不是，我们在接收同步信号之前进行一个睡眠延时，发现打印"Hello World！也需要等待这个睡眠延时的时间，原因就是对无缓存Channel的接收发生在对该Chennel进行的发送完成之前：

```go
package main

import "time"

var done = make(chan bool)
var msg string

func recv() {
    // 对msg进行写入
    msg = "Hello World！"
    // 等待三秒
    time.Sleep(3 * time.Second)
    // 接收同步信号
    <-done
}

func main() {
    go recv()
    // 发送同步信号
    done <- true
    println(msg)
}
```

这里看到了一幅很具体的图展示了什么叫无缓存的Channel：

![不带缓存的Channels](https://garmen-imgsubmit.oss-cn-shenzhen.aliyuncs.com/img/20200822000155.png)



需要注意的是<u>无缓存的Channel 必须再有人接收值的时候才能发送值，否则会产生deallock</u>，代码如下：

```go
package main

import "fmt"

func main() {
    ch := make(chan int)
    // 只有发送，并没有接收！
    ch <- 10
    fmt.Println("发送成功")
}
```

运行之后发生错误：

```go
fatal error: all goroutines are asleep - deadlock!
```

上面的代码会阻塞在ch <- 10这一行代码形成死锁，那如何解决这个问题呢？

可以启用一个goroutine去接收值，例如：

```go
package main

import "fmt"

func recv(c chan int) {
    ret := <-c
    fmt.Println("接收成功", ret)
}

func main() {
    ch := make(chan int)
    go recv(ch) // 启用goroutine从通道接收值
    ch <- 10
    fmt.Println("发送成功")
}
```



还有一个需要注意的地方，也是我一直在怀疑的地方，<u>如果我同时向通道进行数据发送和数据接收，会不会速度太快导致接收方不够时间在发送者继续执行发送前处理数据</u>？进行验证一下：

```go
package main

import "fmt"

func main() {
    ch := make(chan string)
    go func() {
        for m := range ch {
            fmt.Println("processed:",m)
        }
    }()
    ch <- "First"
    ch <- "Second" // 这段有可能不被处理
}
```

进行了多次的测试之后发现有时候"Second"会被处理到，有时候并不会！所以对这里数据处理要注意一下，主要原因是<u>上述代码处理过程中发送方不会被阻塞，除非发送的消息正在被接收方处理这样才会进行阻塞</u>，所以我们的同步操作不可像上面那样编写！



## 2.3 带缓存的Channels

我们可以在使用make函数初始化通道的时候为其指定通道的容量，带缓存的Channel内部持有一个元素队列。队列的最大容量是在调用make函数创建channel时通过第二个参数指定的。例如：

```go
package main

import (
    "fmt"
)

func main() {
    ch = make(chan string, 3) // 创建一个容量为3的有缓冲区通道
    ch <- "A"
    fmt.Println("发送成功")
}
```

向缓存Channel的发送操作就是向内部缓存队列的尾部插入元素，接收操作则是从队列的头部删除元素。

而带缓存的Channels 与阻塞的联系则如下：

- 如果内部缓存队列是满的，那么发送操作将阻塞直到因另一个goroutine执行接收操作而释放了新的队列空间。
- 相反，如果channel是空的，接收操作将阻塞直到有另一个goroutine执行发送操作而向队列插入元素。
- 那么channel的缓存队列将不是满的也不是空的，因此对该channel执行的发送或接收操作都不会发送阻塞。通过这种方式，channel的缓存队列解耦了接收和发送的goroutine。

细心的你想必发现了，为啥上面的代码不像前面的不带缓存的Channel一样报错deadlock？这里联系上下文讲一下我个人的理解，在不带缓存的Channels中就好比快递员送快递，而你的小区没有快递柜和菜鸟驿站啥的，那么他就必须要把这个快递直接送到你手中了，但是根本都没有这个收货人则怎么发出去货物呢？所以则报错了！而在带缓存的Channels中，你有很多个快递柜，你可以将货物暂存在这里，别人拿不拿是别人的事，反正有目的地即可（如果理解错误请纠正我）。

在某些特殊情况下，程序可能需要知道channel内部缓存的容量，可以用内置的cap函数获取：

```go
package main

import (
    "fmt"
)

func main() {
    ch = make(chan string, 3)

    ch <- "A"
    ch <- "B"
    ch <- "C"

    fmt.Println(cap(ch)) // "3"
}
```

同样，对于内置的len函数，如果传入的是channel，那么将返回channel内部缓存队列中有效元素的个数。

```go
fmt.Println(len(ch)) // "3"
```

还有一点需要注意的是，如果Channel为带缓存的，即使我们将其缓存设置为1，也不会进行像前面无缓存那样的阻塞，`main`线程的`done <- true`接收操作将不会被后台线程的`<-done`接收操作阻塞，该程序将无法保证打印出"Hello World！"。如下：

```go
package main

import "time"

// 将其变为有缓存的Chan
var done = make(chan bool, 1)
var msg string

func recv() {
    // 对msg进行写入
    msg = "Hello World！"
    // 等待三秒
    time.Sleep(3 * time.Second)
    // 接收同步信号
    <-done
}

func main() {
    go recv()
    // 发送同步信号
    done <- true
    println(msg)
}
```

你会发现，这个 "Hello World！"永远也打印不出来！

同样参考如下图所示：

![带缓存的Channels](https://garmen-imgsubmit.oss-cn-shenzhen.aliyuncs.com/img/20200822000228.png)





## 2.4 单向的Channels

单向 Channel，既只能读或者只能进行写的Channel。对比普通的Channel：`chan int`，其详细类型如下：

- <u>`chan<- int`表示一个只发送int的channel</u>，只能发送不能接收。
- <u>`<-chan int`表示一个只接收int的channel</u>，只能接收不能发送。

> 注意：箭头`<-`和关键字chan的相对位置表明了channel的方向。这种限制将在编译期检测。

**任何双向channel向单向channel变量的赋值操作都将导致该channel进行隐式转换**。这里并没有反向转换的语法：也就是不能一个将类似`chan<- int`类型的单向型的channel转换为`chan int`类型的双向型的channel。案例如下：

```go
// counter ：进行计数，传入一个只读的chan
func counter(out chan<- int) {
    for x := 0; x < 100; x++ {
        out <- x
    }
    close(out)
}

// squarer ：进行平方计算,out是只写chan，in是只读chan
func squarer(out chan<- int, in <-chan int) {
    for v := range in {
        out <- v * v
    }
    close(out)
}

// printer ：进行打印输出，in是只读chan
func printer(in <-chan int) {
    for v := range in {
        fmt.Println(v)
    }
}

func main() {
    naturals := make(chan int)
    squares := make(chan int)
    go counter(naturals)
    go squarer(squares, naturals)
    printer(squares)
}
```

调用counter(naturals)将导致将`chan int`类型的naturals隐式地转换为`chan<- int`类型只发送型的channel。调用printer(squares)也会导致相似的隐式转换，这一次是转换为`<-chan int`类型只接收型的channel。

总的来说，单向 channel 主要用在<u>函数声明</u>中，有的时候我们会将通道作为参数在多个任务函数间传递，很多时候我们在不同的任务函数中使用通道都会对其进行限制，比如限制通道在函数中只能发送或只能接收。比如：

```go
func foo(ch chan<- int) <-chan int {...}
```

<u>foo 的形参是一个只能写的 channel</u>，那么就表示函数 foo 只会对 ch 进行写，当然你传入的参数可以是个普通 channel。<u>foo 的返回值是一个只能读的 channel</u>，那么表示 foo 的返回值规范用法就是只能读取。

使用单向 channel 编程体现了一种非常优秀的编程范式：**convention over configuration**，中文一般叫做 **约定优于配置**。

还有需要注意的一点是因为关闭操作只用于断言不再向channel发送新的数据，所以只有在发送者所在的goroutine才会调用close函数，**因此对一个只接收的channel调用close将是一个编译错误**。



---

# 3.Channels有缓存和无缓存的区别

简单来说：

- 无缓存的管道，**只要没有协程写入就读出阻塞，没有协程读出，就造成写入阻塞**；
- 有缓存的管道，即使没人写入，也能读出若干默认值，即使没人读出，也能写入若干值；

**关于无缓存或带缓存channels之间的选择**

- **关于无缓存或带缓存channels之间的选择，或者是带缓存channels的容量大小的选择，都可能影响程序的正确性**。
  - 无缓存channel更强地保证了每个发送操作与相应的同步接收操作；
  - 但是对于带缓存channel，这些操作是解耦的。
- 即使我们知道将要发送到一个channel的信息的数量上限，创建一个对应容量大小的带缓存channel也是不现实的，因为<u>这要求在执行任何接收操作之前缓存所有已经发送的值</u>。如果未能分配足够的缓存将导致程序死锁。
- **Channel的缓存也可能影响程序的性能**。想象一家蛋糕店有三个厨师，一个烘焙，一个上糖衣，还有一个将每个蛋糕传递到它下一个厨师的生产线。在狭小的厨房空间环境，每个厨师在完成蛋糕后必须等待下一个厨师已经准备好接受它；这类似于在一个无缓存的channel上进行沟通。
- 如果在每个厨师之间有一个放置一个蛋糕的额外空间，那么每个厨师就可以将一个完成的蛋糕临时放在那里而马上进入下一个蛋糕的制作中；这类似于将channel的缓存队列的容量设置为1。只要每个厨师的平均工作效率相近，那么其中大部分的传输工作将是迅速的，个体之间细小的效率差异将在交接过程中弥补。如果厨师之间有更大的额外空间——也是就更大容量的缓存队列——将可以在不停止生产线的前提下消除更大的效率波动，例如一个厨师可以短暂地休息，然后再加快赶上进度而不影响其他人。
- 另一方面，如果生产线的前期阶段一直快于后续阶段，那么它们之间的缓存在大部分时间都将是满的。<u>相反，如果后续阶段比前期阶段更快，那么它们之间的缓存在大部分时间都将是空的。对于这类场景，额外的缓存并没有带来任何好处</u>。
- 生产线的隐喻对于理解channels和goroutines的工作机制是很有帮助的。例如，如果第二阶段是需要精心制作的复杂操作，一个厨师可能无法跟上第一个厨师的进度，或者是无法满足第三阶段厨师的需求。要解决这个问题，我们可以再雇佣另一个厨师来帮助完成第二阶段的工作，他执行相同的任务但是独立工作。这类似于基于相同的channels创建另一个独立的goroutine。



---

# 他山之石

- [Go语言中文文档-Channel](http://topgoer.com/%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B/channel.html)
- [深入理解 Go Channel](http://legendtkl.com/2017/07/30/understanding-golang-channel/)
- [第八章 Goroutines和Channels](https://www.kancloud.cn/lbb4511/gopl/1107899)
- [Go的50度灰：Golang新开发者要注意的陷阱和常见错误](https://colobu.com/2015/09/07/gotchas-and-common-mistakes-in-go-golang/#%E5%90%91%E6%97%A0%E7%BC%93%E5%AD%98%E7%9A%84Channel%E5%8F%91%E9%80%81%E6%B6%88%E6%81%AF%EF%BC%8C%E5%8F%AA%E8%A6%81%E7%9B%AE%E6%A0%87%E6%8E%A5%E6%94%B6%E8%80%85%E5%87%86%E5%A4%87%E5%A5%BD%E5%B0%B1%E4%BC%9A%E7%AB%8B%E5%8D%B3%E8%BF%94%E5%9B%9E)
- [面向并发的内存模型](https://chai2010.cn/advanced-go-programming-book/ch1-basic/ch1-05-mem.html)





