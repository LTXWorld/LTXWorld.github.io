+++
date = '2024-12-01T12:00:57+08:00'
title = 'Golang源码小试牛刀:Golang并发01_context'
categories = ["核心技术"]
tags = ["Golang","源码","Context"]
+++

# 前言

今天心血来潮，给自己梳理梳理golang中这个十分重要的东东——context，毕竟也是在面试中考察到，但是反问自己的时候却感觉一点也说不出来什么有价值的东西，所以今天我来一次刨根究底。

![](/img/jb/coffee.webp)

# 源码解析

> Package context defines the Context type, which carries deadlines, cancellation signals, and other request-scoped values across API boundaries and between processes.

这是来自于官方的最新说明，意味着context携带终止期限，取消信号，以及跨API，进程之间通信等信息。即context设计的核心目标是

* 任务取消机制：跨API和Goroutine的任务取消信号传播机制
* 超时控制：超时或截止时间控制任务生命周期
* 数据共享：在不同函数调用间共享元数据

明确了这些，我们进入到[源码](https://cs.opensource.google/go/go/+/refs/tags/go1.23.3:src/context/context.go;l=227)中。

## Context接口及错误变量

下面这段是来自于Chatgpt的精简版（去掉了长长的英文注释），可以发现Context是一个接口，其中包含了四个方法等待实现，第二个比较特殊，返回的是一个channel。

```go
type Context interface {
    Deadline() (deadline time.Time, ok bool) // 返回任务截止时间
    Done() <-chan struct{}                  // 返回取消信号的 channel
    Err() error                             // 返回上下文取消的原因
    Value(key any) any                      // 获取上下文中的值
}
```

说到这里我们先暂停一下，提前告知整个context包中一共有多少种上下文类型,接下来就会根据这几种类型分别阐述。

* emptyCtx
* cancleCtx
* timerCtx
* valueCtx
* 以及cancleCtx下的一系列衍生体


定义完了接口，接下来是两个变量:

```go
var Canceled = errors.New("context canceled")
var DeadlineExceeded error = deadlineExceededError{}

type deadlineExceededError struct{}

func (deadlineExceededError) Error() string   { return "context deadline exceeded" }
func (deadlineExceededError) Timeout() bool   { return true }
func (deadlineExceededError) Temporary() bool { return true }
```

根据源码中的注释可以得知

* Canceld是**上下文被取消时**的错误类型，是一个简单的 error 对象，由 errors.New 创建。
* DealineExceeded表示**上下文超出截止时间**的错误类型，具体实现是 deadlineExceededError，一个实现了 error 接口的结构体，可以为这个错误类型提供更加丰富的信息。
* 下面三个实现的来自于error的接口表示这是由于超时而引起的错误，暂时性的。

看到这里，我想你已经猜出为什么要定义这两个变量了，正是能够适配context接口中的Err()方法，至于到底怎么具体实现的Err()方法我们后面再说。

## emptyCtx结构

```go
type emptyCtx struct{}

func (emptyCtx) Deadline() (deadline time.Time, ok bool) {
	return
}

func (emptyCtx) Done() <-chan struct{} {
	return nil
}

func (emptyCtx) Err() error {
	return nil
}

func (emptyCtx) Value(key any) any {
	return nil
}
```

显而易见，这是一个空的上下文实现，通常用作上下文链的根节点或者占位符。注释中也提到，**这是一个没有任何状态信息，不包含取消、截止时间、或值存储功能，不能通过取消函数手动取消的上下文。**

### BackgroundCtx & todoCtx结构

```go
type backgroundCtx struct{ emptyCtx }

func (backgroundCtx) String() string {
	return "context.Background"
}

type todoCtx struct{ emptyCtx }

func (todoCtx) String() string {
	return "context.TODO"
}
```

立马这两个接口就用到了emptyCtx,提供了最基础的上下文功能，为开发中的代码提供一个临时的上下文对象，避免为空。

* Background通常用于上下文链的根节点。
* todo通常用作占位符（见名知意嘛）

对应的还有两个方法来生成二者。
```go
func Background() Context {
	return backgroundCtx{}
}

func TODO() Context {
	return todoCtx{}
}
```


## cancelCtx:支持显示取消的结构

```go
type cancelCtx struct {
	Context

	mu       sync.Mutex            // protects following fields
	done     atomic.Value          // of chan struct{}, created lazily, closed by first cancel call
	children map[canceler]struct{} // set to nil by the first cancel call
	err      error                 // set to non-nil by the first cancel call
	cause    error                 // set to non-nil by the first cancel call
}

func (c *cancelCtx) Value(key any) any {
	if key == &cancelCtxKey {
		return c
	}
	return value(c.Context, key)
}

func (c *cancelCtx) Done() <-chan struct{} {
	d := c.done.Load()
	if d != nil {
		return d.(chan struct{})
	}
	c.mu.Lock()
	defer c.mu.Unlock()
	d = c.done.Load()
	if d == nil {
		d = make(chan struct{})
		c.done.Store(d)
	}
	return d.(chan struct{})
}

func (c *cancelCtx) Err() error {
	c.mu.Lock()
	err := c.err
	c.mu.Unlock()
	return err
}
```

哦，原来你小子嵌套了Context祖宗，你是儿子啊！然后你还多定义了什么：

* mu，很常见的保护并发访问
* done 存储一个懒加载的通道，上下文被取消时关闭此通道；这里的atomic.Value暂且不表。😂
* children,存储派生于该上下文的子上下文，形成一个上下文链（取消时通知他们）
* err，错误，通常用上面那两个错误变量
* casuse，错误原因。

后面的这几个就是实现Context接口中的方法

* value:如果键是cancelCtxKey，直接返回cancelCtx本身；否则交给父上下文处理
* Done:先从done.Load获取已有通道，如果有了，直接返回；如果没有，创建一个新通道并存储（注意加锁后又尝试加载了一次，防止其他协程创建了通道）；**相同上下文返回唯一通道**
  * 这是一种双检查锁的模式
* Err:保证线程安全地返回错误状态

接下来是此结构中最重要的cancel方法，用于取消当前上下文，并传播取消信号到所有的子上下文，整个流程大致总结如下：

* 设置err和cause表示上下文已经被取消
* 关闭Done通道以通知所有监听者
* 遍历递归取消所有子上下文（链条）
* 最后清理资源

```go
// 是否需要将当前上下文从其父上下文的 children 集合中移除，取消的错误信息，取消的原因
func (c *cancelCtx) cancel(removeFromParent bool, err, cause error) {
	if err == nil {
		panic("context: internal error: missing cancel error")
	}
	if cause == nil {
		cause = err
	}
    // 锁定状态，防止重复取消
	c.mu.Lock()
	if c.err != nil {
		c.mu.Unlock()
		return // already canceled
	}
	c.err = err
	c.cause = cause
    // 检查done通道是否已创建，如果没有就存储一个，否则调用close关闭通道
	d, _ := c.done.Load().(chan struct{})
	if d == nil {
		c.done.Store(closedchan)
	} else {
		close(d)
	}
    // 至此，通知所有子上下文关闭,递归地调用cancel方法
	for child := range c.children {
		// NOTE: acquiring the child's lock while holding parent's lock.
		child.cancel(false, err, cause)
	}
    // 清空子上下文集合
	c.children = nil
	c.mu.Unlock()

	if removeFromParent {
		removeChild(c.Context, c)
	}
}
```

### CancelFunc函数类型

```go
// A CancelFunc tells an operation to abandon its work.
// A CancelFunc does not wait for the work to stop.
// A CancelFunc may be called by multiple goroutines simultaneously.
// After the first call, subsequent calls to a CancelFunc do nothing.
type CancelFunc func()
```

是一个函数类型，表示一个无参数、无返回值的函数，并发安全，一次性触发（只有第一次调用会有效）。

之所以这里定义为函数类型，就是为了让其他需要使用到CancelFunc的方法通过**闭包**来绑定具体的逻辑。稍后我们在一些具体方法中就能看到。

###  WithCancel & WithCancelCause

```go
func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {
	c := withCancel(parent)
	return c, func() { c.cancel(true, Canceled, nil) }
}

type CancelCauseFunc func(cause error)

func WithCancelCause(parent Context) (ctx Context, cancel CancelCauseFunc) {
	c := withCancel(parent)
	return c, func(cause error) { c.cancel(true, Canceled, cause) }
}

```

1. WithCancel

* 传入父上下文，创建一个取消上下文，返回此上下文和一个取消函数
* 可以发现正如我们上面所说，利用闭包，直接返回了一个取消函数cancel来触发上下文的取消

2. WithCancelCause

* 将WithCancel中的CancelFunc换为CancelCasuseFunc，带有取消原因。
* 下面的闭包函数中可以传入一个取消原因`cause error`

```go
func withCancel(parent Context) *cancelCtx {
	if parent == nil {
		panic("cannot create context from nil parent")
	}
	c := &cancelCtx{}
	c.propagateCancel(parent, c)
	return c
}
```

这个withCancel就是上面两个方法分别调用的核心方法来创建一个cancelCtx上下文实例。

先判断父上下文是否为空（上下文链必须有一个根），**随后创建一个cancelCtx上下文实例，调用propagateCancel关联父子并监听父上下文的取消事件**。

说到这里可能有些懵逼了，那我们就来看看这个propagateCancel是怎么个事！

![](/img/ys/ye.webp)

### propagateCancel

用于将子取消逻辑与父取消逻辑联系起来形成一个链。具体代码逻辑的解释见注释。

```go
// propagateCancel arranges for child to be canceled when parent is.
// It sets the parent context of cancelCtx.
func (c *cancelCtx) propagateCancel(parent Context, child canceler) {
	c.Context = parent // parent作为基础（即上面的嵌套Context）

	done := parent.Done() // 获取parent的Done通道，记录着取消信号的channel
	if done == nil { // 如果为空，说明没有取消信号，parent永远不会取消
		return // parent is never canceled
	}

	select {
	case <-done: // ?
		// parent is already canceled
		child.cancel(false, parent.Err(), Cause(parent))
		return
	default:
	}
    // 检查parent类型是否为cancelCtx或派生类型
	if p, ok := parentCancelCtx(parent); ok {
		// parent is a *cancelCtx, or derives from one.
		p.mu.Lock()
		if p.err != nil { // 如果错误类型不为空，证明parent已被取消了
			// parent has already been canceled
			child.cancel(false, p.err, p.cause)
		} else { // 否则将child加入到parent的子集中
			if p.children == nil {
				p.children = make(map[canceler]struct{})
			}
			p.children[child] = struct{}{}
		}
		p.mu.Unlock()
		return
	}
    // 如果parent实现了AfterFunc方法，这个之后我们会说，是一种定时回调
	if a, ok := parent.(afterFuncer); ok {
		// parent implements an AfterFunc method.
		c.mu.Lock()
		stop := a.AfterFunc(func() {
			child.cancel(false, parent.Err(), Cause(parent))
		})
		c.Context = stopCtx{
			Context: parent,
			stop:    stop,
		}
		c.mu.Unlock()
		return
	}
    // 用于处理无法直接关联的上下文，goroutine同时监听parent和child的Done通道
	goroutines.Add(1)
	go func() {
		select {
		case <-parent.Done():
			child.cancel(false, parent.Err(), Cause(parent))
		case <-child.Done():
		}
	}()
}
```

所以，回到最开始的withCancel中，构建了一个取消传播链，可以调用CancelFunc来触发取消逻辑，释放资源。

### Cause

从Context中提取取消原因(cause)

```go
func Cause(c Context) error {
	if cc, ok := c.Value(&cancelCtxKey).(*cancelCtx); ok {
		cc.mu.Lock()
		defer cc.mu.Unlock()
		return cc.cause
	}
    return c.Err()
}
```

### afterFuncCtx

在cancelCtx的基础上实现回调。

```go
type afterFuncCtx struct {
	cancelCtx
	once sync.Once // 确保 f 只被执行一次
	f    func()    // 要在上下文完成时调用的回调函数
}
```

AfterFunc传入回调函数，并将afterFuncCtx关联到父上下文
```go
func AfterFunc(ctx Context, f func()) (stop func() bool) {
	a := &afterFuncCtx{
		f: f,
	}
	// 将 afterFuncCtx 作为子上下文，关联到父上下文 ctx
	a.cancelCtx.propagateCancel(ctx, a)

	// 返回的 stop 函数用于停止回调的执行
	return func() bool {
		stopped := false
		a.once.Do(func() {
			stopped = true
		})
		if stopped {
			a.cancel(true, Canceled, nil)
		}
		return stopped
	}
}
```


## timerCtx

这是cancelCtx的扩展，支持设置截止时间的上下文类型，多了timer和deadline字段。

```go
type timerCtx struct {
	cancelCtx
	timer *time.Timer // Under cancelCtx.mu.

	deadline time.Time
}
// 返回当前截止时间以及标志位
func (c *timerCtx) Deadline() (deadline time.Time, ok bool) {
	return c.deadline, true
}

func (c *timerCtx) String() string {
	return contextName(c.cancelCtx.Context) + ".WithDeadline(" +
		c.deadline.String() + " [" +
		time.Until(c.deadline).String() + "])"
}
// 调用cancelCtx的cancel实现取消，多了定时器
func (c *timerCtx) cancel(removeFromParent bool, err, cause error) {
	c.cancelCtx.cancel(false, err, cause)
	if removeFromParent {
		// Remove this timerCtx from its parent cancelCtx's children.
		removeChild(c.cancelCtx.Context, c)
	}
	c.mu.Lock()
	if c.timer != nil {
		c.timer.Stop()
		c.timer = nil
	}
	c.mu.Unlock()
}
```

### WithTimeout & WithDeadline

基于超时持续时间创建上下文；基于指定的截止时间创建上下文

```go
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc) {
	return WithDeadline(parent, time.Now().Add(timeout))
}

func WithDeadline(parent Context, d time.Time) (Context, CancelFunc) {
	return WithDeadlineCause(parent, d, nil)
}
```

可以发现最终还是调用的是WithDeadlineCause这个方法

* 检查父上下文是否有更早的截止时间
* 创建timerCtx
* 计算剩余时间
* 设置定时器
* 返回上下文和取消函数

```go
func WithDeadlineCause(parent Context, d time.Time, cause error) (Context, CancelFunc) {
	if parent == nil {
		panic("cannot create context from nil parent")
	}
	if cur, ok := parent.Deadline(); ok && cur.Before(d) {
		// The current deadline is already sooner than the new one.
		return WithCancel(parent)
	}
	c := &timerCtx{
		deadline: d,
	}
    // 关联到父上下文并计算截止时间的剩余时间
	c.cancelCtx.propagateCancel(parent, c)
	dur := time.Until(d)
	if dur <= 0 {
		c.cancel(true, DeadlineExceeded, cause) // deadline has already passed
		return c, func() { c.cancel(false, Canceled, nil) }
	}
    // 设置定时器，到时间后自动取消上下文
	c.mu.Lock()
	defer c.mu.Unlock()
	if c.err == nil {
		c.timer = time.AfterFunc(dur, func() {
			c.cancel(true, DeadlineExceeded, cause)
		})
	}
	return c, func() { c.cancel(true, Canceled, nil) }
}
```

## valueCtx

实现键值对存储功能的一种上下文类型,适合携带少量元数据。

```go
type valueCtx struct {
	Context
	key, val any
}
```

Value:如果当前上下文没有匹配的键，则调用 value 函数继续在父上下文中查找。

value:递归辅助方法，用于在上下文链中查找指定key对应的值

```go
func (c *valueCtx) String() string {
	return contextName(c.Context) + ".WithValue(" +
		stringify(c.key) + ", " +
		stringify(c.val) + ")"
}

func (c *valueCtx) Value(key any) any {
	if c.key == key {
		return c.val
	}
	return value(c.Context, key)
}

func value(c Context, key any) any {
	for {
		switch ctx := c.(type) {
            // 如果是valueCtx类型
		case *valueCtx:
			if key == ctx.key {
				return ctx.val
			}
			c = ctx.Context //向父上下文继续找
            // 如果是cancelCtx类型
		case *cancelCtx:
			if key == &cancelCtxKey {
				return c
			}
			c = ctx.Context
            // 如果是withoutCancelCtx类型
		case withoutCancelCtx:
			if key == &cancelCtxKey {
				// This implements Cause(ctx) == nil
				// when ctx is created using WithoutCancel.
				return nil
			}
			c = ctx.c
            // 如果是timerCtx类型
		case *timerCtx:
			if key == &cancelCtxKey {
				return &ctx.cancelCtx
			}
			c = ctx.Context
            // 如果是backgroundCtx，todoCtx类型即根上下文
		case backgroundCtx, todoCtx:
			return nil
		default:
			return c.Value(key)
		}
	}
}
```

### withValue方法

```go
func WithValue(parent Context, key, val any) Context {
	if parent == nil {
		panic("cannot create context from nil parent")
	}
	if key == nil {
		panic("nil key")
	}
	if !reflectlite.TypeOf(key).Comparable() {
		panic("key is not comparable")
	}
	return &valueCtx{parent, key, val}
}
```

# 常用的Go Context 类型对比

| **类型**         | **功能描述**                                                                 | **支持取消** | **支持超时/截止时间** | **支持键值对存储** |
|------------------|-----------------------------------------------------------------------------|-------------|-----------------------|-------------------|
| **`emptyCtx`**   | 根上下文类型。用于创建 `context.Background()` 和 `context.TODO()`，无任何功能。 | 否          | 否                    | 否                |
| **`cancelCtx`**  | 支持取消功能，通过 `WithCancel` 或 `WithCancelCause` 创建，可以传播取消信号。  | 是          | 否                    | 否                |
| **`timerCtx`**   | 基于 `cancelCtx`，支持取消和超时，通过 `WithTimeout` 和 `WithDeadline` 创建。 | 是          | 是                    | 否                |
| **`valueCtx`**   | 支持键值对存储，通过 `WithValue` 创建，通常嵌套在其他类型上下文中使用。       | 否          | 否                    | 是                |

---


# 平时是如何使用的

这里举例说明context以及其中的常见方法在代码中是如何使用的。

## WithTimeout&WithDeadline

在这段代码中，`context.WithTimeout`起到了控制数据库操作是否超时的作用

```go
func (m MovieModel) Insert(movie *Movie) error {
	// 插入一条新记录的SQL语句，并返回信息（Postgresql专有)
	query := `
			INSERT INTO movies (title, year, runtime, genres)
			VALUES ($1, $2, $3, $4)
			RETURNING id, created_at, version`

	// 创建一个代表着占位符的movie中的属性切片
	args := []interface{}{movie.Title, movie.Year, movie.Runtime, pq.Array(movie.Genres)}

	// Create a context with a 3-second timeout
	// 如果数据库操作在3s内没有完成，操作自动取消，返回超时错误
	ctx, cancle := context.WithTimeout(context.Background(), 3*time.Second)
	defer cancle()

	// 使用QueryRowContext方法执行,利用传入的ctx进行SQL查询，并使用Scan方法将返回值注入到movie的三个属性中
	return m.DB.QueryRowContext(ctx, query, args...).Scan(&movie.ID, &movie.CreatedAt, &movie.Version)
}
```

