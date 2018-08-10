---
title: golang-http包
date: 2017-08-16 16:21:21
tags: golang
categories: Golang
---

# http服务

http包包含http客户端和服务端的实现,利用Get,Head,Post,以及PostForm实现HTTP或者HTTPS的请求.

## 1. 函数

### 1.1 服务端函数

1. `Handle`将handler按照指定的格式注册到`DefaultServeMux`,`ServeMux`解释了模式匹配规则

   ```go
   func Handle(pattern string, handler Handler)
   ```

2. `HandleFunc`同上，主要用来实现动态文件内容的展示，这点与`ServerFile()`不同的地方。

   ```go
   func HandleFunc(pattern string, handler func(ResponseWriter, *Request))
   ```

3. `ServeFile`利用指定的文件或者目录的内容来响应相应的请求.

   ```go
   func ServeFile(w ResponseWriter, r *Request, name string)
   ```

4. `ListenAndServe`监听TCP网络地址addr然后调用具有handler的Serve去处理连接请求.通常情况下Handler是nil,使用默认的DefaultServeMux

   ```go
   func ListenAndServe(addr string, handler Handler) error
   ```

### 1.2最简单的http服务

```go
package main

import "net/http"

func main() {
    http.ListenAndServe(":8080", nil)
}
```

访问网页后会发现，提示的不是“无法访问”，而是”页面没找到“，说明http已经开始服务了，只是没有找到页面
由此可以看出，访问什么路径显示什么网页 这件事情，和ListenAndServe的第2个参数有关

由1.1的解析可知，第2个参数是一个 **Hander**

在http包中看到这个 **Hander**接口只有一个方法ServeHTTP

```go
type Handler interface {
	ServeHTTP(ResponseWriter, *Request)
}
```

所以只要实现了ServeHTTP(ResponseWriter, *Request)这个方法的struct,那么就可以将这个struct方法放进去，然后被调用

ServeHTTP方法，他需要2个参数，

1. 一个是http.ResponseWriter， 往http.ResponseWriter写入什么内容，浏览器的网页源码就是什么内容
2. 另一个是*http.Request，*http.Request里面是封装了浏览器发过来的请求（包含路径、浏览器类型等等）

### example

```go
package main

import (
    "io"
    "net/http"
)

type test struct{}
//结构体a实现了ServeHTTP
func (*test) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    io.WriteString(w, "hello world")
}

func main() {
    http.ListenAndServe(":8080", &test{})//第2个参数需要实现Hander的struct，a满足
}
```

现在
访问localhost:8080的话，可以看到“hello world”
访问localhost:8080/abc的话，可以看到“hello world”
访问localhost:8080/123的话，可以看到“hello world”
事实上访问任何路径都是“hello world”

当 http.ListenAndServe(“:8080”, &test{})后，开始等待有访问请求

一旦有访问请求过来，http包回去调用test的ServeHTTP这个方法。并把自己已经处理好的`http.ResponseWriter, *http.Request`传进去

而test的ServeHTTP这个方法，拿到`*http.ResponseWriter`后，并往里面写东西，客户端的网页就显示出来了

一、从源码可以理解:这里会将Handler赋值给Server

```go
// ListenAndServe always returns a non-nil error.
func ListenAndServe(addr string, handler Handler) error {
	server := &Server{Addr: addr, Handler: handler}
	return server.ListenAndServe()
}
```

二、

这里是server.ListenAndServe()———–>回去调用go c.server(ctx)

其中c是`c := srv.newConn(rw)`

然后`c.server(ctx)`这个函数中会调用`serverHandler{c.server}.ServeHTTP(w, w.req)`这个方法

这里serverHandler组合了Server结构体

这里当handler为空的时候就调用默认的DefaultServeMux，当不为空的时候就会去调用handler.ServeHTTP

```go
// serverHandler delegates to either the server's Handler or
// DefaultServeMux and also handles "OPTIONS *" requests.
type serverHandler struct {
	srv *Server
}

func (sh serverHandler) ServeHTTP(rw ResponseWriter, req *Request) {
	handler := sh.srv.Handler
	if handler == nil {
		handler = DefaultServeMux
	}
	if req.RequestURI == "*" && req.Method == "OPTIONS" {
		handler = globalOptionsHandler{}
	}
	handler.ServeHTTP(rw, req)
}
```

通过上面的解析就可以知道网页解析是通过调用ServeHTTP方法来的

\###2.3、ServeMux的作用

先看ServeMux的结构体:

```go
type ServeMux struct {
   mu    sync.RWMutex
   m     map[string]muxEntry
   hosts bool // whether any patterns contain hostnames
}

type muxEntry struct {
   explicit bool
   h        Handler
   pattern  string
}
```

从结构体中可以看出ServeMux有一个map属性，map属性的value是个muxEntry类型，这个类型中有一个Handler属性，可以推测看看此ServerMux的m属性的key保存的是url，muxEntry是一个Handler方法

然后看代码

```go
package main

import (
    "net/http"
    "io"
)

type b struct{}

func (*b) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    io.WriteString(w, "hello")
}
func main() {
    mux := http.NewServeMux()//新建一个ServeMux。
    mux.Handle("/h", &b{})//注册路由，把"/"注册给b这个实现Handler接口的struct，注册到map表中。
    http.ListenAndServe(":8080", mux)
}
```

**mux.Handle内部**

```go
//Handle registers the handler for the given pattern.
// If a handler already exists for pattern, Handle panics.
func (mux *ServeMux) Handle(pattern string, handler Handler) {
   mux.mu.Lock()
   defer mux.mu.Unlock()

   if pattern == "" {
      panic("http: invalid pattern " + pattern)
   }
   if handler == nil {
      panic("http: nil handler")
   }
   if mux.m[pattern].explicit {
      panic("http: multiple registrations for " + pattern)
   }

   if mux.m == nil {
      mux.m = make(map[string]muxEntry)
   }
  //将路由作为key，然后将handler和路由以及显示调用设置为true
   mux.m[pattern] = muxEntry{explicit: true, h: handler, pattern: pattern}

   if pattern[0] != '/' {
      mux.hosts = true
   }
   ....
   }
```

所以可以看出ServeMux是通过一个map将路由以及函数存起来的。

```go
func (mux *ServeMux) Handle(pattern string, handler Handler) {}
```

这个函数接收的第一个参数是**路由**，第二个参数是一个**Handler**。这个Handler和上面ListenAndServe的第二个参数是一样的是只有一个ServeHTTP(ResponseWriter, *Request)方法的接口。所以此处的handler需要实现ServeHTTP方法。

运行时，因为第二个参数是mux，所以http会调用mux的ServeHTTP方法。ServeHTTP方法执行时，会检查map表（表里有一条数据，key是“/h”，value是&b{}的ServeHTTP方法）

如果用户访问`/h`的话，mux因为匹配上了，mux的ServeHTTP方法会去调用&b{}的 ServeHTTP方法，从而打印hello
如果用户访问`/abc`的话，mux因为没有匹配上，从而打印404 page not found

### 2.4、ServeMux的HandleFunc方法

ServeMux有一个HandleFunc方法，此方法直接调用handle函数并实现了ServeHTTP

```go
func (mux *ServeMux) HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
   mux.Handle(pattern, HandlerFunc(handler))
}
```

```go
type HandlerFunc func(ResponseWriter, *Request)

// ServeHTTP calls f(w, r).
func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
   f(w, r)
}
```

所以使用HandlerFunc的时候

```go
package main

import (
    "net/http"
    "io"
)

func main() {
    mux := http.NewServeMux()
    mux.HandleFunc("/h", func(w http.ResponseWriter, r *http.Request) {
        io.WriteString(w, "hello")
    })
    mux.HandleFunc("/bye", func(w http.ResponseWriter, r *http.Request) {
        io.WriteString(w, "byebye")
    })
    mux.HandleFunc("/hello", sayhello)
    http.ListenAndServe(":8080", mux)
}

func sayhello(w http.ResponseWriter, r *http.Request) {
    io.WriteString(w, "hello world")
}
```

