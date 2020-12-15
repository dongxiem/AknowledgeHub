# 1.Sync包 简明介绍

`golang` 语言有着天生适合高并发的特性，作为原生支持用户态进程（`Goroutine`）的语言，当涉及到并发编程，多线程编程时候，则离不开锁相关的概念了，于是编写 `golang` 源码的大叔们就给了[src/sync](https://sourcegraph.com/github.com/golang/go@master/-/tree/src/sync) 这么个包，通过浏览源码可以知道[src/sync](https://sourcegraph.com/github.com/golang/go@master/-/tree/src/sync)包中提供了用于同步的一些基本原语，可以说这是一个很重要的的包了，如果掌握这个包，在编写代码的工程中肯定能够大有帮助，这个包大体上有：[sync.Mutex](https://sourcegraph.com/github.com/golang/go@master/-/blob/src/sync/mutex.go#L11)、[sync.RWMutex](https://sourcegraph.com/github.com/golang/go@master/-/blob/src/sync/rwmutex.go#L7)、[sync.WaitGroup](https://sourcegraph.com/github.com/golang/go@master/-/blob/src/sync/waitgroup.go#L7)、[sync.Map](https://sourcegraph.com/github.com/golang/go@master/-/blob/src/sync/map.go#L7)、[sync.Pool](https://sourcegraph.com/github.com/golang/go@master/-/blob/src/sync/pool.go#L44)、[sync.Once](https://sourcegraph.com/github.com/golang/go@master/-/blob/src/sync/once.go#L7)、[sync.Cond](https://sourcegraph.com/github.com/golang/go@master/-/blob/src/sync/cond.go#L5)，也可以看一下官方给的sync包的一些方法解释： [`sync`](https://golang.org/pkg/sync/) ，同时为了节省一下 Clone 源码时间，或者只想简单回顾一下源码，这里给了快捷的入口：[src/sync](https://sourcegraph.com/github.com/golang/go@master/-/tree/src/sync)。

在并发编程中，同步原语或者锁，他们的主要作用是保证多个线程或者多个`goroutine`在访问同一片内存时不会出现混乱的问题，这个是一个非常重要的内容，很多人在编写代码的时候常常没有注重对这些并发访问的内容进行关注，导致出现事故也不清楚是什么原因，而[`sync`](https://golang.org/pkg/sync/) 包中所有的结构都适用于`goroutine`并发执行的情况，需要好好掌握。

注：以下`golang`源码版本为：1.15

---

# 2.sync.Mutex 源码解析

​    

## sync.Mutex 背景更迭

**我认为了解知识的最好方式是从其发展背景进行了解，比如它在发展的过程中，做了哪些改变？是因为什么问题而做了这些改变？**我想如果有这些问题的话，了解会更加深入一些。

我们可以看看`Russ Cox`在 2008 提交的第一版的`Mutex`实现

```go
type Mutex struct {
    key int32;
    sema int32;
}

func xadd(val *int32, delta int32) (new int32) {
    for {
        v := *val;
        if cas(val, v, v+delta) {
            return v+delta;
        }
    }
    panic("unreached")
}

func (m *Mutex) Lock() {
    if xadd(&m.key, 1) == 1 {
        // changed from 0 to 1; we hold lock
        return;
    }
    sys.semacquire(&m.sema);
}

func (m *Mutex) Unlock() {
    if xadd(&m.key, -1) == 0 {
        // changed from 1 to 0; no contention
        return;
    }
    sys.semrelease(&m.sema);
}
```

> 这个版本相对简单，<u>只需要通过`cas`对 `key` 进行加一, 如果`key`的值是从`0`加到`1`, 则直接获得了锁。否则通过`semacquire`进行sleep, 被唤醒的时候就获得了锁</u>。
>
> 2012年， commit [dd2074c8](https://github.com/golang/go/commit/dd2074c82acda9b50896bf29569ba290a0d13b03)做了一次大的改动，它将`waiter count`（等待者的数量）和`锁标识`分开来(内部实现还是合用使用`state`字段)。新来的 goroutine 占优势，会有更大的机会获取锁。
>
> 2015年， commit [edcad863](https://github.com/golang/go/commit/edcad8639a902741dc49f77d000ed62b0cc6956f)，<u>Go 1.5中`mutex`实现为全协作式的，增加了spin机制，一旦有竞争，当前goroutine就会进入调度器</u>。在临界区执行很短的情况下可能不是最好的解决方案。
>
> 2016年 commit [0556e262](https://github.com/golang/go/commit/0556e26273f704db73df9e7c4c3d2e8434dec7be)，<u>Go 1.9中增加了饥饿模式，让锁变得更公平，不公平的等待时间限制在1毫秒</u>，并且修复了一个大bug，唤醒的goroutine总是放在等待队列的尾部会导致更加不公平的等待时间。
>
> 而现在的 Go 1.15 中的也稍微更加复杂了一些，大体上的变化就是上面那几个了，初次看很容易迷糊，绝对不是说像初次版本中只有加锁解锁那么简单，对正常模式和饥饿模式的认识我也是通过源码才得知的，还有这其中对字段的一些共用、标识位的一些位操作都挺麻烦的，需要对源码进行进一步的理解。
>

   

## sync.Mutex 结构体及常量定义

[sync.Mutex](https://sourcegraph.com/github.com/golang/go@master/-/blob/src/sync/mutex.go#L13) 可能是`sync`包中使用最广泛的原语，顾名思义就是相互排斥的锁，它确保同一时刻只有一个协程能访问某对象，也即它允许在共享资源上互斥访问（不能同时访问），初始状态为`unlock`。

`Mutex` 的结构体定义如下：

```go
type Mutex struct {
    state int32
    sema  uint32
}
```

可以看到 `Mutex` 是由 `state` 和 `sema` 两个字段构成的，`int32` 和 `uint32` 均为四个字节，所以一个 `Mutex` 一般需要消耗八个字节的空间。通过源码可以很容易理解 `state` 为状态表示，而 sema 的用途是什么呢？ `sema` 其实是用来控制锁状态的信号量，它是一个非负数。

查看源码我们会发现有以下几个常量：

```go
const (
    mutexLocked = 1 << iota // mutex is locked
    mutexWoken
    mutexStarving
    mutexWaiterShift = iota
    starvationThresholdNs = 1e6
)
```

对这几个常量稍微解释一下：

- **mutexLocked** ：值为 1，第一位为 1，表示 `mutex` 已经被加锁。根据 `mutex.state & mutexLocked` 的结果来判断 `mutex` 的状态：该位为 1 表示已加锁，0 表示未加锁。
- **mutexWoken**：值为 2（二进制：10），第二位为 1，表示 `mutex` 是否被唤醒。根据 `mutex.state & mutexWoken` 的结果判断 `mutex` 是否被唤醒：该位为 1 表示已被唤醒，0 表示未被唤醒。
- **mutexStarving**：值为 4（二进制：100），第三位为 1，表示 `mutex` 是否处于饥饿模式。根据 `mutex.state & mutexWoken` 的结果判断 `mutex` 是否处于饥饿模式：该位为 1 表示处于饥饿模式，0 表示正常模式。
- **mutexWaiterShift**：值为 3，表示 `mutex.state` 右移 3 位后即为等待的 `goroutine` 的数量，也即表示统计阻塞在该`mutex`上的`goroutine`数目需要移位的数值。根据 `mutex.state >> mutexWaiterShift` 得到当前等待的 `goroutine` 数目

`Mutex` 中的 `state`其32位 bit 的分布具体如下所示：

```go
1111 1111 ...... 1111 1 1 1 1
|_________29__________| ↓ ↓ ↓
          ↓             ↓ ↓ \ 表示当前 mutex 是否加锁
          ↓             ↓ \ 表示当前 mutex 是否被唤醒
          ↓             \ 表示 mutex 当前是否处于饥饿状态
          ↓
  存储等待 goroutine 数量
```



还有最后一个常量，这个常量尤其重要，因为它引出了 引出了 `sync.Mutex` 的一个特性：保证公平。怎样来保证公平呢？通过引入正常状态和饥饿状态模式进行。

- **starvationThresholdNs**：值为 1000000 纳秒，即 1ms，表示将 `mutex` 切换到饥饿模式的等待时间阈值。这个常量在源码中有大篇幅的注释，理解这段注释对理解程序逻辑至关重要。

注解原文：[Mutex fairness](https://sourcegraph.com/github.com/golang/go@master/-/blob/src/sync/mutex.go#L42)

​       

## 饥饿模式与正常模式

将上面的注解原文，进行一个简单的翻译处理之后的认识如下。（从源码作者口中讲出的东西，更有说服力！）

饥饿模式是在 Go 语言 [1.9](https://github.com/golang/go/commit/0556e26273f704db73df9e7c4c3d2e8434dec7be) 版本引入的优化，引入的**<u>目的是保证互斥锁的公平性（Fairness）</u>**。互斥锁有两种状态：正常状态和饥饿状态。

**<u>在正常状态下，所有等待锁的goroutine按照FIFO顺序等待</u>**。唤醒的`goroutine`不会直接拥有锁，而是会和新请求锁的goroutine竞争锁的拥有。新请求锁的`goroutine`具有优势：它正在CPU上执行，而且可能有好几个，所以刚刚唤醒的`goroutine`有很大可能在锁竞争中失败。在这种情况下，这个被唤醒的`goroutine`会加入到等待队列的前面。 如果一个等待的`goroutine`超过1ms没有获取锁，那么它将会把锁转变为饥饿模式。

**<u>在饥饿模式下，锁的所有权将从`unlock`的`gorutine`直接交给交给等待队列中的第一个</u>**。新来的`goroutine`将不会尝试去获得锁，即使锁看起来是`unlock`状态, 也不会去尝试自旋操作（等也白等，在饥饿模式下是不会给你的），而是乖乖地待在等待队列的尾部。

如果一个等待的`goroutine`获取了锁，并且满足一以下其中的任何一个条件：（1）它是等待队列中的最后一个；（2）它等待的时候小于1ms。它会将锁的状态转换为正常状态。

相比于饥饿模式，正常模式有更好地性能，因为一个 `goroutine` 可以连续获得好几次 `mutex`，即使有阻塞的等待者。而饥饿模式可以有效防止出现位于等待队列尾部的等待者一直无法获取到 `mutex` 的情况。



## 加锁

[sync.Mutex](https://sourcegraph.com/github.com/golang/go@master/-/blob/src/sync/mutex.go#L13) 有加锁和解锁，它们分别使用 [sync.Mutex.Lock](https://sourcegraph.com/github.com/golang/go@master/-/blob/src/sync/mutex.go#L72:22) 和 [sync.Mutex.Unlock](https://sourcegraph.com/github.com/golang/go@master/-/blob/src/sync/mutex.go#L179) 方法。

互斥锁的加锁是靠 [sync.Mutex.Lock](https://sourcegraph.com/github.com/golang/go@master/-/blob/src/sync/mutex.go#L72:22) 进行完成的，如果参考其他以往版本的 Golang 源码解析，会发现最新版本的 Golang 源码将 [sync.Mutex.Lock](https://sourcegraph.com/github.com/golang/go@master/-/blob/src/sync/mutex.go#L72:22) 进行了简化，方法的主干只保留最常见、简单的情况 — 当锁的状态是 0 时，将 `mutexLocked` 位置成 1：

```go
func (m *Mutex) Lock() {
    // Fast path: grab unlocked mutex.
    // 查看 state 是否为0，如果是则表示可以加锁，将其状态转换为1，当前 goroutine 加锁成功，函数返回
    if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
        if race.Enabled {
            race.Acquire(unsafe.Pointer(m))
        }
        return
    }
    // Slow path (outlined so that the fast path can be inlined)
    m.lockSlow()
}
```

这个 `atomic.CompareAndSwapInt32()` 方法的签名如下：

```go
// CompareAndSwapInt32 executes the compare-and-swap operation for an int32 value.
func CompareAndSwapInt32(addr *int32, old, new int32) (swapped bool)
```

这里对 `atomic.CompareAndSwapInt32()` 方法进行一个解释，`atomic`包是由golang提供的low-level的原子操作封装，主要用来解决进程同步问题，但官方并不建议直接使用。`CompareAndSwapInt32()`就是int32型数字的`compare-and-swap`实现。**<u>`cas(&addr, old, new)`的意思是`if *addr==old, *addr=new`。大部分操作系统支持CAS，x86指令集上的CAS汇编指令是`CMPXCHG`。</u>**

当然，这里还是可以深入理解一下 `CAS` 的底层实现，源码可见：[src/runtime/internal/atomic/asm_amd64.s](https://sourcegraph.com/github.com/golang/go@master/-/blob/src/runtime/internal/atomic/asm_amd64.s)，对汇编语言的了解较少，读起来较为吃力，具体的代码片段及注释可见下面：

```Assembly 
// bool Cas(int32 *val, int32 old, int32 new)
// Atomically:
//	if(*val == old){
//		*val = new;
//		return 1;
//	} else
//		return 0;

// 一个指针在 amd64 下是8个字节，int32分别是占用4字节，最后的返回值是bool占用1字节
// 所以参数及返回值大小总共占17个字节
TEXT runtime∕internal∕atomic·Cas(SB),NOSPLIT,$0-17
	// 将*val指针放入到寄存器BX中
	MOVQ	ptr+0(FP), BX
	// AX要用来存参数old
	MOVL	old+8(FP), AX
	// 把new中的数存到寄存器CX中
	MOVL	new+12(FP), CX
	// 使用了LOCK前缀，保证操作是原子的
	LOCK
	// 0(BX) 可以理解为 *val
	// 把 AX中的数 和 第二个操作数 0(BX)——也就是BX寄存器所指向的地址中存的值 进行比较
	// 如果相等，就把 第一个操作数 CX寄存器中存的值 赋给 第二个操作数 BX寄存器所指向的地址
	// 并将标志寄存器ZF设为1
	// 否则将标志寄存器ZF清零
	CMPXCHGL	CX, 0(BX)
	// SETE的作用是：
	// 如果Zero Flag标志寄存器为1，那么就把操作数设为1，否则把操作数设为0
	// 也就是说，如果上面的比较相等了，就返回true，否则为false
	// ret+16(FP)代表了返回值的地址
	SETEQ	ret+16(FP)
	RET
```

从上面的汇编源码可以知道，大概的流程为：看看这把锁是不是空闲状态，如果是的话，直接原子性地修改一下 `state` 为已被获取就行了。大概了解一下，知道该方法是原子性的即可。

继续回到 [sync.Mutex.Lock](https://sourcegraph.com/github.com/golang/go@master/-/blob/src/sync/mutex.go#L72:22) 当中，接下来判断如果 [sync.Mutex](https://sourcegraph.com/github.com/golang/go@master/-/blob/src/sync/mutex.go#L13) 的状态不为0的时候， [sync.Mutex.Lock](https://sourcegraph.com/github.com/golang/go@master/-/blob/src/sync/mutex.go#L72:22) 就会进入 `m.lockSlow()` 方法，那么`m.lockSlow()` 方法做了些什么事情呢？注意，我将分段解析这个方法，具体合并的注释理解在最后附上。

我们先对其<u>一些变量</u>做一些简单的注解解释：

```go
func (m *Mutex) lockSlow() {
    var waitStartTime int64 // 用来存当前goroutine等待的时间
    starving := false // 用来存当前goroutine是否饥饿
    awoke := false // 用来存当前goroutine是否已唤醒
    iter := 0 // 用来存当前goroutine的循环次数
    old := m.state // 复制一下当前锁的状态
```

   

接下来分成几个部分介绍获取锁的过程：

1. <u>**判断 `Goroutine` 的状态，看是否能够进行自旋等锁；**</u>

```go
    for {
        // 进入到这个循环的，有两种角色goroutine，一种是新来的goroutine。另一种是被唤醒的goroutine；
        // 所以它们可能在这个地方再一起竞争锁，如果新来的goroutine抢成功了，那另一个只能再阻塞着等待，
        // 但超过1ms后，锁会转换成饥饿模式。在这个模式下，所有新来的goroutine必须排在队伍的后面，没有抢锁资格。也即：
        // 如果是饥饿情况之下，就不要自旋了，因为锁会直接交给队列头部的goroutine
        // 如果锁是被获取状态，并且满足自旋条件（canSpin见后文分析），那么就自旋等锁
        if old&(mutexLocked|mutexStarving) == mutexLocked && runtime_canSpin(iter) {
            // 通过了上面的检测，这时进行自旋是有意义的。
            // 通过把 mutexWoken 标识为 true，以通知 Unlock 方法就先不要叫醒其它阻塞着的 goroutine 了
            if !awoke && old&mutexWoken == 0 && old>>mutexWaiterShift != 0 &&
            atomic.CompareAndSwapInt32(&m.state, old, old|mutexWoken) {
                awoke = true
            }
            // 将当前 goroutine 标识为唤醒状态后，执行自旋操作，计数器加一，将当前状态记录到 old，继续循环等待
            runtime_doSpin()
            iter++
            old = m.state
            continue
        }
```

总的来说需要注意**<u>如果是饥饿模式则不进行自旋，因为锁的所有权会直接交给队列头部的goroutine，所以在这个饥饿状态下，无论如何都无法获得mutex</u>**。 

需要了解的是自旋是一种多线程同步机制，**<u>当前的进程在进入自旋的过程中会一直保持 CPU 的占用，持续检查某个条件是否为真</u>**。在多核的 CPU 上，自旋可以避免 Goroutine 的切换，使用恰当会对性能带来很大的增益，但是使用的不恰当就会拖慢整个程序，所以 Goroutine 进入自旋的条件非常苛刻：

1. 互斥锁只有在普通模式才能进入自旋；
2. 需要等待 [runtime_canSpin()](https://sourcegraph.com/github.com/golang/go@8ab020adb27089fa207d015f2f69600ef3d1d307/-/blob/src/sync/runtime.go#L52:1) 返回 True；

而 [runtime_canSpin()](https://github.com/golang/go/blob/cfe3cd903f018dec3cb5997d53b1744df4e53909/src/runtime/proc.go#L5339-L5352)  究竟是做些什么判断呢？其源码片段如下：

```go
// Active spinning for sync.Mutex.
func sync_runtime_canSpin(i int) bool {
    if i >= active_spin || ncpu <= 1 || gomaxprocs <= int32(sched.npidle+sched.nmspinning)+1 {
        return false
    }
    if p := getg().m.p.ptr(); !runqempty(p) {
        return false
    }
    return true
}
```

源码注释认为因为 `sync.Mutex` 是协作的，所以对于 `Spin` 我们应该要保守一些使用，使用 `Spin` 的条件还挺严苛，看看其需要满足什么条件：

1. 旋次数小于active_spin（这里是4）次；
2. 然后应该运行在多内核的机器上，且`GOMAXPROCS`的数目应该要大于1；（如果`GOMAXPROCS`不了解，可以看看 `Goroutine `相对应的 `GMP` 模型）
3. 还有当前机器上至少存在一个正在运行的处理器 P 并且处理的运行队列为空；

如果当前的 `Goroutine` 能够满足以上进入 `Spin`的条件，则会调用 [runtime_doSpin](https://sourcegraph.com/github.com/golang/go@master/-/blob/src/runtime/proc.go#L5595:6) 进行 `Spin`。**<u>所以可以看出来，并不是一直无限自旋下去的，当自旋次数到达 4 次或者其它条件不符合的时候，就改为信号量拿锁了</u>**。



2. **<u>通过自旋等待互斥锁的释放；</u>**

上面已经分析了，`m.lockSlow()` 会调用 [runtime_doSpin](https://sourcegraph.com/github.com/golang/go@master/-/blob/src/runtime/proc.go#L5595:6) 进行 `Spin`进行自旋操作，其源码片段如下：

```go
//go:linkname sync_runtime_doSpin sync.runtime_doSpin
//go:nosplit
func sync_runtime_doSpin() {
	procyield(active_spin_cnt)
}
```

`procyield()` 会执行 30 次的 `PAUSE` 指令，该指令只会占用 CPU 并消耗 CPU 时间：

```assembly
TEXT runtime·procyield(SB),NOSPLIT,$0-0
	MOVL	cycles+0(FP), AX
again:
	PAUSE
	SUBL	$1, AX
	JNZ	again
	RET
```



3. **<u>计算互斥锁的最新状态；</u>**

如果此时 `Goroutine` 不能进行自旋操作，则会进入剩余的代码逻辑；到了这一步， state的状态可能是：

1. 锁还没有被释放，锁处于正常状态；
2. 锁还没有被释放， 锁处于饥饿状态；
3. 锁已经被释放， 锁处于正常状态；
4. 锁已经被释放， 锁处于饥饿状态；

接下来互斥锁会根据上下文计算当前互斥锁最新的状态：

- 如果 `mutex` 当前处于正常模式，将 `new` 的第一位即锁位设置为 1；
- 如果 `mutex` 当前已经被加锁或处于饥饿模式，则当前 `goroutine` 进入等待队列；
- 如果 `mutex` 当前处于饥饿模式，而且 `mutex` 已被加锁，则将 `new` 的第三位即饥饿模式位设置为 1。
- 如果 `goroutine` 已经设置为唤醒状态, 需要清除 `new state` 的唤醒标记

具体源码片段如下：

```go
        // new 复制 state的当前状态， 用来设置新的状态
        // old 是锁当前的状态
        new := old

        // 如果old state状态不是饥饿状态, new state 设置锁， 尝试通过CAS获取锁,
        // 如果old state状态是饥饿状态, 则不设置new state的锁，因为饥饿状态下锁直接转给等待队列的第一个。
        if old&mutexStarving == 0 {
            // 非饥饿模式下，可以抢锁，将 new 的第一位设置为1，即加锁
            new |= mutexLocked
        }

        // 当 mutex 处于加锁状态或饥饿状态的时候，新到来的 goroutine 进入等待队列，将等待队列的等待者的数量加1。
        // old & 0101 != 0，那么 old 的第一位和第三位至少有一个为 1，即 mutex 已加锁或处于饥饿模式。
        if old&(mutexLocked|mutexStarving) != 0 {
            new += 1 << mutexWaiterShift
        }

        // 如果当前goroutine已经处于饥饿状态， 并且old state的已被加锁,
        // 将new state的状态标记为饥饿状态, 将锁转变为饥饿状态。
        // 但如果当前 mutex 未加锁，则不需要切换，Unlock 操作希望饥饿模式存在等待者
        if starving && old&mutexLocked != 0 {
            new |= mutexStarving
        }

        // 如果说当前goroutine是被唤醒状态，我们需要reset这个状态，
        // 因为goroutine要么是拿到锁了，要么是进入sleep了。
        if awoke {
            // new设置为非唤醒状态
            new &^= mutexWoken
        }
```



4. **通过CAS来尝试设置互斥锁的状态**；

`m.lockSlow()` 剩下代码片段如下：

```go
        // 调用 CAS 更新 state 状态
        if atomic.CompareAndSwapInt32(&m.state, old, new) {
            // 如果说old状态不是饥饿状态也不是被获取状态
            // 那么代表当前goroutine已经通过CAS成功获取了锁
            // (能进入这个代码块表示状态已改变，也就是说状态是从空闲到被获取)
            if old&(mutexLocked|mutexStarving) == 0 {
                // 成功上锁
                break // locked the mutex with CAS
            }
            // 如果之前已经等待过了，那么就要放到队列头
            queueLifo := waitStartTime != 0
            // 如果说之前没有等待过，就初始化设置现在的等待时间
            if waitStartTime == 0 {
                waitStartTime = runtime_nanotime()
            }
            // 既然获取锁失败了，就使用sleep原语来阻塞当前goroutine
            // 通过信号量来排队获取锁
            // 如果是新来的goroutine，就放到队列尾部
            // 如果是被唤醒的等待锁的goroutine，就放到队列头部
            runtime_SemacquireMutex(&m.sema, queueLifo, 1)
            
            // 这里sleep完了，被唤醒

            // 如果当前 goroutine 等待时间超过starvationThresholdNs，mutex 进入饥饿模式
            starving = starving || runtime_nanotime()-waitStartTime > starvationThresholdNs
            
            // 再次获取一下锁现在的状态
            old = m.state
            // 如果说锁现在是饥饿状态，就代表现在锁是被释放的状态，当前goroutine是被信号量所唤醒的，
            // 也就是说，锁被直接交给了当前goroutine
            if old&mutexStarving != 0 {
                // 如果说当前锁的状态是被唤醒状态或者被获取状态，或者说等待的队列为空
                // 那么state是一个非法状态，因为当前状态肯定应该有等待的队列，锁也一定是被释放状态且未唤醒
                if old&(mutexLocked|mutexWoken) != 0 || old>>mutexWaiterShift == 0 {
                    throw("sync: inconsistent mutex state")
                }
                // 当前的goroutine获得了锁，那么就把等待队列-1
                delta := int32(mutexLocked - 1<<mutexWaiterShift)
                // 如果不是饥饿模式或只剩一个等待者了，退出饥饿模式
                if !starving || old>>mutexWaiterShift == 1 {
                    // 把状态设置为正常
                    delta -= mutexStarving
                }
                // 设置新state, 因为已经获得了锁，退出、返回
                atomic.AddInt32(&m.state, delta)
                break
            }
            // 如果锁不是饥饿模式，就把当前的goroutine设为被唤醒，并且重置iter(重置spin)
            awoke = true
            iter = 0
        } else {
            // 如果CAS不成功，重新获取锁的state, 从for循环开始处重新开始
            old = m.state
        }
```

   

计算了新的互斥锁状态之后，就会使用 `CAS` 函数 `atomic.CompareAndSwapInt32` 更新该状态，但是如果通过 `CAS` 没有获得锁，则会调用 [sync.runtime_SemacquireMutex](https://sourcegraph.com/github.com/golang/go/-/blob/src/runtime/sema.go#L69:20) 使用信号量保证资源不会被两个 Goroutine 获取，[sync.runtime_SemacquireMutex](https://sourcegraph.com/github.com/golang/go/-/blob/src/runtime/sema.go#L69:20) 的源码如下，可以尝试一下解读：

```go
//go:linkname sync_runtime_SemacquireMutex sync.runtime_SemacquireMutex
func sync_runtime_SemacquireMutex(addr *uint32, lifo bool, skipframes int) {
	semacquire1(addr, lifo, semaBlockProfile|semaMutexProfile, skipframes)
}
```

[semacquire1](https://sourcegraph.com/github.com/golang/go/-/blob/src/runtime/sema.go#L98:5) 会在方法中不断调用尝试获取锁并休眠当前 `Goroutine` 等待信号量的释放，一旦当前 `Goroutine` 可以获取信号量，它就会立刻返回，其源码片段如下：

```go
func semacquire1(addr *uint32, lifo bool, profile semaProfileFlags, skipframes int) {
    gp := getg()
    if gp != gp.m.curg {
        throw("semacquire not on the G stack")
    }

    // Easy case.
    if cansemacquire(addr) {
        return
    }
    // 整体流程注释如下
    // Harder case:
    //	increment waiter count
    //	try cansemacquire one more time, return if succeeded
    //	enqueue itself as a waiter
    //	sleep
    //	(waiter descriptor is dequeued by signaler)
    s := acquireSudog()
    root := semroot(addr)
    t0 := int64(0)
    s.releasetime = 0
    s.acquiretime = 0
    s.ticket = 0
    if profile&semaBlockProfile != 0 && blockprofilerate > 0 {
        t0 = cputicks()
        s.releasetime = -1
    }
    if profile&semaMutexProfile != 0 && mutexprofilerate > 0 {
        if t0 == 0 {
            t0 = cputicks()
        }
        s.acquiretime = t0
    }
    // 注意这个死循环
    for {
        lockWithRank(&root.lock, lockRankRoot)
        // Add ourselves to nwait to disable "easy case" in semrelease.
        atomic.Xadd(&root.nwait, 1)
        // 通过 cansemacquire 检查以避免错过唤醒。 
        if cansemacquire(addr) {
            atomic.Xadd(&root.nwait, -1)
            unlock(&root.lock)
            break
        }
        // Any semrelease after the cansemacquire knows we're waiting
        // (we set nwait above), so go to sleep.
        root.queue(addr, s, lifo)
        goparkunlock(&root.lock, waitReasonSemacquire, traceEvGoBlockSync, 4+skipframes)
        if s.ticket != 0 || cansemacquire(addr) {
            break
        }
    }
    if s.releasetime > 0 {
        blockevent(s.releasetime-t0, 3+skipframes)
    }
    releaseSudog(s)
}
```



通过上面进行 `sleep` 之后，然后被唤醒，会接着继续执行剩下的代码，剩下会重新获得当前的模式，然后进行判断：

- 在正常模式下，这段代码会设置唤醒和饥饿标记、重置迭代次数并重新执行获取锁的循环；
- 在饥饿模式下，当前 Goroutine 会获得互斥锁，如果等待队列中只存在当前 Goroutine，互斥锁还会从饥饿模式中退出；



具体 [sync.Mutex](https://sourcegraph.com/github.com/golang/go@master/-/blob/src/sync/mutex.go#L13) 源码及注释如下：

```go
func (m *Mutex) Lock() {
    // Fast path: grab unlocked mutex.
    // 查看 state 是否为0，如果是则表示可以加锁，将其状态转换为1，当前 goroutine 加锁成功，函数返回
    if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
        if race.Enabled {
            race.Acquire(unsafe.Pointer(m))
        }
        return
    }
    // Slow path (outlined so that the fast path can be inlined)
    m.lockSlow()
}

func (m *Mutex) lockSlow() {
    var waitStartTime int64 // 用来存当前goroutine等待的时间
    starving := false // 用来存当前goroutine是否饥饿
    awoke := false // 用来存当前goroutine是否已唤醒
    iter := 0 // 用来存当前goroutine的循环次数
    old := m.state // 复制一下当前锁的状态
    for {
        // 进入到这个循环的，有两种角色goroutine，一种是新来的goroutine。另一种是被唤醒的goroutine；
        // 所以它们可能在这个地方再一起竞争锁，如果新来的goroutine抢成功了，那另一个只能再阻塞着等待，
        // 但超过1ms后，锁会转换成饥饿模式。在这个模式下，所有新来的goroutine必须排在队伍的后面，没有抢锁资格。也即：
        // 如果是饥饿情况之下，就不要自旋了，因为锁会直接交给队列头部的goroutine
		// 如果锁是被获取状态，并且满足自旋条件（canSpin见后文分析），那么就自旋等锁
        if old&(mutexLocked|mutexStarving) == mutexLocked && runtime_canSpin(iter) {
            // 通过了上面的检测，这时进行自旋是有意义的。
            // 通过把 mutexWoken 标识为 true，以通知 Unlock 方法就先不要叫醒其它阻塞着的 goroutine 了
            if !awoke && old&mutexWoken == 0 && old>>mutexWaiterShift != 0 &&
            atomic.CompareAndSwapInt32(&m.state, old, old|mutexWoken) {
                awoke = true
            }
            // 将当前 goroutine 标识为唤醒状态后，执行自旋操作，计数器加一，将当前状态记录到 old，继续循环等待
            runtime_doSpin()
            iter++
            old = m.state
            continue
        }
        // new 复制 state的当前状态，用来设置新的状态
        // old 是锁当前的状态
        new := old

        // 如果old state状态不是饥饿状态, new state 设置锁， 尝试通过CAS获取锁,
        // 如果old state状态是饥饿状态, 则不设置new state的锁，因为饥饿状态下锁直接转给等待队列的第一个。
        if old&mutexStarving == 0 {
            // 非饥饿模式下，可以抢锁，将 new 的第一位设置为1，即加锁
            new |= mutexLocked
        }

        // 当 mutex 处于加锁状态或饥饿状态的时候，新到来的 goroutine 进入等待队列，将等待队列的等待者的数量加1。
        // old & 0101 != 0，那么 old 的第一位和第三位至少有一个为 1，即 mutex 已加锁或处于饥饿模式。
        if old&(mutexLocked|mutexStarving) != 0 {
            new += 1 << mutexWaiterShift
        }

        // 如果当前goroutine已经处于饥饿状态， 并且old state的已被加锁,
        // 将new state的状态标记为饥饿状态, 将锁转变为饥饿状态。
        // 但如果当前 mutex 未加锁，则不需要切换，Unlock 操作希望饥饿模式存在等待者
        if starving && old&mutexLocked != 0 {
            new |= mutexStarving
        }

        // 如果说当前goroutine是被唤醒状态，我们需要reset这个状态，
        // 因为goroutine要么是拿到锁了，要么是进入sleep了。
        if awoke {
            // new设置为非唤醒状态
            new &^= mutexWoken
        }
        // 调用 CAS 更新 state 状态
        if atomic.CompareAndSwapInt32(&m.state, old, new) {
            // 如果说old状态不是饥饿状态也不是被获取状态
            // 那么代表当前goroutine已经通过CAS成功获取了锁
            // (能进入这个代码块表示状态已改变，也就是说状态是从空闲到被获取)
            if old&(mutexLocked|mutexStarving) == 0 {
                // 成功上锁
                break // locked the mutex with CAS
            }
            // 如果之前已经等待过了，那么就要放到队列头
            queueLifo := waitStartTime != 0
            // 如果说之前没有等待过，就初始化设置现在的等待时间
            if waitStartTime == 0 {
                waitStartTime = runtime_nanotime()
            }
            // 既然获取锁失败了，就使用sleep原语来阻塞当前goroutine
            // 通过信号量来排队获取锁
            // 如果是新来的goroutine，就放到队列尾部
            // 如果是被唤醒的等待锁的goroutine，就放到队列头部
            runtime_SemacquireMutex(&m.sema, queueLifo, 1)

            // 这里sleep完了，被唤醒

            // 如果当前 goroutine 等待时间超过starvationThresholdNs，mutex 进入饥饿模式
            starving = starving || runtime_nanotime()-waitStartTime > starvationThresholdNs

            // 再次获取一下锁现在的状态
            old = m.state
            // 如果说锁现在是饥饿状态，就代表现在锁是被释放的状态，当前goroutine是被信号量所唤醒的，
            // 也就是说，锁被直接交给了当前goroutine
            if old&mutexStarving != 0 {
                // 如果说当前锁的状态是被唤醒状态或者被获取状态，或者说等待的队列为空
                // 那么state是一个非法状态，因为当前状态肯定应该有等待的队列，锁也一定是被释放状态且未唤醒
                if old&(mutexLocked|mutexWoken) != 0 || old>>mutexWaiterShift == 0 {
                    throw("sync: inconsistent mutex state")
                }
                // 当前的goroutine获得了锁，那么就把等待队列-1
                delta := int32(mutexLocked - 1<<mutexWaiterShift)
                // 如果不是饥饿模式或只剩一个等待者了，退出饥饿模式
                if !starving || old>>mutexWaiterShift == 1 {
                    // 把状态设置为正常
                    delta -= mutexStarving
                }
                // 设置新state, 因为已经获得了锁，退出、返回
                atomic.AddInt32(&m.state, delta)
                break
            }
            // 如果锁不是饥饿模式，就把当前的goroutine设为被唤醒，并且重置iter(重置spin)
            awoke = true
            iter = 0
        } else {
            // 如果CAS不成功，重新获取锁的state, 从for循环开始处重新开始
            old = m.state
        }

        if race.Enabled {
            race.Acquire(unsafe.Pointer(m))
        }
    }
```





## 解锁

[sync.Mutex.Unlock]([`sync.Mutex.Unlock`](https://github.com/golang/go/blob/71239b4f491698397149868c88d2c851de2cd49b/src/sync/mutex.go#L179-L192)) 代码比较少，总的来说，该过程会先使用 `AddInt32` 函数快速解锁，这时会发生下面的两种情况：

- 如果该函数返回的新状态等于 0，当前 Goroutine 就成功解锁了互斥锁；
- 如果该函数返回的新状态不等于 0，这段代码会调用 `sync.Mutex.unlockSlow` 方法开始慢速解锁：

其源码片段如下：

```go
func (m *Mutex) Unlock() {
    if race.Enabled {
        _ = m.state
        race.Release(unsafe.Pointer(m))
    }

	// mutex 的 state 减去1，加锁状态 -> 未加锁，并保存到 new
    new := atomic.AddInt32(&m.state, -mutexLocked)
    if new != 0 {
        // Outlined slow path to allow inlining the fast path.
        // To hide unlockSlow during tracing we skip one extra frame when tracing GoUnblock.
        m.unlockSlow(new)
    }
}
```



而 [sync.Mutex.unlockSlow](https://sourcegraph.com/github.com/golang/go@master/-/blob/src/sync/mutex.go#L194:17) 方法<u>**首先会校验锁状态的合法性**</u> — 如果当前互斥锁已经被解锁过了就会直接抛出异常 `sync: unlock of unlocked mutex` 中止当前程序。

在正常情况下会根据当前互斥锁的状态，分别处理正常模式和饥饿模式下的互斥锁，具体看以下源码及注释

```go
func (m *Mutex) unlockSlow(new int32) {
    // 如果state不是处于锁的状态, 那么就是Unlock根本没有加锁的mutex, panic
    if (new+mutexLocked)&mutexLocked == 0 {
        throw("sync: unlock of unlocked mutex")
    }
    // 非饥饿模式，也即正常模式
    if new&mutexStarving == 0 {
        old := new
        for {
            // 没有被阻塞的goroutine。直接返回
            // 有阻塞的goroutine，但处于woken模式，直接返回
            // 有阻塞的goroutine，但被上锁了，可能发生在此for循环内，第一次CAS不成功。因为CAS前可能被新的goroutine抢到锁。直接返回
            // 有阻塞的goroutine，但锁处于饥饿模式，可能发生在被阻塞的goroutine不是被唤醒调度的，而是被正常调度运行的。直接返回
            if old>>mutexWaiterShift == 0 || old&(mutexLocked|mutexWoken|mutexStarving) != 0 {
                return
            }
            // 等待者数量减 1，并将唤醒位改成 1
            new = (old - 1<<mutexWaiterShift) | mutexWoken
            // 设置新的state, 这里通过信号量会唤醒一个阻塞的goroutine去获取锁.
            if atomic.CompareAndSwapInt32(&m.state, old, new) {
                // 唤醒一个阻塞的 goroutine，但不是唤醒第一个等待者
                runtime_Semrelease(&m.sema, false, 1)
                return
            }
            old = m.state
        }
    } else {
        // 饥饿模式：将 mutex 所有权传递给下个等待者。
        // 注意：mutexLocked 没有设置，等待者将在被唤醒后设置它。
        // 在此期间，如果有新的 goroutine来请求锁， 因为 mutex 处于饥饿状态，mutex还是被认为处于锁状态，
        // 新来的 goroutine 不会把锁抢过去.
        runtime_Semrelease(&m.sema, true, 1)
    }
}
```

---

# 3.小结



互斥锁的加锁过程比较复杂，它涉及自旋、信号量以及调度等概念，加锁流程小结如下：

- 如果互斥锁处于初始化状态，就会直接通过置位 `mutexLocked` 加锁；
- 如果互斥锁处于 `mutexLocked` 并且在普通模式下工作，就会进入自旋，执行 30 次 `PAUSE` 指令消耗 CPU 时间等待锁的释放；
- 如果当前 Goroutine 等待锁的时间超过了 1ms，互斥锁就会切换到饥饿模式；
- 互斥锁在正常情况下会通过 [`sync.runtime_SemacquireMutex`](https://github.com/golang/go/blob/a4c579e8f7c8129b2c27779f206ebd2c9b393633/src/runtime/sema.go#L70-L72) 函数将尝试获取锁的 Goroutine 切换至休眠状态，等待锁的持有者唤醒当前 Goroutine；
- 如果当前 Goroutine 是互斥锁上的最后一个等待的协程或者等待的时间小于 1ms，当前 Goroutine 会将互斥锁切换回正常模式；

而解锁流程较为简单，小结如下：

- 当互斥锁已经被解锁时，那么调用 [`sync.Mutex.Unlock`](https://github.com/golang/go/blob/71239b4f491698397149868c88d2c851de2cd49b/src/sync/mutex.go#L179-L192) 会直接抛出异常；
- 当互斥锁处于饥饿模式时，会直接将锁的所有权交给队列中的下一个等待者，等待者会负责设置 `mutexLocked` 标志位；
- 当互斥锁处于普通模式时，如果没有 Goroutine 等待锁的释放或者已经有被唤醒的 Goroutine 获得了锁，就会直接返回；在其他情况下会通过 [`sync.runtime_Semrelease`](https://github.com/golang/go/blob/a4c579e8f7c8129b2c27779f206ebd2c9b393633/src/runtime/sema.go#L65-L67) 唤醒对应的 Goroutine；



---

# 他山之石

- [Go 语言设计与实现](https://draveness.me/golang/)
- [sync.mutex 源代码分析](https://colobu.com/2018/12/18/dive-into-sync-mutex/)
- [源码剖析 golang 中 sync.Mutex](https://purewhite.io/2019/03/28/golang-mutex-source/)







