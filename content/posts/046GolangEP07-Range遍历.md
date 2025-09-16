+++
date = '2025-07-20T10:16:01+08:00'

title = 'GolangEP07-Range遍历(上)'
+++

## 

## Value copy

> In GO,everything we assign is a copy.

变量的值到底是什么？

- 值类型，`string,array,struct`,是数据本身；
- 引用类型，`slice,map,channel,pointer,function`,是数据的引用——但仍是值拷贝，只是这个值中带着地址。

所以本质上二者都是值拷贝。

值拷贝的意义：

- 值传递意味着数据可以被分配到**栈**上，函数返回时，栈上的数据立即销毁，无需 GC；
- **指针数据分配在堆**上，由 GC 管理
- 要践行*通过通信共享内存，而不是共享内存来通信*，传递值的副本可以**避免数据竞争**。

遍历一个很大的 slice 时，除了直接使用索引以外，还可以将 slice 变为指针类型，避免拷贝开销。（但是遍历指针类型对 CPU 来说效率低）

## 循环控制器的求值时机

> The provided expression is evaluated only once,before the beginning of the loop.

只在循环开始前求一次值，之后会使用其副本进行迭代。

```go
s := []int{0, 1, 2}
for range s { // 不需要索引和值，只要按照长度循环相应的次数即可。
    s = append(s, 10)
}
```

- 先进行拷贝，得到的`s_copy`作为循环控制器
- 后续操作，s 在不断变化，但是**循环控制器始终不变**，一直是开始的那个拷贝副本，所以只会执行三次。

为什么要有一个始终不变的循环控制器呢？

为了**保证循环的可预测性，防止意外的无限循环,减少对应的性能开销**。

举一个无限循环的错误示范：

```go
s := []int{0,1,2}
for i := 0; i < len(s); i++ {
    s = append(s, 10)
}
```

这会导致无限循环，因为 s 的长度一直在增长。

channels 和 Arrays 也有类似的行为，具体见下方所示：

### Channels

同样的，在处理多个 channel 时也会遇到类似问题。

```go
ch1 := make(chan int, 3)
go func() {
    ch1 <- 0
    ch1 <- 1
    ch1 <- 2
    close(ch1)
}()

ch2 := make(chan int, 3)
go func() {
    ch2 <- 10
    ch2 <- 11
    ch2 <- 12
    close(ch2)
}()

ch := ch1
for v := range ch {
    fmt.Println(v)
    ch = ch2
}
```

这里打印出的 v 一直是 ch1，因为只在开始时确定一次。

一开始就确定了一个循环控制器，这样以后在 range 中修改也不会修改定住的这个控制器。

日常开发中通常会采用 `select` 语句来处理多个动态数据。

### Array

```go
a := [3]int{0,1,2}
for i, v := range a {
    a[2] = 10
    if i == 2 {
        fmt.Println(v)
    }
}
```

这段代码最终会输出 2 还是 10 呢？答案是 2，循环中的 v 来自于拷贝，而只有`a[i]`访问的是原始的数组。

与最开始的 slice 类似，都有拷贝作为循环控制器:

- 一开始求值得到一个拷贝副本值`a_copy`，接下来的循环都将在`a_copy`上进行
- i,v 都来源于`a_copy`
- 明确使用变量名`a[2]`才是使用原数组

### pointer

当结构体较大时，建议使用指针存入 map，否则会带来两个严重的后果：

- 只是操作副本
- 大的结构体在 map 存取过程中会涉及到数据拷贝

所以使用指针 `mapPointer map[string]*LargeStruct`,可以直接对结构体内的属性值进行操作`mapPointer[id].foo = "bar"`

在 Go1.22 之前，以下代码会出现循环变量的复用问题:

```go
for _, customer := range customers {
    s.m[customer.ID] = &customer
}
```

- 循环变量的复用:`customer`只有一个实例，每次只操作这一个实例。
- 循环赋值
- 获取被复用的地址

如何解决这个问题：

- 每次循环创建一个新的局部变量
- 通过索引获取原始切片的元素地址，而不是循环变量的地址

在 Go1.22 之后，修改了 for 循环的语义,让每次循环背后都做了一次局部变量声明操作：

```go
customer := customers[i]
```

这样就不会出现循环变量的复用问题了。

在 Java 中 `for-each` 循环每次都是一个新的变量。

## 对比 Java

可以看到，首先 Go 中一切皆值拷贝，而 Java，对象变量存的就是引用，**拷贝的是引用的值——地址**。

在传参过程中，Java 传递的是引用的值（地址），这样不会带来拷贝的开销。