# 1.使用无缓存Channel进行goroutine通信

在前面的关于Channel的一些认识当中，我们了解基于无缓存Channels的发送和接收操作将导致两个goroutine做一次同步操作，故无缓存Channels有时候也被称为同步Channels，那么我们就可以使用无缓存的Channel进行简单的goroutine通信了，代码如下：

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    // make一个无缓存channel
    ch := make(chan int)

    go func() {
        fmt.Println("三秒之后开始启动！")
        // 等待三秒
        time.Sleep(3 * time.Second)
        ch <- 1 // 信号发送
    }()
    <- ch // 信号接收
    close(ch) // 关闭通道
    fmt.Println("收到通知！")
}
```

当然对于如何使用`stuct{}`空结构体进行同步Channels的操作我是一直耿耿于怀，将上面的代码改一改，加深一下认识：

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    // 使用strcu{}创建一个无缓存channel
    ch := make(chan struct{})
    go func() {
        fmt.Println("三秒之后开始启动！")
        // 等待三秒
        time.Sleep(3 * time.Second)
        ch <- struct{}{} // 发送信号
    }()
    <- ch // 接收信号
    close(ch)
    fmt.Println("收到通知！")
}
```



---

# 2.结合Select多路复用

一言以蔽之：**Golang 中的 select 提供了对多个 channel 的统一管理**。这个就是最简洁的答案，如果我们想要在程序中对多个Channel的管理，我们可以选择使用select，select语句的一般形式的代码如下：

```go
select {
    case <-ch1:
    // 如果chan1成功读到数据，则进行该case处理语句
    case x := <-ch2:
    // 如果chan2成功读到数据并将该数据赋值给x，则进行该case处理语句
    case ch3 <- y:
    // 如果成功向chan2写入数据，则进行该case处理语句
    default:
    // ...
}
```

select和switch语句稍微有点相似，有case也有最后的default默认分支来设置当其它的操作都不能够马上被处理时程序需要执行哪些逻辑。每一个case代表一个通信操作(在某个channel上进行发送或者接收)并且会包含一些语句组成的一个语句块。

<u>select会等待case中有能够执行的case时去执行</u>。当条件满足时，select才会去通信并执行case之后的语句；这时候其它通信是不会执行的。<u>一个没有任何case的select语句写作select{}，会永远地等待下去</u>。

## 2.1 监听一个或者多个Channel

关于select结合channel还是有几种情况出现的，比如我们的select可以**<u>监听一个或者多个channel</u>**，只要有一个channel做好准备进行数据发送，则select则会马上进行处理，代码如下：

```go
package main

import (
	"fmt"
	"time"
)

func test1(ch chan string) {
	time.Sleep(time.Second * 3)
	ch <- "test1"
}
func test2(ch chan string) {
	time.Sleep(time.Second * 2)
	ch <- "test2"
}

func main() {
	// 2个管道
	chan1 := make(chan string)
	chan2 := make(chan string)
	// 跑2个子协程，写数据
	go test1(output1)
	go test2(output2)

	// 用select监控
	select {
	case s1 := <-chan1:
		fmt.Println("chan1:", s1)
	case s2 := <-chan2:
		fmt.Println("chan2:", s2)
	}
}
```

## 2.2 多个Case随机处理

那肯定有些情况我们不知道那么些个channel哪个先准备好，哪个后准备好，在这种个case同时就绪时的情况下，select会<u>**随机处理**</u>，即随机地选择一个执行，这样来保证每一个channel都有平等的被select的机会。比如下面的代码中，有时候会打印 `ch1:1`，有时候则打印 `ch2:"hello"`，这是一个随机处理的过程！代码如下：

```go
package main

import (
    "fmt"
)

func main() {
    // 创建2个管道
    ch1 := make(chan int, 1)
    ch2 := make(chan string, 1)
    go func() {
        ch1 <- 1
    }()
    go func() {
        ch2 <- "hello"
    }()
    select {
        case value := <-ch1:
        fmt.Println("ch1:", value)
        case value := <-ch2:
        fmt.Println("ch2:", value)
    }
}
```

## 2.3 充分利用Default

我们还可以<u>**利用default**</u>这个巧妙的设定来进行一些判断，比如判断channel是否已经写满，下面的代码会在ch channel中有值时，从其中接收值；无值时什么都不做。这是一个非阻塞的接收操作；反复地做这样的操作叫做“轮询channel”。代码如下：

```go
package main

import (
    "fmt"
    "time"
)

// 判断管道有没有存满
func main() {
    // 创建容量为10的chan
    ch := make(chan string, 10)
    // 开启goroutine写数据
    go write(ch)
    // 消耗数据
    for s := range ch {
        fmt.Println("res:", s)
        time.Sleep(time.Second)
    }
}

func write(ch chan string) {
    for {
        select {
            // 写数据
            case ch <- "hello":
            fmt.Println("write hello")
            // 如果ch通道已满，数据则写不进了，于是走default
            default:
            fmt.Println("channel full")
        }
        time.Sleep(time.Millisecond * 500)
    }
}
```

## 2.4 超时控制

还有其他很多很简单的一些实现，比如可以使用`select+channel`做超时控制，在很多操作情况下都需要超时控制，利用 select 实现超时控制，代码如下：

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    ch := make(chan string)

    go func() {
        // 休眠两秒
        time.Sleep(time.Second * 2)
        // 往通道ch写入数据
        ch <- "result"
    }()

    select {
        case res := <-ch:
        fmt.Println(res)
        case <-time.After(time.Second * 1): 
        // 休眠时间设定为1秒，超过1秒没有得到ch数据则判定为超时
        fmt.Println("timeout")
    }
}
```

上面的代码中，`select`语句会阻塞等待最先返回数据的`channel`，当先接收到`time.After`的通道数据时，`select`则会停止阻塞并执行该`case`的代码。此时就已经实现了对业务代码的超时处理。



---

# 3.并发协程的安全退出

有时候我们需要通知goroutine停止它正在干的事情，比如一个正在执行计算的web服务，然而它的客户端已经断开了和服务端的连接。Golang 没有提供这么一个goroutine中终止另一个goroutine的方法，为啥不直接提供一个goroutine直接终止另外一个goroutine的方法呢？因为这样会**导致goroutine之间的共享变量落在未定义的状态上**。

通常我们可以使用`select`和`default`分支可以很容易实现一个Goroutine的退出控制：

```go
package main

import (
    "fmt"
    "time"
)

func worker(cancel chan bool) {
    for {
        select {
            case <-cancel:
            return
            // 退出
            default:
            fmt.Println("hello")
            // 正常工作
        }
    }
}

func main() {
    cancel := make(chan bool)
    go worker(cancel)

    time.Sleep(3 * time.Second)
    cancel <- true
}
```

假如我们不是使用`cancel <- true`这种传值的方式，而是想通过`close()`方法关闭一个通道进而关闭相对于的协程呢？这就要使用到`ok-idiom`了！

**关于使用`ok-idiom`我们可以结合select进行**，往往我们需要结合`for-select`这种结构来实现，因为select提供了多路复用的能力，所以<u>`for-select`可以让函数具有持续多路处理多个Channel的能力</u>。

需要注意的是在使用ok-idiom过程中进行退出的时候，**select没有感知channel的关闭**，这引出了2个问题：

1. 继续在关闭的通道上读，会读到通道传输数据类型的零值，如果是指针类型，读到nil，继续处理还会产生nil。
2. 继续在关闭的通道上写，将会panic。

问题2可以这样解决，<u>通道只由发送方关闭，接收方不可关闭</u>，即某个写通道只由使用该select的协程关闭，select中就不存在继续在关闭的通道上写数据的问题。关于这点，官方`close()`的注释中也明确讲了：

```go
// The close built-in function closes a channel, which must be either
// bidirectional or send-only. It should be executed only by the sender,
// never the receiver, and has the effect of shutting down the channel after
// the last sent value is received. After the last value has been received
// from a closed channel c, any receive from c will succeed without
// blocking, returning the zero value for the channel element. The form
// x, ok := <-c
// will also set ok to false for a closed channel.
func close(c chan<- Type)
```

问题1可以使用`,ok`来检测通道的关闭，使用情况有2种。

使用`ok-idiom`结合`for-select`结构第一种情况：**如果某个通道关闭后，需要退出协程，直接return即可**。例代码中，该协程需要从in通道读数据，还需要定时打印已经处理的数量，有2件事要做，所以不能使用for-range，需要使用for-select，当in关闭时，`ok=false`，我们直接返回。

```go
package main

import (
    "fmt"
    "time"
)

func worker(cancel chan bool) {
    for {
        select {
            case _, ok := <-cancel:
            // 如果ok为false，则直接返回退出
            if !ok {
                fmt.Println("I am out!")
                // 退出
                return
            }
            // 接收其他值时候进行正常工作
            default:
            fmt.Println("hello")
            // 正常工作
        }
    }
}

func main() {
    cancel := make(chan bool)
    go worker(cancel)

    time.Sleep(3 * time.Second)
    close(cancel)
}
```

使用`ok-idiom`结合`for-select`结构第二种情况：如果**某个通道关闭了，不再处理该通道，而是继续处理其他case，退出是等待所有的通道关闭。**

我们需要**使用select的一个特征：select不会在nil的通道上进行等待**。这种情况，把只读通道设置为nil即可解决。

```go
package main

import (
    "fmt"
    "time"
)

func worker(ch1, ch2 chan int,) {
    for {
        select {
            case _, ok := <-ch1:
            // 如果ok为false，置ch1位nil
            // 再往后的循环中，select不会再处理这个nil的case了
            if !ok {
                fmt.Println("ch1 is over!")
                ch1 = nil
            }
            // 正常处理
            case _, ok := <-ch2:
            // 同上
            if !ok {
                fmt.Println("ch2 is over!")
                ch2 = nil
            }
            // 接收其他值时候进行正常工作
            default:
            time.Sleep(1 * time.Second)
            fmt.Println("I am doing other work!")
            // 正常处理
        }
        // 如果ch1 和 ch2通道都被关闭了，才进行goroutine的退出
        if ch1 == nil && ch2 == nil {
            fmt.Println("worker is done!")
            return
        }
    }
}

func main() {
    ch1 := make(chan int)
    ch2 := make(chan int)
    go worker(ch1, ch2)

    time.Sleep(2 * time.Second)
    // 先关闭ch1
    close(ch1)
    time.Sleep(5 * time.Second)
    // 再关闭ch2
    close(ch2)
    // 延时一下再结束主协程
    time.Sleep(2 * time.Second)
}
```

打印结果如下：

![image-20200823234824352](https://garmen-imgsubmit.oss-cn-shenzhen.aliyuncs.com/img/20200823234824.png)

但问题是**如果我们想要退出两个或者任意多个Goroutine怎么办呢？**这里大概有两种办法：

<u>一种是向所有的goroutine发送同样数量的信号给对应的同步Channel来进行退出提示（上面的示例就是一个例子了！）</u>。但是这样并不是保险的，想想如果在发出发出信号的时候有些goroutine自动退出了，那么是不是<u>Channel中的事件数比需要关闭的goroutine还多</u>？这样一来，我们的发送就直接被阻塞了！除了发送到Channel的事件数目过多的情况，<u>过少的情况也可能出现</u>，比如待关闭的goroutine又生成了其他的goroutine，那样一来就会产生有些需要关闭的goroutine却没有收到退出的消息

> 最重要的一点在于Go的并发十分强大，我们很难知道某一个时刻具体运行着的goroutine数目，所以采用这种方法精确的去关闭多个goroutine是很困难的。

管道的发送操作和接收操作是一一对应的，如果要停止多个Goroutine那么可能需要创建同样数量的管道，这个代价太大了！

而另外一种则是通过Channel进行消息广播，**使用一个专门的通道，发送退出的信号**，我们看看如何进行一步步改进。

**首先我们可以通过不向Channel发送值而是使用`close`关闭一个Channel，从而实现广播的效果！**为什么不使用发送值而是使用close呢？因为当一个goroutine从一个channel中接收到一个值的时候，他会消费掉这个值，这样其它的goroutine就没法看到这条信息了。

比如说，我们启动了10个worker时，只要`main()`执行关闭cancel通道，每一个worker都会都到信号，进而关闭。示例代码如下：

```go
package main

import (
    "fmt"
    "time"
)

func worker(cancel chan bool) {
    for {
        select {
            case <-cancel:
            return
            // 退出
            default:
            fmt.Println("hello")
            // 正常工作
        }
    }
}

func main() {
    // 建立一个通知退出的同步Channel
    cancel := make(chan bool)
    // 创建10个goroutine
    for i := 0; i < 10; i++ {
        go worker(cancel)
    }
    // 休眠一段时间
    time.Sleep(time.Second)
    // 发送终止信号
    close(cancel)
}
```

这里存在的问题就是：当每个Goroutine收到退出指令退出时一般会进行一定的清理工作，但是退出的清理工作并不能保证被完成，因为**`main`线程并没有等待各个工作Goroutine退出工作完成的机制**。我们可以结合`sync.WaitGroup`来改进:

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

func worker(wg *sync.WaitGroup, cannel chan bool) {
    // 在程序处理完所有之后进行减一
    defer wg.Done()

    for {
        select {
            default:
            fmt.Println("hello")
            case <-cannel:
            return
        }
    }
}

func main() {
    cancel := make(chan bool)

    var wg sync.WaitGroup
    for i := 0; i < 10; i++ {
        // 每开始创建一个woker之前将计数加一
        wg.Add(1)
        go worker(&wg, cancel)
    }

    time.Sleep(time.Second)
    close(cancel)
    // 等待所有计数为0时才关闭主线程
    wg.Wait()
}
```

现在每个工作者并发体的创建、运行、暂停和退出都是在`main`函数的安全控制之下了。

小结一下处理并发协程的安全退出的几种方法：

1. 使用`ok-idiom`处理一个或者多个goroutine的关闭，但是多个goroutine的关闭并不推荐使用这种方式进行。
2. 通过Channel进行消息广播，**使用一个专门的Channel，通过`close()`发送退出的信号**。
3. 在第二点的基础上结合`sync.WaitGroup`来改进，完善为**`main`线程等待各个工作Goroutine退出工作完成的机制**



---

# 4.使用管道（Pipeline）优雅的从Channel循环取值

当通过Channel发送有限的数据时，我们可以通过`close()`函数关闭Channel来告知从该Channel接收值的goroutine停止等待。当Channel被关闭时，再继续往该Channel发送值则会引发panic，如果从该Channel里接收的值一直都是类型零值。**那如何判断一个通道是否被关闭了呢？**在前面的关于Channel的认识中我们了解到，可以使用ok-idiom 进行判断，接收操作有一个**变体形式**：它多接收一个结果，多接收的第二个结果是一个布尔值ok，ture表示成功从channels接收到值，false表示channels已经被关闭并且里面没有值可接收。

在下面的代码中，第一个goroutine是一个计数器，用于生成0、1、2、……形式的整数序列，然后通过channel将该整数序列发送给第二个goroutine；第二个goroutine是一个求平方的程序，对收到的每个整数求平方，然后将平方后的结果通过第二个channel发送给第三个goroutine；第三个goroutine是一个打印程序，打印收到的每个整数。

```go
package main

import (
    "fmt"
)

func main() {
    ch1 := make(chan int)
    ch2 := make(chan int)

    // 开启goroutine将0~100的数发送到ch1中
    go func() {
        for i := 0; i < 100; i++ {
            ch1 <- i
        }
        close(ch1)
    }()

    // 开启goroutine从ch1中接收值，并将该值的平方发送到ch2中
    go func() {
        for {
            i, ok := <-ch1 // 通道关闭后再取值，此时ok=false
            if !ok { // 如果ok为false，证明通道已经关闭了，则break，然后执行close
                break
            }
            ch2 <- i * i
        }
        close(ch2)
    }()

    // 在主goroutine中从ch2中接收值打印
    // 通道关闭后会退出for range循环
    for i := range ch2 {
        fmt.Println(i)
    }
}
```

从上面的例子中我们看到有两种方式在接收值的时候判断通道是否被关闭：

- 一种是使用`ok-idiom`
- 另外一种就是使用`for range`了，而我们通常使用的是`for range`的方式。

为什么for range能够起到作用呢？因为range channel 可以直接取到 channel 中的值。当我们使用 range 来操作 channel 的时候，**它依次从channel接收数据，当channel被关闭并且没有值可接收时跳出循环**。这应该和`for range`的语法糖相关，后续了解到`for range`的语法糖的时候，再返回来解决详细的解决这个疑惑！



---

# 5.实现生产者消费者模型

生产者消费者模型是很常见的了，在操作系统看见的次数可不少，这是并发编程中最常见的例子了，该模式主要通过平衡生产线程和消费线程的工作能力来提高程序的整体处理数据的速度。简单地说，就是生产者生产一些数据，然后放到成果队列中，同时消费者从成果队列中来取这些数据。这样就让生产消费变成了异步的两个过程。当成果队列中没有数据时，消费者就进入饥饿的等待中；而当成果队列中数据已满时，生产者则面临因产品挤压导致CPU被剥夺的下岗问题。

```go
package main

import (
    "fmt"
    "time"
)

// Producer ： 生产者，生成 factor 整数倍的序列
func Producer(factor int, out chan<- int) {
    for i := 0; i <= 100; i++ {
        fmt.Println("生产了：", i*factor)
        out <- i*factor
    }
}

// Consumer ：消费者
func Consumer(in <-chan int) {
    for v := range in {
        fmt.Println("消费了：",v)
    }
}

func main() {
    ch := make(chan int, 64) // 成果队列

    go Producer(2, ch) // 生成 2 的倍数的序列
    go Consumer(ch)    // 消费 生成的队列

    // 运行一定时间后退出
    time.Sleep(2 * time.Second)
}
```

还可以进行改进，我们让`main`函数保存阻塞状态不退出，只有当用户输入`Ctrl-C`时才真正退出程序：

```go
package main

import (
    "fmt"
    "os"
    "os/signal"
    "syscall"
)

// Producer ： 生产者，生成 factor 整数倍的序列
func Producer(factor int, out chan<- int) {
    for i := 0; i <= 100; i++ {
        fmt.Println("生产了：", i*factor)
        out <- i*factor
    }
}

// Consumer ：消费者
func Consumer(in <-chan int) {
    for v := range in {
        fmt.Println("消费了：",v)
    }
}

func main() {
    ch := make(chan int, 64) // 成果队列

    go Producer(2, ch) // 生成 3 的倍数的序列
    go Consumer(ch)    // 消费 生成的队列

    // Ctrl+C 退出
    sig := make(chan os.Signal, 1)
    signal.Notify(sig, syscall.SIGINT, syscall.SIGTERM)
    fmt.Printf("quit (%v)\n", <-sig)
}
```



---

# 参考文章

- [Go语言中文文档-Channel](http://topgoer.com/%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B/channel.html)
- [深入理解 Go Channel](http://legendtkl.com/2017/07/30/understanding-golang-channel/)
- [第八章 Goroutines和Channels](https://www.kancloud.cn/lbb4511/gopl/1107899)
- [Go的50度灰：Golang新开发者要注意的陷阱和常见错误](https://colobu.com/2015/09/07/gotchas-and-common-mistakes-in-go-golang/#%E5%90%91%E6%97%A0%E7%BC%93%E5%AD%98%E7%9A%84Channel%E5%8F%91%E9%80%81%E6%B6%88%E6%81%AF%EF%BC%8C%E5%8F%AA%E8%A6%81%E7%9B%AE%E6%A0%87%E6%8E%A5%E6%94%B6%E8%80%85%E5%87%86%E5%A4%87%E5%A5%BD%E5%B0%B1%E4%BC%9A%E7%AB%8B%E5%8D%B3%E8%BF%94%E5%9B%9E)
- [面向并发的内存模型](https://chai2010.cn/advanced-go-programming-book/ch1-basic/ch1-05-mem.html)
- [Golang并发模型：并发协程的优雅退出](https://lessisbetter.site/2018/12/02/golang-exit-goroutine-in-3-ways/)