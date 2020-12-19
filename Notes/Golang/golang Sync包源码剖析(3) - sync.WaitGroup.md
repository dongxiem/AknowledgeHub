# 1.sync.WaitGroup 介绍



[sync.WaitGroup](https://sourcegraph.com/github.com/golang/go@master/-/blob/src/sync/waitgroup.go#L21:2) 按照官方注释给的解释，**它可以等待一组 Goroutine 集合的结束，主 `goroutine` 通过调用 `Add()` 函数来设置一定数量进行等待的 `goroutines` ，然后其余的一些 `goroutines` 则进行各自的运行结束之后再调用 `Done()`，这样一来，等待的主 `goroutine` 会阻塞知道其余所有 `goroutines`都结束**。

> A WaitGroup waits for a collection of goroutines to finish.The main goroutine calls Add to set the number of goroutines to wait for. Then each of the goroutines runs and calls Done when finished. At the same time, Wait can be used to block until all goroutines have finished.

注：以下`golang`源码版本为：1.15

---

# 2.源码解析

## 结构体定义

```go
// A WaitGroup must not be copied after first use.
type WaitGroup struct {
    // 保证 sync.WaitGroup 不会被开发者通过再赋值的方式拷贝；
	noCopy noCopy
    // 存储着状态和信号量；
	state1 [3]uint32
}
```

**第一个参数：`noCopy`**

`noCopy`：它是 `sync` 包下的一个特殊标记，可以嵌入到结构中，在第一次使用后不可复制，使用`go vet`作为检测使用，并因此只能进行指针传递，从而保证全局唯一；

比如下面的演示代码中：

```go
func main () {
    wg := sync.WaitGroup {}
    w := wg
    fmt.Println (w, wg)
}
```

`run` 的时候没问题，但是如果你使用 `go vet` 做个检查就有警告了：

```go
$ go vet proc.go
./prog.go:10:10: assignment copies lock value to yawg: sync.WaitGroup
./prog.go:11:14: call of fmt.Println copies lock value: sync.WaitGroup
./prog.go:11:18: call of fmt.Println copies lock value: sync.WaitGroup
```

所以它能够保证 `sync.WaitGroup` 不会被开发者通过再赋值的方式拷贝；

注：`golang` 的 `vet` 工具是 `golang` 代码静态诊断器，可以用以检查 `golang` 项目中可通过编译但仍可能存在错误的代码，例如无法访问的代码、错误的锁使用、不必要的赋值、布尔运算错误等。

**第二个参数：`state1`**

state1：是用来存放任务计数器和等待者计数器，应该会涉及到一些位操作相关的内容，估计还是挺棘手的，需要注意的是：**WaitGroup 在 64 位和 32 位机器是持有不同的状态**，其相对应的具体内容如下：

|       | state [0] | state [1] | state [2] |
| :---- | :-------- | :-------- | :-------- |
| 64 位 | waiter    | counter   | sema      |
| 32 位 | sema      | waiter    | counter   |

注意其中 `waiter` 是等待者计数，`counter` 是任务计数，`sema` 是信号量。具体可以看看注释（不过这是之前版本的注释，官方并未修改）：

> 64-bit value: high 32 bits are counter, low 32 bits are waiter count.
> 64-bit atomic operations require 64-bit alignment, but 32-bit compilers do not ensure it. 
>
> So we allocate 12 bytes and then use the aligned 8 bytes in them as state, and the other 4 as storage for the sema.

其实如果看其他书籍或者博客，会发现之前版本的 `WaitGroup` 结构体定义并非如此：

```go
  type WaitGroup struct {
    noCopy noCopy  //该WaitGroup对象不允许拷贝使用,只能用指针传递

    state1 [12]byte
    sema   uint32  //信号量
}
```

关于新版本的 `go` 为什么重新设计可以看看注释，我认为应该是为了操作的便利性。

同时`sync.WaitGroup` 提供的私有方法 `sync.WaitGroup.state` 能够帮我们从 `state1` 字段中取出它的状态（`statep = waiter + counter`）和信号量（`semap`）。

```go
// 获取 statep、semap 的值
func (wg *WaitGroup) state() (statep *uint64, semap *uint32) {
    // 根据 state1 的起始地址分析，判断是否为 8 字节对齐的（64位）
    if uintptr(unsafe.Pointer(&wg.state1))%8 == 0 {
        return (*uint64)(unsafe.Pointer(&wg.state1)), &wg.state1[2]
    } else {
        return (*uint64)(unsafe.Pointer(&wg.state1[1])), &wg.state1[0]
    }
}
```



## Add()

`sync.WaitGroup` 对外暴露了三个方法分别是 `sync.WaitGroup.Add`、`sync.WaitGroup.Wait` 和 `sync.WaitGroup.Done`，而有意思的是`sync.WaitGroup.Done`向 `sync.WaitGroup.Add` 方法传入了 -1（没错，就是这么简单，也说明了这个 `delta` 可以为负数），接下来先从`sync.WaitGroup.Add` 开始分析起。

通过 `Add()` 函数我们传进了 `delta` 这么一个值，它要加到 `WaitGroup` 的计数器当中，它可以是负数；如果计数器变为零时，所有被阻塞的 `goroutines` 都会被释放，如果计数器变为负数时，则会报 `panic`（意思就是，计数器不能为负）。

其源码具体如下：

```go
func (wg *WaitGroup) Add(delta int) {
    // 首先获得 statep（waiter + counter） 和 semap
    statep, semap := wg.state()
    // 竞态检测
    if race.Enabled {
        _ = *statep // trigger nil deref early
        if delta < 0 {
            // Synchronize decrements with Wait.
            race.ReleaseMerge(unsafe.Pointer(wg))
        }
        race.Disable()
        defer race.Enable()
    }
    // 首先将 delta 左移32位，再加到 statep 中，此时是叠加到 staep 中的 counter计数器
    state := atomic.AddUint64(statep, uint64(delta)<<32)
    // 将 counter 和 waiter 进行抽离，分别封装为 v、w
    v := int32(state >> 32)
    w := uint32(state)
    
    if race.Enabled && delta > 0 && v == int32(delta) {
        race.Read(unsafe.Pointer(semap))
    }
    // 如果计数器 counter 为负，则panic
    if v < 0 {
        panic("sync: negative WaitGroup counter")
    }
    // waiter值不为0，delta大于零且累加后的counter值和delta相等，
    // 说明Wait()方法没有在Add()方法之后调用，触发panic，因为正确的做法是先Add()后Wait()
    if w != 0 && delta > 0 && v == int32(delta) {
        panic("sync: WaitGroup misuse: Add called concurrently with Wait")
    }
    // Add()添加正常返回
    // 1.counter > 0，说明还不需要释放信号量，可以直接返回；
    // 2.waiter = 0，说明没有等待的goroutine，也不需要释放信号量，可以直接返回；
    if v > 0 || w == 0 {
        return
    }
    // 下面是 counter == 0 并且 waiter > 0的情况
    // 现在若原state和新的state不等，则有以下两种可能：
    // 	- Add 与 Wait 同时调用；
    // 	- 如果counter已经0，但Wait 继续增加等待计数器 waiters，这种情况永远不会触发信号量；
    // 以上两种情况都是错误的，所以触发异常。
    // 注：state := atomic.AddUint64(statep, uint64(delta)<<32)  这一步调用之后，state和*statep的值应该是相等的，除非有以上两种情况发生。
    if *statep != state {
        panic("sync: WaitGroup misuse: Add called concurrently with Wait")
    }
    // 将 waiter 和 counter都置为0
    *statep = 0
    // 当调用计数器归零，也就是所有任务都执行完成时，就会通过 sync.runtime_Semrelease 唤醒处于等待状态的所有 Goroutine
    for ; w != 0; w-- {
        runtime_Semrelease(semap, false, 0)
    }
}
```



捋一捋，以上的 `Add()` 函数主要完成的以下内容：

1. 首先通过 `wg.state()` 得到  `statep`（`waiter + counter`） 和 `semap`，然后将 `Add()` 传进的参数 `delta` 添加至 `counter`，再将`statep` 中的 `waiter` 和 `coutner` 分别抽离出来并封装成为 `v` 和 `w`，然后对 `v` 和 `w`做一些校验，比如
   1. 计数器 `counter` 为负，则触发`panic`；
   2. 还有 `Wait()` 方法没有在 `Add()` 方法之后调用，则触发panic；
2. 如果 `Add()` 添加正常则返回。
3. 再对原`state`和新的`state`不等的两种情况进行判断是否出错，出错则报panic，具体的两种情况为：
   1. Add 与 Wait 同时调用；
   2. 如果counter已经0，但Wait 继续增加等待计数器 waiters，这种情况永远不会触发信号量；
4. 最后对于 `w > 0` 的情况，会进行 for 循环直到调用计数器归零，也就是所有任务都执行完成时，就会通过 `sync.runtime_Semrelease` 唤醒处于等待状态的所有 `Goroutine`。



## Done()

`Done()` 就简单将`counter`值减1，这里就能够理解为什么上面说 `delta` 可以为负数了。

```go
// Done decrements the WaitGroup counter by one.
func (wg *WaitGroup) Done() {
	wg.Add(-1)
}
```



## Wait()

`Wait()` 会在计数器 `counter` 大于 0 并且不存在等待的 `Goroutine` 时，调用 `sync.runtime_Semacquire` 陷入睡眠状态，直到 `couner` 的值为0时进行唤醒。

```go
func (wg *WaitGroup) Wait() {
    // 同样得到 statep（waiter + counter） 和 semap
    statep, semap := wg.state()
    // 竞态检测
    if race.Enabled {
        _ = *statep // trigger nil deref early
        race.Disable()
    }
    // 进行一个死循环判断
    for {
        state := atomic.LoadUint64(statep)
        v := int32(state >> 32)
        w := uint32(state)
        // 如果 counter 为0，则停止死循环，进行 return 跳出
        if v == 0 {
            if race.Enabled {
                race.Enable()
                race.Acquire(unsafe.Pointer(wg))
            }
            return
        }
        // 通过CAS方法进行原子地增加 waiter的值，并且for循环会一直尝试，保证多个goroutine同时调用Wait()也能正确累加waiter；
        // 注意等待者计数在低32位，可以直接加，不需要移位操作
        if atomic.CompareAndSwapUint64(statep, state, state+1) {
            if race.Enabled && w == 0 {
                race.Write(unsafe.Pointer(semap))
            }
            // 同步原语，进行信号量的等待
            runtime_Semacquire(semap)
            // 之前的Add()函数中，触发信号量前会先将counter和waiter置0，所以此时*statep必定为0
            // 若不为0，说明WaitGroup此时又被执行Add()或者Wait()操作了，应panic
            if *statep != 0 {
                panic("sync: WaitGroup is reused before previous Wait has returned")
            }
            if race.Enabled {
                race.Enable()
                race.Acquire(unsafe.Pointer(wg))
            }
            return
        }
    }
}
```

可以看到当 `sync.WaitGroup` 的计数器 `Counter` 归零时，陷入睡眠状态的 Goroutine 就被唤醒，上述方法会立刻返回。



---

# 3.小结



通过源码的解析，可以得到以下的一些认识：

- `WaitGroup` 利用信号量来实现任务结束的通知；
- `Wait()` 可以被调用多次，也即可以同时有多个 Goroutine 等待当前 [`sync.WaitGroup`](https://github.com/golang/go/blob/71239b4f491698397149868c88d2c851de2cd49b/src/sync/waitgroup.go#L20-L29) 计数器的归零，并且每个都会收到完成的通知；
- `sync.WaitGroup.Done` 只是对 `sync.WaitGroup.Add` 方法的简单封装，我们可以向 `sync.WaitGroup.Add` 方法传入任意负数（需要保证计数器非负）快速将计数器归零以唤醒其他等待的 Goroutine；
- `WaitGroup`作为参数传递的时候需要注意传递指针，或者尽量避免传递；
- `WaitGroup`不能保证多个 `goroutine` 执行次序

但是使用的时候需要注意以下几点：

- `Add()`操作必须早于`Wait()`，否则会`panic`；
- `Add()`设置的值必须与实际等待的`goroutine`个数一致，否则会`panic`；
- `WaitGroup` 必须在 `Wait()` 方法返回之后才能被重新使用；
- `WaitGroup`只可保持一份，不可拷贝给其他变量，否则会造成意想不到的BUG；

最后：实在忍不了吐槽一下，虽然很多地方我分析得不是很好，但是 `go夜读` 的一些源码解析也太烂了，直接将注解扔进谷歌翻译/百度翻译，最后也都不审核一下，简直就是不能入眼。

![image-20201006140734579](https://garmen-imgsubmit.oss-cn-shenzhen.aliyuncs.com/img/20201006140734.png)

![image-20201006140833507](https://garmen-imgsubmit.oss-cn-shenzhen.aliyuncs.com/img/20201006140833.png)