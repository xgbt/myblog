---
title: "geeWeb项目笔记"
date: 2022-03-24T21:31:05+08:00
draft: false
tags: [专业知识]
categories: [笔记]
---



# 上下文 Context

1. 对Web服务来说，无非是根据请求`*http.Request`，构造响应`http.ResponseWriter`。

   但是这两个对象提供的接口粒度太细，比如要构造一个完整的响应，需要考虑Header和Body，而 Header 包含了StatusCode，ContentType等每次请求都需要设置的信息。

   因此，如果不进行有效的封装，那么框架的用户将需要写大量重复，繁杂的代码，而且容易出错。

2. 针对使用场景，封装`*http.Request`和`http.ResponseWriter`的方法，简化相关接口的调用，只是设计 Context 的原因之一。

   对于框架来说，还需要支撑额外的功能。例如，解析**动态路由**`/hello/:name`，参数`:name`的值放在哪？框架需要支持**中间件**，中间件产生的信息放在哪？

   Context 随着每一个请求的出现而产生，伴随每一个请求的结束而销毁，和当前请求强相关的信息都应由 Context 承载。

   因此，设计 Context 结构，扩展性和复杂性留在了内部，而对外简化了接口。

   路由的处理函数，以及将要实现的中间件，参数都统一使用 Context 实例， Context 就像一次会话的百宝箱，可以找到任何东西。

## Demo

```go
func main() {
	r := gee.New()
	r.GET("/", func(c *gee.Context) {
		c.HTML(http.StatusOK, "<h1>Hello Gee</h1>")
	})
	r.GET("/hello", func(c *gee.Context) {
		// expect /hello?name=geektutu
		c.String(http.StatusOK, "hello %s, you're at %s\n", c.Query("name"), c.Path)
	})

	r.POST("/login", func(c *gee.Context) {
		c.JSON(http.StatusOK, gee.H{
			"username": c.PostForm("username"),
			"password": c.PostForm("password"),
		})
	})

	r.Run(":9999")
}
```

```bash
$ curl -i http://localhost:9999/
HTTP/1.1 200 OK
Date: Mon, 12 Aug 2019 16:52:52 GMT
Content-Length: 18
Content-Type: text/html; charset=utf-8
<h1>Hello Gee</h1>

$ curl "http://localhost:9999/hello?name=geektutu"
hello geektutu, you're at /hello

$ curl "http://localhost:9999/login" -X POST -d 'username=geektutu&password=1234'
{"password":"1234","username":"geektutu"}

$ curl "http://localhost:9999/xxx"
404 NOT FOUND: /xxx
```



# 路由 Router

路由相关的方法和结构应提取出来，放到文件`router.go`中，方便对 router 的功能进行增强，例如提供动态路由的支持。 其中，router 的 handle 方法作了一个细微的调整，即 handler 的参数，变成了 Context。

## 动态路由

### Trie简介

实现动态路由最常用的数据结构，被称为前缀树(Trie树)。看到名字你大概也能知道前缀树长啥样了：每一个节点的所有的子节点都拥有相同的前缀。这种结构非常适用于路由匹配，比如我们定义了如下路由规则：

- /:lang/doc
- /:lang/tutorial
- /:lang/intro
- /about
- /p/blog
- /p/related

如果用前缀树来表示，是这样的。

![trie tree](C:\Users\Administrator\Desktop\7days-golang-master\gee-web\doc\gee-day3\trie_router.jpg)

HTTP请求的路径恰好是由`/`分隔的多段构成的，因此，每一段可以作为前缀树的一个节点。我们通过树结构查询，如果中间某一层的节点都不满足条件，那么就说明没有匹配到的路由，查询结束。

本项目实现的动态路由具备以下两个功能。

- **参数匹配**`:`。例如 `/p/:lang/doc`，可以匹配 `/p/c/doc` 和 `/p/go/doc`。
- **通配**`*`。例如 `/static/*filepath`，可以匹配`/static/fav.ico`，也可以匹配`/static/js/jQuery.js`，这种模式常用于静态服务器，能够递归地匹配子路径。

### Trie实现

[**geeWeb/trie.go**]()

```go
type node struct {
	pattern  string  // 节点待匹配路由, 例如 /p/:lang
	part     string  // 节点存储的部分路由，例如 :lang
	children []*node // 子节点，例如 [doc, tutorial, intro]
	isWild   bool    // 标记节点是否与 * 和 : 匹配, part 含有 : 或 * 时为true
}
```

与普通的树不同，为了实现动态路由匹配，加上了`isWild`这个参数。

例如：

当匹配 `/p/go/doc/`时，

第一层节点，`p`精准匹配到了`p`，

第二层节点，`go`模糊匹配到`:lang`，那么将会把`lang`这个参数赋值为`go`，继续下一层匹配。

下面将匹配的逻辑，包装为两个辅助函数`matchChild` 和 `matchChildren`。

```go
// 返回第一个匹配成功的节点，用于insert操作
func (n *node) matchChild(part string) *node {
	for _, child := range n.children {
		if child.part == part || child.isWild {
			return child
		}
	}
	return nil
}

// 返回所有匹配成功的节点，用于search
func (n *node) matchChildren(part string) []*node {
	nodes := make([]*node, 0)
	for _, child := range n.children {
		if child.part == part || child.isWild {
			nodes = append(nodes, child)
		}
	}
	return nodes
}
```

对于Route说，最重要的是注册与匹配。

开发服务时，注册路由规则，映射对应handler；用户访问时，匹配路由规则，查找对应handler。

因此，Trie 树需要支持节点的插入与查询。

**插入**：遍历每一层的节点，如果没有匹配到当前`part`的节点，则新建一个并递归插入。

有一点需要注意，`/p/:lang/doc`只有在第三层节点，即`doc`节点，`pattern`才会设置为`/p/:lang/doc`。`p`和`:lang`节点的`pattern`属性皆为空。因此，当匹配结束时，我们可以使用`n.pattern == ""`来判断路由规则是否匹配成功。例如，`/p/python`虽能成功匹配到`:lang`，但`:lang`的`pattern`值为空，因此匹配失败。

**查询**，遍历查询每一层的节点。

退出规则为，如果匹配到了`*`，或者匹配到了第`len(parts)`层节点，且`n.pattern`非空则说明匹配成功。

```go
func (n *node) insert(pattern string, parts []string, idx int) {
	if idx == len(parts) {
		n.pattern = pattern
		return
	}

	part := parts[idx]
	child := n.matchChild(part)
	if child == nil {
		child = &node{
			part:   part,
			isWild: part[0] == ':' || part[0] == '*',
		}
		n.children = append(n.children, child)
	}
	child.insert(pattern, parts, idx+1)
}

func (n *node) search(parts []string, idx int) *node {
	if idx == len(parts) || strings.HasPrefix(n.part, "*") {
		if n.pattern == "" {
			return nil
		}
		return n
	}

	part := parts[idx]
	children := n.matchChildren(part)
	for _, child := range children {
		ret := child.search(parts, idx+1)
		if ret != nil {
			return ret
		}
	}

	return nil
}
```

### 应用到Router

**getRoute** 函数中，还解析了`:`和`*`两种匹配符的参数，返回一个 map 。

例如：

`/p/go/doc` 匹配到 `/p/:lang/doc`，解析结果为：`{lang: "go"}`

`/static/css/geektutu.css` 匹配到 `/static/*filepath`，解析结果为`{filepath: "css/geektutu.css"}`。

```go
func (r *router) addRoute(method string, pattern string, handler HandlerFunc) {
	if _, ok := r.roots[method]; !ok {
		r.roots[method] = new(node)
	}

	parts := parsePattern(pattern)
	r.roots[method].insert(pattern, parts, 0)

	key := method + "-" + pattern
	r.handlers[key] = handler
}

func (r *router) getRoute(method string, pattern string) (*node, map[string]string) {
	// 获取Method根节点
	root, ok := r.roots[method]
	if !ok {
		return nil, nil
	}

	// pattern存储的是原始路由，例如 p/go/doc
	searchParts := parsePattern(pattern)
	n := root.search(searchParts, 0)
	if n != nil {
		params := make(map[string]string)
		// n.pattern存储的是解析路由，例如p/:lang/doc
		parts := parsePattern(n.pattern)
		for idx, part := range parts {
			if part[0] == ':' {
				params[part[1:]] = searchParts[idx]
			}
			if part[0] == '*' && len(part) > 1 {
				params[part[1:]] = strings.Join(searchParts[idx:], "/")
				break
			}
		}
		return n, params
	}

	return nil, nil
}
```

## Demo

```go
func main() {
	r := gee.New()
	r.GET("/", func(c *gee.Context) {
		c.HTML(http.StatusOK, "<h1>Hello Gee</h1>")
	})

	r.GET("/hello", func(c *gee.Context) {
		// expect /hello?name=geektutu
		c.String(http.StatusOK, "hello %s, you're at %s\n", c.Query("name"), c.Path)
	})

	r.GET("/hello/:name", func(c *gee.Context) {
		// expect /hello/geektutu
		c.String(http.StatusOK, "hello %s, you're at %s\n", c.Param("name"), c.Path)
	})

	r.GET("/assets/*filepath", func(c *gee.Context) {
		c.JSON(http.StatusOK, gee.H{"filepath": c.Param("filepath")})
	})

	r.Run(":9999")
}
```

```bash
$ curl "http://localhost:9999/hello/geektutu"
hello geektutu, you're at /hello/geektutu

$ curl "http://localhost:9999/assets/css/geektutu.css"
{"filepath":"css/geektutu.css"}
```



# 分组 Group

所谓分组，是指路由的分组。如果没有路由分组，我们需要针对每一个路由进行控制。但是真实的业务场景中，往往某一组路由需要相似的处理。例如：

- 以`/post`开头的路由匿名可访问。
- 以`/admin`开头的路由需要鉴权。
- 以`/api`开头的路由是 RESTful 接口，可以对接第三方平台，需要三方平台鉴权。

大部分情况下的路由分组，是以相同的前缀来区分的。

中间件可以给框架提供无限的扩展能力，应用在分组上，可以使得分组控制的收益更为明显，而不是共享相同的路由前缀这么简单。例如`/admin`的分组，可以应用鉴权中间件；`/`分组应用日志中间件，`/`是默认的最顶层的分组，也就意味着给所有的路由，即整个框架增加了记录日志的能力。

## 分组嵌套

将`Engine`作为最顶层分组，也就是说`Engine`拥有`RouterGroup`所有的能力。

```go
type (
	RouterGroup struct {
		prefix      string        // 前缀，例如 / , /api
		middlewares []HandlerFunc // 存储当前分组应用的中间件
		parent      *RouterGroup  // 存储当前分组的父节点
		engine      *Engine       // 所有路由分组都共享同一个Engine实例
	}

	Engine struct {
		*RouterGroup
		router        *router
		groups        []*RouterGroup     // store all groups
		htmlTemplates *template.Template // for html render
		funcMap       template.FuncMap   // for html render
	}
)
```

这样，就可以将和路由有关的函数，都交给`RouterGroup`实现。

```go
func (group *RouterGroup) addRoute(method string, comp string, handler HandlerFunc) {
	pattern := group.prefix + comp
	log.Printf("Route %4s - %s", method, pattern)
	// 由于`Engine`从某种意义上继承了`RouterGroup`的所有属性和方法，因为 (*Engine).engine 是指向自己的。
	group.engine.router.addRoute(method, pattern, handler)
}
```

## Demo

```go
func main() {
	r := gee.New()
	r.GET("/index", func(c *gee.Context) {
		c.HTML(http.StatusOK, "<h1>Index Page</h1>")
	})
	v1 := r.Group("/v1")
	{
		v1.GET("/", func(c *gee.Context) {
			c.HTML(http.StatusOK, "<h1>Hello Gee</h1>")
		})

		v1.GET("/hello", func(c *gee.Context) {
			// expect /hello?name=geektutu
			c.String(http.StatusOK, "hello %s, you're at %s\n", c.Query("name"), c.Path)
		})
	}
	v2 := r.Group("/v2")
	{
		v2.GET("/hello/:name", func(c *gee.Context) {
			// expect /hello/geektutu
			c.String(http.StatusOK, "hello %s, you're at %s\n", c.Param("name"), c.Path)
		})
		v2.POST("/login", func(c *gee.Context) {
			c.JSON(http.StatusOK, gee.H{
				"username": c.PostForm("username"),
				"password": c.PostForm("password"),
			})
		})

	}

	r.Run(":9999")
}
```

```bash
$ curl "http://localhost:9999/v1/hello?name=geektutu"
hello geektutu, you're at /v1/hello

$ curl "http://localhost:9999/v2/hello/geektutu"
hello geektutu, you're at /hello/geektutu
```



# 中间件 middlewares

简单说就是非业务的技术类组件。Web 框架本身不可能去理解所有的业务，因而不可能实现所有的功能。因此，框架需要有一个插口，允许用户自己定义功能嵌入到框架中，仿佛这个功能是框架原生支持的一样。

因对中间件而言，需要考虑2个比较关键的点：

- 插入点在哪？使用框架的人并不关心底层逻辑的具体实现，如果插入点太底层，中间件逻辑就会非常复杂。如果插入点离用户太近，那和用户直接定义一组函数，每次在 Handler 中手工调用没有太大区别。
- 中间件的输入是什么？中间件的输入，决定了扩展能力。暴露的参数太少，用户发挥空间有限。

## 中间件设计

中间件的定义与路由映射的 Handler 一致，处理的输入是`Context`对象。

插入点是框架接收到请求初始化`Context`对象后，允许用户使用自己定义的中间件做一些额外的处理，例如记录日志等，以及对`Context`进行二次加工。

另外通过调用`(*Context).Next()`函数，中间件可等待用户自己定义的 `Handler`处理结束后，再做一些额外的操作，即 中间件支持用户在请求被处理的前后，做一些额外的操作。

例如，我们希望最终能够支持如下定义的中间件，

[**geeWeb/logger.go**]()

```go
func Logger() HandlerFunc {
	return func(c *Context) {
		// 启动计时器
		t := time.Now()
		// 执行其他的中间件或用户定义的HandlerFunc
		c.Next()
		// 计算执行时间
		log.Printf("[%d] %s in %v", c.StatusCode, c.Req.RequestURI, time.Since(t))
	}
}
```

其中，

中间件是应用在`RouterGroup`上的，应用在最顶层的 Group即相当于作用于全局，此时所有的请求都会被中间件处理。

## Context 修改

之前的框架设计是：接收到请求，匹配路由，该请求的所有信息都保存在`Context`中。

中间件也不例外，接收到请求后，应查找所有应作用于该路由的中间件，保存在`Context`中，然后依次进行调用。

> 为什么依次调用后，还需要保存在`Context`中保存？因为中间件不仅作用在处理流程前，也可以作用在处理流程后，即在用户定义的 Handler 处理完毕后，还可以执行剩下的操作。

因此，`Context`新增了2个参数，并定义了`Next`方法：

[**geeWeb/context.go**]()

```go
type Context struct {
   // 原始对象
   Writer http.ResponseWriter
   Req    *http.Request
   // 请求信息
   Method string
   Path   string
   Params map[string]string
   // 回应信息
   StatusCode int
   // 中间件数组及其下标idx
   handlers []HandlerFunc
   idx      int
   // engine 指针
   engine *Engine
}

func newContext(w http.ResponseWriter, req *http.Request) *Context {
   return &Context{
      Path:   req.URL.Path,
      Method: req.Method,
      Req:    req,
      Writer: w,
      idx:    -1,
   }
}

```

`c.Next()`表示执行其他的中间件或 用户定义的`Handler`，支持设置多个中间件，依次进行调用.

```go
func (c *Context) Next() {
   for c.idx++; c.idx < len(c.handlers); c.idx++ {
      c.handlers[c.idx](c)
   }
}
```

`idx`记录当前执行到第几个中间件，当在中间件中调用`Next`方法时，控制权交给了下一个中间件，直到调用到最后一个中间件，然后再从后往前，调用每个中间件在执行`Next`方法之后的代码。

---

如果将用户在映射路由时定义的`Handler`添加到`c.handlers`列表中，结果会怎样？

```go
func A(c *Context) {
    part1
    c.Next()
    part2
}
func B(c *Context) {
    part3
    c.Next()
    part4
}
```

假设我们应用了中间件 A、B，以及路由映射的 Handler。

`c.handlers`为[A, B, Handler]，`c.idx`初始化为-1。调用`c.Next()`，接下来的流程如下：

- c.idx++，c.idx为 0
- 0 < 3，调用 c.handlers[0]，即 A
- 执行 part1，调用 c.Next()
- c.idx++，c.idx为 1
- 1 < 3，调用 c.handlers[1]，即 B
- 执行 part3，调用 c.Next()
- c.idx++，c.idx为 2
- 2 < 3，调用 c.handlers[2]，即Handler
- Handler 调用完毕，返回到 B 中的 part4，执行 part4
- part4 执行完毕，返回到 A 中的 part2，执行 part2
- part2 执行完毕，结束。

一句话说清楚重点，最终的顺序是`part1 -> part3 -> Handler -> part 4 -> part2`，满足了我们对中间件的要求。

## 其他部分代码修改

[**geeWeb/geeWeb.go**]()

- `Use`函数将中间件应用到某个 Group 。

```go
// Use 向分组增加中间件
func (group *RouterGroup) Use(middlewares ...HandlerFunc) {
   group.middlewares = append(group.middlewares, middlewares...)
}

```

- `ServeHTTP` 函数也有变化，当接收到一个具体请求时，要判断该请求适用于哪些中间件，这里简单通过 URL前缀判断。得到中间件列表后，赋值给 `c.handlers`。

```go
func (engine *Engine) ServeHTTP(w http.ResponseWriter, req *http.Request) {
	var middlewares []HandlerFunc
	for _, group := range engine.groups {
		if strings.HasPrefix(req.URL.Path, group.prefix) {
			middlewares = append(middlewares, group.middlewares...)
		}
	}
	c := newContext(w, req)
	c.handlers = middlewares
	engine.router.handle(c)
}
```

[**geeWeb/router.go**]()

- `handle` 函数中，将从路由匹配得到的 Handler 添加到 `c.handlers`列表中，最后执行`c.Next()`，执行所有中间件和匹配到的handler。

```go
func (r *router) handle(c *Context) {
   n, params := r.getRoute(c.Method, c.Path)

   if n != nil {
      key := c.Method + "-" + n.pattern
      c.Params = params
      c.handlers = append(c.handlers, r.handlers[key])
   } else {
      c.handlers = append(c.handlers, func(c *Context) {
         c.String(http.StatusNotFound, "404 NOT FOUND: %s\n", c.Path)
      })
   }

   c.Next()
}
```

## Demo

```go
func onlyForV2() gee.HandlerFunc {
	return func(c *gee.Context) {
		// Start timer
		t := time.Now()
		// if a server error occurred
		c.Fail(500, "Internal Server Error")
		// Calculate resolution time
		log.Printf("[%d] %s in %v for group v2", c.StatusCode, c.Req.RequestURI, time.Since(t))
	}
}

func main() {
	r := gee.New()
	r.Use(gee.Logger()) // global midlleware
	r.GET("/", func(c *gee.Context) {
		c.HTML(http.StatusOK, "<h1>Hello Gee</h1>")
	})

	v2 := r.Group("/v2")
	v2.Use(onlyForV2()) // v2 group middleware
	{
		v2.GET("/hello/:name", func(c *gee.Context) {
			// expect /hello/geektutu
			c.String(http.StatusOK, "hello %s, you're at %s\n", c.Param("name"), c.Path)
		})
	}

	r.Run(":9999")
}
```

```bash
$ curl http://localhost:9999/
>>> log
2019/08/17 01:37:38 [200] / in 3.14µs

(2) global + group middleware
$ curl http://localhost:9999/v2/hello/geektutu
>>> log
2019/08/17 01:38:48 [200] /v2/hello/geektutu in 61.467µs for group v2
2019/08/17 01:38:48 [200] /v2/hello/geektutu in 281µs
```



# 模板 HTML Template

## 服务端渲染

目前最流行的是前后端分离的开发模式，即 Web 后端提供 RESTful 接口，返回结构化的数据(通常为 JSON 或者 XML；前端使用 AJAX 技术请求到所需的数据，利用 JavaScript 进行渲染。

这种开发模式前后端解耦，优势突出。后端专心解决资源利用，并发，数据库等问题，只需要考虑数据如何生成；前端专注于界面设计实现，只需要考虑拿到数据后如何渲染即可。而且前后端分离还有另外一个不可忽视的优势。后端只关注于数据，接口返回值是结构化的，与前端解耦。同一套后端服务能够同时支撑小程序、移动APP、PC端 Web 页面。

但前后分离的一大问题在于，页面是在客户端渲染的，比如浏览器。这对于爬虫并不友好，Google 爬虫已经能够爬取渲染后的网页，但是短期内爬取服务端直接渲染的 HTML 页面仍是主流。

## 静态文件

要做到服务端渲染，第一步便是要支持 JS、CSS 等静态文件。

之前设计动态路由的时候，支持通配符`*`匹配多级子路径，

例如：路由规则`/assets/*filepath`，可以匹配`/assets/`开头的所有的地址，如`/assets/js/geektutu.js`，匹配后，参数`filepath`就赋值为`js/geektutu.js`。

将所有的静态文件放在`/usr/web`目录下，那么`filepath`的值即是该目录下文件的相对地址。映射到真实的文件后，将文件返回，静态服务器就实现了。

找到文件后，如何返回？，这一步，`net/http`库已经实现了。因此，gee 框架要做的，仅仅是解析请求的地址，映射到服务器上文件的真实地址，然后交给`http.FileServer`处理就好了。

本部分为`RouterGroup`添加了2个方法，其中`Static`这个方法是暴露给用户的。用户可以将磁盘上的某个文件夹`root`映射到路由`relativePath`。

[**geeWeb/geeWeb.go**]()

```go
// create static handler
func (group *RouterGroup) createStaticHandler(relativePath string, fs http.FileSystem) HandlerFunc {
   absolutePath := path.Join(group.prefix, relativePath)
   fileServer := http.StripPrefix(absolutePath, http.FileServer(fs))
   return func(c *Context) {
      file := c.Param("filepath")
      // Check if file exists and/or if we have permission to access it
      if _, err := fs.Open(file); err != nil {
         c.Status(http.StatusNotFound)
         return
      }

      fileServer.ServeHTTP(c.Writer, c.Req)
   }
}

// Static serve static files
func (group *RouterGroup) Static(relativePath string, root string) {
   handler := group.createStaticHandler(relativePath, http.Dir(root))
   urlPattern := path.Join(relativePath, "/*filepath")
   // Register GET handlers
   group.GET(urlPattern, handler)
}
```

- 使用样例：

```go
r := gee.New()
r.Static("/assets", "/usr/geektutu/blog/static")
// 或相对路径 r.Static("/assets", "./static")
r.Run(":9999")
```

用户访问`localhost:9999/assets/js/geektutu.js`，

最终返回`/usr/geektutu/blog/static/js/geektutu.js`。

## HTML 模板渲染

Go内置了`text/template`和`html/template`2个模板标准库。

其中[html/template](https://golang.org/pkg/html/template/)为 HTML 提供了较为完整的支持，包括普通变量渲染、列表渲染、对象渲染等。

gee 框架的模板渲染直接使用了`html/template`提供的能力。

[**geeWeb/geeWeb.go**]()

- 首先为 Engine 示例添加了 `*template.Template` 和 `template.FuncMap`对象，前者将所有的模板加载进内存，后者是所有的自定义模板渲染函数。]
- 另外，给用户分别提供了设置自定义渲染函数`funcMap`和加载模板`LoadHTMLGlob`方法。

```go
Engine struct {
   *RouterGroup
   router        *router
   groups        []*RouterGroup     // store all groups
   htmlTemplates *template.Template // for html render
   funcMap       template.FuncMap   // for html render
}

// SetFuncMap for custom render function
func (engine *Engine) SetFuncMap(funcMap template.FuncMap) {
	engine.funcMap = funcMap
}

func (engine *Engine) LoadHTMLGlob(pattern string) {
	engine.htmlTemplates = template.Must(template.New("").Funcs(engine.funcMap).ParseGlob(pattern))
}
```

[**geeWeb/context.go**]()

- 接下来，对原来的 `(*Context).HTML()`方法做了些小修改，使之支持根据模板文件名选择模板进行渲染。

```go
// HTML 构造返回HTML的HTTP响应
// HTML template render
// refer https://golang.org/pkg/html/template/
func (c *Context) HTML(code int, name string, data interface{}) {
   c.SetHeader("Content-Type", "text/html")
   c.Status(code)
   if err := c.engine.htmlTemplates.ExecuteTemplate(c.Writer, name, data); err != nil {
      c.Fail(500, err.Error())
   }
}
```

- 并且，我们又在 `Context` 中添加了成员变量 `engine *Engine`，这样就能够通过 Context 访问 Engine 中的 HTML 模板。

```go
type Context struct {
   // 原始对象
   Writer http.ResponseWriter
   Req    *http.Request
   // 请求信息
   Method string
   Path   string
   Params map[string]string
   // 回应信息
   StatusCode int
   // 中间件数组及其下标idx
   handlers []HandlerFunc
   idx      int
   // engine 指针
   engine *Engine
}
```

[**geeWeb/geeWeb.go**]()

在geeWeb实例中，实例化 Context 的时候，要记得给 `c.engine` 赋值。

```go
func (engine *Engine) ServeHTTP(w http.ResponseWriter, req *http.Request) {
	// ...
	c := newContext(w, req)
	c.handlers = middlewares
	c.engine = engine
	engine.router.handle(c)
}
```

## demo

```go
type student struct {
	Name string
	Age  int8
}

func FormatAsDate(t time.Time) string {
	year, month, day := t.Date()
	return fmt.Sprintf("%d-%02d-%02d", year, month, day)
}

func main() {
	r := gee.New()
	r.Use(gee.Logger())
	r.SetFuncMap(template.FuncMap{
		"FormatAsDate": FormatAsDate,
	})
	r.LoadHTMLGlob("templates/*")
	r.Static("/assets", "./static")

	stu1 := &student{Name: "Geektutu", Age: 20}
	stu2 := &student{Name: "Jack", Age: 22}
	r.GET("/", func(c *gee.Context) {
		c.HTML(http.StatusOK, "css.tmpl", nil)
	})
	r.GET("/students", func(c *gee.Context) {
		c.HTML(http.StatusOK, "arr.tmpl", gee.H{
			"title":  "gee",
			"stuArr": [2]*student{stu1, stu2},
		})
	})

	r.GET("/date", func(c *gee.Context) {
		c.HTML(http.StatusOK, "custom_func.tmpl", gee.H{
			"title": "gee",
			"now":   time.Date(2019, 8, 17, 0, 0, 0, 0, time.UTC),
		})
	})

	r.Run(":9999")
}
```

然后浏览器访问主页，模板正常渲染，CSS 静态文件加载成功。

# 错误恢复 Panic Recover

之前实现了中间件机制，错误处理也可以作为一个中间件，增强 gee 框架的能力。

新增文件 **gee/recovery.go**，在这个文件中实现中间件 `Recovery`。

```go
package gee

import (
	"fmt"
	"log"
	"net/http"
	"runtime"
	"strings"
)

// print stack trace for debug
func trace(message string) string {
	var pcs [32]uintptr
	n := runtime.Callers(3, pcs[:]) // skip first 3 caller

	var str strings.Builder
	str.WriteString(message + "\nTraceback:")
	for _, pc := range pcs[:n] {
		fn := runtime.FuncForPC(pc)
		file, line := fn.FileLine(pc)
		str.WriteString(fmt.Sprintf("\n\t%s:%d", file, line))
	}
	return str.String()
}

func Recovery() HandlerFunc {
	return func(c *Context) {
		defer func() {
			if err := recover(); err != nil {
				message := fmt.Sprintf("%s", err)
				log.Printf("%s\n\n", trace(message))
				c.Fail(http.StatusInternalServerError, "Internal Server Error")
			}
		}()

		c.Next()
	}
}
```

- `Recovery` 的实现非常简单，使用 defer 挂载上错误恢复的函数，在这个函数中调用 `recover()`，捕获 panic，并且将堆栈信息打印在日志中，向用户返回 *Internal Server Error*。

-  `trace()` 函数，这个函数是用来获取触发 panic 的堆栈信息。

  其中，调用了 `runtime.Callers(3, pcs[:])`，Callers 用来返回调用栈的程序计数器, 第 0 个 Caller 是 Callers 本身，第 1 个是上一层 trace，第 2 个是再上一层的 `defer func`。因此，为了日志简洁一点，跳过了前 3 个 Caller。

  接下来，通过 `runtime.FuncForPC(pc)` 获取对应的函数，在通过 `fn.FileLine(pc)` 获取到调用该函数的文件名和行号，打印在日志中。

## Demo

```go
package main

import (
	"net/http"

	"gee"
)

func main() {
	r := gee.Default()
	r.GET("/", func(c *gee.Context) {
		c.String(http.StatusOK, "Hello Geektutu\n")
	})
	// index out of range for testing Recovery()
	r.GET("/panic", func(c *gee.Context) {
		names := []string{"geektutu"}
		c.String(http.StatusOK, names[100])
	})

	r.Run(":9999")
}
```

```bash
$ curl "http://localhost:9999"
Hello Geektutu
$ curl "http://localhost:9999/panic"
{"message":"Internal Server Error"}
$ curl "http://localhost:9999"
Hello Geektutu
```

我们可以在后台日志中看到如下内容，引发错误的原因和堆栈信息都被打印了出来，通过日志，我们可以很容易地知道，在*day7-panic-recover/main.go:47* 的地方出现了 `index out of range` 错误。

```bash
2020/01/09 01:00:10 Route  GET - /
2020/01/09 01:00:10 Route  GET - /panic
2020/01/09 01:00:22 [200] / in 25.364µs
2020/01/09 01:00:32 runtime error: index out of range
Traceback:
        /usr/local/Cellar/go/1.12.5/libexec/src/runtime/panic.go:523
        /usr/local/Cellar/go/1.12.5/libexec/src/runtime/panic.go:44
        /tmp/7days-golang/day7-panic-recover/main.go:47
        /tmp/7days-golang/day7-panic-recover/gee/context.go:41
        /tmp/7days-golang/day7-panic-recover/gee/recovery.go:37
        /tmp/7days-golang/day7-panic-recover/gee/context.go:41
        /tmp/7days-golang/day7-panic-recover/gee/logger.go:15
        /tmp/7days-golang/day7-panic-recover/gee/context.go:41
        /tmp/7days-golang/day7-panic-recover/gee/router.go:99
        /tmp/7days-golang/day7-panic-recover/gee/gee.go:130
        /usr/local/Cellar/go/1.12.5/libexec/src/net/http/server.go:2775
        /usr/local/Cellar/go/1.12.5/libexec/src/net/http/server.go:1879
        /usr/local/Cellar/go/1.12.5/libexec/src/runtime/asm_amd64.s:1338

2020/01/09 01:00:32 [500] /panic in 395.846µs
2020/01/09 01:00:38 [200] / in 6.985µs
```

