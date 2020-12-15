# 1.sync.RWMutex 介绍

[Sync.RWMutex](https://sourcegraph.com/github.com/golang/go@master/-/blob/src/sync/rwmutex.go#L5) 官方给出的定义是读/写互斥锁，锁可以由任意数量的读者或单个写者持有。 `RWMutex`的零值是未锁定的互斥锁。

> A RWMutex is a reader/writer mutual exclusion lock. The lock can be held by an arbitrary number of readers or a single writer. The zero value for a RWMutex is an unlocked mutex.

其包含了四种方法，如下所示：

```go
func (rw *RWMutex) Lock         // 提供写锁加锁操作
func (rw *RWMutex) RLock        // 提供读锁加锁操作
func (rw *RWMutex) RUnlock      // 提供读锁解锁操作
func (rw *RWMutex) Unlock       // 提供写锁解锁操作
```

这四个方法的源码解析在下面可以深入理解，可以大概了解到在 `RWMutex` 读操作可并发重入，而写操作是互斥的，读写锁通常用互斥锁、条件变量、信号量实现。

以下有四个场景可以先了解一下，方便理解该读/写互斥锁的源码：

**（1）：写操作如何防止写操作？**

- **写操作依赖互斥锁阻止其他的写操作**：读写锁包含一个互斥锁`(Mutex)`，写锁定必须要先获取该互斥锁，如果互斥锁已被协程A获取（或者协程A在阻塞等待读结束），意味着协程 A 获取了互斥锁，那么协程B只能阻塞等待该互斥锁。

**（2）：写操作是如何阻止读操作的？**

- 首先了解一下最多可支持的并发读者：`RWMutex.readerCount`是个整型值，用于表示读者数量，不考虑写操作的情况下，每次读锁定将该值+1，每次解除读锁定将该值 -1，所以`readerCount`取值为`[0, N]`，N为读者个数，实际上**最大可支持`2^30`个并发读者**。
- **写操作将`readerCount`变成负值来阻止读操作的**：当写锁定进行时，会先将`readerCount`减去`2^30`，从而`readerCount`变成了负值，此时再有读锁定到来时检测到`readerCount`为负值，便知道有写操作在进行，只好阻塞等待。而真实的读操作个数并不会丢失，只需要将`readerCount`加上`2^30`即可获得。

**（3）：读操作是如何阻止写操作的？**

- **读操作通过`readerCount`来将来阻止写操作的**：读锁定会先将`RWMutext.readerCount`加1，此时写操作到来时发现读者数量不为0，会阻塞等待所有读操作结束。

**（4）：为什么写锁定不会被饿死？**

- 首先解释一下为什么可能有饿死的情况发生：写操作要等待读操作结束后才可以获得锁，写操作等待期间可能还有新的读操作持续到来，如果写操作等待所有读操作结束，很可能被饿死。
- **可以通过`RWMutex.readerWait`解决这个问题**写操作到来时，会把`RWMutex.readerCount`值拷贝`到RWMutex.readerWait`中，用于标记排在写操作前面的读者个数。前面的读操作结束后，除了会递减`RWMutex.readerCount`，还会递减`RWMutex.readerWait`值，当`RWMutex.readerWait`值变为0时唤醒写操作。**所以，写操作就相当于把一段连续的读操作划分成两部分，前面的读操作结束后唤醒写操作，写操作结束后唤醒后面的读操作**。

注：以下源码解析基于 `go 1.15`

---

# 2.源码解析

## 结构体及其常量定义

`RWMutex` 包含了五个字段及一个常量，其结构体定义如下：

```go
type RWMutex struct {
	w           Mutex  // 互斥锁
	writerSem   uint32 // 写操作等待读操作完成的信号量
	readerSem   uint32 // 读操作等待写操作完成的信号量
	readerCount int32  // 读锁计数器
	readerWait  int32  // 获取写锁时当前需要等待的读锁释放数量
}

// 最大只支持 1 << 30 个读锁
const rwmutexMaxReaders = 1 << 30
```



关于信号量（`semaphore`）的问题，这里做一个补充：

这是一个由 `Edsger Dijkstra` 提出的数据结构，解决很多关于同步的问题时，它提供了两种操作的整数：

- 获取（`acquire`，又称 `wait`、`decrement` 或者 P）
- 释放（`release`，又称 `signal`、increment 或者 V）

获取操作把信号量减一，如果减一的结果是非负数，那么线程可以继续执行。如果结果是负数，那么线程将会被阻塞，除非有其它线程把信号量增加回非负数，该线程才有可能恢复运行）。

释放操作把信号量加一，如果当前有被阻塞的线程，那么它们其中一个会被唤醒，恢复执行。

Go 语言的运行时提供了 `runtime_SemacquireMutex` 和 `runtime_Semrelease` 函数，像 sync.RWMutex 这些对象的实现会用到这两个函数。



## 写锁加锁 Lock()

当资源的使用者想要获取写锁时，需要调用 `sync.RWMutex.Lock` 方法：

```go
func (rw *RWMutex) Lock() {
    // 竞态检测
    if race.Enabled {
        _ = rw.w.state
        race.Disable()
    }
    // 1.使用 Mutex 锁，解决与其他写者的竞争
    rw.w.Lock()
    
    // 2.判断当前是否存在读锁：先通过原子操作改变readerCount（readerCount-rwmutexMaxReaders），
    // 使其变为负数，告诉 RUnLock 当前存在写锁等待；
    // 然后再加回 rwmutexMaxReaders 并赋给r，若r仍然不为0, 代表当前还有读锁
    r := atomic.AddInt32(&rw.readerCount, -rwmutexMaxReaders) + rwmutexMaxReaders
    
    // 3.如果仍然有其他 Goroutine 持有互斥锁的读锁（r != 0）
    // 会先将 readerCount 的值加到 readerWait中，防止源源不断的读者进来导致写锁饿死，
    // 然后该 Goroutine 会调用 sync.runtime_SemacquireMutex 进入休眠状态，
    // 并等待所有读锁所有者执行结束后释放 writerSem 信号量将当前协程唤醒。
    if r != 0 && atomic.AddInt32(&rw.readerWait, r) != 0 {
        // 阻塞写锁
        runtime_SemacquireMutex(&rw.writerSem, false, 0)
    }
    // 竞态检测
    if race.Enabled {
        race.Enable()
        race.Acquire(unsafe.Pointer(&rw.readerSem))
        race.Acquire(unsafe.Pointer(&rw.writerSem))
    }
}
```

其主要的内容：

1. 使用 `sync.Mutex` 中的互斥锁 `sync.Mutex.Lock()` 先解决与其他写者的竞争问题；
2. 判断当前是否存在读锁：先通过原子操作改变`readerCount（readerCount-rwmutexMaxReaders）`，使其变为负数，告诉 RUnLock 当前存在写锁等待，然后再加回`rwmutexMaxReaders` 并赋给r，若r仍然不为0，代表当前还有读锁
3. 判断是否还有其他 `Goroutine` 持有`RWMutex`互斥锁的读锁（r != 0），如果有则会先将当前的 `readerCount` 的数量加到 `readerWait`中，从而防止后面源源不断的读者请求读锁，从而进来导致写锁饿死的情况发生，然后该 `Goroutine` 会调用 `sync.runtime_SemacquireMutex` 进入休眠状态，并等待当前持有读锁的 `Goroutine` 结束后释放 `writerSem` 信号量将当前 `Goroutine` 唤醒。

注意：<u>**上面的 `readerWait` 就是用以解决写锁饥饿的问题**</u>，代码与注释已经解释得很清楚了，不再阐述了。



## 写锁释放 UnLock()

写锁的释放会调用 `sync.RWMutex.Unlock` 方法：

```go
func (rw *RWMutex) Unlock() {
    // 竞态检测
	if race.Enabled {
		_ = rw.w.state
		race.Release(unsafe.Pointer(&rw.readerSem))
		race.Disable()
	}
	// 1.释放读锁：通过调用 atomic.AddInt32 函数将 readerCount 加上 rwmutexMaxReaders 从而变回正数；；
	r := atomic.AddInt32(&rw.readerCount, rwmutexMaxReaders)
    // 若超过读锁的最大限制, 触发panic
	if r >= rwmutexMaxReaders {
		race.Enable()
		throw("sync: Unlock of unlocked RWMutex")
	}
	// 2.通过 for 循环触发所有由于获取读锁而陷入等待的 Goroutine，也即解除阻塞的读锁(若有)
	for i := 0; i < int(r); i++ {
		runtime_Semrelease(&rw.readerSem, false, 0)
	}
	// 3.调用 sync.Mutex.Unlock 方法释放写互斥锁
	rw.w.Unlock()
    
    // 是否开启竞态检测
	if race.Enabled {
		race.Enable()
	}
}
```



若抛开竞态检测的代码的话，其实这个写锁释放的代码也是挺简洁的，主要完成了以下几件事情：

1. 释放读锁：通过调用 `atomic.AddInt32` 函数将 `readerCount` 加上 `rwmutexMaxReaders` 从而变回正数；
2. 通过 for 循环触发所有由于获取读锁而陷入等待的 Goroutine，也即解除阻塞的读锁(若有)；
3. 调用 `sync.Mutex.Unlock()` 方法释放写互斥锁。



## 读锁加锁 RLock()

读锁操作调用 `RLock()`

```go
func (rw *RWMutex) RLock() {
    // 是否开启检测race
    if race.Enabled {
        _ = rw.w.state
        race.Disable()
    }
    //这里分两种情况:
	// 1.此时无写锁 (readerCount + 1) > 0,那么可以上读锁, 并且readerCount原子加1(读锁可重入[只要匹配了释放次数就行])。
	// 2.此时有写锁 (readerCount + 1) < 0,所以通过readerSem读信号量, 使读操作睡眠等待。
    if atomic.AddInt32(&rw.readerCount, 1) < 0 {
        // 当前有个写锁, 读操作需要阻塞等待写锁释放；
        // 其实做的事情是将 goroutine 排到G队列的后面,挂起 goroutine
        runtime_SemacquireMutex(&rw.readerSem, false, 0)
    }
    // 是否开启检测race
    if race.Enabled {
        race.Enable()
        race.Acquire(unsafe.Pointer(&rw.readerSem))
    }
}
```



该读锁加锁整体流程较为简单，分两种情况进行判断：

1. 此时无写锁 `(readerCount + 1) > 0`（注意，在写锁是加锁那里，我们对readerCount 进行了`readerCount-rwmutexMaxReaders`处理），那么可以上读锁, 并且`readerCount`原子加1（读锁可重入[只要匹配了释放次数就行]）；
2. 此时有写锁 `(readerCount + 1) < 0,`所以通过`readerSem`读信号量, 使读操作睡眠等待；

## 读锁释放 RUnlock()

`RUnLock()` 方法对读锁进行解锁：

```go
func (rw *RWMutex) RUnlock() {
    // 竞态检测
    if race.Enabled {
        _ = rw.w.state
        race.ReleaseMerge(unsafe.Pointer(&rw.writerSem))
        race.Disable()
    }
    // 写锁等待状态，检查当前是否可以进行获取；
    // 首先将 readerCount 减1并赋予r，然后分两种情况判断
    //  1.若r大于等于0，读锁直接解锁成功，直接结束本次操作；
    //  2.若r小于0，有一个正在执行的写操作，在这时会调用sync.RWMutex.rUnlockSlow 方法；
    if r := atomic.AddInt32(&rw.readerCount, -1); r < 0 {
        // Outlined slow-path to allow the fast-path to be inlined
        rw.rUnlockSlow(r)
    }
    if race.Enabled {
        race.Enable()
    }
}

func (rw *RWMutex) rUnlockSlow(r int32) {
    // r + 1 == 0 表示本来就没读锁, 直接执行RUnlock()
    // r + 1 == -rwmutexMaxReaders 表示执行Lock()再执行RUnlock()
	if r+1 == 0 || r+1 == -rwmutexMaxReaders {
		race.Enable()
		throw("sync: RUnlock of unlocked RWMutex")
	}
	// 如果当前有写锁等待，则减少一个readerWait的数目
    if atomic.AddInt32(&rw.readerWait, -1) == 0 {
		// 写锁前的最后一个读锁唤醒写锁执行
        runtime_Semrelease(&rw.writerSem, false, 1)
	}
}
```



关于读锁释放的详细如下：

1. 首先`readerCount` 减1，然后进行两种情况的判断：
   1. 若 r 大于等于0，读锁直接解锁成功，直接结束本次操作；
   2. 若 r 小于0， 有一个正在执行的写操作，在这时会调用`sync.RWMutex.rUnlockSlow` 方法；
2. 然后倘若上面判断 r 小于0，则进入 `rUnlockSlow()` 慢解锁，先进行一个判断，若有以下两种情况发生：`r + 1 == 0`表示直接执行`RUnlock()` ，`r + 1 == -rwmutexMaxReaders`表示执行 `Lock()` 再执行 `RUnlock()`，这两种情况都会进行报错。
3. 如果没有上述两种情况发生，则`sync.RWMutex.rUnlockSlow` 会减少获取锁的写操作等待的读操作数 `readerWait` ，并在所有读操作都被释放之后触发写操作的信号量 `writerSem`，该信号量被触发时，调度器就会唤醒尝试获取写锁的 `Goroutine`。

---

# 3.小结

之前的`sync.Mutex` 分析起来简直是复杂得不得了，但是这个读写互斥锁 `sync.RWMutex` 看起来功能非常复杂源码却相对较为简单，因为它建立在 `sync.Mutex` 上，复杂的工作都被`sync.Mutex` 完成了，所以其简单很多了，总的来说读锁和写锁的关系：

1. 读锁不能阻塞读锁，引入`readerCount`实现；
2. 读锁需要阻塞写锁，直到当前的所有读锁都释放，引入`readerSem`实现（同时防止写锁饥饿）；
3. 写锁需要阻塞读锁，直到所有写锁都释放，引入`writerSem`实现；
4. 写锁需要阻塞写锁，引入`Metux`实现；

使用好读写互斥锁可以得到更细粒度的控制，能够在读操作远远多于写操作的时候提升性能，所以对于究竟是使用`sync.Mutex` 还是 `sync.RWMutex` 就要看具体的业务分析了。

----

# 他山之石

- [sync.RWMutex：Solving readers-writers problems](https://medium.com/golangspec/sync-rwmutex-ca6c6c3208a0)





















