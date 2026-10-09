---
title: Gin 访问日志定制：从 LogFormatter 接入结构化日志
date: 2023-01-03 23:22:30
updated: 2026-10-09
author: ws
description: 理解 Gin 的 Logger 中间件，把访问日志写进 zap 并加上请求 ID
categories: ["Go"]
tags: ["Go", "gin", "日志"]
cover:
---

## 引言

Gin 自带的访问日志默认打到标准输出，格式固定、没有请求 ID，接采集系统时只能当纯文本处理；网上常见的做法是写一个 `gin.LoggerWithFormatter` 格式化函数，但多数示例只改了显示样式，日志依旧进不了结构化管道。这篇文章先把 `LoggerWithFormatter` 的内部流程和 `LogFormatterParams` 的每个字段讲清楚，再看一段常见的 `logFormatter` 里签名不匹配、假行号、错误日志缺字段等问题，最后给出两个自写中间件 `RequestID()` 和 `AccessLog()`：用 zap 结构化记录 method、path、status、latency、client_ip、request_id，慢请求单独告警，4xx/5xx 分级，敏感参数脱敏。文中的代码在 Go 1.25.9、Gin v1.12.0、zap v1.28.0 下编译运行过，每条日志都附真实输出。

## LoggerWithFormatter 到底做了什么

`gin.LoggerWithFormatter(f)` 只是 `LoggerWithConfig` 的语法糖，两者最终都落到同一个闭包：

```go
// gin v1.12.0 logger.go（节选）
func LoggerWithFormatter(f LogFormatter) HandlerFunc {
	return LoggerWithConfig(LoggerConfig{Formatter: f})
}

// ... LoggerWithConfig 内部返回的中间件：

return func(c *Context) {
	// Start timer
	start := time.Now()
	path := c.Request.URL.Path
	raw := c.Request.URL.RawQuery

	// Process request
	c.Next()

	// Log only when it is not being skipped
	if _, ok := skip[path]; ok || (conf.Skip != nil && conf.Skip(c)) {
		return
	}

	param := LogFormatterParams{Request: c.Request, isTerm: isTerm, Keys: c.Keys}
	param.TimeStamp = time.Now()
	param.Latency = param.TimeStamp.Sub(start)
	param.ClientIP = c.ClientIP()
	param.Method = c.Request.Method
	param.StatusCode = c.Writer.Status()
	param.ErrorMessage = c.Errors.ByType(ErrorTypePrivate).String()
	param.BodySize = c.Writer.Size()
	if raw != "" && !conf.SkipQueryString {
		path = path + "?" + raw
	}
	param.Path = path

	fmt.Fprint(out, formatter(param))
}
```

整条链路可以画成：

```text
请求进入
  │
  ▼
LoggerWithConfig 返回的中间件
  ├─ start := time.Now()              计时开始
  ├─ 提前保存 path / rawQuery
  ├─ c.Next() ───────────────────────▶ 后续中间件与业务 handler
  │                                    （panic 被 Recovery 捕获、AbortWithStatus 等）
  ├─ ◀───────────────────────────────
  ├─ SkipPaths / Skip 过滤
  ├─ 组装 LogFormatterParams（此时 StatusCode、Latency 才确定）
  ├─ fmt.Fprint(out, formatter(param))  自定义格式化函数在这里被调用
  ▼
请求结束
```

几个直接影响定制方式的结论：

- `formatter` 的返回值就是整行日志的内容，gin 只负责把它 `Fprint` 到 `Output`（默认 `gin.DefaultWriter`，即 `os.Stdout`）。想把访问日志写进 zap，正确做法是自写中间件，而不是格式化一个字符串再塞给 zap。
- 计时范围是 `c.Next()` 前后，覆盖它后面的所有中间件和 handler；`Latency` 是未经截断的 `time.Duration`，截断发生在默认格式化函数里。
- `ErrorMessage` 来自 `c.Errors.ByType(gin.ErrorTypePrivate)`，也就是 handler 里 `c.Error(err)` 收集的错误；业务之外用它做判断没什么信息量。
- `SkipPaths`、`Skip`、`SkipQueryString` 是 `LoggerConfig` 的能力，后者是较新版本才有的（v1.12.0 已具备）：`SkipQueryString: true` 会把 query 整体从 `Path` 里去掉。注意它解决的是"不打印整段 query"，不是"只打敏感字段以外的部分"。

### LogFormatterParams 字段逐个说明

| 字段 | 类型 | 含义与来源 |
| --- | --- | --- |
| `Request` | `*http.Request` | 本次请求对象，可继续取 Header、URL 等 |
| `TimeStamp` | `time.Time` | 响应结束时刻，在 `c.Next()` 返回后取值 |
| `StatusCode` | `int` | `c.Writer.Status()`，默认 200 |
| `Latency` | `time.Duration` | `TimeStamp - start`，含全部后续处理耗时 |
| `ClientIP` | `string` | `c.ClientIP()`，结果受 `TrustedProxies` 配置影响 |
| `Method` | `string` | HTTP 方法 |
| `Path` | `string` | `URL.Path`；`SkipQueryString=false` 且 query 非空时拼成 `path?raw` |
| `ErrorMessage` | `string` | `c.Errors.ByType(ErrorTypePrivate).String()` |
| `BodySize` | `int` | `c.Writer.Size()`，响应体字节数 |
| `Keys` | `map[any]any` | `c.Keys`，即 `c.Set` 存进去的请求级键值 |
| `isTerm` | `bool`（私有） | `Output` 是否是终端，决定颜色是否输出 |

颜色相关的三个方法都挂在参数上：`StatusCodeColor()`（1xx 白、2xx 绿、4xx 黄、5xx 红）、`MethodColor()`（GET 蓝、POST 青、DELETE 红等）、`LatencyColor()`（按 100ms 开始分档，v1.12.0 中可用），以及判断是否该输出颜色的 `IsOutputColor()`：

```go
// gin v1.12.0
func (p *LogFormatterParams) IsOutputColor() bool {
	return consoleColorMode == forceColor || (consoleColorMode == autoColor && p.isTerm)
}
```

`isTerm` 在 `LoggerWithConfig` 里计算：`Output` 必须是 `*os.File`，且 `TERM != "dumb"`、文件描述符确实是终端。后台服务重定向输出时它是 `false`，颜色自动关闭；调试时可以 `gin.ForceConsoleColor()` 强制打开，或 `gin.DisableConsoleColor()` 关掉。

默认 formatter 里有一段容易看漏的延迟截断：

```go
// gin v1.12.0 defaultLogFormatter（节选）
switch {
case param.Latency > time.Minute:
	param.Latency = param.Latency.Truncate(time.Second * 10)
case param.Latency > time.Second:
	param.Latency = param.Latency.Truncate(time.Millisecond * 10)
case param.Latency > time.Millisecond:
	param.Latency = param.Latency.Truncate(time.Microsecond * 10)
}
```

也就是说默认日志的 `1.234567s` 会被显示成 `1.23s` 这种量级，精度随耗时变化。不少网上传抄的 `logFormatter` 里还在用一段只处理分钟档的截断逻辑：

```go
if param.Latency > time.Minute {
	// Truncate in a golang < 1.8 safe way
	param.Latency = param.Latency - param.Latency%time.Second
}
```

这段逻辑对正数 Duration 来说等价于 `Truncate(time.Second)`，本身没错，但它只处理了分钟档，而且注释里"golang < 1.8"的说法早就过时了。

## 一种常见的错误写法

先看一段很常见的 `logFormatter`：它试图把访问日志交给项目的日志封装，同时手动拼一行文本：

```go
func logFormatter(param gin.LogFormatterParams) string {
	var statusColor, methodColor, resetColor string
	if param.IsOutputColor() {
		statusColor = param.StatusCodeColor()
		methodColor = param.MethodColor()
		resetColor = param.ResetColor()
	}

	if param.Latency > time.Minute {
		param.Latency = param.Latency - param.Latency%time.Second
	}

	if len(param.ErrorMessage) > 0 {
		log.Error("%s,%s,%s", param.ClientIP, param.Method, param.ErrorMessage)
	}

	return fmt.Sprintf("%v\t[INFO]\tmiddleware/recorder.go:28\t[%s %3d %s| %13v | %15s |%s %-7s %s %#v]\n",
		param.TimeStamp.Format("2006-01-02 15:04:05.000000"),
		statusColor, param.StatusCode, resetColor,
		param.Latency,
		param.ClientIP,
		methodColor, param.Method, resetColor,
		param.Path,
	)
}

func Recorder() gin.HandlerFunc {
	return gin.LoggerWithFormatter(logFormatter)
}
```

这段代码看起来没问题，实际有 7 个隐患，按严重程度排列：

**1. `log.Error("%s,%s,%s", ...)` 和 `Error(args ...interface{})` 签名对不上。** 项目的 `log.Error` 是 `SugaredLogger` 风格的可变参数，收到的是 `[]interface{}{"%s,%s,%s", ip, method, err}`，整个切片又被当成**一个**参数传下去。实际输出是：

```text
{"level":"error","msg":"[%s,%s,%s 10.0.0.1 GET broken pipe]"}
```

格式串原样出现，参数被方括号包着挤成一条消息。更隐蔽的是：这种签名会把 `zap.Field` 当成普通参数。把字段传给 `SugaredLogger` 的实测输出：

```text
{"level":"error","msg":"access error{client_ip 15 0 10.0.0.1 <nil>} {method 15 0 GET <nil>}"}
{"level":"error","msg":"access error","client_ip":"10.0.0.1","method":"GET"}
```

前一条是 `sugar.Error(msg, zap.String(...))`，字段被打成结构体字面量；后一条是底层 `*zap.Logger` 的正确输出。写成 `sugar.Errorf("client_ip=%s method=%s", ...)` 可用，但那只是格式化字符串，采集端仍然拿不到字段。

**2. 硬编码 `middleware/recorder.go:28` 行号是假的。** 日志里印的行号应该由 zap 的 caller 机制在真实调用点生成（见 `AddCallerSkip`），写死在格式串里，代码一改行号就骗人；而真正有价值的位置信息（访问日志是在哪个中间件、哪个 handler 产生的）反而丢了。

**3. `%#v` 输出多余引号。** `%#v` 是 Go 语法表示，字符串路径会被加引号：

```text
"/ping"
```

采集端还得先剥一层引号。

**4. 错误日志缺字段。** `log.Error` 只带了 `client_ip`、`method`、`error` 三个值，没有 path、status、latency、request_id，事后排查"哪个接口的哪个请求报错了"只能靠猜时间线。

**5. 所有请求一视同仁。** 不论 200 还是 500，访问日志都是 `[INFO]` 一行文本，慢请求没有任何标记；线上想看"过去五分钟 5xx 和 P99 慢请求"，只能对纯文本做正则统计。

**6. 没有请求 ID。** 一次请求的访问日志、业务日志、下游调用日志之间无法串联，日志检索只能靠时间窗口。

**7. 颜色代码在生产环境不起作用。** `Output` 不是终端时 `IsOutputColor()` 恒为 false，这套 ANSI 颜色只在本地开发终端里有用，占用的是格式化函数的复杂度。

## 推荐的实现：两个中间件分别负责 ID 和日志

思路是把职责拆开，用两个中间件替代单个字符串格式化函数——`RequestID()` 只负责生成/透传请求 ID，`AccessLog()` 在请求结束后用 zap 记录结构化字段。

### 请求 ID 中间件

```go
const (
	// RequestIDHeader 是请求 ID 的 HTTP 头名称。
	RequestIDHeader = "X-Request-ID"
	// requestIDKey 是请求 ID 在 gin.Context 里的键。
	requestIDKey = "request_id"
	// slowThreshold 超过这个耗时的请求单独用 Warn 级别记录。
	slowThreshold = 300 * time.Millisecond
)

// RequestID 复用上游传入的请求 ID，没有就生成 UUID，
// 同时写回响应头并放进 Context，方便业务代码和日志串联。
func RequestID() gin.HandlerFunc {
	return func(c *gin.Context) {
		rid := c.GetHeader(RequestIDHeader)
		if rid == "" {
			rid = uuid.NewString()
		}
		c.Set(requestIDKey, rid)
		c.Writer.Header().Set(RequestIDHeader, rid)
		c.Next()
	}
}

// GetRequestID 供业务代码读取当前请求 ID。
func GetRequestID(c *gin.Context) string {
	rid, _ := c.Get(requestIDKey)
	s, _ := rid.(string)
	return s
}
```

上游网关通常已经生成了 trace ID，直接复用能少一层映射；没有就 `uuid.NewString()` 生成。ID 放进 `c.Set`，业务 handler 可以把它回显到响应体里，出错时用户报一个 ID 就能定位整条链路。

### 结构化访问日志中间件

```go
// AccessLog 在请求处理结束后打一条结构化访问日志。
func AccessLog() gin.HandlerFunc {
	return func(c *gin.Context) {
		start := time.Now()
		path := c.Request.URL.Path
		rawQuery := c.Request.URL.RawQuery

		c.Next()

		latency := time.Since(start)
		status := c.Writer.Status()

		fields := []zap.Field{
			zap.String("request_id", GetRequestID(c)),
			zap.String("method", c.Request.Method),
			zap.String("path", path),
			zap.Int("status", status),
			zap.Duration("latency", latency),
			zap.Int("size", c.Writer.Size()),
			zap.String("client_ip", c.ClientIP()),
			zap.String("user_agent", c.Request.UserAgent()),
		}
		if rawQuery != "" {
			fields = append(fields, zap.String("query", redactQuery(rawQuery)))
		}
		if len(c.Errors) > 0 {
			fields = append(fields, zap.String("errors", c.Errors.ByType(gin.ErrorTypePrivate).String()))
		}

		switch {
		case status >= 500:
			logx.L().Error("http_request", fields...)
		case status >= 400:
			logx.L().Warn("http_request", fields...)
		case latency >= slowThreshold:
			logx.L().Warn("slow_http_request", fields...)
		default:
			logx.L().Info("http_request", fields...)
		}
	}
}
```

和默认格式相比，这里刻意做了这些事：

- **字段化**：method、path、status、latency 等都是独立字段，采集端可以直接做聚合和告警；`method` 和 `status` 用 `zap.Int`/`zap.String` 强类型，数字就是数字。
- **分级**：5xx 记 Error、4xx 记 Warn、超过 300ms 的成功请求记 Warn 并把消息改成 `slow_http_request`，其余 Info。这样"错误率"和"慢请求"都有独立的日志级别可以利用，不用把文本 dump 出来再统计。
- **请求 ID 进每一条日志**，与业务日志共用同一个 ID。
- **不记录 Authorization、Cookie**：整个中间件没有引用这两个头，从源头上避免把凭证写进日志。字段是白名单式添加的，新增字段需要显式写一行，不会被"顺手打印整个请求"坑到。
- **query 脱敏**：

```go
// sensitiveKeys 是不允许以明文写进日志的 query 参数。
var sensitiveKeys = []string{"token", "access_token", "password", "secret", "api_key"}

// redactQuery 把敏感 query 参数替换成 ***，其余原样保留。
func redactQuery(raw string) string {
	values, err := url.ParseQuery(raw)
	if err != nil {
		return "(invalid-query)"
	}
	for _, key := range sensitiveKeys {
		if values.Has(key) {
			values.Set(key, "***")
		}
	}
	return values.Encode()
}
```

`url.ParseQuery` 把 query 解析成键值对，敏感键统一替换成 `***` 再 `Encode` 回字符串。注意 `Encode` 会做百分号转义，日志里看到的是 `token=%2A%2A%2A`。

### 中间件顺序决定 panic 请求能不能被记录

注册顺序是：

```go
r.Use(middleware.RequestID(), middleware.AccessLog(), gin.Recovery())
```

gin 的中间件按注册顺序形成调用链，先注册的在外层。`AccessLog` 的 `c.Next()` 内部调用 `Recovery`，handler 里 panic 时 `Recovery` 会恢复并写入 500 响应，控制权回到 `AccessLog`，所以这条请求的日志照样能打出来（`status` 字段为 500）。如果顺序反过来写成 `Recovery` 在外、`AccessLog` 在内，panic 会跳过 `AccessLog` 的剩余代码直接进 Recovery，这一条访问日志就丢了。真实日志里能同时看到 Recovery 的堆栈和访问日志：

```text
[Recovery] 2026/10/09 - 10:22:06 panic recovered:
boom
/private/var/folders/.../gindemo/main.go:42 (0x104d0029b)
	main.func6: panic("boom")

{"level":"error","time":"2026-10-09 10:22:06.942","msg":"http_request","request_id":"a7801fcf-2766-44fd-b50f-9d5d89f46db3","method":"GET","path":"/panic","status":500,"latency":"664.792µs","size":0,"client_ip":"127.0.0.1","user_agent":"curl/8.7.1"}
```

恢复堆栈由 `gin.Recovery()` 写到 stderr，结构化访问日志由 zap 写到 stdout，两者互不干扰；如果想把堆栈也收进 zap，需要用 `gin.CustomRecovery` 自己接。

## 完整示例与运行

示例的目录结构如下，日志器本身用最小实现（`logx`），重点是中间件：

```text
gindemo/
├── go.mod
├── main.go
├── logx/logx.go
└── middleware/access.go
```

`logx` 只做一件事：构建一个输出到 stdout 的 JSON Logger，且初始化是显式的，避免 `init()` 副作用：

```go
// Package logx 提供示例用的最小 zap Logger：JSON 输出、stdout。
package logx

import (
	"os"

	"go.uber.org/zap"
	"go.uber.org/zap/zapcore"
)

var logger *zap.Logger

// Init 显式初始化，避免在 init() 里产生副作用。
func Init() {
	encCfg := zapcore.EncoderConfig{
		TimeKey:        "time",
		LevelKey:       "level",
		MessageKey:     "msg",
		CallerKey:      zapcore.OmitKey,
		StacktraceKey:  zapcore.OmitKey,
		LineEnding:     zapcore.DefaultLineEnding,
		EncodeLevel:    zapcore.LowercaseLevelEncoder,
		EncodeTime:     zapcore.TimeEncoderOfLayout("2006-01-02 15:04:05.000"),
		EncodeDuration: zapcore.StringDurationEncoder,
	}
	core := zapcore.NewCore(
		zapcore.NewJSONEncoder(encCfg),
		zapcore.AddSync(os.Stdout),
		zapcore.InfoLevel,
	)
	logger = zap.New(core)
}

// L 返回全局 Logger。
func L() *zap.Logger { return logger }

// Sync 在进程退出前刷盘。
func Sync() {
	_ = logger.Sync()
}
```

`EncodeDuration` 选 `StringDurationEncoder`，`zap.Duration("latency", ...)` 输出 `"401.079083ms"` 这种可读形式，而不是整数纳秒。

路由和启动代码：

```go
func main() {
	logx.Init()
	defer logx.Sync()

	gin.SetMode(gin.ReleaseMode)
	r := gin.New()
	r.Use(middleware.RequestID(), middleware.AccessLog(), gin.Recovery())

	r.GET("/ping", func(c *gin.Context) {
		c.JSON(http.StatusOK, gin.H{"message": "pong", "request_id": middleware.GetRequestID(c)})
	})
	r.GET("/users/:id", func(c *gin.Context) {
		c.JSON(http.StatusOK, gin.H{"id": c.Param("id")})
	})
	r.GET("/slow", func(c *gin.Context) {
		time.Sleep(400 * time.Millisecond)
		c.JSON(http.StatusOK, gin.H{"ok": true})
	})
	r.GET("/forbidden", func(c *gin.Context) {
		c.AbortWithStatus(http.StatusForbidden)
	})
	r.GET("/fail", func(c *gin.Context) {
		err := errors.New("db timeout")
		_ = c.Error(err)
		c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
	})
	r.GET("/panic", func(c *gin.Context) {
		panic("boom")
	})

	if err := r.Run("127.0.0.1:18080"); err != nil {
		logx.L().Fatal("server exit", zap.Error(err))
	}
}
```

依赖版本：`github.com/gin-gonic/gin v1.12.0`、`github.com/google/uuid v1.6.0`、`go.uber.org/zap v1.28.0`。启动后发几个不同类型的请求：

```bash
go run .
curl -s http://127.0.0.1:18080/ping
curl -s -H 'X-Request-ID: trace-abc123' 'http://127.0.0.1:18080/ping?token=supersecret&q=go'
curl -s http://127.0.0.1:18080/users/42
curl -s -o /dev/null -w 'forbidden -> %{http_code}\n' http://127.0.0.1:18080/forbidden
curl -s -o /dev/null -w 'slow -> %{http_code}\n' http://127.0.0.1:18080/slow
curl -s -o /dev/null -w 'fail -> %{http_code}\n' http://127.0.0.1:18080/fail
curl -s -o /dev/null -w 'panic -> %{http_code}\n' http://127.0.0.1:18080/panic
```

响应和退出码：

```text
{"message":"pong","request_id":"2a8eb890-ff36-4594-8e09-17d3603c12c1"}
{"message":"pong","request_id":"trace-abc123"}
{"id":"42"}
forbidden -> 403
slow -> 200
fail -> 500
panic -> 500
```

服务端日志（每条一行 JSON）：

```text
{"level":"info","time":"2026-10-09 10:22:06.482","msg":"http_request","request_id":"2a8eb890-ff36-4594-8e09-17d3603c12c1","method":"GET","path":"/ping","status":200,"latency":"71.375µs","size":70,"client_ip":"127.0.0.1","user_agent":"curl/8.7.1"}
{"level":"info","time":"2026-10-09 10:22:06.492","msg":"http_request","request_id":"trace-abc123","method":"GET","path":"/ping","status":200,"latency":"14µs","size":46,"client_ip":"127.0.0.1","user_agent":"curl/8.7.1","query":"q=go&token=%2A%2A%2A"}
{"level":"info","time":"2026-10-09 10:22:06.501","msg":"http_request","request_id":"011906e5-a51d-4a75-9362-dc90eabe5673","method":"GET","path":"/users/42","status":200,"latency":"16.75µs","size":11,"client_ip":"127.0.0.1","user_agent":"curl/8.7.1"}
{"level":"warn","time":"2026-10-09 10:22:06.511","msg":"http_request","request_id":"b3b60509-12cd-44f7-826b-05c2a07698db","method":"GET","path":"/forbidden","status":403,"latency":"33.625µs","size":0,"client_ip":"127.0.0.1","user_agent":"curl/8.7.1"}
{"level":"warn","time":"2026-10-09 10:22:06.922","msg":"slow_http_request","request_id":"432e24df-1be7-412d-aef0-e10f03f83047","method":"GET","path":"/slow","status":200,"latency":"401.079083ms","size":11,"client_ip":"127.0.0.1","user_agent":"curl/8.7.1"}
{"level":"error","time":"2026-10-09 10:22:06.931","msg":"http_request","request_id":"268f587e-3189-4e8d-b5cb-1198054b1cdb","method":"GET","path":"/fail","status":500,"latency":"37.542µs","size":22,"client_ip":"127.0.0.1","user_agent":"curl/8.7.1","errors":"Error #01: db timeout\n"}
```

对着这几条日志可以核验全文的结论：第二条的 `request_id` 是调用方传入的 `trace-abc123`，说明透传生效；`token` 被替换成 `%2A%2A%2A`，`q=go` 保留；403 是 warn、慢请求是 `slow_http_request`、`/fail` 是 error，分级符合设计；`errors` 字段带上了 `c.Error` 收集的业务错误。

## 踩坑与边界

- **`c.Writer.Size()` 不能当业务指标**：没写响应体时它是 0（`AbortWithStatus` 后就是 0），走 Hijack、SSE、流式输出的路径还可能不准。想要精确流量统计应该在网关或 `ResponseWriter` 包装层做。
- **panic 日志别写两遍**：`gin.Recovery()` 默认把堆栈写到 `gin.DefaultErrorWriter`（stderr），访问日志只记 status=500。如果还想要结构化堆栈，用 `gin.CustomRecovery` 把 `err` 和 `stack` 收集成字段，而不是让两套机制各写一份。
- **goroutine 里不要直接用 `gin.Context`**：请求结束后 `gin.Context` 会被对象池复用，异步任务要用 `c.Copy()`。更常见的做法是把需要的字段（request_id、业务参数）提前取出来传进 goroutine，日志用 zap，不走 Context。
- **不要无条件信任客户端的 `X-Request-ID`**：它可以很长、可以带换行符（日志注入）、可以伪造。网关层最好统一生成并覆盖，或者至少校验格式、限制长度、剔除控制字符。
- **高 QPS 下成功请求要采样**：每请求一行在几万 QPS 下会把磁盘和采集成本推高。常见策略是 2xx 按比例采样（例如 1/100），4xx/5xx 和慢请求全量，采样率作为字段一起打出来，统计时按权重还原。
- **`SkipQueryString` 和手工脱敏是两回事**：`SkipQueryString: true` 是把 query 整个丢掉，适合"query 全都不需要"的场景；需要保留非敏感参数时，还是得像本文的 `redactQuery` 一样按 key 脱敏。
- **格式化函数不是日志管道**：`LogFormatter` 只是"把参数拼成字符串"，在这个函数里做网络请求、查库、写文件都会直接拖慢请求。要写结构化日志，就用中间件在 `c.Next()` 之后记录。

## 总结

- `gin.LoggerWithFormatter` 的链路是：计时 → `c.Next()` → 组装 `LogFormatterParams` → `Fprint(formatter(param))`；formatter 只应做字符串格式化。
- `LogFormatterParams` 提供了 status、latency、client IP、`c.Errors` 等现成信息，但错误信息字段有限、格式固定，认真做可观测性要走自定义中间件。
- 常见错误写法的核心 bug 是可变参数签名不匹配，`%s` 不会被替换；配套问题是假行号、`%#v` 引号、错误日志缺 method/path/request_id、无分级、无慢请求标记。
- 正确做法是用两个中间件替代单个字符串格式化函数：`RequestID` 解决链路串联，`AccessLog` 用 zap 打结构化字段，按 5xx/4xx/慢请求/正常四档分级告警。
- 中间件注册顺序必须是 `RequestID → AccessLog → Recovery`，否则 panic 请求会丢访问日志。
- 敏感信息靠"白名单字段 + query 脱敏"控制；高 QPS 场景给成功请求加采样，错误与慢请求保持全量。
