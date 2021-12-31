---
title: "Golang的错误处理"
date: 2021-12-15T16:29:53+08:00
draft: false
tags: [Golang]
categories: [笔记]

---

# 异常（panic）和错误（error）

## panic

panic会中止程序执行进入异常处理逻辑，panic可以在当前函数或者调用链向上的任何一层被[defer recover](https://go.dev/blog/defer-panic-and-recover)捕获处理，没被捕获的panic会使程序打印堆栈后异常退出。

panic很少在代码中出现，一般用来表示除数为0、内存不足、强制类型转换失败等语言层级的异常，如果出现意味着程序员本身的实现问题，所以程序员一般不会着重考虑和防御它们。

## error

error会显得更平凡，所有满足error这个interface的值都可以当做一个合法的错误：

```go
type error interface {
    // 输出对该错误的文本描述
    Error() string
}
```

在Go语言中，error是[一个平凡的值](https://go.dev/blog/errors-are-values)，运行时不会用特殊的逻辑对待它，不处理它不会让程序打印堆栈异常退出。

error会在代码中大量出现，表示连接超时、JSON解析失败、文件不存在等用户层级的错误，往往和业务直接相关，程序眼需要着重考虑它们。

## 总结

- panic：不着重考虑，没有明确的抛出位置；
- error：着重考虑，抛出位置固定而明显，只要函数返回值元组中最后一个是error类型，用户就需要考虑防御和处理。

换句话说，Go语言中error是值这个特征在鼓励用户考虑和处理每个错误，写出鲁棒的程序。



# “好”的错误

例如，有个天气API服务，它在收到请求时会先查询Redis中有没有对应缓存，如果没有缓存再请求外部服务，最后返回结果给调用方。

有一天，调用方说接口不工作了，你查了日志，错误信息是[`context deadline exceeded`](https://pkg.go.dev/context#pkg-variables)，这个错误意味着代码里有地方发生了超时，可是在哪里呢？是缓存还是外部API？



好的错误是有根本原因和调用链上下文的错误，这样的错误容易排查，如果你看到的错误信息是这样的：

```go
reading cache: redis GET: context deadline exceeded
```

或者带有调用堆栈的：

```go
context deadline exceeded
goroutine 1 [running]:
main.Example(0x19010001)
           /Users/hello/main.go
           temp/main.go:8 +0x64
main.main()
           /Users/bill/main.go
           temp/main.go:4 +0x32
```

那么排错工作都会容易得多。

实际应用中,比起堆栈，人工添加的文本错误上下文更易于阅读，有着更高的信息密度，而且看到的人就算没有接触相关代码也有可能理解错误原因。



# 为错误提供文本上下文

Go 1.13中新增了[`fmt.Errorf()`](https://pkg.go.dev/fmt#Errorf)用于为错误提供上下文，[errors](https://pkg.go.dev/errors)标准库新增`Is()`, `As()`, `Unwrap()`用于便利化错误的鉴别和比较。

`fmt.Errorf()`的用法大概是这样的，回忆刚刚的那种“好错误”：

```go
reading cache: redis GET: context deadline exceeded
```

这个错误就像在说故事一样，从左到右层层递进，每层`fmt.Errorf()`叙述自己想要做的事情，然后用`:`分隔下一层，下一层用`%w`指代。

下面这段伪代码中，`FindWeather()`调用`ReadCache()`，请试想`ReadCache()`报错时，`FindWeather()`调用者收到的错误：

```go
func FindWeather(city string) (weather string, err error) {
	weather, err = ReadCache(city)
	if err != nil {
		err = fmt.Errorf("reading cache: %w", err)
		return
	}
	
	if weather != "" {
		// cache hit
		return 
	}
	
	// cache missed, query for data source and update cache
	// ...
}

func ReadCache(city string) (weather string, err error) {
	cacheKey := "city-" + city
	weather, err = cache.Get(cacheKey)
	if err == redis.Nil {
		// cache missed
		err = nil
		return 
	} else if err != nil {
		err = fmt.Errorf("redis Get: %w", err)
		return
	}
	
	return 
}

```

Go语言的静态分析工具能检查出代码中忘记处理的错误，不规范的错误上下文格式等（`%d`,`%f`, `%v`, `%w`, `%x`分不清），一般使用它们的合集版本[golangci-lint](https://github.com/golangci/golangci-lint).

# 其他为错误添加上下文的方式

如果有必要，也可以使用其他为错误添加上下文的方案

- [使用pkg/errors为错误添加堆栈](https://pkg.go.dev/github.com/pkg/errors#hdr-Retrieving_the_stack_trace_of_an_error_or_wrapper)；
- 如果是HTTP服务，使用panic和recover中间件，[知乎的实现](https://zhuanlan.zhihu.com/p/48039838)就是个例子，我猜想上下文也是以堆栈的方式呈现。

# 参考文献

1. https://nanmu.me/zh-cn/posts/2021/error-handling-in-go/
2. https://go.dev/blog/defer-panic-and-recover
3. https://go.dev/blog/errors-are-values
