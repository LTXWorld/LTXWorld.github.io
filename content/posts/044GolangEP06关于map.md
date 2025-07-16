+++
date = '2025-07-16T09:52:36+08:00'
title = 'GolangEP06:关于map-上'
categories = ["核心技术"]
tags = ["Golang","源码","Map"]
+++

## 引子

在做两数之和时，操作 map 遇到了问题，可以看我下面的代码。

```go
func twoSum(nums []int, target int) []int {
	sMap := make(map[int]int)
	for i, x := range nums {
		sMap[x] = i
	}
	var ans []int
	for i, y := range nums {
		// 如果另一个值在 map 中存在的话即下标大于等于 0;且要保证不能使用两次相同的元素
		p := sMap[target-y]
		if p >= 0 && p != i {
			ans = append(ans, i)
			ans = append(ans, p)
			break
		}
	}
	return ans
}
```

像极了一个 golang 新手的操作，特别是在判断 key 是否存在于哈希表中时，用值（下标）是否大于 0 来判断。（当然前面的哈希表的定义也有问题）

而 go 中判断某 key 是否存在于哈希表中使用的是如下语法

```go
if val, ok := map[key]; ok {
    ...
}
```

用这个 ok 来判断是否存在。

还有一点是，就拿我上面错误的判断方法来说，我进行 debug 时发现：为什么不在 map 中的 key 它们的值都返回的是 0 呢？0在题目中的含义可是下标 0啊。

这就又牵扯到了 go 中 map 的原理了，直接访问一个不存在的 key 会返回该值类型的零值，对于 int 来说就是 0 了。

## Map底层原理

使用 go 时，一开始刷到的八股就是 map 类型线程不安全，多个 goroutine 不能同时安全读写同一个 map。

但是 why？老规矩，先从其底层结构看起吧。

### 底层结构

```go
type hmap struct {
    count     int     // map 中存储的元素个数
    B         uint8   // 当前 bucket 的数量为 2^B 个
    buckets   unsafe.Pointer // 指向 bucket 数组
    oldbuckets unsafe.Pointer // 指向旧的 buckets（在扩容时使用）
    ...
}
```

看了这些结构内容后，很明显，bucket 数组很重要(其实就是 hash table),这是我们剖析 map 的核心。

下面这张图可以粗略地展示 bucket数组以及其指向的的 bucket空间。

![1](/img/golangPic/bucket.png)

可以看到 bucket数组中存放的是指针，指向底层的 bucket 空间，而 bucket 空间由四部分组成

- `topHash`存储哈希值的高八位,用于快速查找——不会立即比较key，而是先根据 topHash 进行快速比较，因为**字符串比较十分昂贵**
- `keys`，要求类型必须是可比较的，有哪些不能比较呢：切片，函数等类型
- `values` 没有特殊要求
- `*overflows`指向溢出桶

### 初始化

默认一个 bucket 最多可以存储 8 个键值对。与 slice 类似，其初始化也分为不带初始容量和带初始容量。

```go
map1 := make(map[string]int)
map2 := make(map[string]int, 10)
```

- 面对不带容量的，go 会对其进行懒加载，要用的时候才去分配 bucket
- 对于带容量的，go 会根据容量来计算出一个合适的 **B 值**（上面提到过的 $2^B$ 桶数量）

显然，带有初始容量会更高效，减少扩容次数（这一点在 slice 中多次提到）。

最后需要注意一点，如果不用 make 进行初始化而是 var，则会遇到问题

```go
var map3 map[string]int
map3["l"]=20 
```

这里会发生 panic，因为这是一个 nil map，不能直接存入键值对。为什么呢？

在 slice 文章中我们提到过 nil slice 和 empty slice 的区别——nil slice 等于 nil，而empty 是长度为 0 的。nil 是 empty 的子集。

同样的 nil Map 和 empty Map。我们将二者进行输出，看看结果如何。

```go
func main() {
	map1 := make(map[string]int)
    var map3 map[string]int
	println(map1)
	println(map3)
}
// 结果如下
0x14000052708
0x0
```

- nil 输出为`0x0`，意味着 go **未分配任何底层内存结构**给它，只是一个 nil 变量
- empty 输出了具体地址，已经分配了 hmap 结构的内存。

所以我们对 nil 进行写入数据会引发 panic。

## 总结

那么关于 Map 的剖析上篇就到这里结束了，我们着重分析了其底层结构 hmap 的内容、初始化的方式以及可能出现的问题。

下一篇文章我们进入底层的扩容机制以及回答开始的问题，为什么 map 并发不安全。

**这里是LTX，感谢您阅读这篇博客，人生海海，和自己对话，像只蝴蝶纵横四海。**