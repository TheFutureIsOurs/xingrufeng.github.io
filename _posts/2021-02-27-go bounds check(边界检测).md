---
layout: post
title: "go bounds check(边界检测)"
date: 2021-02-27
tags:
 - go
---

> 本文作者：刘代明,现在奇虎360搜索技术部任web服务端技术专家职位。 祝大家阅读愉快。

## 前言

我们在阅读go源码时，很多地方在操作数组或切片时，会有bounds check的注释，
比如go map在对tophash指定索引赋值(go map的源码阅读见 [go map源码阅读及与php map实现对比](https://www.imflybird.cn/2021/02/23/go-map%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB%E5%8F%8A%E4%B8%8Ephp-map%E5%AE%9E%E7%8E%B0%E5%AF%B9%E6%AF%94/)这篇文章)：

```go

dst.b.tophash[dst.i&(bucketCnt-1)] = top // mask dst.i as an optimization, to avoid a bounds check

```

类似的地方还有很多，

那bounds check是什么呢？为什么要这么做呢？该如何做呢？

## bounds check是什么

我们知道，go是一个编译型语言，而且还是一个内存安全的语言。在操作数组或切片的索引时，我们不用手动检查是否出现了数组越界，go的运行时会帮我们检查，如果出现数组越界，会抛出panic。

而bounds check说的就是对数组或切片的索引操作进行越界检查。


## 为什么要这么做

虽然有go 运行时帮我们来做越界检查，但是相应的，大量的bounds check无法使我们的程序达到最高的运行效率。那我们能不能减少或消除运行时的bounds check呢？


## 该如何做

幸好，go 编译器提供了检测命令来帮助我们来找出哪些地方进行了bounds check.

我们可以通过查看ssa，具体下面两个方法来查看：

> 1.直接查看ssa生成，GOSSAFUNC=func go build go file
> 
> 2.使用命令查看ssa的check_bce阶段：go build -gcflags="-d=ssa/check_bce/debug"
> 
> 其结构为：-d=ssa/<phase>/<flag>[=<value>|<function_name>]
> 
> value默认为1



如：

```go

package main
func f1(s []int) {
	_ = s[0]
}
func main() {}


```

使用方法一，可以看到如下信息：

![](https://www.imflybird.cn/static/img/2020/GO/ssa-bce.png)

使用方法二，可以看到如下信息：

```go

# bce
./main.go:5:7: Found IsInBounds

```

实际代码中，推荐使用方法2来定位，输出更简洁。

我们看下消除情况：

```go

func f1(s []int) {
	_ = s[0] // bounds check
	_ = s[1] // bounds check
	_ = s[2] // bounds check
}

func f2(s []int) {
	_ = s[2] // bounds check
	_ = s[0] // bounds check 消除
	_ = s[1] // bounds check 消除
}

```

上面中f2中，因为检测了了s[2]，编译器可以消除s[0]和s[1]的边界检查。

再如：

```go

_ = dst.b.tophash[dst.i&(bucketCnt-1)] // 边界消除。&操作符保证了index落在范围内

_ = dst.b.tophash[dst.i%(bucketCnt-1)] // bounds check。%操作符无法保证index落在范围内，可能有负数

```

有一种情况：

```go

var s[]int
var index int
_ = s[:index] // bounds check
_ = s[index:] // bounds check

```

为什么会这样？因为s[a:b]中需要 0 <= a <= len(s), a <= b <= cap(s)

所以需要检测index是否越界。

我们可以利用边界检测消除来提升程序性能，如下，我们通过给编译器增加提示来达到边界消除：

```go

func fd(is []int, bs []byte) {
	if len(is) >= 256 {
		for _, n := range bs {
			_ = is[n] // bounds check
		}
	}
}

func fd2(is []int, bs []byte) {
	if len(is) >= 256 {
		is = is[:256] // 给编译器增加一个提示，可以避免下面for中的边界检测
		for _, n := range bs {
			_ = is[n] // 边界检测消除
		}
	}
}

```


参考：

[Bounds Check Elimination](https://go101.org/article/bounds-check-elimination.html)



