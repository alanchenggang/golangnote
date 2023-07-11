1. os.Stat 通过返回err判断文件是否存在

   ```go
   if _, err := os.Stat("go.mod"); err == nil {
   		base.Fatalf("go mod init: go.mod already exists")
   }
   ```

2. os.Getwd 获取当前目录

   ```go
   	cwd, err = os.Getwd()
   	if err != nil {
   		base.Fatalf("go: %v", err)
   	}
   ```

   

3. switch

   ```go
   switch env {
   	default:
   		base.Fatalf("go: unknown environment setting GO111MODULE=%s", env)
   	case "auto":
   		mustUseModules = ForceUseModules
   	case "on", "":
   		mustUseModules = true
   	case "off":
   		if ForceUseModules {
   			base.Fatalf("go: modules disabled by GO111MODULE=off; see 'go help modules'")
   		}
   		mustUseModules = false
   ```

   4. filepath.SplitList(path)

      > SplitList 拆分由特定于操作系统的 ListSeparator 连接的路径列表，通常在 PATH 或 GOPATH 环境变量中找到。与 strings.Split 不同，SplitList 在传递空字符串时返回一个空切片。

   5. sync.Once

      ![image-20230712004458870](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/note/image-20230712004458870.png)
   
      >`sync.Once` 是 Go 语言中的一个并发原语，用于确保某个操作只执行一次。它的常见用法是在程序启动时执行一些初始化操作，例如初始化全局变量或者建立一些连接。
      >
      >`sync.Once` 只有一个方法 `Do`，该方法接收一个函数作为参数。`Do` 方法会保证该函数只被执行一次，即使在多个 goroutine 中同时调用 `Do` 方法也不会重复执行该函数。
      >
      >```go
      >package main
      >
      >import (
      >    "fmt"
      >    "sync"
      >)
      >
      >var (
      >    once     sync.Once
      >    database map[string]string
      >)
      >
      >func initialize() {
      >    fmt.Println("Initializing database...")
      >    database = make(map[string]string)
      >    database["foo"] = "bar"
      >}
      >
      >func main() {
      >    // 这里调用了两次 Do 方法，但 initialize 函数只会被执行一次
      >    once.Do(initialize)
      >    once.Do(initialize)
      >
      >    fmt.Println(database["foo"])
      >}
      >```
      >
      >在上面的代码中，我们定义了一个 `once` 变量作为 `sync.Once` 的实例，以及一个 `database` 变量作为我们要初始化的全局变量。在 `initialize` 函数中，我们初始化了 `database` 变量，并输出一条日志。
      >
      >在 `main` 函数中，我们两次调用了 `once.Do(initialize)` 方法，但是 `initialize` 函数只会被执行一次。最后我们输出了 `database` 变量中的一个值，以验证初始化操作是否成功。
      >
      >需要注意的是，`sync.Once` 只能保证函数只被执行一次，但是无法保证函数执行的顺序。如果多个 goroutine 同时调用 `Do` 方法，那么它们可能会在不同的时间执行 `initialize` 函数。如果需要保证函数的执行顺序，可以使用其他的并发原语，例如 `sync.Mutex`
   
   6. 