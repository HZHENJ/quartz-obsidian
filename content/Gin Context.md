%% [[Gin]] %%
%% [[Gin Engine]] %%
```
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