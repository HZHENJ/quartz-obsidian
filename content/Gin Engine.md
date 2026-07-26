%% [[Gin]] %%
# Engine

Engine是Gin框架的核心：

- 运行前：注册路由，配置中间件
- 运行时：通过`ServeHTTP()`接受请求，路由匹配，执行中间件链，返回响应
- 管理：Context池化，模版渲染，错误处理

```Go
type Engine struct {
	RouterGroup
	routeTreesUpdated sync.Once
	RedirectTrailingSlash bool
	RedirectFixedPath bool
	HandleMethodNotAllowed bool
	ForwardedByClientIP bool
	AppEngine bool
	UseRawPath bool
	UseEscapedPath bool
	UnescapePathValues bool
	RemoveExtraSlash bool
	RemoteIPHeaders []string
	TrustedPlatform string
	MaxMultipartMemory int64
	UseH2C bool
	ContextWithFallback bool

	delims           render.Delims
	secureJSONPrefix string
	HTMLRender       render.HTMLRender
	FuncMap          template.FuncMap
	allNoRoute       HandlersChain
	allNoMethod      HandlersChain
	noRoute          HandlersChain
	noMethod         HandlersChain
	pool             sync.Pool
	trees            methodTrees
	maxParams        uint16
	maxSections      uint16
	trustedProxies   []string
	trustedCIDRs     []*net.IPNet
}
```

Engine中的内容太多了，下面会挑重要的内容写...
# ServeHTTP实现

实现了[[http.Handler]]接口

```Go
// ServeHTTP conforms to the http.Handler interface.
func (engine *Engine) ServeHTTP(w http.ResponseWriter, req *http.Request) {
	engine.routeTreesUpdated.Do(func() {
		engine.updateRouteTrees()
	})
	
	c := engine.pool.Get().(*Context)
	c.writermem.reset(w)
	c.Request = req
	c.reset()
	
	engine.handleHTTPRequest(c)
	
	engine.pool.Put(c)
}
```

在Engine中定义的`routeTreesUpdated sync.Once`确保路由树只初始化一次，`sync.Once.Do`保证传入的函数在整个程序生命周期内只执行一次，即使成千上万个请求同时到达，这是一种设计模式，[[Lazy Initialization|懒加载（Lazy Initialization）]]，不在`New()`里做，而是在第一个请求到达时才做，加快启动速度。

```Go
engine.routeTreesUpdated.Do(func() {
	engine.updateRouteTrees()
})
```

接下来从pool（[[Gin Context|Context]]对象池）中取一个复用的Context，因为在面对高并发的场景，如果用`new()`则每一次new都会进行一次堆内存空间的分配，导致GC频繁扫描，性能下降。而从对象池中复用Context，分配次数变少，GC压力极小，高性能。本质是**避免反复向runtime申请和归还堆内存，从而减少GC的负担。**

接下来，`c.writermem.reset(w)`请求回写通道，并且将请求挂载到Context中，`c.reset()`清空Context的内容并放回对象池。这里进行`c.reset()`并不会将req清空，所以不需要担心顺序问题，
Context相关的具体内容在[[Gin Context]]中。

```Go
c := engine.pool.Get().(*Context)
c.writermem.reset(w)
c.Request = req // 将请求挂载到Context上
c.reset() // 清空
engine.handleHTTPRequest(c)
engine.pool.Put(c) // 将context放回对象池
```

