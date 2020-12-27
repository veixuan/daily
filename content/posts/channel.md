---
# 常用定义
title: "channel"
date: 2020-12-27T20:44:26+08:00
lastmod: 2020-12-27T20:44:26+08:00
tags: ["golang"] 
categories: ["golang"]             
author: "Neo"          
# 用户自定义
# 你可以选择 关闭(false) 或者 打开(true) 以下选项
comment: false   # 关闭评论
toc: true       # 关闭文章目录
reward: true	 # 关闭打赏
---

`golang` 中`channel`的基本使用

<!--more-->

## channel

>   `channel`是golang中的一等公民，和`goroutine` 一起搭配使用，大大降低了`golang`中并发编程的难度.
>
>   主要用于`goroutine `之间进行消息、事件等传播和通信 

## 基本语法 

| 语法                        | 说明                                    |
| --------------------------- | --------------------------------------- |
| `c1 := make(chan bool)`     | 创建一个无缓冲的bool类型`channel`       |
| `c2 := make(chan<- string)` | 创建一个单向`channel`，只能用于发送数据 |
| `c3 := make(<-chan string)` | 创建一个单向`channel`，只能用于接收数据 |
| `c4 := make(chan int, 3)`   | 创建一个缓冲长度是3的`channel`          |

说明: 双向的channel 可以隐式转换成单向`channel`，反之不行  

## `channel` 操作方式

| 操作                 | 说明                  |
| -------------------- | --------------------- |
| `close(channel)`     | 关闭一个`channel`     |
| `channel <- "hello"` | 往channel中写入hello  |
| `str := <-channel`   | 从channel中取出一个值 |
| `cap(channel)`       | 获取channel的容量     |
| `len(channel)`       | 获取channel的长度     |

 说明: 这五种操作都是同步的，也就是线程安全的 

## 操作说明 

| 操作                     | 关闭     | 发送数据     | 接收数据     |
| ------------------------ | -------- | ------------ | ------------ |
| `nil 零值channel`        | `panic`  | `block`      | `block`      |
| 非零值已关闭的`channel`  | `panic`  | `panic`      | 永不阻塞     |
| 非零值未关闭的``channel` | 成功关闭 | 阻塞或者成功 | 阻塞或者成功 |

## 底层结构 

我们通过 `make(chan string,100)` 这种方式创建一个`channel` , 本质是调用`makechan` 方法创建一个`hchan` 类型的指针 

```go
_ = make(chan string)
// go tool compile -S main.go

makechan(t *chantype, size int) *hchan
```

### `hchan`

```go
// /usr/local/go/src/runtime/chan.go:32 
// go version go1.16beta1

type hchan struct {
  // chan 里面的元素数量
	qcount   uint           // total data in the queue
  // chan 底层的循环队列长度 
  // 只针对 缓冲类型的channel 
	dataqsiz uint           // size of the circular queue
  // 指向底层循环队列的指针 
	buf      unsafe.Pointer // points to an array of dataqsiz elements
  // chan 里面的元素大小 
	elemsize uint16
  // chan 是否关闭的状态 
	closed   uint32
  // chan 里面的元素类型 
	elemtype *_type // element type
  // 已发送的元素在循环队列的索引 
	sendx    uint   // send index
  // 已接收的元素在循环队列的索引 
	recvx    uint   // receive index
  // 等待接收的goroutine 队列 
	recvq    waitq  // list of recv waiters
  // 等待发送的goroutine 队列 
	sendq    waitq  // list of send waiters

	// lock protects all fields in hchan, as well as several
	// fields in sudogs blocked on this channel.
	//
	// Do not change another G's status while holding this lock
	// (in particular, do not ready a G), as this can deadlock
	// with stack shrinking.
  // 锁 用于保证channel中的元素读写都是线程安全的 
	lock mutex
}

// waitq 是一个goroutine的双向链表 
type waitq struct {
	first *sudog  // sudog 是对一个goroutine的封装 
	last  *sudog
}
```

结构图示 

![channel.png](https://i.loli.net/2020/12/27/v1FdlwXNEDq6Y9I.png)

## 创建一个`channel`的过程

源码 

```go
const (
	maxAlign  = 8  
	hchanSize = unsafe.Sizeof(hchan{}) + uintptr(-int(unsafe.Sizeof(hchan{}))&(maxAlign-1))
	debugChan = false
)

func makechan(t *chantype, size int) *hchan {
	elem := t.elem

	// 省略中间的校验代码

  // 两个数相乘 返回结果 并且判断是否越界 
  // 计算要分配的内存大小 
	mem, overflow := math.MulUintptr(elem.size, uintptr(size))
	if overflow || mem > maxAlloc-hchanSize || size < 0 {
		panic(plainError("makechan: size out of range"))
	}
	var c *hchan
	switch {
	case mem == 0:
    // 要分配的内存==0 有两种可能性
    // 1. 无缓冲类型的channel  size == 0 
    // 2. 非缓冲类型的channel 但是 elem.size == 0 那么肯定是 struct{}{} 
    // 只需要分配 hchansize 的内存大小就可以 
		// Queue or element size is zero.
		c = (*hchan)(mallocgc(hchanSize, nil, true))
		// Race detector uses this location for synchronization.
    // buf 无用 直接指向channel的起始位置
		c.buf = c.raceaddr()
	case elem.ptrdata == 0:
    // 元素类型不包含指针 
    // 只做一次内存分配 分配的大小是 hchansize + mem(count * size )
		// Elements do not contain pointers.
		// Allocate hchan and buf in one call.
		c = (*hchan)(mallocgc(hchanSize+mem, nil, true))
    // 计算buf指向的位置 
		c.buf = add(unsafe.Pointer(c), hchanSize)
	default:
		// Elements contain pointers.
    // 包含指针 
    // 两次内存分配 
		c = new(hchan)
		c.buf = mallocgc(mem, elem, true)
	}

	c.elemsize = uint16(elem.size)
	c.elemtype = elem
	c.dataqsiz = uint(size)
  // 锁初始化 
	lockInit(&c.lock, lockRankHchan)

	if debugChan {
		print("makechan: chan=", c, "; elemsize=", elem.size, "; dataqsiz=", size, "\n")
	}
  // 返回 指针类型 
	return c
}

// 计算是否越界 
func MulUintptr(a, b uintptr) (uintptr, bool) {
  // 判断 a、b 是否小于 2^32
	if a|b < 1<<(4*sys.PtrSize) || a == 0 {
		return a * b, false
	}
  // 越界兜底 
	overflow := b > MaxUintptr/a
	return a * b, overflow
}
```

>   总结:
>
>   -   要分配的内存是0，那么做一次内存分配，分配的长度是channel对齐后的长度，这种有两种可能 
>   -   `无缓冲性channel`，因为`size == 0``
>   -   ``struct{}类型的有缓冲channel`，因为`elem.size == 0` 
>   -   要分配的内存长度不是0，但是存储的元素是非指针类型，那么做一次内存分配 
>   -   要分配的内存长度不是0，存储的原始是指针类型，做两次内存分配 

## 发送数据的过程 

```go

strChan <- "hello"
// 通过汇编可以发现，调用的是runtime的.chansend1 方法 
runtime.chansend1 
func chansend1(c *hchan, elem unsafe.Pointer) {
	chansend(c, elem, true, getcallerpc())
}
```

send函数逻辑如下 

```go
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
  // channel 是nil 
	if c == nil {
    // 非block  直接返回false 
		if !block {
			return false
		}
    // 挂起当前goroutine 
		gopark(nil, nil, waitReasonChanSendNilChan, traceEvGoStop, 2)
		throw("unreachable")
	}

	if debugChan {
		print("chansend: chan=", c, "\n")
	}

	if raceenabled {
		racereadpc(c.raceaddr(), callerpc, funcPC(chansend))
	}

	// Fast path: check for failed non-blocking operation without acquiring the lock.
	//
	// After observing that the channel is not closed, we observe that the channel is
	// not ready for sending. Each of these observations is a single word-sized read
	// (first c.closed and second full()).
	// Because a closed channel cannot transition from 'ready for sending' to
	// 'not ready for sending', even if the channel is closed between the two observations,
	// they imply a moment between the two when the channel was both not yet closed
	// and not ready for sending. We behave as if we observed the channel at that moment,
	// and report that the send cannot proceed.
	//
	// It is okay if the reads are reordered here: if we observe that the channel is not
	// ready for sending and then observe that it is not closed, that implies that the
	// channel wasn't closed during the first observation. However, nothing here
	// guarantees forward progress. We rely on the side effects of lock release in
	// chanrecv() and closechan() to update this thread's view of c.closed and full().
  
  // 未关闭 并且channel 已经满了  直接返回false 
	if !block && c.closed == 0 && full(c) {
		return false
	}

	var t0 int64
	if blockprofilerate > 0 {
		t0 = cputicks()
	}

  // 加锁 
	lock(&c.lock)

  // 已经关闭了 直接panic 
	if c.closed != 0 {
		unlock(&c.lock)
		panic(plainError("send on closed channel"))
	}

  // 从等待接收的go routine 队列中取出一个等待的线程 
	if sg := c.recvq.dequeue(); sg != nil {
		// Found a waiting receiver. We pass the value we want to send
		// directly to the receiver, bypassing the channel buffer (if any).
    // 如果存在等待的线程，直接发送 
    // 1. 拷贝当前的数据buf到等待的线程的buf 
    // 2. 拉起等待的goroutine 
		send(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true
	}

  // 缓冲性队列未满 
	if c.qcount < c.dataqsiz {
		// Space is available in the channel buffer. Enqueue the element to send.
		qp := chanbuf(c, c.sendx)
		if raceenabled {
			racenotify(c, c.sendx, nil)
		}
    // eq: 等待入队的元素的地址 
    // qp: 循环队列中下一个可用的index 对应的地址 
    // 将eq的数据复制到qp 
		typedmemmove(c.elemtype, qp, ep)
    // send index ++ 
		c.sendx++
    // index 已满，重置(环形)
		if c.sendx == c.dataqsiz {
			c.sendx = 0
		}
    // 队列中的元素大小+1
		c.qcount++
    // 解锁 
		unlock(&c.lock)
    // 返回成功 
		return true
	}

  // 非block的情况  
  // 直接解锁 返回失败 
	if !block {
		unlock(&c.lock)
		return false
	}

	// Block on the channel. Some receiver will complete our operation for us.
  // channel 已经满了  发送方会被block住 
  // 获取当前线程 
	gp := getg()
	mysg := acquireSudog()
	mysg.releasetime = 0
	if t0 != 0 {
		mysg.releasetime = -1
	}
	// No stack splits between assigning elem and enqueuing mysg
	// on gp.waiting where copystack can find it.
	mysg.elem = ep
	mysg.waitlink = nil
	mysg.g = gp
	mysg.isSelect = false
	mysg.c = c
	gp.waiting = mysg
	gp.param = nil
  
  // 当前的线程进入等待发送的队列 
	c.sendq.enqueue(mysg)
	// Signal to anyone trying to shrink our stack that we're about
	// to park on a channel. The window between when this G's status
	// changes and when we set gp.activeStackChans is not safe for
	// stack shrinking.
	atomic.Store8(&gp.parkingOnChan, 1)
  // 挂起当前线程 
	gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanSend, traceEvGoBlockSend, 2)
	// Ensure the value being sent is kept alive until the
	// receiver copies it out. The sudog has a pointer to the
	// stack object, but sudogs aren't considered as roots of the
	// stack tracer.
	KeepAlive(ep)

	// someone woke us up.
  // 队列中空闲 被唤醒 
	if mysg != gp.waiting {
		throw("G waiting list is corrupted")
	}
	gp.waiting = nil
	gp.activeStackChans = false
	closed := !mysg.success
	gp.param = nil
	if mysg.releasetime > 0 {
		blockevent(mysg.releasetime-t0, 2)
	}
	mysg.c = nil
	releaseSudog(mysg)
	if closed {
		if c.closed == 0 {
			throw("chansend: spurious wakeup")
		}
		panic(plainError("send on closed channel"))
	}
	return true
}
```

> 总结：
>
> - channel是nil，非block直接返回失败，block时直接永远挂起
> - 非block，但是channel 已经满了，直接返回失败
> - 加锁 
> - 从当前channel的等待队列上 获取一个等待的goroutine，如果存在，直接将当前数据拷贝到等待的线程中，返回true 
> - 没有等待的线程，队列未满，直接放置在队列的环形数组中 
> - block并且队列已满，挂起当前线程，等待被唤醒 

## 数据接收 

```go
// runtime.chanrecv1
func chanrecv1(c *hchan, elem unsafe.Pointer) {
	chanrecv(c, elem, true)
}

func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
	// 删除debug的内容 
  // channel 是nil 
	if c == nil {
    // 非block 直接return 
		if !block {
			return
		}
    // 永远挂起当前线程 
		gopark(nil, nil, waitReasonChanReceiveNilChan, traceEvGoStop, 2)
		throw("unreachable")
	}

	// Fast path: check for failed non-blocking operation without acquiring the lock.
  // 非block 并且是空 
  // 1. 可能已经被关闭 
	if !block && empty(c) {
		// After observing that the channel is not ready for receiving, we observe whether the
		// channel is closed.
		//
		// Reordering of these checks could lead to incorrect behavior when racing with a close.
		// For example, if the channel was open and not empty, was closed, and then drained,
		// reordered reads could incorrectly indicate "open and empty". To prevent reordering,
		// we use atomic loads for both checks, and rely on emptying and closing to happen in
		// separate critical sections under the same lock.  This assumption fails when closing
		// an unbuffered channel with a blocked send, but that is an error condition anyway.
		if atomic.Load(&c.closed) == 0 {
			// Because a channel cannot be reopened, the later observation of the channel
			// being not closed implies that it was also not closed at the moment of the
			// first observation. We behave as if we observed the channel at that moment
			// and report that the receive cannot proceed.
			return
		}
		// The channel is irreversibly closed. Re-check whether the channel has any pending data
		// to receive, which could have arrived between the empty and closed checks above.
		// Sequential consistency is also required here, when racing with such a send.
		if empty(c) {
			// The channel is irreversibly closed and empty.
			if raceenabled {
				raceacquire(c.raceaddr())
			}
			if ep != nil {
				typedmemclr(c.elemtype, ep)
			}
			return true, false
		}
	}

	var t0 int64
	if blockprofilerate > 0 {
		t0 = cputicks()
	}

  // 加锁 
	lock(&c.lock)

  // channel 已关闭，并且环形队列里没有元素
	if c.closed != 0 && c.qcount == 0 {
		if raceenabled {
			raceacquire(c.raceaddr())
		}
    // 从一个已关闭的 channel 执行接收操作，且未忽略返回值
    // 那么接收的值将是一个该类型的零值
		unlock(&c.lock)
		if ep != nil {
      // typedmemclr 根据类型清理相应地址的内存
			typedmemclr(c.elemtype, ep)
		}
    // 从一个已关闭的 channel 接收，selected 会返回true
		return true, false
	}

  
  // 等待发送的队列不为空，优先从等待发送的队列上获取 
  // 1. buf 已经满了 
  // 2. 无缓冲型
	if sg := c.sendq.dequeue(); sg != nil {
		// Found a waiting sender. If buffer is size 0, receive value
		// directly from sender. Otherwise, receive from head of queue
		// and add sender's value to the tail of the queue (both map to
		// the same buffer slot because the queue is full).
		recv(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true, true
	}

  // 缓冲型队列 
  // 从队列中获取一个 
	if c.qcount > 0 {
		// Receive directly from queue
		qp := chanbuf(c, c.recvx)
		if raceenabled {
			racenotify(c, c.recvx, nil)
		}
		if ep != nil {
			typedmemmove(c.elemtype, ep, qp)
		}
		typedmemclr(c.elemtype, qp)
		c.recvx++
		if c.recvx == c.dataqsiz {
			c.recvx = 0
		}
		c.qcount--
		unlock(&c.lock)
		return true, true
	}

	if !block {
		unlock(&c.lock)
		return false, false
	}

  // 没有数据 block
	// no sender available: block on this channel.
	gp := getg()
	mysg := acquireSudog()
	mysg.releasetime = 0
	if t0 != 0 {
		mysg.releasetime = -1
	}
	// No stack splits between assigning elem and enqueuing mysg
	// on gp.waiting where copystack can find it.
	mysg.elem = ep
	mysg.waitlink = nil
	gp.waiting = mysg
	mysg.g = gp
	mysg.isSelect = false
	mysg.c = c
	gp.param = nil
  // 将当前线程放入带等待接收的队列 
	c.recvq.enqueue(mysg)
	// Signal to anyone trying to shrink our stack that we're about
	// to park on a channel. The window between when this G's status
	// changes and when we set gp.activeStackChans is not safe for
	// stack shrinking.
	atomic.Store8(&gp.parkingOnChan, 1)
	gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanReceive, traceEvGoBlockRecv, 2)

	// someone woke us up
  // 被唤醒 
	if mysg != gp.waiting {
		throw("G waiting list is corrupted")
	}
	gp.waiting = nil
	gp.activeStackChans = false
	if mysg.releasetime > 0 {
		blockevent(mysg.releasetime-t0, 2)
	}
	success := mysg.success
	gp.param = nil
	mysg.c = nil
	releaseSudog(mysg)
	return true, success
}
```

>    总结：
>
>   -   channel是nil，非block直接返回失败，block时会永远挂起当前线程 
>   -   channel 已关闭，并且度列中没有元素 返回失败 
>   -   等待发送队列不为空，从等待队列上获取一个等待的线程，直接进行数据copy 
>   -   buf未满，直接放置buf中
>   -   buf已满，挂起当前线程，等待被唤醒 

## close chan

```go
func closechan(c *hchan) {
  // nil 直接panic 
	if c == nil {
		panic(plainError("close of nil channel"))
	}

  // 加锁 
	lock(&c.lock)
  // 已经被关闭 直接panic 
	if c.closed != 0 {
		unlock(&c.lock)
		panic(plainError("close of closed channel"))
	}

	if raceenabled {
		callerpc := getcallerpc()
		racewritepc(c.raceaddr(), callerpc, funcPC(closechan))
		racerelease(c.raceaddr())
	}

  // 标记已经close 
	c.closed = 1

	var glist gList

	// release all readers
  // 释放所有的等待消费的线程 
	for {
		sg := c.recvq.dequeue()
		if sg == nil {
			break
		}
		if sg.elem != nil {
			typedmemclr(c.elemtype, sg.elem)
			sg.elem = nil
		}
		if sg.releasetime != 0 {
			sg.releasetime = cputicks()
		}
		gp := sg.g
		gp.param = unsafe.Pointer(sg)
		sg.success = false
		if raceenabled {
			raceacquireg(gp, c.raceaddr())
		}
		glist.push(gp)
	}

	// release all writers (they will panic)
  // 释放所有的等待发送的线程 
	for {
		sg := c.sendq.dequeue()
		if sg == nil {
			break
		}
    // 会panic 
		sg.elem = nil
		if sg.releasetime != 0 {
			sg.releasetime = cputicks()
		}
		gp := sg.g
		gp.param = unsafe.Pointer(sg)
		sg.success = false
		if raceenabled {
			raceacquireg(gp, c.raceaddr())
		}
		glist.push(gp)
	}
  // 解锁 
	unlock(&c.lock)

	// Ready all Gs now that we've dropped the channel lock.
	for !glist.empty() {
		gp := glist.pop()
		gp.schedlink = 0
		goready(gp, 3)
	}
}
```

>   总结:
>
>   -   关闭一个nil channel 会panic 
>   -   关闭一个已经关闭的channel 会panic 
>   -   所有等待消费的线程不会panic 
>   -   所有等待发送的线程会panic 