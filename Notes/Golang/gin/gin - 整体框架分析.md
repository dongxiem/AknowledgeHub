# 1.Gin 简明介绍

gin 是什么？官方给的解释为：Gin 是一个用 Go (Golang) 编写的 HTTP web 框架。 它是一个类似于 [martini](https://github.com/go-martini/martini) 但拥有更好性能的 API 框架, 优于 [httprouter](https://github.com/julienschmidt/httprouter)，速度提高了近 40 倍。

> Gin is a HTTP web framework written in Go (Golang). It features a Martini-like API with much better performance – up to 40 times faster. If you need smashing performance, get yourself some Gin.

![Gin](https://garmen-imgsubmit.oss-cn-shenzhen.aliyuncs.com/img/20200922104010.jpeg)



gin 有一下的一些特性：

1. 快速：基于 Radix 树的路由，小内存占用。没有反射。可预测的 API 性能。
2. 支持中间件：传入的 HTTP 请求可以由一系列中间件和最终操作来处理。 例如：Logger，Authorization，GZIP，最终操作 DB。
3. Crash 处理：Gin 可以 catch 一个发生在 HTTP 请求中的 panic 并 recover 它。这样，你的服务器将始终可用。例如，你可以向 Sentry 报告这个 panic！
4. JSON 验证：Gin 可以解析并验证请求的 JSON，例如检查所需值的存在。
5. 路由组：更好地组织路由。是否需要授权，不同的 API 版本…… 此外，这些组可以无限制地嵌套而不会降低性能。
6. 错误管理：Gin 提供了一种方便的方法来收集 HTTP 请求期间发生的所有错误。最终，中间件可以将它们写入日志文件，数据库并通过网络发送。
7. 内置渲染：Gin 为 JSON，XML 和 HTML 渲染提供了易于使用的 API。
8. 可扩展性：新建一个中间件非常简单，去查看示例代码吧。



<!--more-->

在学习 gin 之前我也用过 golang 自带的`net/http`来实现一个HTTP服务。，虽然`net/http`看着很便捷、很简单，但是它也存在很多不足：

1. 不能单独的对请求方法(POST,GET等)注册特定的处理函数
2. 不支持Path变量参数
3. 不能很很好的获取参数
4. 不支持参数校验
5. 不支持参数绑定
6. 不能更好的多种格式输出
7. 性能一般
8. 扩展性不足
9. ……

而 gin 作为这么一个那么优秀的 web 框架，弥补了`net/http`的一些不足，同时还增加了很多日常Web开发使用的功能，性能也这么好，可以让我们更好的进行Web开发，这就让人很有理由去学习这个框架了啊！

注：以下代码有下横线的地方，我将源码地址链接上了，使用`ctrl+鼠标左键`即可在浏览器中展开相对应的代码片段，如果Chrome安装插件 [Sourcegraph](https://chrome.google.com/webstore/detail/sourcegraph/dgjhfomjieaadpoljlnidmbgkdffpack) 会发现更加便捷地查看 Github 中的代码。



---

# 2.快速入门

我们可以从官网给的一个简单的入门程序来看看其大概是如何工作的！

```go
package main

import "github.com/gin-gonic/gin"

func main() {
    r := gin.Default()
    r.GET("/ping", func(c *gin.Context) {
        c.JSON(200, gin.H{
            "message": "pong",
        })
    })
    r.Run() // 监听并在 0.0.0.0:8080 上启动服务
}
```

关于上面的一些解释：

1. 首先，我们使用了`gin.Default()`生成了一个实例，这个实例即 WSGI 应用程序。
2. 接下来，我们使用`r.GET("/", ...)`声明了一个路由，告诉 Gin 什么样的URL 能触发传入的函数，这个函数返回我们想要显示在用户浏览器中的信息。
3. 最后用 `r.Run()`函数来让应用运行在本地服务器上，默认监听端口是 _8080_，可以传入参数设置端口，例如`r.Run(":9999")`即运行在 _9999_端口。

运行之后打印如下：

```go
[GIN-debug] [WARNING] Creating an Engine instance with the Logger and Recovery middleware already attached.

[GIN-debug] [WARNING] Running in "debug" mode. Switch to "release" mode in production.
 - using env:	export GIN_MODE=release
 - using code:	gin.SetMode(gin.ReleaseMode)
```

我们使用浏览器访问：http://localhost:8080，可以看到

![image-20200921101745940](https://garmen-imgsubmit.oss-cn-shenzhen.aliyuncs.com/img/20200921101746.png)

我们可以看见官方给出的示例非常简单了，但是就是这么简单的代码，包含了Gin的核心调用顺序：

1. 找到第一个入口 [gin.Default()](https://sourcegraph.com/github.com/gin-gonic/gin/-/blob/gin.go#L159:1)，由此展开整个gin框架。
2. 第二关键处：[r.Run()](https://github.com/gin-gonic/gin/blob/master/gin.go#L305:23) ，后续会逐步分析到。
3. 通过这2个关键入口，庖丁解牛，把整个 gin 框架拆解开。



---

# 3.框架分析

首先对于[gin.Default()](https://sourcegraph.com/github.com/gin-gonic/gin/-/blob/gin.go#L159:1)，其代码片段如下：

```go
// Default 会返回已连接Logger和Recovery中间件的Engine实例。
func Default() *Engine {
    debugPrintWARNINGDefault()
    // 重点关注这个New()
    engine := New()
    engine.Use(Logger(), Recovery())
    return engine
}
```

从上面的Default()，我们可以看到代码其实很短，其核心在于[New()](https://sourcegraph.com/github.com/gin-gonic/gin/-/blob/gin.go#L129:6)，让我们看看如何新创建一个引擎，代码片段如下：

```go
// New 会返回一个新的空白Engine实例，不附加任何中间件。
// 默认的配置为:
// - RedirectTrailingSlash:  true
// - RedirectFixedPath:      false
// - HandleMethodNotAllowed: false
// - ForwardedByClientIP:    true
// - UseRawPath:             false
// - UnescapePathValues:     true
func New() *Engine {
    debugPrintWARNINGNew()
    // 1.初始化框架对象
    engine := &Engine{
        RouterGroup: RouterGroup{
            Handlers: nil,
            basePath: "/",
            root:     true,
        },
        FuncMap:                template.FuncMap{},
        RedirectTrailingSlash:  true,
        RedirectFixedPath:      false,
        HandleMethodNotAllowed: false,
        ForwardedByClientIP:    true,
        AppEngine:              defaultAppEngine,
        UseRawPath:             false,
        RemoveExtraSlash:       false,
        UnescapePathValues:     true,
        MaxMultipartMemory:     defaultMultipartMemory,
        trees:                  make(methodTrees, 0, 9),
        delims:                 render.Delims{Left: "{{", Right: "}}"},
        secureJSONPrefix:       "while(1);",
    }
    engine.RouterGroup.engine = engine
    // 2.初始化pool
    engine.pool.New = func() interface{} {
        // 关键调用: 初始化上下文对象
        return engine.allocateContext()
    }
    return engine
}
```

这些配置看起来有点摸不着头脑？没关系，我们继续分析！

我们可以看看这个[Engine](https://sourcegraph.com/github.com/gin-gonic/gin/-/blob/gin.go#L56:6)里面到底藏这些什么？按照来看，其应该为**<u>最核心的部分了！</u>**，其对应的代码片段如下：

```go
type Engine struct {
    // 路由组
    RouterGroup
    // 开启自动重定向开关
    RedirectTrailingSlash bool
    // 重定向修复当前请求路径开关
    RedirectFixedPath bool
    // Method Not Allowed 处理方法开关
    HandleMethodNotAllowed bool

    ForwardedByClientIP    bool

    AppEngine bool
    // 参数寻找开关
    UseRawPath bool
    // 取消转义开关
    UnescapePathValues bool

    MaxMultipartMemory int64

    RemoveExtraSlash bool

    delims           render.Delims
    secureJSONPrefix string
    HTMLRender       render.HTMLRender
    FuncMap          template.FuncMap
    allNoRoute       HandlersChain
    allNoMethod      HandlersChain
    noRoute          HandlersChain
    noMethod         HandlersChain
    pool             sync.Pool 			// 临时对象池
    trees            methodTrees
    maxParams        uint16
}
```



`Engine`是框架的实例，它包含`muxer`、中间件和还有一些配置设置。它可以通过`New()`或者`Default()`来创建一个`Engine`实例。

关于上面的几个参数，具体解释如下：

1. `RouterGroup`

   -  路由组

2. `RedirectTrailingSlash`

   - 这个是开启自动重定向（Enables automatic redirection）的开关，如果当前路由不能匹配尾斜杆的路径情况下可以开启。
   - 比如需要请求`/foo/`，但是只存在`/foo`这个路由，则可以通过开启重定向进行定位了。

3. `RedirectFixedPath`

   - 如果启用，则路由器将尝试修复当前请求路径（如果未为其注册句柄）。
   - 首先会删除多余的路径元素，例如../或//，然后不区分大小写。
   - 经过上面的处理之后，还有相对应的路径存在则进行重定向到已更正的路径。
   - 例如：`/ FOO`和 `/..//Foo` 可以重定向到`/ foo`。

4. `HandleMethodNotAllowed`

   - 如果启用，当发出的当前请求不能进行路由请求时，也即请求收到的回复是405(Method Not Allowed)时，路由器会去检查是否有其他方法能够允许当前请求（比如Post回复405，会尝试Get能不能行得通）
   - 如果其他方法也不被允许，该请求会委派给 `NotFound` 处理。

5. `UseRawPath`

   - 如果启用该开关，则 `url.RawPath` 会被用来寻找参数

6. `UnescapePathValues`

   - 如果为True，则路径值将被取消转义。
   - 需要注意的是：如果上面的 `UseRawPath` 为false，则 `UnescapePathValues` 实际上是为True。而且此时 `url.Path` 是不可替代的，会使用原`url`的路径进行处理。

7. `MaxMultipartMemory`

   - 该值是给 `http.Request's ParseMultipartForm` 这个方法进行调用的。

8. `RemoveExtraSlash`

   - `RemoveExtraSlash` 是可以从URL解析的参数，即使使用额外的斜杠也是如此。

9. `pool`

   - 临时对象池：用于处理 context 

   

----



再来对于[r.Run()](https://github.com/gin-gonic/gin/blob/master/gin.go#L305:23)，这个一看就显得很明显了，是启动的方法了，**<u>具体是如何启动的呢？</u>**看看接下来的代码片段：

```go
// 注意：除非发生错误，否则该方法将无限期地阻止调用goroutine。
func (engine *Engine) Run(addr ...string) (err error) {
    defer func() { debugPrintError(err) }()
    // 1.获取端口值。
    address := resolveAddress(addr)
    debugPrint("Listening and serving HTTP on %s\n", address)
    // 2.使用 标准库 `http.ListenAndServe()` 启动 web 监听服务，处理HTTP请求。
    err = http.ListenAndServe(address, engine)
    return
}
```

Run方法会将路由器连接到`http`服务器并开始监听和服务HTTP请求，主要还是使用了 `http.ListenAndServe(addr, router)` 这个方法了。这里面具体就做了两件事：

1. 获取端口值。
2. 使用 标准库 `http.ListenAndServe()` 启动 web 监听服务，处理HTTP请求。

首先让我们先来看看如何获取端口值的，[Run.resolveAddress(addr)](https://sourcegraph.com/github.com/gin-gonic/gin@master/-/blob/utils.go#L136:21) 的具体代码片段如下：

```go
func resolveAddress(addr []string) string {
    switch len(addr) {
        case 0:
        if port := os.Getenv("PORT"); port != "" {
            debugPrint("Environment variable PORT=\"%s\"", port)
            return ":" + port
        }
        debugPrint("Environment variable PORT is undefined. Using port :8080 by default")
        return ":8080"
        case 1:
        return addr[0]
        default:
        panic("too many parameters")
    }
}
```

可以看到这个方法会传进来一个切片，去**<u>对该切片的长度进行判断</u>**：

1. 如果切片长度为0，证明使用户并未给出端口值，那么则使用HTTP默认的`"8080"`端口。
2. 如果用户给出了自己想要使用的端口，则直接使用该端口。
3. 如果传入的切片长度不为0也不为1，则证明传入了多个端口值了，返回一个Panic。



然后再看看 [http.ListenAndServe()](https://sourcegraph.com/github.com/golang/go@master/-/blob/src/net/http/server.go?utm_source=chrome-extension#L3136:6) 这个方法，其需要传入了两个参数，第一个参数我们上面也分析了为`address` 也即端口地址，第二个参数为`engine`，这是干嘛用的？我也不知道，得看看下面的源码才能知道，不过想想，引擎是干嘛的？我对引擎的理解也就是让它去自动去获取一些参数，然后再自动进行一些处理。

这里涉及到了 go 的标准库`net.http` 包，其代码片段如下：

```go
// ListAndServe 始终返回非零错误
func ListenAndServe(addr string, handler Handler) error {
    server := &Server{Addr: addr, Handler: handler}
    return server.ListenAndServe()
}
```

[http.ListenAndServe()](https://sourcegraph.com/github.com/golang/go@master/-/blob/src/net/http/server.go?utm_source=chrome-extension#L3136:6) 做了些什么呢？首先  [http.ListenAndServe()](https://sourcegraph.com/github.com/golang/go@master/-/blob/src/net/http/server.go?utm_source=chrome-extension#L3136:6) 监听一个 TCP 网络地址，然后调用 Serve 的 handler 去处理即将到来连接的请求，而且接受的连接配置为启用TCP保持活动状态（`keep-alives`）。

需要注意的是handler 通常为 nil，而且这种情况下 `DefaultServeMux` 会被使用，我们还需要留意到这个`Handler`，这就是传入的`egine`所对应的吗？

然后发现 [Handler](https://sourcegraph.com/github.com/golang/go@b4ea67200977b99ede1885ed77e034a2fdf434f5/-/blob/src/net/http/server.go#L86) 其实是个接口类型，其具体的代码片段如下：

```go
type Handler interface {
    ServeHTTP(ResponseWriter, *Request)
}
```

首先，我们可以很清晰的知道了**<u>`Handler {}` 是个接口类型</u>**，然后会发现`Handler` 内部<u>只有一个 `ServeHTTP()` 方法声明</u>，此时我们再返回到 `gin`的源码去寻找是否有关于 `ServeHTTP()` 方法的实现，果不其然真的有 [ServeHTTP()](https://sourcegraph.com/github.com/gin-gonic/gin@master/-/blob/gin.go#L370:1) 的实现，而关于接口的知识，假如有疑惑，可以去翻翻书或者找一些博客，又或者等我这个曾经也带有疑惑的人给你解答，此处插个旗。

话说回来，gin 是如何实现 [ServeHTTP()](https://sourcegraph.com/github.com/gin-gonic/gin@master/-/blob/gin.go#L370:1) 的呢？其代码片段如下：

```go
// ServeHTTP conforms to the http.Handler interface.
func (engine *Engine) ServeHTTP(w http.ResponseWriter, req *http.Request) {
    // 1.从临时对象池 pool 中获取 context 上下文对象
    c := engine.pool.Get().(*Context)
    c.writermem.reset(w)
    c.Request = req
    c.reset()
    // 2.处理 HTTP 请求
    engine.handleHTTPRequest(c)
    // 3.使用完 context 对象, 将其归还给 pool 
    engine.pool.Put(c)
}
```

通过源码的注释我们可以知道， `struct gin.Engine {}` 是 `interface http.Handler{}` 接口的实现，而关键方法[ServeHTTP()](https://sourcegraph.com/github.com/gin-gonic/gin@master/-/blob/gin.go#L370:1) 是符合 [http.Handler](https://sourcegraph.com/github.com/golang/go@b4ea67200977b99ede1885ed77e034a2fdf434f5/-/blob/src/net/http/server.go#L86) 接口的方法，可不要小看这个方法了，这个方法可以说是 gin 的核心代码了！

它主要就是做了三件事，上面代码也给了注释了：

1. 从临时对象池 pool 中获取 `context` 上下文对象，对该 `context` 进行一些处理。
2. 处理 `Http` 请求。
3. 使用完 `context` 对象, 将其归还给 `pool`。

关于 Context 的一些具体我在以后会具体单独拉出来讲，这是 `golang` 中很重要的一个部分，也是这个 gin 中很重要的一部分了。这里先对第二步请求 `Http` 请求的方法 [gin.handleHTTPRequest()](https://sourcegraph.com/github.com/gin-gonic/gin@master/-/blob/gin.go#L392:15) 重点了解一下，其代码稍长，代码片段如下：

```go
func (engine *Engine) handleHTTPRequest(c *Context) {
    httpMethod := c.Request.Method
    rPath := c.Request.URL.Path
    unescape := false
    if engine.UseRawPath && len(c.Request.URL.RawPath) > 0 {
        rPath = c.Request.URL.RawPath
        unescape = engine.UnescapePathValues
    }

    if engine.RemoveExtraSlash {
        rPath = cleanPath(rPath)
    }

    // Find root of the tree for the given HTTP method
    t := engine.trees
    for i, tl := 0, len(t); i < tl; i++ {
        if t[i].method != httpMethod {
            continue
        }
        root := t[i].root
        // Find route in tree
        value := root.getValue(rPath, c.params, unescape)
        if value.params != nil {
            c.Params = *value.params
        }
        if value.handlers != nil {
            c.handlers = value.handlers
            c.fullPath = value.fullPath
            // 主要看这里
            c.Next()
            c.writermem.WriteHeaderNow()
            return
        }
        if httpMethod != "CONNECT" && rPath != "/" {
            if value.tsr && engine.RedirectTrailingSlash {
                redirectTrailingSlash(c)
                return
            }
            if engine.RedirectFixedPath && redirectFixedPath(c, root, engine.RedirectFixedPath) {
                return
            }
        }
        break
    }

    if engine.HandleMethodNotAllowed {
        for _, tree := range engine.trees {
            if tree.method == httpMethod {
                continue
            }
            if value := tree.root.getValue(rPath, nil, unescape); value.handlers != nil {
                c.handlers = engine.allNoMethod
                serveError(c, http.StatusMethodNotAllowed, default405Body)
                return
            }
        }
    }
    c.handlers = engine.allNoRoute
    serveError(c, http.StatusNotFound, default404Body)
}
```

其中，[c.Next()](https://sourcegraph.com/github.com/gin-gonic/gin@master/-/blob/context.go#L162:19) 是最核心的代码： c是 `Context` 对象，这就引出了 `Context` 的实现细节。没办法，只能继续查看 [c.Next()](https://sourcegraph.com/github.com/gin-gonic/gin@master/-/blob/context.go#L162:19) 完成了一些什么事情，其相对于的代码片段如下：

```go
// Next 应该只在中间件内部使用 
// Next 在正在调用中的 handler 执行在链中(in the chain)挂起的处理程序(pending handlers)
func (c *Context) Next() {
    c.index++
    // 逐个遍历，根据不同的参数 c 执行 handle 方法
    for c.index < int8(len(c.handlers)) {
        c.handlers[c.index](c)
        c.index++
    }
}
```



可以知道，主要的进行处理的代码就在这一块了。最开始的时候 `c.index` 为0值，所以会执行 `c.handlers` 里面的第一个handler，然后一个个执行下去。

 



---

# 他山之石

- [gin](https://github.com/gin-gonic/gin)
- [gin中文文档](https://gin-gonic.com/zh-cn/docs/)
- [Golang Gin 实战（一）| 快速安装入门](https://www.flysnow.org/2019/12/10/golang-gin-quick-start.html)



