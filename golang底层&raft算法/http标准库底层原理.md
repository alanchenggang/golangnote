#### Q1. 基本使用

![image-20230808140532702](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/note/image-20230808140532702.png)

http 协议下，交互框架是由客户端（Client）和服务端（Server）两个模块组成的 C-S 架构

```go
import (
	"fmt"
	"net/http"
)
// Path: server\main.go
func main() {
    // 注册路径和处理函数
	http.HandleFunc("/ping", func(w http.ResponseWriter, r *http.Request) {
		fmt.Println("receive ping")
		w.Write([]byte("pong"))
	})
    // 监听本机:8080端口并启动 就会陷入阻塞
	http.ListenAndServe(":8080", nil)
}



import (
	"io"
	"net/http"
	"testing"
)
// Path: client\main_test.go
func TestConnect(t *testing.T) {
    // 发送post请求
	response, err := http.Post("http://localhost:8080/ping", "application/json", nil)
	if err != nil {
		t.Error(err)
		return
	}
    // 读取响应体信息
	body, err := io.ReadAll(response.Body)
	if err != nil {
		return
	}
    // 延迟关闭响应体io流
	defer response.Body.Close()
	t.Log(string(body))
	t.Error("test")
}

```

![image-20230808140354865](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/note/image-20230808140354865.png)

| **模块**         | **文件**              |
| ---------------- | --------------------- |
| 服务端           | net/http/server.go    |
| 客户端——主流程   | net/http/client.go    |
| 客户端——构造请求 | net/http/request.go   |
| 客户端——网络交互 | net/http/transport.go |

#### Q2. 服务端数据结构

```go
//基于面向对象的思想，整个 http 服务端模块被封装在 Server 类当中.
//Handler 是 Server 中最核心的成员字段，实现了从请求路径 path 到具体处理函数 handler 的注册和映射能力.
//在用户构造 Server 对象时，倘若其中的 Handler 字段未显式声明，则会取 net/http 包下的单例对象 DefaultServeMux（ServerMux 类型） 进行兜底.
// A Server defines parameters for running an HTTP server.
// The zero value for Server is a valid configuration.
type Server struct {
	// server 地址
	Addr string
    // 路由处理器
	Handler Handler // handler to invoke, http.DefaultServeMux if nil
	...
}
//Handler 是一个 interface，定义了方法： ServeHTTP.
//该方法的作用是，根据 http 请求 Request 中的请求路径 path 映射到对应的 handler 处理函数，对请求进行处理和响应.
// A Handler responds to an HTTP request.
type Handler interface {
	ServeHTTP(ResponseWriter, *Request)
}

// ServeMux 是对 Handler 的具体实现，内部通过一个 map 维护了从 path 到 handler 的映射关系.
// ServeMux 是一个 HTTP 请求多路复用器。它将每个传入请求的 URL 与注册模式列表进行匹配，并调用与 URL 最匹配的模式的处理程序。
type ServeMux struct {
	mu    sync.RWMutex
	m     map[string]muxEntry
	es    []muxEntry // slice of entries sorted from longest to shortest.
	hosts bool       // whether any patterns contain hostnames
}

// muxEntry 为一个 handler 单元，内部包含了请求路径 path + 处理函数 handler 两部分
type muxEntry struct {
	h       Handler
	pattern string
}
```

