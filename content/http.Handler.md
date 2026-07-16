%% [[Gin Engine]] %%
http.Handler是一个接口，属于Go标准库的`net/http`包。
```
type Handler interface {  
    ServeHTTP(ResponseWriter, *Request)  
}
```
实现了ServeHTTP方法就相当于实现了Handler接口，ServerHTTP方法的作用是，**处理一个已经到达服务器的HTTP请求，并且回写响应**，即负责处理`net/http`已经接收到并解析好的请求，本质上是一个HTTP请求入口。
# 参考
- https://pkg.go.dev/net/http
- https://geektutu.com/post/gee-day1.html
