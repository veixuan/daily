---
# 常用定义
title: "Httpserver"
date: 2020-12-26T11:13:11+08:00
lastmod: 2020-12-26T11:13:11+08:00
tags: ["golang", "popular", "http","server"] 
categories: ["golang"]             
author: "Neo"          
# 用户自定义
# 你可以选择 关闭(false) 或者 打开(true) 以下选项
comment: true # 关闭评论
toc: false       # 关闭文章目录
reward: false	 # 关闭打赏
---
`golang`中标准库`http server`的使用和实现。

<!--more-->

# 基本姿势

## 最小化实现

```go
func Serve(addr string) {
	http.HandleFunc("/now", func(writer http.ResponseWriter, request *http.Request) {
		writer.Write([]byte("Hi, now is " + time.Now().String()))
	})
	log.Fatal(http.ListenAndServe(addr, nil))
}
```

## 自定义实现 

```go
type IndexHandler struct{}

func (i *IndexHandler) ServeHTTP(writer http.ResponseWriter, request *http.Request) {
	writer.Write([]byte("Hi, now is " + time.Now().String()))
}

func Serve(addr string) {
	mux := http.NewServeMux()
	mux.Handle("/hello", &IndexHandler{})
	server := &http.Server{
		Addr:              addr,
		Handler:           mux,
		TLSConfig:         nil,
		ReadTimeout:       time.Second,
		ReadHeaderTimeout: time.Millisecond * 500,
		WriteTimeout:      time.Millisecond * 500,
	}
	log.Fatal(server.ListenAndServe())
}
```

## 支持`https`

```go
// 1. 生成服务端私钥  2048单位bit key的长度 
openssl genrsa -out server.key 2048
// 2. 根据私钥生成自签发的证书 
openssl req -new -x509 -key server.key -out server.crt -days 365
// 当前目录下生成 server.crt和server.key两个文件 

// 修改http server 启动代码
log.Fatal(server.ListenAndServeTLS("server.crt", "server.key"))

// 验证  -k 允许连接到 SSL 站点，而不校验证书 
curl -k https://localhost:8000/hello
```

## 支持 `http2`

>   `GO`的`http`库默认支持`HTTP/2`协议，只要我们使用TLS则会默认启动HTTP/2特性

## 支持`quic`

```go
// 安装依赖 
// go get -u -v github.com/lucas-clemente/quic-go

// brew install mkcert
// mkcert -install 
func Serve(addr string) {
	mux := http.NewServeMux()
	mux.Handle("/hello", &IndexHandler{})
	log.Fatal(http3.ListenAndServe(addr, "install.pem", "install-key.pem", mux))
}
```

# 实现原理 

## 概念 

-   [ ] `Request` http 请求
-   [ ] `Response` http 响应，是server端需要返回给客户端的信息
-   [ ] `Handler` 处理请求和生成返回信息的处理逻辑
-   [ ] `ServeMux` 多路复用器

## 处理流程

>   后面再填坑

# 常见实现 

## 获取`url query` 参数 

```go
// http://localhost:8000/hello?name=张三 
	query := request.URL.Query()
	name := query.Get("name")
```

## 获取 `url path` 参数

```go
// 可以考虑使用第三方的http router 库 
// 比如 
// 1. https://github.com/julienschmidt/httprouter
// 2. https://github.com/gorilla/mux 

// 原生库实现起来比较trick 
```

## 获取 `post`参数

```go
	body, _ := ioutil.ReadAll(request.Body)
```

## 静态文件服务器

```go
	fileServer := http.FileServer(http.Dir("./"))
	go http.ListenAndServe(":4000", fileServer)
```

## `cors `

### 背景

>   浏览器通过同源策略来防止恶意网站窃取数据

### 术语解释

同源(`Same origin policy`)协议、域名、端口都相同` **也就是说非同源的资源之间不能相互通信**

以 `http://example.com/hello`为例

| 域名                              | 是否同源 | 说明                                 |
| --------------------------------- | -------- | ------------------------------------ |
| `http://example.com/world`        | 同源     | 域名、端口、协议都相同，只是路径不同 |
| `https://example.com/world`       | 不同源   | 协议不同，http协议与https协议        |
| `https://example.com:10086/world` | 不同源   | 端口不同                             |
| `https://example001.com/world`    | 不同源   | 域名不同                             |

### 问题 

同源策略的出发点是通过禁止非同源之间通信来避免恶意网站窃取数据，但是同时也会导致非同源之间的合理通信无法实施，比如 `ajax`

### 解决方案 

`Cross-origin resource sharing `跨域资源共享。浏览器和服务端之间共同完成。服务端实现`CORS接口`

```go
// 使用 https://github.com/rs/cors 
	fileServer := http.FileServer(http.Dir("./"))

	c := cors.New(cors.Options{
		AllowedOrigins:   []string{"http://foo.com", "http://foo.com:8080"},
		AllowCredentials: true,
		// Enable Debugging for testing, consider disabling in production
		Debug: true,
	})
	go http.ListenAndServe(":4000", c.Handler(fileServer))
```

## 静态资源`bind` 

-   go1.16 新版将集成原生支持 
-   可以使用开源库 比如 `gobind` 、`vfsgen` 等 

## 优雅退出 

>   1.8 之后，默认的`shutdown`方法已实现优化退出 

```go
defer server.Shutdown(context.Background())

func (srv *Server) Shutdown(ctx context.Context) error {
  
  // 1. 标记退出状态  cas实现 
	srv.inShutdown.setTrue()

  // 2. 加锁 
	srv.mu.Lock()
  // 3. 关闭 listener fd 新的连接无法建立 即不会接受新的请求 
	lnerr := srv.closeListenersLocked()
  // 4. 把server 的done chan给close掉，通知等待的worekr退出
	srv.closeDoneChanLocked()
  // 5. 执行回调方法，我们可以注册shutdown的回调方法
	for _, f := range srv.onShutdown {
		go f()
	}
	srv.mu.Unlock()

  // 6. 500ms 轮询一次 
	ticker := time.NewTicker(shutdownPollInterval)
	defer ticker.Stop()
	for {
    // 已经没有空闲的连接了 
		if srv.closeIdleConns() && srv.numListeners() == 0 {
			return lnerr
		}
		select {
		case <-ctx.Done():
			return ctx.Err()
		case <-ticker.C:
		}
	}
}
```

# 性能分析 

## 集成 pprof 

```go
	_ "net/http/pprof"
```

## 集成 fgprof 

`https://github.com/felixge/fgprof`

>   默认的pprof只能分析`on-cpu`，要分析`off-cpu`可以使用上面的库 

```go
	mux.Handle("/hello", &HelloHandler{})
	mux.Handle("/debug/fgprof", fgprof.Handler())

// go tool pprof --http=:6061 'http://localhost:8000/debug/fgprof?seconds=3'
```

