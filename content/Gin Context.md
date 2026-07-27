%% [[Gin]] %%
%% [[Gin Engine]] %%

```Go
type Context struct {
	// HTTP 原语
	writermem responseWriter
	Request   *http.Request
	Writer    ResponseWriter
	
	// 路由匹配结果
	Params   Params
	handlers HandlersChain
	index    int8
	fullPath string
	
	// 引擎反向引用与路由内部状态
	engine       *Engine
	params       *Params
	skippedNodes *[]skippedNode
	
	// 跨中间件数据共享
	mu sync.RWMutex
	Keys map[any]any
	
	// 错误收集
	Errors errorMsgs
	
	// 内容协商
	Accepted []string
	
	// 查询/表单缓存
	queryCache url.Values
	formCache url.Values
	
	// Cookie
	sameSite http.SameSite
}
```

对Web服务来说，无非是根据请求\*http.Request，构造响应http.ResponseWriter。但是这两个对象提供的接口粒度太细，比如我们要构造一个完整的响应，需要考虑消息头(Header)和消息体(Body)，而 Header 包含了状态码(StatusCode)，消息类型(ContentType)等几乎每次请求都需要设置的信息。因此，如果不进行有效的封装，那么框架的用户将需要写大量重复，繁杂的代码，而且容易出错。针对常用场景，能够高效地构造出 HTTP 响应是一个好的框架必须考虑的点。

针对使用场景，封装\*http.Request和http.ResponseWriter的方法，简化相关接口的调用，只是设计 Context 的原因之一。对于框架来说，还需要支撑额外的功能。例如，将来解析动态路由`/hello/:name`，参数`:name`的值放在哪呢？再比如，框架需要支持中间件，那中间件产生的信息放在哪呢？Context 随着每一个请求的出现而产生，请求的结束而销毁，和当前请求强相关的信息都应由 Context 承载。因此，设计 Context 结构，扩展性和复杂性留在了内部，而对外简化了接口。路由的处理函数，以及将要实现的中间件，参数都统一使用 Context 实例， Context 就像一次会话的百宝箱，可以找到任何东西。

Go 标准库的 HTTP 处理模型极其简单

```Go
type Handler interface {
	ServeHTTP(ResponseWriter, *Request)
}
```

当开始写Web应用的时候，就会涌现出很多问题：路由参数在哪？ /user/123 里的 "123" 怎么拿到？认证中间件怎么把 "当前用户" 传给这个 handler？怎么把 JSON body 解析成结构体？怎么统一处理 panic？怎么记录这次请求的耗时？

Context可以理解为一次HTTP请求从进入Gin到响应结束期间的“请求上下文对象“。它把请求、响应、路径参数、中间件状态、错误和请求级数据等内容集中封装在一个对象中。

Gin 的 Handler 签名因此变成：

```Go
type HandlerFunc func(*Context) // 标准库是 func(ResponseWriter, *Request)
```

从职责角度看，Context 扮演了三个角色：

| 角色    | 职责       | 关键方法                              |
| ----- | -------- | --------------------------------- |
| 输入管理器 | 从请求中提取数据 | Param, Query, Bind, FormFile      |
| 流程控制器 | 驱动中间件链执行 | Next, Abort, IsAborted            |
| 输出管理器 | 构造并发送响应  | JSON, XML, HTML, String, Redirect |

这三个角色并不是割裂的，它们共享同一个 Context 实例。这意味着：流程控制器可以在任意时刻中止输入/输出的处理，输入管理器提取的数据可以影响输出管理器的行为。

# 流程控制器

- handlers 函数数组
- index 指针

```Go
type Context struct {
	handlers HandlersChain
	index int8 
}
```

`Next()` 不是简单地"调用下一个函数"，而是在**递归推进指针**。每个中间件内部调用 `c.Next()` 时，控制权交给内层，内层执行完毕后**回到外层继续执行**。这就是经典的洋葱模型。

```Go
func (c *Context) Next() {
	c.index++
	for c.index < safeInt8(len(c.handlers)) {
		if c.handlers[c.index] != nil {
			c.handlers[c.index](c)
		}
		c.index++
	}
}
```

那么Abort如何阻止后续执行，Abort 只是把指针跳到 63。

为什么是 63？因为 int8 最大 127，`>>1` 得 63。任何合理的 handler 链长度都不会超过 63（加上 Abort 后 index 还会被 `Next()` 的 `index++` 再递增）。

所以跳过去之后，`Next()` 的循环条件 `c.index < len(c.handlers)` 永远不成立，后续 handler 全部跳过。

```Go
const abortIndex int8 = math.MaxInt8 >> 1 // = 63

func (c *Context) Abort() {
	c.index = abortIndex
}
```

# 输入管理器

三种情况：

1. 路由参数：从URL到Context
2. Query参数
3. Bind

路由匹配的细节会在Router里面详细讲

## 1. 路由参数

假设注册路由是：`GET /user/:id/order/:oid` 请求路由是：`GET /user/123/order/456`，通过Radix Tree匹配成功后，直接把解析好的参数切片写入 Context。`c.Params = [{Key:"id", Value:"123"}, {Key:"oid", Value:"456"}]`，之后业务代码通过 `c.Param("id")` 获取。

**为什么 Params 是切片而不是 map？** 因为路由参数很少（通常 1-3 个），切片的线性搜索比 map 的哈希计算更快，且内存开销更小。

## 2.Query参数

标准库每次调用`r.URL.Query().Get(key)`都会重新解析整个 query string 并 `map.clone()`。
三次调用 = 三次解析 + 三次 clone。Gin 的做法：**解析一次，缓存起来，后续零开销读取。**

整个 Query 参数体系只有一个缓存字段`c.queryCache` 和一个初始化入口`initQueryCache()`，其余方法都是对这张表的读取封装：

- `Query(key)`string 有就取，不区分"值为空"和"不存在"
- `DefaultQuery(k, d)` string 不存在时返回默认值
- `GetQuery(key)` (string, bool) 区分"值为空"和"不存在"
- `QueryArray(key)` \[]string 取同名参数的完整列表
- `GetQueryArray(key)` (\[]string, bool) 同上 + 区分是否存在
- `QueryMap(key)` map\[string]string   解析方括号语法 user\[name]=John
- `GetQueryMap(key)` (map\[string]string, bool)

所有方法最终都落到 GetQueryArray，它直接从缓存 map 里取值。GetQuery调用 GetQueryArray只取\[0]，Query 调用 GetQuery扔掉bool，层层包装、各司其职。

核心机制：懒加载，只在首次访问的时候执行

```Go
func (c *Context) initQueryCache() {
	if c.queryCache == nil {
		if c.Request != nil && c.Request.URL != nil {
			c.queryCache = c.Request.URL.Query()
		} else {
			c.queryCache = url.Values{}
		}
	}
}
```

## 3.Bind

Query 和 Param 解决的是"拿单个值"的问题，但你最终要的是把请求数据填充到一个结构体里，那么不管数据来源是什么（JSON body、query string、表单、URI 参数），都用同一套结构体 tag 来表达映射关系，用同一个方法签名来执行绑定。

# 输入输出

输出管理器解决的核心问题：**怎么把 Go 的数据结构变成 HTTP 响应发出去？** 标准库只给了`w.Write([]byte)`，所有格式转换都要手动做。Gin 的目标是`c.JSON(200, data)`一行搞定。


