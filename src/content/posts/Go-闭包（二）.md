---
title: "Go: 闭包（二）"
pubDatetime: 2026-09-11T00:00:00+08:00
description: "理解 Go 闭包的变量捕获、生命周期、常见用法以及并发场景中的注意事项。"
tags:
  - Go
  - Grammar
---

## 一、闭包捕获变量的方式

先看一个小例子：

```go
count := 0
f := func() {
	count++
}
f()
f()
fmt.Println(count) // 输出 2
```

这里 `f` 捕获的是 `count` 这个**变量本身**，即闭包和外层代码共享同一个 `count`。所以两次调用 `f()` 修改的是同一个变量，最终输出 2。这和把值拷贝一份存进去是完全不同的。

### 两个注意点

**1、闭包可能导致被捕获变量逃逸**，如果闭包的生命周期超过了声明该变量的函数调用，那这个变量就不能随着原栈帧失效，编译期需要保证它继续存活，此时通常会发生逃逸。

每次创建闭包，背后可能会有堆分配。大多数场景这点开销无关紧要，但如果你在热点路径上大量创建闭包（比如每个 Http 请求都 `makeLogger` 一次），这就是需要考虑的性能问题了。

可以使用：

```sh
go build -gcflags="-m" .
```

来观察编译期的逃逸分析结果。

**2、闭包活着，变量就活着**，既然共享同一个变量，那么变量的生命周期就被闭包接管了：闭包不被回收，被捕获的变量就不能被 GC。这带来一个隐蔽的内存问题：

```go
func handlerWithCache() http.HandlerFunc {
	bigCache := loadHugeCache() // 几百 MB

	return func(w http.ResponseWriter, r *http.Request) {
		uid := r.URL.Query().Get("uid")
		_ = bigCache.users[uid] // 只用了一小块
		w.Write([]byte("ok"))
	}
}
```

闭包只要引用了 `bigCache`，哪怕只用其中一个字段，整个 `bigCache` 都会因为闭包的存在而无法释放。反过来说，闭包并不会拖住它**没有引用**的外层变量——编译器只保留实际被引用的那些；但只要引用了，哪怕只用一处，整个对象都逃不掉。写长生命周期的闭包（常驻的 Handler、全局注册的回调）时，要留意它把什么拖住了。

记住**逃逸**和**延长生命周期**这两个特点。接下来的场景，都是它们的具体表现。

## 二、依赖注入

函数在 Golang 中可以来去自如，可以当参数传、当返回值返回。而闭包在此基础上更进一步：返回的函数带着自己捕获的变量一起走。

于是最自然的用法出现了——**依赖注入**：

```go
func makeUserHandler(db *sql.DB) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		rows, err := db.Query("SELECT id, name FROM users")
		if err != nil {
			http.Error(w, err.Error(), http.StatusInternalServerError)
			return
		}
		defer rows.Close()
		// ...
	}
}
```

`makeUserHandler` 在启动时调用一次，产生的 Handler 就永久"记住"了 `db`。之后每个请求到来时，只需调用 `handler(w, r)`。

这其实就是**依赖注入**，只是不需要任何框架。因为函数是值，依赖可以直接塞进函数里，构造函数是组装点，返回值是**携带依赖的运行单元**。

## 三、中间件

把上面这个思路再推一步，就是中间件。以一个认证中间件举例：

```go
func AuthMiddleware(jwtManager *JWTManager) gin.HandlerFunc {
	return func(c *gin.Context) {
		token := strings.TrimPrefix(c.GetHeader("Authorization"), "Bearer ")

		claims, err := jwtManager.Parse(token)
		if err != nil {
			c.AbortWithStatus(http.StatusUnauthorized)
			return
		}

		c.Set("user_id", claims.UserID)
		c.Next()
	}
}
```

这里返回的匿名函数捕获了 `jwtManager` 这个对象，初始化时就可以给它注入：

```go
authMiddleware := AuthMiddleware(jwtManager)
```

然后在路由中使用这个中间件：

```go
router.Use(authMiddleware)
```

此后每一个请求都会使用同一个`jwtManager`

组装与执行分离，依赖在组装期一次性注入，运行期无需额外查找、直接可用——这正是闭包"预捕获"特性的工程价值。

## 四、预绑定参数

### 先看一个简单例子

```go
func makeLogger(prefix string) func(string) {
	return func(message string) {
		fmt.Printf("[%s] %s\n", prefix, message)
	}
}

dbLog := makeLogger("DB")     // 捕获 prefix = "DB"
httpLog := makeLogger("HTTP") // 捕获 prefix = "HTTP"

dbLog("connected")    // [DB] connected
httpLog("GET /users") // [HTTP] GET /users
```

其实 `dbLog` 和 `httpLog`两个函数执行同一套逻辑，唯一区别是捕获的 `prefix` 不同。闭包在这里相当于**把部分参数提前绑定，生成一个新函数**。这在函数式编程里叫偏应用（partial application），在 Go 里它就是闭包最朴素的用法。

### Functional Options

如果配置项越来越多，构造函数会变成这样：

```go
func NewServer(port int, debug bool, tlsConfig *tls.Config, readTimeout time.Duration, /* ... */) *Server
```

问题很明显：

- **参数膨胀**，调用处一长串裸值，`NewServer(8080, false, nil, 30*time.Second)`
- **无法演进**。给构造函数加一个参数，是所有调用方的编译错误，对库的作者来说就是 breaking change。
- **校验无处安放**。端口合法范围、超时不能为负——这些规则和参数本身分离了。

Functional Options 用闭包把每个配置项变成一个"配置动作"：

```go
type Server struct {
	port        int
	debug       bool
	readTimeout time.Duration
}

type Option func(*Server)

func WithPort(port int) Option {
	return func(s *Server) {
		if port <= 0 || port > 65535 {
			panic("invalid port")
		}
		s.port = port
	}
}

func WithDebug(debug bool) Option {
	return func(s *Server) {
		s.debug = debug
	}
}
```

注意这里的质变：`WithPort(8080)` 返回的闭包捕获了 `port = 8080`，并把端口合法性校验**和参数绑定在了一起**。配置结构体做不到这一点——结构体只是数据，闭包是"数据 + 行为"。（示例里校验失败用 panic 简化；真实库一般让 `Option` 返回 `error` 而不是 panic，此处从略。）

组装时依次执行这些"配置动作"：

```go
func NewServer(options ...Option) *Server {
	server := &Server{port: 8080} // 默认值集中在组装处
	for _, option := range options {
		option(server) // 每个闭包修改同一个 server
	}
	return server
}

server := NewServer(
	WithPort(9000),
	WithDebug(true),
)
```

**这里多个闭包共享并依次修改同一个 `server` 变量**——这正是第一节"捕获变量本身、共享变量"的直接应用。看懂了这个，`grpc.DialOption`、`zap` 的 `zap.Option` 这些真实代码你也能一眼看穿。

## 五、执行时机

闭包捕获的变量是调用时才取值的，这个特性在 `defer` 上表现得最戏剧化：

```go
func main() {
	i := 0
	defer fmt.Println("值拷贝：", i) // defer 语句的实参立即求值 → 打印 0
	defer func() {
		fmt.Println("闭包：", i) // 执行时才取值 → 打印 1
	}()
	i++
}
```

输出：

```go
闭包： 1
值拷贝： 0
```

`defer fmt.Println(i)` 在注册那一刻就把 `i` 的值定死了；而 `defer func(){ ... }()` 注册的是函数，函数体里的 `i` 要到函数真正执行时才读取——那时 `i` 已经是 1。

这个区别决定了两类写法：

```go
func handle() {
	start := time.Now()
	defer func() {
		log.Printf("cost=%s", time.Since(start)) // 必须闭包：结束时才算耗时
	}()
	// ...
}
```

反过来，如果你**希望**固定住当下的值，就传参而不是捕获：

```go
for _, f := range files {
	defer closeFile(f) // 每个 defer 绑定各自的 f，正确
}
```

这里 `closeFile(f)` 的实参在每次迭代时立即求值，所以五个 defer 各关各的文件。如果写成 `defer func() { f.Close() }()`，所有闭包共享循环变量 `f`（1.22 之前）——就变成了五个闭包都关最后一个文件。理解了执行时机，这类 bug 是完全可以推理出来的，不需要背。

顺带一提：`defer` 注册在循环体里会把所有闭包压到函数退出时才执行，文件句柄可能因此耗尽。配合上面这条，就能写出两种正确的循环清理模式：把循环体抽成函数，或注册时即传参。

## 六、并发

### 循环变量捕获

```go
for i := 0; i < 10; i++ {
	go func() {
		fmt.Println(i)
	}()
}
```

Go 1.22 之前，这段代码大概率打印十个相同的数字。原因用第一节的机制就能推出来：循环里只声明**一个** `i`，十个闭包捕获的是同一个变量。等 goroutine 被调度执行时，循环早已跑完，它们读到的都是终值。

官方最终也认为这是语言的坑，**Go 1.22 起循环变量改为每次迭代新声明**，每个闭包捕获的是各自的副本，上面代码自然输出 0 到 9。

修复手法：

```go
// 修复方式一：再声明一次，遮蔽外层变量
//（Go 1.22 起循环变量已按迭代独立，这行在新版本中是冗余的，保留也无害）
for i := 0; i < 10; i++ {
	i := i
	go func() { fmt.Println(i) }()
}

// 修复方式二：通过参数传递，值拷贝进闭包
for i := 0; i < 10; i++ {
	go func(n int) { fmt.Println(n) }(i)
}
```

两种方式做了同一件事：**切断闭包与外层变量的共享**。方式二更地道——依赖就应该出现在参数列表里，这既解决了共享问题，也让依赖显式可见。

### 数据竞争

```go
count := 0
var wg sync.WaitGroup
for i := 0; i < 10; i++ {
	wg.Add(1)
	go func() {
		defer wg.Done()
		count++ // 十个 goroutine 通过闭包并发读写同一个变量
	}()
}
wg.Wait()
```

这里的 `count++` 不是原子操作（读、加、写三步），多个 goroutine 交织执行会丢更新。用 `go run -race` 或 `go test -race` 可以让竞态在运行时现形。但要厘清责任：**问题不在闭包，在于多个 goroutine 共享了一个可变变量**。闭包只是把"共享"这件事变得方便了，方便到有隐蔽性——你看不到任何传参，就意识不到发生了共享。

判断准则很简单：**闭包捕获的变量，会被多个 goroutine 同时读写吗？** 会，就需要 `sync.Mutex`、`sync/atomic` 或 channel 来保护；或者干脆用参数把值拷进去，从根上取消共享。

```go
var count atomic.Int64
var wg sync.WaitGroup
for i := 0; i < 10; i++ {
	wg.Add(1)
	go func() {
		defer wg.Done()
		count.Add(1)
	}()
}
wg.Wait()
```

## 七、隐式依赖

看这个函数：

```go
func() { ... }
```

它看似不依赖任何东西，实则背后可能有一堆依赖：

```go
func() {
	db.Query(...)    // 连了数据库
	redis.Get(...)   // 连了缓存
	logger.Info(...) // 依赖日志组件
	cfg.Timeout      // 依赖全局配置
}
```

依赖没有出现在函数签名里，就没办法从调用处看出这个函数的真实复杂度，测试时也不知道该 mock 什么。被捕获的依赖越多，这个函数越像一个黑盒。

所以，在实践中需要控制闭包的职责，例如：

- 闭包负责**组装**（捕获依赖、对外提供统一入口）；
- 业务逻辑收敛到**显式参数的纯函数**里，便于单测。

让闭包当粘合层，不让它当业务层。
