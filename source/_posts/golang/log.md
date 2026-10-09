---
title: 用 zap + lumberjack 搭建生产级 Go 日志
date: 2023-01-03 23:22:30
updated: 2026-10-09
author: ws
description: 讲透 zap 的 Core/Encoder/WriteSyncer，配好切割、级别与调用位置
categories: ["Go"]
tags: ["Go", "zap", "日志"]
cover:
---

## 引言

项目上线后，日志要在三个场景里同时可用：出问题时能按请求追到具体代码行、磁盘写满前能自动切割、接采集系统时能被解析成字段。很多项目里的日志封装习惯用 `fmt.Sprintf` 拼字符串起步，等日志量上来才发现字段对不上、行号指向封装函数、日志文件把磁盘占满。这篇文章先看一种常见的日志封装写法，拆解其中几个容易在生产里出问题的点，再把 zap 的 Core、Encoder、WriteSyncer 讲清楚，最后给出一个可以直接拿去用的日志包，并用真实输出验证 caller 行号、动态级别、Fatal 退出和 lumberjack 切割这些容易踩坑的地方。文中的代码在 Go 1.25.9、zap v1.28.0、lumberjack v2.2.1 下全部编译运行过。

## 字符串日志的问题，以及 zap 的定位

先看两种日志在采集端的差距。字符串日志：

```text
2026-10-09 10:00:00 [INFO] user login user=ws uid=1001 note=hello world
```

结构化日志（zap 的 JSON 输出）：

```json
{"level":"info","time":"2026-10-09 10:00:00.123456","caller":"demo/main.go:35","msg":"user login","user":"ws","uid":1001}
```

字符串日志里 `note=hello world` 带了空格，采集时按空格切分就错了；想按 `uid` 做聚合查询，还得写正则；再加两个字段，正则就要跟着改。结构化日志把每个字段单独编码，`uid` 是数字，`user` 是字符串，采集端直接按 key 取，这是后面接入 Loki、ES 的前提。

zap 是 Uber 开源的日志库，它的 API 分两套：

| | `*zap.Logger` | `*zap.SugaredLogger` |
| --- | --- | --- |
| 字段 | 强类型 `zap.Field` | 任意参数，fmt 风格 |
| 性能 | 更快，可做到禁用级别零分配 | 慢一些，内部走 `fmt.Sprint` 类似的路径 |
| 适用 | 热点路径、生产日志 | 从 fmt 风格迁移、参数不固定的场景 |

在 Apple M1 Pro、Go 1.25.9、JSON encoder 写内存 sink 的条件下实测（每个基准 100 万次迭代）：

| 写法 | ns/op | B/op | allocs/op |
| --- | --- | --- | --- |
| `zap.Logger.Info` 带两个字段 | 350.6 | 128 | 1 |
| `zap.SugaredLogger.Infow` 带两个键值 | 556.2 | 256 | 1 |
| 级别关闭时 `Logger.Debug`（字段已构造） | 38.0 | 128 | 1 |
| 级别关闭时 `Logger.Check` | 3.4 | 0 | 0 |
| `fmt.Fprintf` | 65.5 | 0 | 0 |
| 标准库 `log.Printf` | 137.2 | 0 | 0 |

两点结论：一，`SugaredLogger` 大约是 `Logger` 的 1.5 倍开销，不是不能用，但热点路径优先用 `Logger`；二，禁用级别时 `Debug` 仍有 128B 分配，那是**调用方构造字段切片**的成本，`Logger.Check` 把构造推迟到确认要写之后，才是真正的零分配路径。这组数字只反映 CPU 侧开销，真正写文件时 IO 才是大头。

## Core 三件套：Logger 只是入口

`zap.Logger` 本身几乎不做决策，它把日志交给 `zapcore.Core`，Core 由三个部件组成：

```text
业务代码
  │  log.Info("user login", zap.String("user", "ws"))
  ▼
┌────────────┐  Enabled(level)?  ┌──────────────────────────────┐
│ zap.Logger │ ────────────────▶ │ Core                         │
│ 入口/封装   │                   │  ├─ LevelEnabler（AtomicLevel）│
└────────────┘                   │  ├─ Encoder（Console / JSON） │
                                 │  └─ WriteSyncer（多路输出）   │
                                 └───────────────┬──────────────┘
                                                 │ 编码后的一行字节
                                       ┌─────────┴──────────┐
                                       ▼                    ▼
                                  os.Stdout        lumberjack → 文件
```

- **LevelEnabler**：这条日志要不要写。`AtomicLevel` 是它最常见的实现，支持运行时改级别。
- **Encoder**：怎么写。`ConsoleEncoder` 给人看，`JSONEncoder` 给机器采集，字段编码规则来自 `EncoderConfig`。
- **WriteSyncer**：写去哪。`zapcore.AddSync` 把任意 `io.Writer` 适配成 WriteSyncer，`MultiWriteSyncer` 把多个目标串成一条扇出。

理解了这三件套，后面所有配置代码都只是"往哪个部件塞什么参数"。

## 一种常见的日志封装及其隐患

下面这段代码看起来没问题，实际有 8 个隐患，会在生产中逐个咬人：

```go
var logger *zap.SugaredLogger

func init() {
	const logDirPath = "./log"
	if !utils.IsPathExist(logDirPath) {
		utils.CreatDir(logDirPath) // 返回值被忽略
	}
	logger = initLogger(filepath.Join(logDirPath, "acc.log"), "debug")
}

func Error(args ...interface{}) {
	logger.Error(args) // args 整个切片被当成一个参数
}

func initLogger(logPath string, loglevel string) *zap.SugaredLogger {
	// ...
	atomicLevel := zap.NewAtomicLevel()
	atomicLevel.SetLevel(level)
	core := zapcore.NewCore(
		zapcore.NewConsoleEncoder(encoderConfig),
		zapcore.NewMultiWriteSyncer(zapcore.AddSync(os.Stdout), zapcore.AddSync(write)),
		level, // 传的是 level，不是 atomicLevel
	)
	development := zap.Development() // 注释写着"堆栈跟踪"，其实不会加堆栈
	logger := zap.New(core, caller, development, zap.AddCallerSkip(1))
	return logger.Sugar()
}
```

逐条说明：

1. **`init()` 里创建目录、打开文件**：只要这个包被 import，副作用就发生；目录创建失败没法上报，只能祈祷；单元测试想换个目录也做不到。更稳妥的做法是提供显式 `Init(cfg)`。
2. **`AtomicLevel` 白建了**：创建、设置完 `atomicLevel`，传给 `NewCore` 的却是普通变量 `level`，动态改级别能力等于没接上。
3. **级别有两套开关**：core 的 `"debug"` 级别和 `config.IsDebug()` 各管一半。配置说 debug 但级别是 info，`Debug` 调用静默消失；两者不一致时没人能快速说清日志为什么没出来。应该让级别成为唯一事实来源。
4. **`logger.Error(args)` 参数直传**：`args` 是 `[]interface{}`，传给 `SugaredLogger.Error(args ...interface{})` 时整个切片变成**一个**参数。把这段代码实际跑一遍：

   ```go
   func logErrorBad(l *zap.SugaredLogger, args ...interface{}) {
       l.Error(args)
   }

   logErrorBad(sugar, "%s,%s,%s", "10.0.0.1", "GET", "broken pipe")
   ```

   ```text
   {"level":"error","msg":"[%s,%s,%s 10.0.0.1 GET broken pipe]"}
   {"level":"error","msg":"client_ip=10.0.0.1 method=GET error=broken pipe"}
   {"level":"error","msg":"access error{client_ip 15 0 10.0.0.1 <nil>} {method 15 0 GET <nil>}"}
   {"level":"error","msg":"access error","client_ip":"10.0.0.1","method":"GET"}
   ```

   第一条是这种直传写法的输出：格式串原样保留，参数被方括号括起来。第二条是 `sugar.Errorf`，能用但不结构化。第三条是把 `zap.Field` 传给 `SugaredLogger`：字段被当成普通参数，打出了结构体字面量。只有第四条（`Logger.Error` + `zap.Field`）才是想要的结果。
5. **死配置**：设置了 `NameKey: "logger"`、`EncodeName: zapcore.FullNameEncoder`，但从未调用 `logger.Named("xxx")`，这两个配置永远不会出现在输出里。
6. **`AddSync` 重复包裹**：`write` 已经是 `zapcore.AddSync(&hook)` 的返回值（`WriteSyncer`），外面再套一层 `zapcore.AddSync(write)` 是无害但多余的。
7. **`Development()` 不等于带堆栈**：手动 `zap.New(core, opts...)` 时，`zap.Development()` 只影响 DPanic 级别的行为，`AddStacktrace` 需要单独加（v1.28.0 源码里 `Development()` 只做 `log.development = true`）。开发模式下期待的堆栈，用下面这段验证：

   ```text
   2026-10-09 10:23:09.492087	[WARN]	demo/main.go:37	慢查询	{"sql": "select * from orders", "cost": "820ms"}
   main.main
   	/private/var/folders/.../zapdemo/cmd/demo/main.go:37
   runtime.main
   	/Users/wangshang/.g/go/src/runtime/proc.go:285
   ```

   这是补上 `zap.AddStacktrace(zap.WarnLevel)` 之后的效果；只有 `zap.Development()` 时这一段不会出现。
8. **`Fatal` 会跳过 defer**：`logger.Fatal` 内部是 `os.Exit(1)`，进程直接结束。真实验证：

   ```text
   2026-10-09 10:23:11.272440	[INFO]	fatal/main.go:21	开始初始化
   2026-10-09 10:23:11.272870	[FATAL]	fatal/main.go:22	数据库连不上，进程退出	{"addr": "127.0.0.1:5432"}
   exit status 1
   ```

   示例里注册了一个打印 `defer: 清理资源并 Sync` 的 defer，一行都没有输出，程序退出码 1，`defer` 里的刷盘也不会执行。

## 从零实现日志包

针对上面这些问题，下面实现一个可直接使用的日志包，目标很明确：显式初始化、级别单一来源、caller 指向业务代码、Console/JSON 可切换、文件自动切割。本文代码位于 `zapdemo/log/log.go`，下面按主题拆开讲解。

### 依赖与配置结构体

```go
package log

import (
	"fmt"
	"os"
	"path/filepath"

	"go.uber.org/zap"
	"go.uber.org/zap/zapcore"
	"gopkg.in/natefinch/lumberjack.v2"
)

// Config 是 Init 的全部可配置项。
type Config struct {
	Dir         string        // 日志目录
	Filename    string        // 日志文件名
	Level       zapcore.Level // 初始级别
	Console     bool          // 是否同时写终端
	JSON        bool          // true 用 JSON 编码，适合采集
	Development bool          // 开发模式：Warn 及以上自动附带堆栈
	MaxSize     int           // 单个文件上限，单位 MB
	MaxBackups  int           // 保留旧文件个数
	MaxAge      int           // 旧文件保留天数
	Compress    bool          // 是否压缩旧文件
}
```

lumberjack 的推荐导入路径是 `gopkg.in/natefinch/lumberjack.v2`（v2.2.1）；`github.com/natefinch/lumberjack` 这个路径也能编译，但它不带 go.mod（对应 v2.0.0），不建议使用。所有参数都从 `Config` 传入，测试时换一个 `Dir` 就行。

### 显式 Init：创建目录、组装 Core

```go
var (
	logger *zap.Logger
	sugar  *zap.SugaredLogger
	level  = zap.NewAtomicLevelAt(zap.InfoLevel)
)

// Init 显式初始化日志组件，必须在打第一条日志之前调用。
func Init(cfg Config) error {
	if err := os.MkdirAll(cfg.Dir, 0o755); err != nil {
		return fmt.Errorf("log: 创建日志目录失败: %w", err)
	}

	// lumberjack 实现了 io.WriteCloser，负责按大小/时间切割日志。
	rotator := &lumberjack.Logger{
		Filename:   filepath.Join(cfg.Dir, cfg.Filename),
		MaxSize:    cfg.MaxSize,
		MaxBackups: cfg.MaxBackups,
		MaxAge:     cfg.MaxAge,
		Compress:   cfg.Compress,
	}

	level.SetLevel(cfg.Level)

	encCfg := encoderConfig(cfg.JSON)
	var enc zapcore.Encoder
	if cfg.JSON {
		enc = zapcore.NewJSONEncoder(encCfg)
	} else {
		enc = zapcore.NewConsoleEncoder(encCfg)
	}

	writers := []zapcore.WriteSyncer{zapcore.AddSync(rotator)}
	if cfg.Console {
		writers = append(writers, zapcore.AddSync(os.Stdout))
	}

	core := zapcore.NewCore(enc, zapcore.NewMultiWriteSyncer(writers...), level)

	opts := []zap.Option{
		zap.AddCaller(),
		zap.AddCallerSkip(1), // 跳过本包的封装函数，让 caller 指向真正的业务代码
	}
	if cfg.Development {
		// zap.Development() 只影响 DPanic 的行为；
		// 手动 New(core) 时想附带堆栈，必须再显式 AddStacktrace。
		opts = append(opts, zap.Development(), zap.AddStacktrace(zap.WarnLevel))
	}
	logger = zap.New(core, opts...)
	sugar = logger.Sugar()
	return nil
}
```

几个关键点：

- `os.MkdirAll` 一次把多级目录建好，错误带回调用方，初始化失败可以立即终止或上报。
- 传给 `NewCore` 的第三个参数是包级 `level`，它正是 `AtomicLevel`，动态改级别能力从这一步就接上了。
- `MultiWriteSyncer` 把 `lumberjack` 和 `os.Stdout` 串起来，一行日志同时进文件和终端；没开 `Console` 时就只写文件。

### Encoder 配置：格式、级别、时间

```go
// encoderConfig 决定每个字段怎么编码。
func encoderConfig(jsonMode bool) zapcore.EncoderConfig {
	cfg := zapcore.EncoderConfig{
		TimeKey:        "time",
		LevelKey:       "level",
		NameKey:        "logger",
		CallerKey:      "caller",
		FunctionKey:    zapcore.OmitKey,
		MessageKey:     "msg",
		StacktraceKey:  "stacktrace",
		LineEnding:     zapcore.DefaultLineEnding,
		EncodeDuration: zapcore.StringDurationEncoder,
		EncodeCaller:   zapcore.ShortCallerEncoder,
		EncodeTime:     zapcore.TimeEncoderOfLayout("2006-01-02 15:04:05.000000"),
		EncodeLevel: func(l zapcore.Level, enc zapcore.PrimitiveArrayEncoder) {
			enc.AppendString(fmt.Sprintf("[%s]", l.CapitalString()))
		},
	}
	if jsonMode {
		// JSON 是给机器看的，level 保持小写裸值更通用。
		cfg.EncodeLevel = zapcore.LowercaseLevelEncoder
	}
	return cfg
}
```

- `EncodeTime`：`2006-01-02 15:04:05.000000` 是 Go 的布局字符串，参考时间是 `2006-01-02 15:04:05`，`.000000` 表示固定输出 6 位小数（微秒）。输出就是 `2026-10-09 10:23:09.383247`。代价是它不是标准时间格式，采集端需要一条 parse 规则；如果只服务 ELK/Loki，可以直接用 `zapcore.RFC3339TimeEncoder`。
- `EncodeLevel`：自定义成 `[INFO]`、`[ERROR]`，终端里一眼能扫到级别；`jsonMode` 下换回小写裸值，机器解析更通用。
- `EncodeCaller: ShortCallerEncoder`：输出 `demo/main.go:34`，只保留最后两级路径，避免长路径把行撑爆。
- `EncodeDuration: StringDurationEncoder`：`zap.Duration("cost", 820*time.Millisecond)` 会输出 `"820ms"`，比用秒数浮点表示更易读。
- `NameKey`/`FunctionKey`：不用 `logger.Named()` 和函数名时直接不配置，避免死配置。

Console 和 JSON 由 `NewConsoleEncoder` / `NewJSONEncoder` 决定。终端调试用 Console，日志采集用 JSON；两者共用同一份 `EncoderConfig`，切换只需要一个 `Config.JSON` 开关。

### caller 行号：AddCallerSkip 解决什么

封装一层后，`zap.AddCaller()` 拿到的调用栈是：业务代码 → `log.Info` → `logger.Info` → `check`。如果不跳过，`caller` 会指向封装函数自己。用一个最小程序对比（`cmd/callerskip`）：

```go
// wrap.go
// wrapNoSkip 的 Logger 没有 AddCallerSkip：
// 报出来的 caller 是下面这行 l.Info，即封装函数自己。
func wrapNoSkip(l *zap.Logger) {
	l.Info("无 skip")
}

// wrapWithSkip 的 Logger 带 AddCallerSkip(1)：
// caller 会跳过本函数，指向调用 wrapWithSkip 的位置。
func wrapWithSkip(l *zap.Logger) {
	l.Info("有 skip")
}
```

```go
// main.go：newLogger(skip) 里按需追加 zap.AddCallerSkip(1)
raw := newLogger(false)
withSkip := newLogger(true)

wrapNoSkip(raw) // caller 指向 wrap.go
wrapWithSkip(withSkip) // caller 指向这行
```

真实输出：

```text
10:31:46.876	INFO	callerskip/wrap.go:8	无 skip
10:31:46.877	INFO	callerskip/main.go:15	有 skip
```

结论：**封装了几层就要跳过几层**。本文的日志包只有一个转发层，所以 `AddCallerSkip(1)`；如果外面再套一层业务封装，就要继续加。另外 zap 的 caller 和 stacktrace 共用同一个 skip，跳过封装层后，行号和堆栈的第一帧都会从业务代码开始，不会混进封装函数。

### 级别：AtomicLevel 是唯一事实来源

```go
// IsDebug 报告当前是否允许输出 Debug 日志。
// 直接读 AtomicLevel，级别的唯一事实来源就是它。
func IsDebug() bool { return level.Level() <= zapcore.DebugLevel }

// SetLevel 运行时切换级别，立即对所有已创建的 Logger 生效。
func SetLevel(l zapcore.Level) { level.SetLevel(l) }

// Sync 把缓冲区刷到磁盘，程序退出前调用。
func Sync() error { return logger.Sync() }
```

有些封装会把调试开关（比如 `config.IsDebug()`）和 core 级别做成两套开关，容易出现"配置说 debug、日志却不出来"的矛盾。这里把级别收敛成一个事实来源：`IsDebug()` 读的就是 `AtomicLevel` 的当前值，调试开关和日志级别永远一致。如果级别来自配置文件，不用手写 `switch`，`zapcore.ParseLevel("debug")` 直接返回 `zapcore.Level` 和错误。

### 封装函数：结构化字段优先

```go
// Debug 输出调试日志；级别高于 Debug 时连字段构造都省掉。
func Debug(msg string, fields ...zap.Field) {
	if IsDebug() {
		logger.Debug(msg, fields...)
	}
}

// Info 输出普通信息日志。
func Info(msg string, fields ...zap.Field) { logger.Info(msg, fields...) }

// Warn 输出警告日志。
func Warn(msg string, fields ...zap.Field) { logger.Warn(msg, fields...) }

// Error 输出错误日志。
func Error(msg string, fields ...zap.Field) { logger.Error(msg, fields...) }

// Fatal 输出日志后调用 os.Exit(1)，之后的 defer 不会执行。
func Fatal(msg string, fields ...zap.Field) { logger.Fatal(msg, fields...) }

// Debugf / Infof / Errorf 保留 Sprintf 风格，给习惯格式化字符串、不方便构造 Field 的场景用。
func Debugf(template string, args ...any) {
	if IsDebug() {
		sugar.Debugf(template, args...)
	}
}
func Infof(template string, args ...any)  { sugar.Infof(template, args...) }
func Errorf(template string, args ...any) { sugar.Errorf(template, args...) }
```

这里的关键设计是签名：`msg string, fields ...zap.Field`。`zap.Field` 是强类型，不会出现把整个 `[]interface{}` 切片当成一个参数传下去的情况。`Fatal` 保留同样的签名，但注释里要写明它会调用 `os.Exit(1)`。热点路径如果字段构造很贵，用 `logger.Check(zap.DebugLevel, "msg")` 拿到 `*zapcore.CheckedEntry`，非 nil 才 `Write` 字段，对应前面表里 3.4ns/0 alloc 的那一行。

## 完整示例与真实输出

示例程序 `cmd/demo/main.go` 覆盖 Debug/Info/Infof/Warn/Error、结构化字段、动态调级：

```go
func main() {
	dev := flag.Bool("dev", false, "开发模式：Warn 及以上自动附带堆栈")
	jsonMode := flag.Bool("json", false, "用 JSON 编码输出")
	flag.Parse()

	if err := log.Init(log.Config{
		Dir:         "logs",
		Filename:    "acc.log",
		Level:       zap.DebugLevel,
		Console:     true,
		JSON:        *jsonMode,
		Development: *dev,
		MaxSize:     10,
		MaxBackups:  30,
		MaxAge:      7,
		Compress:    true,
	}); err != nil {
		panic(err)
	}
	defer func() { _ = log.Sync() }()

	log.Debug("加载配置", zap.String("file", "config.yaml"))
	log.Info("user login", zap.String("user", "ws"), zap.Int("uid", 1001))
	log.Infof("user %s login from %s", "ws", "10.0.0.1")
	log.Warn("慢查询", zap.String("sql", "select * from orders"), zap.Duration("cost", 820*time.Millisecond))
	log.Error("写入失败", zap.Error(errors.New("connection reset by peer")))

	// 动态把级别调到 Warn，之后的 Info 会被直接丢弃。
	log.SetLevel(zap.WarnLevel)
	log.Info("这行不会被输出")
	log.Warn("warn 仍然可见")

	// main 正常返回时，defer 里的 Sync 会执行。
}
```

编译检查用 `go vet ./...`，没有输出即通过。运行 `go run ./cmd/demo`：

```text
2026-10-09 10:23:09.382788	[DEBUG]	demo/main.go:34	加载配置	{"file": "config.yaml"}
2026-10-09 10:23:09.383247	[INFO]	demo/main.go:35	user login	{"user": "ws", "uid": 1001}
2026-10-09 10:23:09.383263	[INFO]	demo/main.go:36	user ws login from 10.0.0.1
2026-10-09 10:23:09.383273	[WARN]	demo/main.go:37	慢查询	{"sql": "select * from orders", "cost": "820ms"}
2026-10-09 10:23:09.383282	[ERROR]	demo/main.go:38	写入失败	{"error": "connection reset by peer"}
2026-10-09 10:23:09.383291	[WARN]	demo/main.go:43	warn 仍然可见
```

第 34 到 43 行都和源码行号一一对应，说明 caller 指向的正是业务代码；`SetLevel(zap.WarnLevel)` 之后，第 42 行的 Info 消失、第 43 行的 Warn 保留，说明 AtomicLevel 工作正常。同一份代码切到 JSON（`go run ./cmd/demo -json`）：

```text
{"level":"debug","time":"2026-10-09 10:23:09.595002","caller":"demo/main.go:34","msg":"加载配置","file":"config.yaml"}
{"level":"info","time":"2026-10-09 10:23:09.595224","caller":"demo/main.go:35","msg":"user login","user":"ws","uid":1001}
{"level":"info","time":"2026-10-09 10:23:09.595234","caller":"demo/main.go:36","msg":"user ws login from 10.0.0.1"}
```

字段顺序和内容与 Console 一致，只是编码方式换成了 JSON，`level` 变成小写裸值。加 `-dev` 时 Warn 及以上会带堆栈（见前文 caller 一节）。文件侧同样有输出，`logs/acc.log` 与终端内容一致，这是 `MultiWriteSyncer` 的效果。

## 磁盘占用怎么估

lumberjack 的切割判断基于**未压缩**字节数：当前文件写满 `MaxSize` 就改名为带时间戳的备份，再开新文件；`MaxBackups` 和 `MaxAge` 决定备份的保留策略；`Compress` 只影响备份文件占用的磁盘，不会改变切割频率。

估算上限的公式：

```text
最坏占用 ≈ MaxSize × (MaxBackups + 1) + 余量
```

`+1` 是正在写的当前文件，单位是 MB。`MaxSize=10`、`MaxBackups=30` 时上限约 310MB，再乘压缩收益。用 1MB 阈值做了个切割实验（测试 payload 是重复文本，压缩率偏高）：

```text
-rw-------@ 1 wangshang  staff    10K Oct  9 10:24 rotate-2026-10-09T02-24-52.090.log.gz
-rw-------@ 1 wangshang  staff    50K Oct  9 10:24 rotate.log
```

写进去约 1.1MB，当前文件留下 50KB，切走的 1MB 压缩成 10KB 的 `.gz`。备份文件名的格式是 `文件名-时间戳.log.gz`，时间戳默认用 UTC（v2.2.x 可开 `LocalTime: true`）。真实业务文本的压缩比一般在 5:1 到 15:1，重复度越高收益越大；预算时可以按不压缩的 310MB 留足，再按实际压缩比回收。

还有一个只在"短命进程"上出现的坑：切割后的压缩在 lumberjack 的后台 goroutine 里做，**进程如果立刻退出，压缩会被打断，留下一个空壳 `.gz` 和一个未压缩的备份**。长驻服务不受影响；CLI 工具在最后一次写日志后要留一个短暂的等待（我们的测试程序里加了 `time.Sleep(time.Second)`）。

## 踩坑与边界

- **Fatal 不执行 defer**：`log.Fatal` 底层是 `os.Exit(1)`，注册在它后面的刷盘、关连接都不会跑。要么先显式清理再调用，要么把 Fatal 换成 Error + 自己控制退出。
- **Sync 的错误在部分平台无意义**：stdout 不是普通文件时 `os.Stdout.Sync()` 会报错（macOS 上实测是 `sync /dev/stdout: bad file descriptor`），所以调用处统一 `_ = log.Sync()`，别把它的返回值当成关键错误处理。
- **多进程写同一个文件不安全**：lumberjack 的切割只在单个进程内加锁，两个进程同时写同一个文件会互相覆盖、切割错乱。要么每进程一个文件名（带 PID），要么交给集中式日志系统。
- **相对路径取决于工作目录**：`Dir: "logs"` 是相对进程启动目录的。用 systemd、supervisor 部署时要确认 `WorkingDirectory`，或者直接传绝对路径。
- **热点路径字段构造要算账**：`zap.Field` 在调用前构造，禁用的 Debug 仍会分配字段切片。高频调用点用 `IsDebug()` 包一层，或者用 `Logger.Check`。
- **`logger.Named` / 函数名不是默认值**：不开 `Named()` 就不要配 `NameKey`；`FunctionKey` 默认是 `OmitKey`，显式写出来反而容易让人以为会输出函数名。

## 总结

- zap 的架构是 **Logger → Core（LevelEnabler + Encoder + WriteSyncer）**，配置文件切割、输出目标、格式时，先想清楚在改哪个部件。
- 级别只保留一个事实来源：`AtomicLevel`。`IsDebug()` 读它，`SetLevel` 改它，`NewCore` 用它。
- 封装一层日志就要 `AddCallerSkip(1)`；zap 的 caller 和 stacktrace 共用 skip，行号和堆栈都会指向业务代码。
- 手动 `zap.New(core)` 时 `Development()` 不带堆栈，必须额外 `AddStacktrace`。
- 字段走 `zap.Field`，别把格式化字符串传进 `args ...interface{}`；`SugaredLogger` 只建议用在参数不固定、或从 fmt 风格平滑迁移的场景。
- 磁盘预算按 `MaxSize × (MaxBackups + 1)` 估算，`Compress` 只省空间不改变切割频率；短命进程要给后台压缩留时间。
