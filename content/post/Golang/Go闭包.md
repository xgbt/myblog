---
title: "Golang闭包"
date: 2022-1-13T16:29:53+08:00
draft: false
tags: [Golang]
categories: [笔记]

---

# **闭包（closure）**

case 1

```go
func foo1(x *int) func() {
  return func() {
    *x = *x + 1
    fmt.Printf("foo1 val = %d\n", *x)
  }
}
```

case 2

```go
func foo2(x int) func() {
  return func() {
    x = x + 1
    fmt.Printf("foo2 val = %d\n", x)
  }
}
```

case1~2：值传递（by value） vs. 引用传递（by reference）

Go是没有**引用传递**的，即使是`foo1()`在参数上加了`*`，内部实现机制仍旧是**值传递**，只不过传递的是指针的数值。

# 

什么是闭包呢？摘用Wikipedia上的一句定义：

> a **closure** is a record storing **a function** together with **an environment**.
> **闭包**是由**函数**和与其相关的引用**环境**组合而成的实体 。

因此**闭包**的核心就是：**函数**和**环境**。其实这里就已经可以回答本文题目的问题：**闭包究竟包了什么？答案是：函数和环境。**

跳回`foo1()`和`foo2()`的例子，来解释**闭包**的**函数和环境**。

```go
func foo1(x *int) func() {
    return func() {
        *x = *x + 1
        fmt.Printf("foo1 val = %d\n", *x)
    }
}
func foo2(x int) func() {
    return func() {
        x = x + 1
        fmt.Printf("foo1 val = %d\n", x)
    }
}

// Q1第一组实验
x := 133
f1 := foo1(&x)
f2 := foo2(x)
f1() // foo1 val = 134
f2() // foo2 val = 134
f1() // foo1 val = 135
f2() // foo2 val = 135
// Q1第二组
x = 233
f1() // foo1 val = 234
f2() // foo2 val = 136
f1() // foo1 val = 235
f2() // foo2 val = 137
// Q1第三组
foo1(&x)() // foo1 val = 236
foo2(x)() // foo2 val = 237
foo1(&x)() // foo1 val = 237
foo2(x)() // foo2 val = 238
foo2(x)() // foo2 val = 238
```

定义了`x=133`之后，我们获取得到了`f1=foo1(&x)`和`f2=foo2(x)`。这里`f1\f2`就是**闭包的函数**，也就是`foo1()\foo2()`的内部匿名函数；而**闭包的环境**即外部函数`foo1()\foo2()`的变量`x`（因为内部匿名函数引用到的相关变量只有`x`，因此这里简化为变量`x`）。

**闭包的函数**做的事情归纳为：1). 将环境的变量`x`自增1；2). 打印环境变量`x`。

**闭包的环境**则是其外部函数获取到的变量`x`。

因此**Q1第一组实验**的答案为：

```go
f1() // foo1 val = 134
f2() // foo2 val = 134
f1() // foo1 val = 135
f2() // foo2 val = 135
```

这是因为闭包`f1\f2`都保存了`x=133`时的整个环境，每次调用闭包`f1\f2`都会执行一次自增+打印的内部匿名函数。因此第一次输出都是`(133+1=)134`，第二次输出都是`(134+1=)135`。

那么**Q1第二组实验**的答案呢？

```go
f1() // foo1 val = 234
f2() // foo2 val = 136
f1() // foo1 val = 235
f2() // foo2 val = 137
```

**有趣的事情发生了！**`f1`的值居然发生了显著性的变化！通过这组实验，能够更好地解释**其（函数）相关的引用环境**其实就是产生这个闭包的时候的外部函数的环境，因此变量`x`的可见性和作用域也与外部函数相同，又因为`foo1`是“引用传递”，变量`x`的作用域不局限在`foo1()`中，因此当`x`发生变化的时候，闭包`f1`内部也变化了。这个也正好是"**反之，那么闭包其实不再封闭，全局可见的变量的修改，也会对闭包内的这个变量造成影响**"的证明。

**Q1的第三组实验**的答案：

```go
foo1(&x)() // foo1 val = 236
foo2(x)() // foo2 val = 237
foo1(&x)() // foo1 val = 237
foo2(x)() // foo2 val = 238
foo2(x)() // foo2 val = 238
```

因为`foo1()`返回的闭包都会修改变量`x`的数值，因此调用`foo1()()`之后，变量`x`必然增加1。而`foo2()`返回的闭包仅仅修改其内部环境的变量`x`而对调用外部的变量`x`不影响，且每次调用`foo2()`返回的闭包是独立的，和其他调用`foo2()`的闭包不相关，因此最后两次的调用，打印的数值都是相同的；第一次调用和第二次调用`foo2()`发现打印出来的数值增加了1，是因为两次调用之间传入的`x`的数值分别是236和237，而不是说第二次在第一次基础上增加了1，这点需要补充说明。







# **闭包的延迟绑定**

调用`f7()`的时候分别会打印什么？

```go
func foo7(x int) []func() {
    var fs []func()
    values := []int{1, 2, 3, 5}
    for _, val := range values {
        fs = append(fs, func() {
            fmt.Printf("foo7 val = %d\n", x+val)
        })
    }
    return fs
}
// Q4实验：
f7s := foo7(11)
for _, f7 := range f7s {
    f7()
}
```

答案是：

```go
foo7 val = 16
foo7 val = 16
foo7 val = 16
foo7 val = 16
```

**闭包**是一段**函数**和**相关的引用环境**的实体。case7的问题中，函数是打印变量`val`的值，引用环境是变量`val`。仅仅是这样的话，遍历到val=1的时候，记录的不应该是val=1的环境吗？

上文在闭包解释最后，还有一句话：闭包保存/记录了它**产生**时的外部函数的所有环境。如同普通变量/函数的**定义**和实际**赋值/调用或者说执行**，是两个阶段。

闭包也是一样，for-loop内部仅仅是声明了一个闭包，`foo7()`返回的也仅仅是一段闭包的函数定义，只有在外部执行了`f7()`时才真正执行了闭包，此时才闭包内部的变量才会进行赋值的操作。



这就是闭包的神奇之处，它会保存相关引用的环境，也就是说，`val`这个变量在闭包内的生命周期得到了保证。因此在执行这个闭包的时候，会去外部环境寻找最新的数值。

**临时case**：

```go
func foo0() func() {
    x := 1
    f := func() {
        fmt.Printf("foo0 val = %d\n", x)
    }
    x = 11
    return f
}

foo0()() // 猜猜我会输出什么？
```

既然我说会**在执行的时候去外部环境寻找最新的数值**，那`x`的最新数值就是11，果然，最后输出的就是11。



# **Goroutine的延迟绑定**



**case3、case4和case5**不是闭包，**case3**只是遍历了内部的slice并且打印，**case4**是在遍历时通过协程调用了打印函数打印，**case5**也是在遍历slice时调用了内部匿名函数打印。

**Q2的case3问题**的答案先丢出来：

```go
func foo3() {
    values := []int{1, 2, 3, 5}
    for _, val := range values {
        fmt.Printf("foo3 val = %d\n", val)
    }
}

foo3()
//foo3 val = 1
//foo3 val = 2
//foo3 val = 3
//foo3 val = 5
```

中规中矩，遍历输出slice的内容：1，2，3，5。

**Q2的case4问题**的答案再丢出来：

```go
func show(v interface{}) {
    fmt.Printf("foo4 val = %v\n", v)
}
func foo4() {
    values := []int{1, 2, 3, 5}
    for _, val := range values {
        go show(val)
    }
}

foo4()
//foo3 val = 2
//foo3 val = 3
//foo3 val = 1
//foo3 val = 5
```

嗯，因为Go Routine的执行顺序是随机并行的，因此执行多次`foo4()`输出的顺序不一行相同，但是一定打印了“1，2，3，5”各个元素。

最后是**Q2的case5问题**的答案：

```go
func foo5() {
    values := []int{1, 2, 3, 5}
    for _, val := range values {
        go func() {
            fmt.Printf("foo5 val = %v\n", val)
        }()
    }
}

foo5()
//foo3 val = 5
//foo3 val = 5
//foo3 val = 5
//foo3 val = 5
```

居然都打印了5，惊不惊喜，意不意外？！相信看过子标题的你，一定不意外了（捂脸）。是的，接下来就要讲讲**Go Routine的延迟绑定**：

这个问题的本质同**闭包的延迟绑定**，或者说，这段匿名函数的对象就是**闭包**。

在我们调用`go func() { xxx }()`的时候，只要没有真正开始执行这段代码，那它还只是一段函数声明。

而在这段匿名函数被执行的时候，才是内部变量寻找真正赋值的时候。

在**case5**中，for-loop的遍历几乎是“瞬时”完成的，4个Go Routine真正被执行在其后。

这个匿名函数就是一个闭包吗？

**闭包**真正被执行的时候，for-loop结束了，但是`val`的生命周期在闭包内部被延长了且被赋值到最新的数值5。



所以，**Go Routine的匿名函数的延迟绑定本质就是闭包，实际生成中注意下这种写法~**

# 参考文献

1. https://zhuanlan.zhihu.com/p/92634505
