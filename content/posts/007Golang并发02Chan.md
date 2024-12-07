+++
date = '2024-12-05T15:40:56+08:00'
title = 'Golang源码小试牛刀:Golang并发02_Chan'
+++

# 前言

# 使用示例

## 创建channel

```go
ch1 := make(chan int)
ch2 := make(chan int, 2)
```

底层实际上调用的是[makechan](#makechan函数)方法

## 发送数据到channel

```go
ch <- 1
```

底层实际调用的是chansend1,而chansend1最终也是调用[chansend](#chansend函数),将block参数设置为true——当前发送操作是**阻塞的**

## 从channel中读取数据

```go
i <- ch
i, ok <- ch
```

底层实际调用的是chanrecv1和chanrecv2，最终都去调用了chanrecv。

# 源码


## 常量

## hchan结构体

```go
type hchan struct {
	qcount   uint           // total data in the queue
	dataqsiz uint           // size of the circular queue
	buf      unsafe.Pointer // points to an array of dataqsiz elements
	elemsize uint16
	synctest bool // true if created in a synctest bubble
	closed   uint32
	timer    *timer // timer feeding this chan
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

* qcount:通道中存储的数据元素总数
* dataqsiz:环形队列大小
* buf:执行环形队列的内存缓冲区，存放实际数据——环形缓存区域，本质上是一个带有头尾指针的固定长度的数组
* sendx,recvx:发收操作的队列位置
* recvq,sendq:等待队列

可以发现，hchan使用环形队列表示缓冲区并且采用lock确保并发访问的安全性。

## waitq队列

```go
type waitq struct {
	first *sudog
	last  *sudog
}
```

管理等待发送或接受的goroutine队列——存储阻塞在通道上的 goroutine 信息。
可以实现快速插入last和删除first，如果通道阻塞，goroutine会封装为sudog并挂入waitq队列中。

### sudog结构体

```go
type sudog struct {
	g *g

	next *sudog
	prev *sudog
	elem unsafe.Pointer // data element (may point to stack)

	acquiretime int64
	releasetime int64
	ticket      uint32

	isSelect bool

	success bool

	waiters uint16

	parent   *sudog // semaRoot binary tree
	waitlink *sudog // g.waiting list or semaRoot
	waittail *sudog // semaRoot
	c        *hchan // channel
}
```

用于管理Goroutine在通道操作时的阻塞与唤醒操作,**sudog 充当连接 Goroutine 和通道的桥梁**

* c *hchan:表示当前阻塞的通道
* elem:指向与通道操作相关的元素，例如保存发送数据
* next,prev:waitq的双向链表中的sudog节点
* waitlink,waittail:用于 semaRoot（信号量队列）的队列链接。
* parent:用于semaRoot的二叉树链接
* g:当前阻塞的Goroutine对象
* isSelect:当前sudog是否参与了select操作，可能存在唤醒竞争问题
* success:当前通道操作是否完成
* acquiretime,releasetime:sudog被加入的时间和被唤醒的时间

可以发现有一些是跟信号量有关的操作semaRoot，是同步操作，与本节关系不太大。

至此我们可以得知，**hchan管理通道的核心状态，waitq双向链表作为等待队列连接多个阻塞的Goroutine,这些Goroutine由sudog记录，专门用于在 Goroutine 被挂起时保存其状态，并将其与特定资源（例如通道、锁或其他同步原语）关联起来。**

* 作为发送方，检查缓冲区是否未满，如果没满就写入缓冲；如果满了就阻塞到waitq中；如果由接收方等待，则唤醒接收方
* 作为接收方，检查缓冲区是否非空，如果非空直接读取数据，如果为空阻塞waitq；如果有发送者，唤醒发送方。

### enqueue

```go
func (q *waitq) enqueue(sgp *sudog) {
	sgp.next = nil
	x := q.last
	if x == nil {
		sgp.prev = nil
		q.first = sgp
		q.last = sgp
		return
	}
	sgp.prev = x
	x.next = sgp
	q.last = sgp
}
```

将一个sudog（阻塞的goroutine结构）添加到waitq队列中——通道在运行时实现发送和接受阻塞的核心机制：

* 获取当前队列尾节点，如果为空意味着队列为空，将sudog作为第一个节点同时也是最后一个节点
* 如果队列非空，改变prev为当前尾节点并将sudog作为新的队尾。

### dequeue

```go
func (q *waitq) dequeue() *sudog {
	for {
		sgp := q.first
		if sgp == nil {
			return nil
		}
		y := sgp.next
		if y == nil {
			q.first = nil
			q.last = nil
		} else {
			y.prev = nil
			q.first = y
			sgp.next = nil // mark as removed (see dequeueSudoG)
		}
		if sgp.isSelect {
			if !sgp.g.selectDone.CompareAndSwap(0, 1) {
				// We lost the race to wake this goroutine.
				continue
			}
		}

		return sgp
	}
}
```

出队操作，返回被移除的sudog

* 如果队列头为空，意味着队列为空
* 如果只有一个结点，移除后队列设置为空
* 如果有多个节点，移除头结点并更新队列（这里的核心逻辑都是找到第二个节点）
* 如果该goroutine是由于`select`放入队列，则需要检查是否有竞争条件——看是否有其他的Goroutine先一步唤醒了该sgp，如果是则不能将其移除，继续尝试出队下一个结点。

## makechan()函数

```go
func makechan(t *chantype, size int) *hchan {
	elem := t.Elem

	// compiler checks this but be safe.
	if elem.Size_ >= 1<<16 {
		throw("makechan: invalid channel element type")
	}
    // 检查通道结构体是否正确对齐，保证运行时安全性
	if hchanSize%maxAlign != 0 || elem.Align_ > maxAlign {
		throw("makechan: bad alignment")
	}

	mem, overflow := math.MulUintptr(elem.Size_, uintptr(size))
	if overflow || mem > maxAlloc-hchanSize || size < 0 {
		panic(plainError("makechan: size out of range"))
	}

	var c *hchan
	switch {
	case mem == 0:
		// Queue or element size is zero.
		c = (*hchan)(mallocgc(hchanSize, nil, true))
		c.buf = c.raceaddr()
	case !elem.Pointers():
		c = (*hchan)(mallocgc(hchanSize+mem, nil, true))
		c.buf = add(unsafe.Pointer(c), hchanSize)
	default:
		// Elements contain pointers.
		c = new(hchan)
		c.buf = mallocgc(mem, elem, true)
	}

	c.elemsize = uint16(elem.Size_)
	c.elemtype = elem
	c.dataqsiz = uint(size)
	if getg().syncGroup != nil {
		c.synctest = true
	}
	lockInit(&c.lock, lockRankHchan)

	if debugChan {
		print("makechan: chan=", c, "; elemsize=", elem.Size_, "; dataqsiz=", size, "\n")
	}
	return c
}
```

**根据给定的通道类型和大小，初始化一个通道的底层结构体hchan并分配必要的内存.**

* 1 验证通道中的元素类型和通道对齐
* 2 计算缓冲区大小:元素大小 × 容量，检查是否溢出或超出允许的最大分配值
* 3 分配内存并初始化:
  * 如果缓冲区为0只分配hchan的结构体内存；
  * 如果存储元素不包含指针，将hchan和缓冲区内存一次性分配在一起；
  * 如果存储元素包含指针，分开分配，保证垃圾回收器能够正确追踪指针（详情见垃圾回收博文）
* 4 初始化通道，设置元素大小，类型，通道容量，初始化锁
* 5 最终返回通道hchan指针

## send()函数

```go
func send(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {
	if c.synctest && sg.g.syncGroup != getg().syncGroup {
		unlockf()
		panic(plainError("send on synctest channel from outside bubble"))
	}
	if raceenabled {
		if c.dataqsiz == 0 {
			racesync(c, sg)
		} else {
			// Pretend we go through the buffer, even though
			// we copy directly. Note that we need to increment
			// the head/tail locations only when raceenabled.
			racenotify(c, c.recvx, nil)
			racenotify(c, c.recvx, sg)
			c.recvx++
			if c.recvx == c.dataqsiz {
				c.recvx = 0
			}
			c.sendx = c.recvx // c.sendx = (c.sendx+1) % c.dataqsiz
		}
	}
	if sg.elem != nil {
		sendDirect(c.elemtype, sg, ep)
		sg.elem = nil
	}
	gp := sg.g
	unlockf()
	gp.param = unsafe.Pointer(sg)
	sg.success = true
	if sg.releasetime != 0 {
		sg.releasetime = cputicks()
	}
	goready(gp, skip+1)
}
```

处理数据从发送端到接收端的实际传输

* 1 进行调试和同步测试，确保通道的发送和接收在相同的同步组中
* 2 触发数据竞争检测:如果无缓冲，调用racesync同步收发；如果有缓冲，通过`racenotify`模拟数据传输和缓冲区索引更新
* 3 调用`sendDirect`*将发送者sudog的数据直接复制到接收者Gorontine的sudog结构中*
* 4 标记发送成功，记录性能追踪的释放时间，调用`goready`唤醒接收者Goroutine

## chansend()函数

```go
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
	if c == nil {
		if !block {
			return false
		}
		gopark(nil, nil, waitReasonChanSendNilChan, traceBlockForever, 2)
		throw("unreachable")
	}

	if debugChan {
		print("chansend: chan=", c, "\n")
	}

	if raceenabled {
		racereadpc(c.raceaddr(), callerpc, abi.FuncPCABIInternal(chansend))
	}

	if c.synctest && getg().syncGroup == nil {
		panic(plainError("send on synctest channel from outside bubble"))
	}

	if !block && c.closed == 0 && full(c) {
		return false
	}

	var t0 int64
	if blockprofilerate > 0 {
		t0 = cputicks()
	}

	lock(&c.lock)

	if c.closed != 0 {
		unlock(&c.lock)
		panic(plainError("send on closed channel"))
	}

	if sg := c.recvq.dequeue(); sg != nil {
		// Found a waiting receiver. We pass the value we want to send
		// directly to the receiver, bypassing the channel buffer (if any).
		send(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true
	}

	if c.qcount < c.dataqsiz {
		// Space is available in the channel buffer. Enqueue the element to send.
		qp := chanbuf(c, c.sendx)
		if raceenabled {
			racenotify(c, c.sendx, nil)
		}
		typedmemmove(c.elemtype, qp, ep)
		c.sendx++
		if c.sendx == c.dataqsiz {
			c.sendx = 0
		}
		c.qcount++
		unlock(&c.lock)
		return true
	}

	if !block {
		unlock(&c.lock)
		return false
	}

	// Block on the channel. Some receiver will complete our operation for us.
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
	c.sendq.enqueue(mysg)

	gp.parkingOnChan.Store(true)
	reason := waitReasonChanSend
	if c.synctest {
		reason = waitReasonSynctestChanSend
	}
	gopark(chanparkcommit, unsafe.Pointer(&c.lock), reason, traceBlockChanSend, 2)

	KeepAlive(ep)

	// someone woke us up.
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

本方法*向c目标通道发送数据ep，并标志是否采用阻塞方式，记录调用方的pc。返回发送成功或者失败*

* 1 通道为nil的处理
* 2 竞态检测
* 3 通道未关闭且已满
* 4 获取通道互斥锁，**如果通道已关闭立即解锁并抛出panic**
* 5 优先处理接收处理:如果有Goroutine正在等待接收，直接调用[send](#send函数)发送——**会绕过缓冲区**
* 6 如果缓冲区未满，将数据写入缓冲区
* 7 如果是阻塞模式（没有等待的接收者且缓存区已满或无缓存），那么当前Goroutine会阻塞，创建一个sudog结构体mysg记录发送者的信息，直到接收者处理当前发送操作。
* 8 如果发送者被唤醒，验证状态正常后进行资源的清理释放上面的sudog避免内存泄漏；并进行性能统计记录阻塞时间。

总结来看，一共分为三种可能的发送方式

* 同步发送（有正在等待的）
* 异步发送（写入缓存）
* 阻塞发送（放入wait队列中）

## chanrecv()函数

### 异常检查

```go
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {

	if debugChan {
		print("chanrecv: chan=", c, "\n")
	}

	if c == nil {
		if !block {
			return
		}
		gopark(nil, nil, waitReasonChanReceiveNilChan, traceBlockForever, 2)
		throw("unreachable")
	}

	if c.synctest && getg().syncGroup == nil {
		panic(plainError("receive on synctest channel from outside bubble"))
	}

	if c.timer != nil {
		c.timer.maybeRunChan()
	}

	if !block && empty(c) {
		if atomic.Load(&c.closed) == 0 {
			return
		}

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
    .......
}
```

* 如果通道为空且不阻塞，立即返回；如果通道为空且阻塞，不能从空的通道接收会直接挂起
* 同步组校验与定时器检查
* 进行快速路径检查（这里有些模糊）

### 同步接收

```go
	var t0 int64
	if blockprofilerate > 0 {
		t0 = cputicks()
	}

	lock(&c.lock)

	if c.closed != 0 {
		if c.qcount == 0 {
			if raceenabled {
				raceacquire(c.raceaddr())
			}
			unlock(&c.lock)
			if ep != nil {
				typedmemclr(c.elemtype, ep)
			}
			return true, false
		}

	} else {

		if sg := c.sendq.dequeue(); sg != nil {
			recv(c, sg, ep, func() { unlock(&c.lock) }, 3)
			return true, true
		}
	}
```

* 对通道加锁
* 如果通道关闭且缓冲区为空，如不符合接收recv请求的状态，直接返回。
* 如果sendq中有sudog等待的发送者，调用recv函数完成同步接收

### 异步接收

```go
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
```

* 如果缓冲区有数据，从缓冲区根据recvx索引读取
* 更新通道的信息（索引，缓冲区数据数量）
* 返回true，表示接收成功

### 阻塞接收

```go
	if !block {
		unlock(&c.lock)
		return false, false
	}

	gp := getg()
	mysg := acquireSudog()
	mysg.releasetime = 0
	if t0 != 0 {
		mysg.releasetime = -1
	}

	mysg.elem = ep
	mysg.waitlink = nil
	gp.waiting = mysg

	mysg.g = gp
	mysg.isSelect = false
	mysg.c = c
	gp.param = nil
	c.recvq.enqueue(mysg)
	if c.timer != nil {
		blockTimerChan(c)
	}

	gp.parkingOnChan.Store(true)
	reason := waitReasonChanReceive
	if c.synctest {
		reason = waitReasonSynctestChanReceive
	}
	gopark(chanparkcommit, unsafe.Pointer(&c.lock), reason, traceBlockChanRecv, 2)
```

* 如果sendq中没有待发送的goroutine且缓冲区为空，即上面那两个if都不满足，则进入阻塞阶段。
* 与chansend中逻辑相似，将当前goroutine休眠并创建一个新的sudog保存被挂起的goroutine当前状态。

```go
	// someone woke us up
	if mysg != gp.waiting {
		throw("G waiting list is corrupted")
	}
	if c.timer != nil {
		unblockTimerChan(c)
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
```

被调度器唤醒后完成阻塞接收，进行参数检查，解除通道的绑定并释放创建的这个sudog。

## 参考资料

1. https://github.com/golang/go/blob/master/src/runtime/chan.go#L176
2. https://juejin.cn/post/7125610595801530376#heading-4