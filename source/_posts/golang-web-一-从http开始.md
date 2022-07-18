---
title: golang-web(一) 从http开始
tags:
  - golang
  - golang-web
categories: golang-web
index_img: /img/post/03.jpg
banner_img: /img/post/03.jpg
abbrlink: 6b5ac4cc
date: 2022-07-18 14:24:58
---

## 从 `http` 包开始

我们都知道在 golang 中构建一个 http 服务很容易

```go
package main  
  
import "net/http"

func main(){
	http.HandleFunc("/ping", func(w http.ResponseWriter, r *http.Request) {  
	   _, _ = w.Write([]byte("pong"))  
	})
	_ = http.ListenAndServe(":8080", nil)
}
```

有过现代 web api 开发的经验的人都能从中发现对应轻量级框架 的影子(如 express.js、 php slim)。
对应的概念有 **路由**、**处理函数**、**请求对象/结构体**、**响应对象/结构体**

因为 `http1.1 - 2` 是通过 tcp 实现的，
上面这个的代码隐藏了一些细节，我们来通过源码闭环整个 注册路由，监听与请求过程

### 注册路由
1. 用了一个 DefaultServeMux 结构， 来处理路由与处理函数的绑定
```go
// net/http/server.go:2488
func HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {  
   DefaultServeMux.HandleFunc(pattern, handler)  
}

// net/http/server.go:2472
func (mux *ServeMux) HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {  
   if handler == nil {  
      panic("http: nil handler")  
   }   
   mux.Handle(pattern, HandlerFunc(handler))  
}
```

2. 里面的实现的比较简陋, 就是一个路由与 handler 函数 对应的 大 map 。

```go
// net/http/server.go:2429
func (mux *ServeMux) Handle(pattern string, handler Handler) {  
   mux.mu.Lock()  
   defer mux.mu.Unlock()  
  
   if pattern == "" {  
      panic("http: invalid pattern")  
   }   
   if handler == nil {  
      panic("http: nil handler")  
   }   
   if _, exist := mux.m[pattern]; exist {  
      panic("http: multiple registrations for " + pattern)  
   }  
   if mux.m == nil {  
      mux.m = make(map[string]muxEntry)  
   }   
   e := muxEntry{h: handler, pattern: pattern}  
   mux.m[pattern] = e  
   if pattern[len(pattern)-1] == '/' {  
      mux.es = appendSorted(mux.es, e)  
   }  
   if pattern[0] != '/' {  
      mux.hosts = true  
   }  
}
```

注册完之后，就看看 监听与处理部分

### 事件监听与任务分派

1. 先是打开了一个 tcp socket 并调用了 Serve 方法去监听

```go
// net/http/server.go:3182
func ListenAndServe(addr string, handler Handler) error {  
   server := &Server{Addr: addr, Handler: handler}  
   return server.ListenAndServe()  
}

// net/http/server.go:2918
func (srv *Server) ListenAndServe() error {  
   if srv.shuttingDown() {  
      return ErrServerClosed  
   }  
   addr := srv.Addr  
   if addr == "" {  
      addr = ":http"  
   }  
   ln, err := net.Listen("tcp", addr)  
   if err != nil {  
      return err  
   }
   return srv.Serve(ln)  
}

```

2. 开启监听 ，并用协程去处理真正的逻辑
```go
// net/http/server.go:2971
func (srv *Server) Serve(l net.Listener) error {  
// ......
   ctx := context.WithValue(baseCtx, ServerContextKey, srv)
   for {  
   // 等待请求 
      rw, err := l.Accept()  
   // ...省略
      connCtx := ctx  
   // ...省略 
      go c.serve(connCtx)  
   }}
```

### 路由匹配与逻辑处理
```go
func (c *conn) serve(ctx context.Context) {  
   // ... net/http/server.go:1929   
   serverHandler{c.server}.ServeHTTP(w, w.req)  
   // ...
}
```

**最重点**就是这一句，`serverHandler{c.server}` 调用 `ServeHTTP` 方法 ，`serverHandler{c.server}` 的类型在这里实际上是我们注入 `DefaultServeMux ServeMux` 调用了自己的 `ServeHTTP()`  其它框架无法就是对于这个 `Handler` 的实现与替换

```go
// net/http/server:2415
func (mux *ServeMux) ServeHTTP(w ResponseWriter, r *Request) {  
   if r.RequestURI == "*" {  
      if r.ProtoAtLeast(1, 1) {  
         w.Header().Set("Connection", "close")  
      }      w.WriteHeader(StatusBadRequest)  
      return  
   }  
   h, _ := mux.Handler(r)  
   // h 就是 Handler 的一个实例
   h.ServeHTTP(w, r)  
}

 
// net/http/server:2396
func (mux *ServeMux) handler(host, path string) (h Handler, pattern string) {  
   mux.mu.RLock()  
   defer mux.mu.RUnlock()  
  
   // Host-specific pattern takes precedence over generic ones  
   if mux.hosts {  
      h, pattern = mux.match(host + path)  
   }   if h == nil {  
      h, pattern = mux.match(path)  
   }   if h == nil {  
      h, pattern = NotFoundHandler(), ""  
   }  
   return  
}
```

去 map 里面匹配到了 注入的 HandlerFunc 最终调用了 HandlerFunc 的 ServeHttp ,执行了我们最开始的逻辑函数

```go
//server:2046
func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {  
   f(w, r)  
}
```

整体的流程下来就是 
**路由注册，开启监听 ，监听到请求，开协程去匹配路由，并根据注册的处理函数来处理请求**
官方包已经给了一整套完整的实现，有改进空间的点在于 路由注册/匹配这一块，再补上 log config 等一些基础包，每个人都可以实现自己的框架

### 自定义 Handler

上面的例子中，我们使用了默认的 `http.DefaultServeMux `  来实现 `handle` 
`Handler Handler // handler to invoke, http.DefaultServeMux if nil`  

 `handle` 最主要就做了两件事
第一是定义了 map 来存储路由与处理逻辑函数
第二个就是实现了 `Handler interface ` 定义了一个 `ServeHTTP(ResponseWriter, *Request)` 来处理 传入的 http  请求。

这两件事，不依赖官方，我们自己去做可行？很显然可行，到这里我们已经往自定义框架的路上走了一大步。

#### Pong http 服务

一起来实现一个 pong http 服务，不管你请求什么 我都会响应 pong 

```go
package main  
  
import "net/http"

type pongHandler struct {  
   content string  
}  
  
func (pH *pongHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {  
   _, _ = w.Write([]byte(pH.content))  
}  
  
func main() {  
   _ = http.ListenAndServe(":8080", &pongHandler{"pong"})
}
```

现在我们就得到了一个不管你怎么访问我都是响应 pong 的 http 服务， 而没有使用  `ServeMux`  ，接下来 我们来实现简易路由的存储与匹配

#### 路由的存储与匹配

```go
package main  
  
import (  
   "net/http"  
)  
  
type pingHandler struct {  
   content string  
   trees   map[string]*node  
}  
  
type node struct {  
   method  string  
   handler func(http.ResponseWriter, *http.Request)  
}  
  
func (pH *pingHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {  
  
   if n, ok := pH.trees[r.Method+r.RequestURI]; !ok {  
      _, _ = w.Write([]byte("404 Not Found"))  
      return  
   } else {  
      if n.method != r.Method {  
         _, _ = w.Write([]byte("Method Not Match"))  
      } else {  
         n.handler(w, r)  
      }   
   }
}  
  
func (pH *pingHandler) Get(route string, logic func(w http.ResponseWriter, r *http.Request)) {  
   n := node{http.MethodGet, logic}  
   pH.trees[http.MethodGet+route] = &n  
}  
  
func (pH *pingHandler) Post(route string, logic func(w http.ResponseWriter, r *http.Request)) {  
   n := node{http.MethodPost, logic}  
   pH.trees[http.MethodPost+route] = &n  
}  
  
func main() {  
  
   r := pingHandler{"pong", make(map[string]*node)}  
  
   r.Get("/ping", func(w http.ResponseWriter, r *http.Request) {  
      _, _ = w.Write([]byte(r.RequestURI + ":" + r.Method))  
   })  
   r.Post("/ping", func(w http.ResponseWriter, r *http.Request) {  
      _, _ = w.Write([]byte(r.RequestURI + ":" + r.Method))  
   })  
   _ = http.ListenAndServe(":8080", &r)
}
```

几行代码就生成了一个简易的路由存储与匹配机制，对路由存储与匹配感兴趣的小伙伴可以参考
[github.com/gorilla/mux](https://github.com/gorilla/mux)  功能更丰富、 [github.com/julienschmidt/httprouter](https://github.com/julienschmidt/httprouter)  性能更好，对应的知识点 有 [[前缀树trie算法]]  ，基于 hash 的路由匹配...  此处就不再深入了，接下来的一章，我们去看看官方推荐的 gin 框架相关功能是如何实现的？


### 其它细节点待补充 

- TODO 
	- socket 细节
	- 路由匹配算法
	- 服务抽象思维与接口实现重写



