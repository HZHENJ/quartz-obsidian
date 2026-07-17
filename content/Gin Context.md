%% [[Gin]] %%
%% [[Gin Engine]] %%
```Go
type Context struct {
	writermem responseWriter
	Request   *http.Request
	Writer    ResponseWriter
	
	Params   Params
	handlers HandlersChain
	index    int8
	fullPath string
	
	engine       *Engine
	params       *Params
	skippedNodes *[]skippedNode
	
	mu sync.RWMutex
	
	Keys map[any]any
	Errors errorMsgs
	Accepted []string
	
	queryCache url.Values
	formCache url.Values
	sameSite http.SameSite
}
```
Context可以理解为一次HTTP请求从进入Gin到响应结束期间的“请求上下文对象“。它把请求、响应、路径参数、中间件状态、错误和请求级数据等内容集中封装在一个对象中。
# 输入输出

```Go
writermem responseWriter // 值类型，存在 Context 内部
Request *http.Request // 标准库的请求对象
Writer ResponseWriter // 接口，指向 &writermem
Params Params // 路径参数，如 /user/:id → {id: "123"}
```
# 中间件控制
```Go
type Context struct {
	...
	handlers HandlersChain   // 待执行的中间件 + handler 列表
	index    int8            // 当前执行到第几个，初始 -1
	fullPath string          // 匹配到的路由模板，如 "/user/:id"
	...
}
```
这三个字段共同实现了Gin的洋葱模型中间件，来看`Next()`，每个handler内部如果调了，`c.Next()`，就会继续往下走，然后回来继续执行`c.Next()`。
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
把 `index` 设成一个很大的值（63），后续 `Next()` 里的 `for c.index < len(c.handlers)` 就不成立了——后续 handler 全部跳过。
```Go
// abortIndex represents a typical value used in abort functions.
const abortIndex int8 = math.MaxInt8 >> 1 // 63
...
func (c *Context) Abort() {
	c.index = abortIndex
}
```
# 数据存储
```Go
Keys map[any]any // c.Set("user", userObj) / c.Get("user")
Errors errorMsgs // c.Error(err) 积累的错误
Accepted []string // c.NegotiateFormat() 的内容协商结果
queryCache url.Values // c.Request.URL.Query() 的缓存
formCache url.Values // c.Request.PostForm 的缓存
sameSite http.SameSite // Cookie 的 SameSite 属性
```
# 内部引用
```Go
engine *Engine // 指回 Engine，访问配置和资源
params *Params // 指针，指向预分配的底层数组，路由匹配时覆盖写入
skippedNodes *[]skippedNode // 路由匹配时记录跳过的节点（用于大小写不敏感查找）
mu sync.RWMutex // 保护 Keys map 的并发读写

Params Params // c.Params，你平时用的 c.Param("id")
params *Params // 指向底层数组，路由匹配时 root.getValue(..., c.params, ...) 直接往里写
```