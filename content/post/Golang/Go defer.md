---
title: "Golang的defer"
date: 2022-2-11T16:29:53+08:00
draft: false
tags: [Golang]
categories: [笔记]

---



# 1、多个defer语句，按先进后出的方式执行

```go
package main

import "fmt"

func main() {
    var whatever [5]struct{}
    for i := range whatever {
        defer fmt.Println(i)
    }
}
```

```
4
3
2
1
0
```

# 2、defer声明时，对应的参数会实时解析

```go
package main

import "fmt"

func main() {
	i := 1
	fmt.Println("i =", i)
	defer fmt.Print(i)
}

```

```
i = 1
1
```

## defer后面跟无参函数、有参函数和方法：

```go
package main

import "fmt"

func test(a int) {//无返回值函数
	defer fmt.Println("1、a =", a) //方法
	defer func(v int) { fmt.Println("2、a =", v)} (a) //有参函数
	defer func() { fmt.Println("3、a =", a)} () //无参函数
	a++
}
func main() {
	test(1)
}

```

```go
3、a = 2
2、a = 1
1、a = 1
```

方法中的参数a，有参函数中的参数v，会请求参数，直接把参数代入，所以输出的都是1。

a++变成2之后，3个defer语句以后声明先执行的顺序执行，无参函数中使用的a现在已经是2了，故输出2。

# 3、可读取函数返回值（return返回机制）

**defer、return、返回值三者的执行逻辑应该是：**

- return最先执行，return将结果写入返回值中；
- 接着defer开始执行一些收尾工作；
- 最后函数携带**当前返回值**（可能和最初的返回值不相同）退出。

## （0）defer放在return后面

defer放在return后面时，就不会被执行。

```go
package main

import "fmt"

func f(i int) int{
	return i
	defer fmt.Print("i =", i)
	return i+1
}

func main() {
	f(1)
}
```

没有输出，因为return i之后函数就已经结束了，不会执行defer。

## （1）※无名返回值

```go
package main

import (
	"fmt"
)

func a() int {
	var i int
	defer func() {
		i++
		fmt.Println("defer2:", i) 
	}()
	defer func() {
		i++
		fmt.Println("defer1:", i) 
	}()
	return i
}

func main() {
	fmt.Println("return:", a()) 
}

```

```
defer1: 1
defer2: 2
return: 0
```

返回值由变量i赋值，相当于返回值=i=0。第二个defer中i++ = 1， 第一个defer中i++ = 2，所以最终i的值是2。但是返回值已经被赋值了，即使后续修改i也不会影响返回值。最终返回值返回，所以main中打印0。

## （2）※有名返回值

```go
package main

import (
	"fmt"
)

func b() (i int) {
	defer func() {
		i++
		fmt.Println("defer2:", i)
	}()
	defer func() {
		i++
		fmt.Println("defer1:", i)
	}()
	return i //或者直接写成return
}

func main() {
	fmt.Println("return:", b())
}

```

```
defer1: 1
defer2: 2
return: 2
```

这里已经指明了返回值就是i，所以后续对i进行修改都相当于在修改返回值，所以最终函数的返回值是2。



## （3）函数返回值为地址

```go
package main

import (
	"fmt"
)

func c() *int {
	var i int
	defer func() {
		i++
		fmt.Println("defer2:", i)
	}()
	defer func() {
		i++
		fmt.Println("defer1:", i)
	}()
	return &i
}

func main() {
	fmt.Println("return:", *(c()))
}

```

```
defer1: 1
defer2: 2
return: 2

```

另一个例子

```go
func f() (r int) {
    defer func(r int) {
          r = r + 5
    }(r)
    return 1
}
```

# 4、defer与闭包

```go
package main

import "fmt"

type Test struct {
	name string
}
func (t *Test) pp() {
	fmt.Println(t.name)
}
func main() {
	ts := []Test{{"a"}, {"b"}, {"c"}}
	for _, t := range ts {
		defer t.pp()
	}
}

```

```
c
c
c
```

for结束时t.name=“c”，接下来执行的那些defer语句中用到的t.name的值均为”c“。

**如果修改为：**

```go
package main

import "fmt"

type Test struct {
	name string
}
func pp(t Test) {
	fmt.Println(t.name)
}
func main() {
	ts := []Test{{"a"}, {"b"}, {"c"}}
	for _, t := range ts {
		defer pp(t)
	}
}

```

```
c
b
a

```

defer语句中的参数会实时解析，所以在碰到defer语句的时候就把该时的t代入了。

**再次修改代码：**

```go
package main

import "fmt"

type Test struct {
	name string
}
func (t *Test) pp() {
	fmt.Println(t.name)
}

func main() {
	ts := []Test{{"a"}, {"b"}, {"c"}}
	for _, t := range ts {
		tt := t
		println(&tt)
		defer tt.pp()
	}
}

```

```
0xc000010200
0xc000010210
0xc000010220
c
b
a
```

:=用来声明并赋值，连续使用2次a:=1就会报错，但是在for循环内，可以看出每次tt:=t时，tt的地址都不同，说明他们是不同的变量，所以并不会报错。

每次都有一个新的变量tt:=t，所以每次在执行defer语句时，对应的tt不是同一个,

（for循环中实际上生成了3个不同的tt）

，所以输出的结果也不相同。

# 5、defer用于关闭文件和互斥锁

```go
func ReadFile(filename string) ([]byte, error) {
    f, err := os.Open(filename)
    if err != nil {
        return nil, err
    }
    defer f.close()
    return ReadAll()
}

```

```go
var mu sync.Mutex
var m = make(map[string]int)
 
func lookup(key string) int {
    mu.Lock()
    defer mu.Unlock()
    return m[key]
}

```

# 6、“解除”对所在函数的依赖

```go
package main

import "fmt"
import "time"

type User struct {
	username string
}

func (this *User) Close() {
	fmt.Println(this.username, "Closed !!!")
}

func main() {
	u1 := &User{"jack"}
	defer u1.Close()
	u2 := &User{"lily"}
	defer u2.Close()
	time.Sleep(10 * time.Second)
	fmt.Println("Done !")

}

```

# 7、defer与panic

（1）在panic语句后面的defer语句不被执行

（2）在panic语句前的defer语句会被执行

# 8、调用os.Exit时defer不会被执行

```go
func deferExit() {
    defer func() {
        fmt.Println("defer")
    }()
    os.Exit(0)
}
```

当调用os.Exit()方法退出程序时，defer并不会被执行，上面的defer并不会输出。

# 参考文献

1. https://blog.csdn.net/Cassie_zkq/article/details/108567205
