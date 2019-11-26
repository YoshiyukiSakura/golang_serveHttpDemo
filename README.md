### 啥是ServeHTTP
在golang中，你要构建一个web服务，必然要用到`http.ListenAndServe`(除非你自己从头在net/tcp基础上写)，来看看它的方法签名：`http.ListenAndServe(address, handler)`,可以看到第二个参数必须要有一个`handler`，然后看看`handler`的定义。
```
type Handler interface {
	ServeHTTP(ResponseWriter, *Request)
}
```
很显然，要用官方的这个`http.ListenAndServe`,可以自己定义一个struct，并在上面实现`ServeHTTP`方法。



### 简单Demo
```
type myHandler int

func (m myHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	_, err := fmt.Fprintln(w, "hello this is golang ServeHTTP")
	if err != nil {
		fmt.Println("发生内部错误:", err)
	}else{
	    fmt.Println("有人访问:", r.Method)   
	}
}

func main() {
	var hello myHandler
	fmt.Println("开始监听")
	http.ListenAndServe(":8080", hello)
}
```

### 使用nil作为handler
正常思考，`http.ListenAndServe(":8080", nil)`这种操作应该是不合法的，因为nil显然没有实现`ServeHTTP`方法。
试试传入一个nil，会发生什么  
```
#代码
http.ListenAndServe(":8080", nil)
#居然跑起来了，然后试着访问这个服务
curl http://127.0.0.1:8080/status/
404 page not found
```
很神奇，返回了`404 page not found`，深入`http`包下面的`server.go`可以发现一段代码
```
#大概在server.go的两千多行位置，取决于go version
func (mux *ServeMux) handler(host, path string) (h Handler, pattern string) {
	mux.mu.RLock()
	defer mux.mu.RUnlock()

	// Host-specific pattern takes precedence over generic ones
	if mux.hosts {
		h, pattern = mux.match(host + path)
	}
	if h == nil {
		h, pattern = mux.match(path)
	}
	if h == nil {
		h, pattern = NotFoundHandler(), ""//就是这里调用了404方法
	}
	return
}
```
继续在`server.go`里探索一下，发现了nil可以传入并且跑起来的原因
```
#大概在server.go的两千七百多行位置
func (sh serverHandler) ServeHTTP(rw ResponseWriter, req *Request) {
     handler := sh.srv.Handler
     if handler == nil {
             handler = DefaultServeMux//传入nil，则go会用自带的DefaultServeMux替换
     }
     if req.RequestURI == "*" && req.Method == "OPTIONS" {
             handler = globalOptionsHandler{}
     }
     handler.ServeHTTP(rw, req)
}
```
得出结论：___在listenAndServe方法中传入nil，其实就是调用DefaultServeMux___  

### DefaultServeMux咋用
非重点，简单略过
```
type mytype struct{}
func (t *mytype) ServeHTTP(w http.ResponseWriter, r *http.Request) {
     fmt.Fprintf(w, "Hello there from mytype")
}
func StatusHandler(w http.ResponseWriter, r *http.Request) {
     fmt.Fprintf(w, "OK")
}
func main() {
     t := new(mytype)
     http.Handle("/", t)
     http.HandleFunc("/status/", StatusHandler)
     http.ListenAndServe(":8080", nil)
}
```
横向观察市面上的框架，用不用DefaultServeMux的都有，优点是比较简单，例如自带路由匹配功能，缺点是深入定制比较麻烦，比如添加中间件

### 实现自己的Mux
先对Mux做一个释义，mux是洋文multiplexer的缩略写法，中文可以译作`多路复用器`，在web服务编程里面，我们可以特化地认为，mux就是路由管理对象，所有path都应该注册到mux里面，然后请求来的时候，由mux对象进行分发，调用对应的`handle`上面的`ServeHTTP`  
试写一个demo  
```
func StatusHandler(w http.ResponseWriter, r *http.Request) {
	fmt.Println("真正的中间业务逻辑")
	fmt.Fprintf(w, "OK")
}

func RunSomeCode(handler http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		log.Printf("前置中间件Got a %s request for: %v", r.Method, r.URL)
		handler.ServeHTTP(w, r)
		log.Println("后置中间件Handler finished processing request")
	})
}
func main() {
	mux := http.NewServeMux()
	mux.HandleFunc("/status/", StatusHandler)
	WrappedMux := RunSomeCode(mux)
	http.ListenAndServe(":8080", WrappedMux)
}
```
这段代码可以解答两个问题：
1. 如何定义自己的mux
2. 为什么要定义自己的mux
针对第二个问题，我们可以看到这一行`WrappedMux := RunSomeCode(mux)`，我们将已经定义好的mux对象像套娃一样又包装一次，再观察`RunSomeCode`就很容易理解，我们要如何在golang服务端编程中，实现方便高效可复用的middleware了  

### mux上面的常用属性、方法
TDB   
最后祝你身体健康
