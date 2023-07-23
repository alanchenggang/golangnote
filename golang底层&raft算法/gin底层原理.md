#### Q1.  Gin 的背景(**https://github.com/gin-gonic/gin**)

支撑研发团队选择 Gin 作为 web 框架的原因包括：

- 支持中间件操作（ handlersChain 机制 ）
- 更方便的使用（ gin.Context ）
-  更强大的路由解析能力（ radix tree 路由树 ）

#### Q2. Gin 与 net/http 的关系

**Gin 是在 Golang HTTP 标准库 net/http 基础之上的再封装**

![image-20230723213221467](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/note/image-20230723213221467.png)

**可以看出，在 net/http 的既定框架下，gin 所做的是提供了一个 gin.Engine 对象作为 Handler 注入其中，从而实现路由注册/匹配、请求处理链路的优化.**

#### Q3. Gin的基本使用

 Gin 框架的使用风格：

- 构造 gin.Engine 实例：gin.Default()
- 路由组注册中间件：Engine.Use()
- 路由组注册 POST 方法下的 handler：Engine.POST()
- 启动 http server：Engine.Run()

```go
import "github.com/gin-gonic/gin"


func main() {
    // 创建一个 gin Engine，本质上是一个 http Handler
    mux := gin.Default()
    // 注册中间件
    mux.Use(myMiddleWare)
    // 注册一个 path 为 /ping 的处理函数
    mux.POST("/ping", func(c *gin.Context) {
        c.JSON(http.StatusOK, "pone")
    })
    // 运行 http 服务
    if err := mux.Run(":8080"); err != nil {
        panic(err)
    }
}
```

#### Q4. 注册Handler流程

```go
type Engine struct {
    //路由组
	RouterGroup
	...
    //context 对象池
	pool             sync.Pool
    //方法路由树
	trees            methodTrees
    ... 
}

// net/http 包下的 Handler interface
type Handler interface {
	ServeHTTP(ResponseWriter, *Request)
}

// ServeHTTP conforms to the http.Handler interface.
func (engine *Engine) ServeHTTP(w http.ResponseWriter, req *http.Request) {
    // 从 sync.Pool 中获取一个 *Context 对象，用于处理当前请求
	c := engine.pool.Get().(*Context)
    // 重置 *Context 的 ResponseWriter 和 Request 字段
	c.writermem.reset(w)
	c.Request = req
    // 重置 *Context 的其他字段，确保它处于初始状态
	c.reset()
    // 处理 HTTP 请求，调用相关的处理逻辑
	engine.handleHTTPRequest(c)
    // 将处理完成的 *Context 对象放回 sync.Pool 中，以便重复使用
	engine.pool.Put(c)
}
```

**Engine 为 Gin 中构建的 HTTP Handler，其实现了 net/http 包下 Handler interface 的抽象方法： Handler.ServeHTTP，因此可以作为 Handler 注入到 net/http 的 Server 当中.**

>Engine包含的核心内容包括：
>
>-  路由组 RouterGroup
>
>- Context 对象池 pool：基于 sync.Pool 实现，作为复用 gin.Context 实例的缓冲池. (复用逻辑删除对象)
>
>- 路由树数组 trees：共有 9 棵路由树，对应于 9 种 http 方法. 路由树基于压缩前缀树实现
>
>  九种http方法展示如下
>
>  ```go
>  const (
>      MethodGet     = "GET"
>      MethodHead    = "HEAD"
>      MethodPost    = "POST"
>      MethodPut     = "PUT"
>      MethodPatch   = "PATCH" // RFC 5789
>      MethodDelete  = "DELETE"
>      MethodConnect = "CONNECT"
>      MethodOptions = "OPTIONS"
>      MethodTrace   = "TRACE"
>  )
>  ```

##### Q4.01 RouterGroup 构成

```go

//RouterGroup 是路由组的概念，其中的配置将被从属于该路由组的所有路由复用
type RouterGroup struct {
	Handlers HandlersChain // 路由组共同的 handler 处理函数链. 组下的节点将拼接 RouterGroup 的公用 handlers 和自己的 handlers，组成最终使用的 handlers 链
	basePath string //路由组的基础路径. 组下的节点将拼接 RouterGroup 的 basePath 和自己的 path，组成最终使用的 absolutePath
	engine   *Engine // 指向路由组从属的 Engine
	root     bool //标识路由组是否位于 Engine 的根节点. 当用户基于 RouterGroup.Group 方法创建子路由组后，该标识为 false
}


// HandlersChain 是由多个路由处理函数 HandlerFunc 构成的处理函数链. 在使用的时候，会按照索引的先后顺序依次调用 HandlerFunc
// HandlersChain defines a HandlerFunc slice.
type HandlersChain []HandlerFunc

// HandlerFunc defines the handler used by gin middleware as return value.
type HandlerFunc func(*Context)
```

##### Q4.02  注册流程入口

```go
func main() {
    // 创建一个 gin Engine，本质上是一个 http Handler
    mux := gin.Default()
    // 注册中间件
    mux.Use(myMiddleWare)
    // 注册一个 path 为 /ping 的处理函数
    mux.POST("/ping", func(c *gin.Context) {
        c.JSON(http.StatusOK, "pone")
    })
    // ...
}
```

##### Q4.03 初始化 Engine

>方法调用：gin.Default -> gin.New
>
>- 创建一个 gin.Engine 实例
>- 创建 Enging 的首个 RouterGroup，对应的处理函数链 Handlers 为 nil，基础路径 basePath 为 "/"，root 标识为 true
>- 构造了 9 棵方法路由树，对应于 9 种 http 方法
>- 创建了 gin.Context 的对象池

```go
// Default returns an Engine instance with the Logger and Recovery middleware already attached.
func Default() *Engine {
	debugPrintWARNINGDefault()
    // 创建一个 gin.Engine 实例
	engine := New()
    // 注册gin写好的中间件
	engine.Use(Logger(), Recovery())
	return engine
}


func New() *Engine {
	debugPrintWARNINGNew()
    // 创建 gin Engine 实例
	engine := &Engine{
        //创建默认root路由组实例
		RouterGroup: RouterGroup{
			Handlers: nil,
			basePath: "/",
			root:     true,
		},
		FuncMap:                template.FuncMap{},
		RedirectTrailingSlash:  true,
		RedirectFixedPath:      false,
		HandleMethodNotAllowed: false,
		ForwardedByClientIP:    true,
		RemoteIPHeaders:        []string{"X-Forwarded-For", "X-Real-IP"},
		TrustedPlatform:        defaultPlatform,
		UseRawPath:             false,
		RemoveExtraSlash:       false,
		UnescapePathValues:     true,
		MaxMultipartMemory:     defaultMultipartMemory,
        //创建9 棵路由压缩前缀树，对应 9 种 http 方法
		trees:                  make(methodTrees, 0, 9),
		delims:                 render.Delims{Left: "{{", Right: "}}"},
		secureJSONPrefix:       "while(1);",
		trustedProxies:         []string{"0.0.0.0/0", "::/0"},
		trustedCIDRs:           defaultTrustedCIDRs,
	}
	engine.RouterGroup.engine = engine
     //创建gin.Context 对象池   
	engine.pool.New = func() any {
		return engine.allocateContext(engine.maxParams)
	}
	return engine
}
```

##### Q4.04 注册middleware

>**通过 Engine.Use 方法可以实现中间件的注册，会将注册的 middlewares 添加到 RouterGroup.Handlers 中. 后续 RouterGroup 下新注册的 handler 都会在前缀中拼上这部分 group 公共的 handlers.**

```go
func (engine *Engine) Use(middleware ...HandlerFunc) IRoutes {
	engine.RouterGroup.Use(middleware...)
	...
	return engine
}

// Use adds middleware to the group
func (group *RouterGroup) Use(middleware ...HandlerFunc) IRoutes {
	group.Handlers = append(group.Handlers, middleware...)
	return group.returnObj()
}
```

##### Q4.05 注册 handler

![image-20230723220522704](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/note/image-20230723220522704.png)

以 http post 为例，注册 handler 方法调用顺序为 RouterGroup.POST-> RouterGroup.handle，接下来会完成三个步骤：

-  拼接出待注册方法的完整路径 absolutePath  calculateAbsolutePath(relativePath)
-  拼接出代注册方法的完整处理函数链 handlers  group.combineHandlers(handlers)
-  以 absolutePath 和 handlers 组成 kv 对添加到路由树中   group.engine.addRoute(httpMethod, absolutePath, handlers)

```go
func (group *RouterGroup) handle(httpMethod, relativePath string, handlers HandlersChain) IRoutes {
	absolutePath := group.calculateAbsolutePath(relativePath)
	handlers = group.combineHandlers(handlers)
	group.engine.addRoute(httpMethod, absolutePath, handlers)
	return group.returnObj()
}


func (group *RouterGroup) calculateAbsolutePath(relativePath string) string {
	return joinPaths(group.basePath, relativePath)
}

func joinPaths(absolutePath, relativePath string) string {
	if relativePath == "" {
		return absolutePath
	}

	finalPath := path.Join(absolutePath, relativePath)
	if lastChar(relativePath) == '/' && lastChar(finalPath) != '/' {
		return finalPath + "/"
	}
	return finalPath
}

func (group *RouterGroup) combineHandlers(handlers HandlersChain) HandlersChain {
    // 计算最终HandlersChain长度
	finalSize := len(group.Handlers) + len(handlers)
    // HandlersChain长度不能大于62 否则报错
    // abortIndex represents a typical value used in abort functions.
    // const abortIndex int8 = math.MaxInt8 >> 1 63
	assert1(finalSize < int(abortIndex), "too many handlers")
    // 创建新的HandlersChain
	mergedHandlers := make(HandlersChain, finalSize)
    // 将原来的HandlersChain深copy到新创建的HandlersChain切片中
	copy(mergedHandlers, group.Handlers)
    // 将新的Handlers深拷贝到新创建的HandlersChain切片末尾
	copy(mergedHandlers[len(group.Handlers):], handlers)
	return mergedHandlers
}


func (engine *Engine) addRoute(method, path string, handlers HandlersChain) {
    //检查路由的路径 path 是否以斜杠 / 开头，如果不是，则触发断言错误
	assert1(path[0] == '/', "path must begin with '/'")
    // 检查 HTTP 方法 method 是否为空字符串，如果是，则触发断言错误
	assert1(method != "", "HTTP method can not be empty")
    // 检查 handlers 切片中是否至少包含一个处理函数，如果没有，则触发断言错误
	assert1(len(handlers) > 0, "there must be at least one handler")

	debugPrintRoute(method, path, handlers)
    // 正常请求条件: path以斜杠"/"开头&&HTTP方法method 不为空字符串&&handlers切片中至少包含一个处理函数
    
    //根据指定的 HTTP 方法，从 engine.trees 中获取对应的路由树的根节点
	root := engine.trees.get(method)
    // 如果还没有为指定的 HTTP 方法创建路由树，则执行以下步骤：
	if root == nil {
        // 创建一个新的根节点 root，表示一个新的空路由树
		root = new(node)
        //设置根节点的完整路径为斜杠 /，表示根节点对应的路径是根路径。
		root.fullPath = "/"
        // 将新的方法树添加到 engine.trees 中，包括 HTTP 方法和对应的根节点。
		engine.trees = append(engine.trees, methodTree{method: method, root: root})
	}
    // 将新的路由添加到对应的路由树中，其中 path 是路由的路径，handlers 是路由的处理函数链。
	root.addRoute(path, handlers)

    // 计算当前路由的路径中的参数个数 paramsCount，并与 engine 的 maxParams 进行比较，如果大于 maxParams，则更新 maxParams。
	// Update maxParams
	if paramsCount := countParams(path); paramsCount > engine.maxParams {
		engine.maxParams = paramsCount
	}
	// 计算当前路由的路径中的段（部分路径）个数 sectionsCount，并与 engine 的 maxSections 进行比较，如果大于 maxSections，则更新 maxSections。
	if sectionsCount := countSections(path); sectionsCount > engine.maxSections {
		engine.maxSections = sectionsCount
	}
}
```

#### Q5. 启动服务流程

##### Q5.01 启动服务

```go
// Run attaches the router to a http.Server and starts listening and serving HTTP requests.
// It is a shortcut for http.ListenAndServe(addr, router)
// Note: this method will block the calling goroutine indefinitely unless an error happens.
// 一键启动 Engine.Run 方法后，底层会将 gin.Engine 本身作为 net/http 包下 Handler interface 的实现类，并调用 http.ListenAndServe 方法启动服务.
func (engine *Engine) Run(addr ...string) (err error) {
	// ...
    // ListenerAndServe 方法本身会基于主动轮询 + IO 多路复用的方式运行，因此程序在正常运行时，会始终阻塞于 Engine.Run 方法，不会返回.
	err = http.ListenAndServe(address, engine.Handler())
	return
}
```

##### Q5.02 处理请求

在服务端接收到 http 请求时，会通过 Handler.ServeHTTP 方法进行处理. 而此处的 Handler 正是 gin.Engine，其处理请求的核心步骤如下：(代码见**Q4. 注册Handler流程**)

- 对于每个 http 请求，会为其分配一个 gin.Context，在 handlers 链路中持续向下传递
- 调用 Engine.handleHTTPRequest 方法，从路由树中获取 handlers 链，然后遍历调用
- 处理完 http 请求后，会将 gin.Context 进行回收. 整个回收复用的流程基于对象池管理

![image-20230723223346602](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/note/image-20230723223346602.png)

>Engine.handleHTTPRequest 方法核心步骤分为三步：
>
>- 根据 http method 取得对应的 methodTree
>- 根据 path 从 methodTree 中找到对应的 handlers 链
>- 将 handlers 链注入到 gin.Context 中，通过 Context.Next 方法按照顺序遍历调用 handler
>
>此处根据 path 从路由树寻找 handlers 的逻辑位于 root.getValue 方法中



```go
func (engine *Engine) handleHTTPRequest(c *Context) {
    // 获取当前 HTTP 请求的方法（GET、POST、PUT等）。
	httpMethod := c.Request.Method
    // 获取当前 HTTP 请求的路径。
	rPath := c.Request.URL.Path
    // 用于标记是否需要对路径进行解码，默认为 false
	unescape := false
    // 如果 engine.UseRawPath 为真，并且原始路径 c.Request.URL.RawPath 不为空，则使用原始路径 RawPath 替代 rPath，并根据 engine.UnescapePathValues 决定是否对路径进行解码。
	if engine.UseRawPath && len(c.Request.URL.RawPath) > 0 {
		rPath = c.Request.URL.RawPath
		unescape = engine.UnescapePathValues
	}
	// 如果 engine.RemoveExtraSlash 为真，则调用 cleanPath 函数处理路径，移除多余的斜杠。
	if engine.RemoveExtraSlash {
		rPath = cleanPath(rPath)
	}

     // 获取当前引擎中存储的所有路由树
	// Find root of the tree for the given HTTP method
	t := engine.trees
    // 循环遍历所有路由树，找到与当前 HTTP 方法匹配的树。
	for i, tl := 0, len(t); i < tl; i++ {
		if t[i].method != httpMethod {
			continue
		}
        // 找到匹配的树后，获取其根节点 root
		root := t[i].root
        // 在当前树中查找与请求路径匹配的路由。getValue 方法是路由树的核心，用于查找匹配的路由节点，并返回匹配的节点信息 value。
		// Find route in tree
		value := root.getValue(rPath, c.params, c.skippedNodes, unescape)
        // 如果 value 中包含参数信息，则将参数绑定到 *Context 的 Params 字段。
		if value.params != nil {
			c.Params = *value.params
		}
        // 如果 value 中包含处理函数链（HandlersChain），表示找到了匹配的路由，执行以下步骤：
		if value.handlers != nil {
            // 将处理函数链绑定到 *Context 的 handlers 字段
			c.handlers = value.handlers
            // 将匹配的完整路径绑定到 *Context 的 fullPath 字段
			c.fullPath = value.fullPath
            // 调用 Next 方法开始执行处理函数链，执行中间件和最终的业务逻辑处理函数。
			c.Next()
            // 在 Next 方法执行完成后，执行 WriteHeaderNow 方法，确保状态码被写入响应头。
			c.writermem.WriteHeaderNow()
			return
		}
        // 如果没有找到匹配的路由，并且当前 HTTP 方法不是 CONNECT，且请求路径不是根路径 /，执行以下步骤：
		if httpMethod != http.MethodConnect && rPath != "/" {
            // 如果路由树支持尾部斜杠重定向（value.tsr 为真），且引擎允许尾部斜杠重定向（engine.RedirectTrailingSlash 为真），则执行 redirectTrailingSlash 函数进行重定向，并直接返回，结束整个处理流程。
			if value.tsr && engine.RedirectTrailingSlash {
				redirectTrailingSlash(c)
				return
			}
            // 如果引擎允许固定路径重定向（engine.RedirectFixedPath 为真），且找到了固定路径的重定向目标，执行 redirectFixedPath 函数进行重定向，并直接返回，结束整个处理流程。
			if engine.RedirectFixedPath && redirectFixedPath(c, root, engine.RedirectFixedPath) {
				return
			}
		}
		break
	}

    // 如果引擎允许处理不允许的 HTTP 方法（engine.HandleMethodNotAllowed 为真），执行以下步骤：
	if engine.HandleMethodNotAllowed {
        // 遍历所有的路由树
		for _, tree := range engine.trees {
            // 如果树的 HTTP 方法与当前请求的 HTTP 方法相同，则跳过当前树
			if tree.method == httpMethod {
				continue
			}
            // 否则，通过调用 getValue 方法查找当前请求路径在该树中是否存在匹配的路由，如果存在，则表示该路径对应的 HTTP 方法不允许，执行以下步骤：
			if value := tree.root.getValue(rPath, nil, c.skippedNodes, unescape); value.handlers != nil {
                // 将 `*Context` 的 `handlers` 字段设置为 `engine.allNoMethod`，表示找不到对应的 HTTP 方法处理函数
				c.handlers = engine.allNoMethod
                // 调用 `serveError` 方法返回状态码为 `http.StatusMethodNotAllowed`（405）的错误响应，默认错误信息为 `default405Body`。
				serveError(c, http.StatusMethodNotAllowed, default405Body)
				return
			}
		}
	}
    // 如果没有找到匹配的路由或重定向，将 *Context 的 handlers 字段设置为 engine.allNoRoute，表示找不到对应的路由处理函数
	c.handlers = engine.allNoRoute
    // 调用 serveError 方法返回状态码为 http.StatusNotFound（404）的错误响应，默认错误信息为 default404Body
	serveError(c, http.StatusNotFound, default404Body)
}
```

#### Q6. Gin的路由树

##### Q6.01 策略与原理

1. 前缀树(https://leetcode.cn/problems/implement-trie-prefix-tree/)

   >前缀树又称 trie 树，是一种基于字符串公共前缀构建索引的树状结构，核心点包括：
   >
   >- 除根节点之外，每个节点对应一个字符
   >- 从根节点到某一节点，路径上经过的字符串联起来，即为该节点对应的字符串
   >- 尽可能复用公共前缀，如无必要不分配新的节点

2. 压缩前缀树

   > 压缩前缀树又称基数树或 radix 树，是对前缀树的改良版本，优化点主要在于空间的节省，核心策略体现在：
   >
   > ​			倘若某个子节点是其父节点的唯一孩子，则与父节点进行合并

   **在 gin 框架中，使用的正是压缩前缀树的数据结构**

3. 为什么使用压缩前缀树

   >与压缩前缀树相对的就是使用 hashmap(Spring MVC)，以 path 为 key，handlers 为 value 进行映射关联，这里选择了前者的原因在于：
   >
   >- path 匹配时不是完全精确匹配，比如末尾 ‘/’ 符号的增减、全匹配符号 '*' 的处理等，map 无法胜任（模糊匹配部分的代码于本文中并未体现，大家可以深入源码中加以佐证）
   >- 路由的数量相对有限，对应数量级下 map 的性能优势体现不明显，在小数据量的前提下，map 性能甚至要弱于前缀树
   >-  path 串通常存在基于分组分类的公共前缀，适合使用前缀树进行管理，可以节省存储空间


4. 补偿策略

   >在 Gin 路由树中还使用一种补偿策略，在组装路由树时，会将注册路由句柄数量更多的 child node 摆放在 children 数组更靠前的位置.
   >
   >这是因为某个链路注册的 handlers 句柄数量越多，一次匹配操作所需要花费的时间就越长，且被匹配命中的概率就越大，因此应该被优先处理.

![image-20230723225231850](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/note/image-20230723225231850.png)

##### Q6.02 核心数据结构

```go
// 对应于 9 种 http method，共有 9 棵 methodTree
// 每棵 methodTree 会通过 root 指向 radix tree 的根节点
type methodTree struct {
	method string // 请求类型
	root   *node // 节点
}

type node struct {
    // 节点的相对路径，拼接上 RouterGroup 中的 basePath 作为前缀后才能拿到完整的路由 path
	path      string
    // 由各个子节点 path 首字母组成的字符串，子节点顺序会按照途径的路由数量 priority进行排序
	indices   string
	wildChild bool
	nType     nodeType
    // 途径本节点的路由数量，反映出本节点在父节点中被检索的优先级
	priority  uint32
    // 子节点列表
	children  []*node // child nodes, at most 1 :param style node at the end of the array
    // 当前节点对应的处理函数链
	handlers  HandlersChain
    // path 拼接上前缀后的完整路径
	fullPath  string
}
```

##### Q6.03 注册到路由树

```go
// 插入新路由
func (n *node) addRoute(path string, handlers HandlersChain) {
    fullPath := path
    // 每有一个新路由经过此节点，priority 都要加 1
    n.priority++


    // 加入当前节点为 root 且未注册过子节点，则直接插入由并返回
    if len(n.path) == 0 && len(n.children) == 0 {
        n.insertChild(path, fullPath, handlers)
        n.nType = root
        return
    }


// 外层 for 循环断点
walk:
    for {
        // 获取 node.path 和待插入路由 path 的最长公共前缀长度
        i := longestCommonPrefix(path, n.path)
    
        // 倘若最长公共前缀长度小于 node.path 的长度，代表 node 需要分裂
        // 举例而言：node.path = search，此时要插入的 path 为 see
        // 最长公共前缀长度就是 2，len(n.path) = 6
        // 需要分裂为  se -> arch
                        -> e    
        if i < len(n.path) {
        // 原节点分裂后的后半部分，对应于上述例子的 arch 部分
            child := node{
                path:      n.path[i:],
                // 原本 search 对应的参数都要托付给 arch
                indices:   n.indices,
                children: n.children,              
                handlers:  n.handlers,
                // 新路由 see 进入时，先将 search 的 priority 加 1 了，此时需要扣除 1 并赋给 arch
                priority:  n.priority - 1,
                fullPath:  n.fullPath,
            }


            // 先建立 search -> arch 的数据结构，后续调整 search 为 se
            n.children = []*node{&child}
            // 设置 se 的 indice 首字母为 a
            n.indices = bytesconv.BytesToString([]byte{n.path[i]})
            // 调整 search 为 se
            n.path = path[:i]
            // search 的 handlers 都托付给 arch 了，se 本身没有 handlers
            n.handlers = nil           
            // ...
        }


        // 最长公共前缀长度小于 path，正如 se 之于 see
        if i < len(path) {
            // path see 扣除公共前缀 se，剩余 e
            path = path[i:]
            c := path[0]            


            // 根据 node.indices，辅助判断，其子节点中是否与当前 path 还存在公共前缀       
            for i, max := 0, len(n.indices); i < max; i++ {
               // 倘若 node 子节点还与 path 有公共前缀，则令 node = child，并调到外层 for 循环 walk 位置开始新一轮处理
                if c == n.indices[i] {                   
                    i = n.incrementChildPrio(i)
                    n = n.children[i]
                    continue walk
                }
            }
            
            // node 已经不存在和 path 再有公共前缀的子节点了，则需要将 path 包装成一个新 child node 进行插入      
            // node 的 indices 新增 path 的首字母    
            n.indices += bytesconv.BytesToString([]byte{c})
            // 把新路由包装成一个 child node，对应的 path 和 handlers 会在 node.insertChild 中赋值
            child := &node{
                fullPath: fullPath,
            }
            // 新 child node append 到 node.children 数组中
            n.addChild(child)
            n.incrementChildPrio(len(n.indices) - 1)
            // 令 node 指向新插入的 child，并在 node.insertChild 方法中进行 path 和 handlers 的赋值操作
            n = child          
            n.insertChild(path, fullPath, handlers)
            return
        }


        // 此处的分支是，path 恰好是其与 node.path 的公共前缀，则直接复制 handlers 即可
        // 例如 se 之于 search
        if n.handlers != nil {
            panic("handlers are already registered for path '" + fullPath + "'")
        }
        n.handlers = handlers
        // ...
        return
}  
    
    func (n *node) insertChild(path string, fullPath string, handlers HandlersChain) {
    // ...
    n.path = path
    n.handlers = handlers
    // ...
}
    
    //呼应于补偿策略，下面这段代码体现了，在每个 node 的 children 数组中，child node 在会依据 priority 有序排列，保证 priority 更高的 child node 会排在数组前列，被优先匹配.
    func (n *node) incrementChildPrio(pos int) int {
    cs := n.children
    cs[pos].priority++
    prio := cs[pos].priority
    // Adjust position (move to front)
    newPos := pos
    for ; newPos > 0 && cs[newPos-1].priority < prio; newPos-- {
        // Swap node positions
        cs[newPos-1], cs[newPos] = cs[newPos], cs[newPos-1]
    }
    // Build new index char string
    if newPos != pos {
        n.indices = n.indices[:newPos] + // Unchanged prefix, might be empty
            n.indices[pos:pos+1] + // The index char we move
            n.indices[newPos:pos] + n.indices[pos+1:] // Rest without char at 'pos'
    }
    return newPos
}
```

##### Q6.04 检索路由树

```go
type nodeValue struct {
    // 处理函数链
    handlers HandlersChain
    // ...
}

// 从路由树中获取 path 对应的 handlers 
func (n *node) getValue(path string, params *Params, skippedNodes *[]skippedNode, unescape bool) (value nodeValue) {
    var globalParamsCount int16
// 外层 for 循环断点
walk: 
    for {
        prefix := n.path
        // 待匹配 path 长度大于 node.path
        if len(path) > len(prefix) {
            // node.path 长度 < path，且前缀匹配上
            if path[:len(prefix)] == prefix {
                // path 取为后半部分
                path = path[len(prefix):]
                // 遍历当前 node.indices，找到可能和 path 后半部分可能匹配到的 child node
                idxc := path[0]
                for i, c := range []byte(n.indices) {
                    // 找到了首字母匹配的 child node
                    if c == idxc {
                        // 将 n 指向 child node，调到 walk 断点开始下一轮处理
                        n = n.children[i]
                        continue walk
                    }
                }
                // ...
            }
        }
        // 倘若 path 正好等于 node.path，说明已经找到目标
        if path == prefix {
            // ...
            // 取出对应的 handlers 进行返回 
            if value.handlers = n.handlers; value.handlers != nil {
                value.fullPath = n.fullPath
                return
            }
            // ...           
        }
        // 倘若 path 与 node.path 已经没有公共前缀，说明匹配失败，会尝试重定向，此处不展开
        // ...
 }  
```

#### Q7. Gin.Context

##### Q7.01 核心数据结构

![image-20230723230251874](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/note/image-20230723230251874.png)

> gin.Context 的定位是对应于一次 http 请求，贯穿于整条 handlersChain 调用链路的上下文，其中包含了如下核心字段：
>
> • Request/Writer：http 请求和响应的 reader、writer 入口
> • handlers：本次 http 请求对应的处理函数链
> • index：当前的处理进度，即处理链路处于函数链的索引位置
> • engine：Engine 的指针
> • mu：用于保护 map 的读写互斥锁
> • Keys：缓存 handlers 链上共享数据的 map

```go

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

	// This mutex protects Keys map.
	mu sync.RWMutex

	// Keys is a key/value pair exclusively for the context of each request.
	Keys map[string]any

	// Errors is a list of errors attached to all the handlers/middlewares who used this context.
	Errors errorMsgs

	// Accepted defines a list of manually accepted formats for content negotiation.
	Accepted []string

	// queryCache caches the query result from c.Request.URL.Query().
	queryCache url.Values

	// formCache caches c.Request.PostForm, which contains the parsed form data from POST, PATCH,
	// or PUT body parameters.
	formCache url.Values

	// SameSite allows a server to define a cookie attribute making it impossible for
	// the browser to send this cookie along with cross-site requests.
	sameSite http.SameSite
}
```

##### Q7.02 复用策略

gin.Context 作为处理 http 请求的通用数据结构，不可避免地会被频繁创建和销毁. 为了缓解 GC 压力，gin 中采用对象池 sync.Pool 进行 Context 的缓存复用，处理流程如下：

-  http 请求到达时，从 pool 中获取 Context，倘若池子已空，通过 pool.New 方法构造新的 Context 补上空缺
-  http 请求处理完成后，将 Context 放回 pool 中，用以后续复用

sync.Pool 并不是真正意义上的缓存，将其称为回收站或许更加合适，放入其中的数据在逻辑意义上都是已经被删除的，但在物理意义上数据是仍然存在的，这些数据可以存活两轮 GC 的时间，在此期间倘若有被获取的需求，则可以被重新复用.

##### Q7.03 分配与回收时机(代码见于Q4.00)

gin.Context 分配与回收的时机是在 gin.Engine 处理 http 请求的前后，位于 Engine.ServeHTTP 方法当中：

-  从池中获取 Context
-  重置 Context 的内容，使其成为一个空白的上下文
- 调用 Engine.handleHTTPRequest 方法处理 http 请求
- 请求处理完成后，将 Context 放回池中

##### Q7.04 使用时机

###### Q7.04.01 handlesChain 入口(代码见于Q5.02)

在 Engine.handleHTTPRequest 方法处理请求时，会通过 path 从 methodTree 中获取到对应的 handlers 链，然后将 handlers 注入到 Context.handlers 中，然后启动 Context.Next 方法开启 handlers 链的遍历调用流程.

###### Q7.04.02 handlesChain 遍历调用

推进 handlers 链调用进度的方法正是 Context.Next. 可以看到其中以 Context.index 为索引，通过 for 循环依次调用 handlers 链中的 handler.

```go
// Next 方法用于在 Gin 框架中依次执行注册的中间件和处理函数链
// 通过逐个调用 HandlersChain 中的中间件和处理函数，实现请求处理流程的链式执行
// 每次调用 Next 方法时，index 自增 1，表示执行下一个中间件或处理函数。当 index 大于或等于 handlers 切片的长度时，说明所有中间件和处理函数都已经执行完毕，该请求的处理流程结束
// 在每个中间件或处理函数中，可以根据需要选择是否继续执行 Next 方法，从而决定是否进入下一个中间件或处理函数。
func (c *Context) Next() {
    // 将 *Context 中的 index 属性自增 1(初始化为-1)。index 用于记录当前正在执行的中间件或处理函数的索引。
    c.index++
    for c.index < int8(len(c.handlers)) {
        // 根据当前 index 从 handlers 切片中获取对应的中间件或处理函数，并传入当前的 *Context 对象 c 进行调用
        // 由于 handlers 是一个 HandlersChain，其中保存了注册的中间件和处理函数，所以 c.handlers[c.index] 实际上是获取当前需要执行的中间件或处理函数
        c.handlers[c.index](c)
        // 将 index 属性自增 1，使得下一次循环时执行下一个中间件或处理函数
        c.index++
    }
}
```

由于 Context 本身会暴露于调用链路中，因此用户可以在某个 handler 中通过手动调用 Context.Next 的方式来打断当前 handler 的执行流程，提前进入下一个 handler 的处理中.

由于此时本质上是一个方法压栈调用的行为，因此在后置位 handlers 链全部处理完成后，最终会回到压栈前的位置，执行当前 handler 剩余部分的代码逻辑.

![image-20230723231332654](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/note/image-20230723231332654.png)

结合下面的代码示例来说，用户可以在某个 handler 中，于调用 Context.Next 方法的前后分别声明前处理逻辑和后处理逻辑，这里的“前”和“后”相对的是后置位的所有 handler 而言.(Spring AOC 环绕通知 ？？？？)

```go
func myHandleFunc(c *gin.Context){
    // 前处理
    preHandle()  
    c.Next()
    // 后处理
    postHandle()
}
```

此外，可以在某个 handler 中通过调用 Context.Abort 方法实现 handlers 链路的提前熔断.

其实现原理是将 Context.index 设置为一个过载值 63(前文提到的 **abortIndex**)，导致 Next 流程直接终止. 这是因为 handlers 链的长度必须小于 63，否则在注册时就会直接 panic. 因此在 Context.Next 方法中，一旦 index 被设为 63，则必然大于整条 handlers 链的长度，for 循环便会提前终止.

此外，用户还可以通过 Context.IsAbort 方法检测当前 handlerChain 是出于正常调用，还是已经被熔断

```go
func (c *Context) IsAborted() bool {
    return c.index >= abortIndex
}
```

###### Q7.04.03 共享数据存取

gin.Context 作为 handlers 链的上下文，还提供对外暴露的 Get 和 Set 接口向用户提供了共享数据的存取服务，相关操作都在读写锁的保护之下，能够保证并发安全.

```go

func (c *Context) Get(key string) (value any, exists bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()
    value, exists = c.Keys[key]
    return
}

func (c *Context) Set(key string, value any) {
    c.mu.Lock()
    defer c.mu.Unlock()
    if c.Keys == nil {
        c.Keys = make(map[string]any)
    }


    c.Keys[key] = value
}
```

#### Q8. 总结全文

- gin 将 Engine 作为 http.Handler 的实现类进行注入，从而融入 Golang net/http 标准库的框架之内
- gin 中基于 handler 链的方式实现中间件和处理函数的协调使用
- gin 中基于压缩前缀树的方式作为路由树的数据结构，对应于 9 种 http 方法共有 9 棵树
- gin 中基于 gin.Context 作为一次 http 请求贯穿整条 handler chain 的核心数据结构
- gin.Context 是一种会被频繁创建销毁的资源对象，因此使用对象池 sync.Pool 进行缓存复用
