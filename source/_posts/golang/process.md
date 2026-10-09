---
title: Go 管理外部进程：启动、采集输出与优雅退出
date: 2023-01-02 23:22:30
updated: 2026-10-09
author: ws
description: 用 os/exec 封装一个能真正停止的后台进程管理器，讲透管道、context 与信号
categories: ["Go"]
tags: ["Go", "进程管理"]
cover:
---

用 Go 启动外部程序本来只需要 `os/exec` 几行代码，但要封装成一个“能后台运行、能采集输出、能真正停掉、还能整树杀掉子进程”的管理器，就要同时处理好管道时序、context、信号和错误语义。很多项目里的进程管理代码看起来能跑，实际上 `Stop` 只是发了个通道消息，`Error()` 永远是 nil。

这篇文章先看一种常见的错误写法，用可复现的运行输出证明它的缺陷；再从头实现一个正确的进程管理器：`Start` 异步返回、逐行采集 stdout/stderr、`Stop` 先 SIGTERM 后 SIGKILL、杀整棵进程组，并附上完整的测试与真实输出。适合需要调用 FFmpeg、脚本、常驻 Worker 之类外部程序的读者。

## 先看一种常见的错误写法

这段代码看起来没什么问题（省略了 import）：

```go
type Process struct {
	exit   chan struct{}
	err    error
	stdout func(*bufio.Reader)
	stderr func(*bufio.Reader)
}

// NewProcess 与 WithStdOut/WithStdErr 是普通构造代码，直接看 Run 和 Stop：

func (p *Process) Run(userCmd string) *Process {
	ctx, cancelFunc := context.WithCancel(context.Background())
	defer cancelFunc()

	go func() {
		var (
			err    error
			stdout io.ReadCloser
			stderr io.ReadCloser
		)
		defer func() {
			if err != nil {
				p.exit <- struct{}{} // 先发信号
				p.err = err          // 再写错误
			}
		}()
		cmd := exec.CommandContext(ctx, "/bin/sh", "-c", userCmd)
		if stdout, err = cmd.StdoutPipe(); err != nil {
			return
		} else {
			go p.stdout(bufio.NewReader(stdout))
		}
		if stderr, err = cmd.StderrPipe(); err != nil {
			return
		} else {
			go p.stderr(bufio.NewReader(stderr))
		}
		if err = cmd.Start(); err != nil {
			return
		}
		if err = cmd.Wait(); err != nil {
			return
		}
		p.exit <- struct{}{}
	}()

	<-p.exit // 卡在这里等 goroutine
	return p
}

func (p *Process) Stop() {
	p.exit <- struct{}{}
}

func (p *Process) Error() error {
	return p.err
}
```

把这段代码放进一个专门的 `naive` 包，逐条跑出问题。

### 问题一：Run 是同步的

`Run` 最后停在 `<-p.exit` 上，一直等到进程结束才返回，所谓“后台运行”根本不存在。跑一条要 2 秒的命令：

```text
$ go run ./probe synchronous
Run("sleep 2") 阻塞了 2s
```

真实的服务里，这意味着 `Run` 会占住调用它的 goroutine（通常是 HTTP handler 或启动流程），进程跑多久就卡多久。

### 问题二：没设置回调就 panic

`p.stdout(bufio.NewReader(stdout))` 在 `p.stdout` 为 nil 时，等于调用一个 nil 函数值，采集 goroutine 直接崩溃并带走整个进程。注意 panic 发生在**子 goroutine 里**，recover 根本来不及。下面的探针只设置了 stderr 回调，故意漏掉 stdout：

```text
$ go run ./probe nilstdout
panic: runtime error: invalid memory address or nil pointer dereference
[signal SIGSEGV: segmentation violation code=0x2 addr=0x0 pc=...]
goroutine 20 [running]:
example.com/goprocess/naive.(*Process).Run.func1.gowrap1()
	.../naive/naive.go:57 +0x24
created by example.com/goprocess/naive.(*Process).Run.func1 in goroutine 19
	.../naive/naive.go:57 +0x238
exit status 2
```

### 问题三：Stop 在进程结束后死锁

`exit` 是无缓冲通道。进程自然结束时，goroutine 已经发过一次信号、没人接收第二次；此时再调 `Stop()`，它会永久阻塞在 `p.exit <- struct{}{}`：

```text
$ go run ./probe stophang
进程已自然退出，Run 返回
Stop 阻塞超过 1 秒：exit 通道无人接收，死锁
```

在 Web 服务里，这是典型的“进程已经退出，但清理逻辑卡死”的事故来源。

### 问题四：Stop 的“能停”是副作用，而且没有优雅退出

一个正在运行的长命令，`Stop()` 之后确实看不到进程了——但杀它的不是 `Stop`。`Stop` 只是让卡在 `<-p.exit` 的 `Run` 收到信号并返回，`Run` 返回时执行 `defer cancelFunc()`，`CommandContext` 的内部 goroutine 看到 context 被取消，调用默认的 `Cancel`（即 `Process.Kill()`）发送 **SIGKILL**。实测：

```text
$ go run ./probe stopkill
子进程 PID = 72882
Stop 之后 p.Error() = <nil>
Stop 之后进程已退出: no such process
goroutine 数：停止前 1，停止后 2
```

这段输出暴露了三件事：

1. 进程是被“Run 退出”这个副作用顺手 SIGKILL 掉的。如果以后有人去掉那个 `defer cancelFunc()`，`Stop` 就只剩发通道消息，什么都不会发生。
2. 用的是 SIGKILL，进程没有任何机会 flush 数据、关闭连接、写日志——不是“停止”，是“击毙”。
3. goroutine 从 1 涨到 2 且不再降回来：`cmd.Wait()` 返回错误后，defer 里先执行 `p.exit <- struct{}{}`，没有接收者，goroutine 永久阻塞，**后面的 `p.err = err` 永远执行不到**，所以 `Error()` 返回 nil。错误不仅没返回给调用方，连存都没存上。

### 问题五：context 只是摆设

context 唯一的用途是在 `Run` 返回时兜底 kill，调用方拿不到 `cancel`，没法做超时、没法做“收到退出信号后停止”，`CommandContext` 提供的能力基本被浪费了。

上面的问题可以总结成一句话：**启动时序错（Run 同步）、回调可能为 nil、停止依赖副作用且无升级路径、错误丢失、没有进程组概念**。下面按正确的时序和分层逐块实现。

## 正确时序：Start → 读管道 → Wait

`os/exec` 的 `Cmd` 有三种收尾顺序，只有一种是安全的：

| 顺序 | 结果 |
| --- | --- |
| `Start` → `Wait` → 读管道 | 输出可能已丢，`Wait` 会关掉管道读端 |
| `Start` → 读管道与 `Wait` 并发 | 依赖实现细节，尾部输出有概率丢 |
| `Start` → 读管道到 EOF → `Wait` | 正确：输出读完再回收进程 |

`Wait` 的文档写得很直接：“`Wait` 在观察到命令退出后会关闭管道，因此调用者通常不需要自己关闭；但在所有管道读取完成之前调用 `Wait` 是错误的行为。”原因在源码里：`Wait` 收尾时会 `closeDescriptors(c.parentIOPipes)`，如果读取协程还没读完，管道缓冲区里剩下的输出就一起被丢掉了。

所以管理器内部按下面的时序工作：

```text
Start()
  │  cmd.Start()
  ├─ go 读 stdout ─┐
  ├─ go 读 stderr ─┴─ wg.Wait()
  ├─ go { wg.Wait(); cmd.Wait(); close(done) }
  └─ 立即返回
EOF 从哪来：进程退出 → 内核关闭子进程的描述符 → 读端读到 EOF
```

三个细节：

- 子进程退出后，管道写端（在子进程一侧）被内核关闭，读取协程读到 EOF 然后结束，`wg.Wait()` 才能返回。**反过来说，读端必须一直读**：调用方不关心输出时也要 `io.Copy(io.Discard, r)` 把管道读空，否则管道缓冲区写满，子进程会阻塞在 write 上，永远不退出。
- 每行输出用 `bufio.Scanner` 读，但 Scanner 默认单行上限是 64KB，超长行会导致 `Scan()` 返回 false。这里把上限提到 10MB，避免日志里偶尔一条超长记录就把采集悄悄截断。
- `done` 通道在 `cmd.Wait()` 返回之后关闭，`Wait()`/`Stop()` 都通过它等待收尾完成，避免第二个问题里的“重复发信号”。

## 数据结构与 Start

先把 `Process` 的结构定下来。回调、超时在 `Start` 之前设置好，运行期的字段全部用互斥锁保护：

```go
// Process 管理一个外部命令的生命周期：启动、采集输出、等待、停止。
type Process struct {
	name    string
	args    []string
	timeout time.Duration

	stdoutFn func(line string)
	stderrFn func(line string)

	mu      sync.Mutex
	cmd     *exec.Cmd
	cancel  context.CancelFunc
	done    chan struct{}
	waitErr error
}
```

`New` 和一组 builder 方法负责配置：

```go
// New 创建进程管理器；timeout 是停止时的优雅退出等待时间，默认 5 秒。
func New(name string, args ...string) *Process {
	return &Process{name: name, args: args, timeout: 5 * time.Second}
}

func (p *Process) WithStdout(fn func(line string)) *Process { p.stdoutFn = fn; return p }
func (p *Process) WithStderr(fn func(line string)) *Process { p.stderrFn = fn; return p }
func (p *Process) WithStopTimeout(d time.Duration) *Process { p.timeout = d; return p }
```

`Start` 是核心。注意它只负责“把进程拉起来、把读取协程挂上去”，收尾全部交给后台 goroutine：

```go
// Start 启动进程并立即返回，输出采集在后台进行。
func (p *Process) Start() error {
	p.mu.Lock()
	defer p.mu.Unlock()
	if p.cmd != nil {
		return errors.New("process: already started")
	}

	ctx, cancel := context.WithCancel(context.Background())
	cmd := exec.CommandContext(ctx, p.name, p.args...)
	// ctx 取消时先体面退出：向整个进程组发 SIGTERM。
	cmd.Cancel = func() error { return terminateGroup(cmd) }
	// 超过 timeout 仍未退出，exec 会兜底 Kill 主进程。
	cmd.WaitDelay = p.timeout
	// 让子进程自成进程组，停止时可以一次杀掉整棵进程树。
	configureProcAttr(cmd)

	stdout, err := cmd.StdoutPipe()
	if err != nil {
		cancel()
		return err
	}
	stderr, err := cmd.StderrPipe()
	if err != nil {
		cancel()
		return err
	}
	if err := cmd.Start(); err != nil {
		cancel()
		return err
	}

	done := make(chan struct{})
	p.cmd, p.cancel, p.done = cmd, cancel, done

	var wg sync.WaitGroup
	wg.Add(2)
	go func() { defer wg.Done(); scanLines(stdout, p.stdoutFn) }()
	go func() { defer wg.Done(); scanLines(stderr, p.stderrFn) }()

	// 正确时序：先等读取协程读到 EOF（进程退出后管道自动关闭），
	// 再调用 cmd.Wait 回收进程资源。反序会提前关掉管道读端。
	go func() {
		wg.Wait()
		err := cmd.Wait()
		p.mu.Lock()
		p.waitErr = err
		p.mu.Unlock()
		close(done)
	}()
	return nil
}
```

两个读取协程共用同一个 `scanLines`，回调为 nil 时也要把管道读空：

```go
// scanLines 逐行读输出；fn 为 nil 时仍然读完，否则管道写满会卡住子进程。
func scanLines(r io.Reader, fn func(string)) {
	if fn == nil {
		_, _ = io.Copy(io.Discard, r)
		return
	}
	sc := bufio.NewScanner(r)
	sc.Buffer(make([]byte, 0, 64*1024), 10*1024*1024)
	for sc.Scan() {
		fn(sc.Text())
	}
	// 忽略 scanner.Err()：进程被停止时管道关闭导致的读错误不是主要错误，
	// cmd.Wait 返回的才是退出原因。
}
```

`Start` 在 5 秒任务上只花不到 2 毫秒就返回，异步性用测试固定下来：

```text
=== RUN   TestStartReturnsImmediately
    process_test.go:54: Start 耗时: 1.616917ms（命令本身要跑 5 秒）
```

## Wait 与错误语义

`Wait` 阻塞到 `done` 关闭，然后把真正的退出错误返回给调用方：

```go
// Wait 阻塞直到进程退出且输出读取完毕，返回退出错误。
func (p *Process) Wait() error {
	p.mu.Lock()
	done := p.done
	p.mu.Unlock()
	if done == nil {
		return errors.New("process: not started")
	}
	<-done
	p.mu.Lock()
	defer p.mu.Unlock()
	return p.waitErr
}
```

这里要接受一个不那么直观的事实：**被 Stop 掉的进程，`Wait` 返回的不是 `context.Canceled`**。`exec` 源码里，进程被信号杀死时会生成 `*exec.ExitError`（`state.Success()` 为 false），它的优先级高于 context 的错误；只有进程捕获信号后以 0 退出，调用方才会看到 `context.Canceled`。实测：

```text
=== RUN   TestCaptureOutputAndStop
    第一行输出: "tick 0"
    SIGTERM 后 1.041459ms 内退出
    Wait 返回: signal: terminated
    ExitCode=-1（-1 表示被信号终止）
```

所以调用方判断“业务正常结束”用 `err == nil`，判断“被我停掉的”应该看 `errors.As(err, &exitErr) && exitErr.ExitCode() == -1`，不要假设 `errors.Is(err, context.Canceled)` 一定成立。自动退出的进程如果退出码非 0，同样通过 `*exec.ExitError` 取退出码：

```text
=== RUN   TestExitErrorAndStderr
    退出码: 7
    stderr: [boom]
```

## Stop：SIGTERM、WaitDelay 与进程组

Stop 的策略只有一条：**先礼后兵，而且礼和兵都给整个进程组**。

- `cancel()` 触发 `cmd.Cancel`，向进程组发 SIGTERM；
- 进程若在 `p.timeout` 内退出，`done` 关闭，Stop 返回；
- 超过 `timeout` 还有活口，`syscall.Kill(-pid, SIGKILL)` 把整组强杀。

Go 1.20 起 `exec.Cmd` 配套提供了 `Cancel` 和 `WaitDelay` 两个字段：`CommandContext` 默认的取消行为是直接 `Kill`，把它改成“发 SIGTERM”就能获得优雅退出窗口；`WaitDelay` 则限制 context 取消或进程退出后等待 I/O 收尾的时间，超时后 exec 会关闭管道并让 `Wait` 返回 `ErrWaitDelay`。它们和 Stop 的关系是：context 取消负责**第一刀**，`WaitDelay` 是 exec 内部的**保险**（超时后 Kill 主进程），Stop 里的进程组强杀是**最后一道保险**，专门处理“主进程死了但子进程还抱着管道不放”的情况。

```go
// Stop 停止进程：先发 SIGTERM 给整个进程组，等 timeout 后 SIGKILL。
// 进程已经退出时返回 nil。
func (p *Process) Stop() error {
	p.mu.Lock()
	cmd, cancel, done := p.cmd, p.cancel, p.done
	p.mu.Unlock()
	if cmd == nil {
		return errors.New("process: not started")
	}
	select {
	case <-done:
		return nil // 已经退出
	default:
	}
	cancel() // 触发 cmd.Cancel：SIGTERM 给进程组
	select {
	case <-done:
		return nil
	case <-time.After(p.timeout + time.Second):
	}

	// 进程（或它留下的子进程）无视了 SIGTERM，强杀整组。
	if err := killGroup(cmd); err != nil {
		return fmt.Errorf("process: force kill: %w", err)
	}
	select {
	case <-done:
		return nil
	case <-time.After(3 * time.Second):
		return errors.New("process: still running after SIGKILL")
	}
}
```

平台相关部分放在 `proc_unix.go`。`Setpgid` 让子进程成为新进程组的组长，之后用 `-pid` 作为目标就能把信号发给整组；`ESRCH` 表示进程已经不在了，此时返回 `os.ErrProcessDone`，免得 exec 把“取消一个已退出的进程”当成错误记录下来：

```go
//go:build !windows

// configureProcAttr 让子进程自成进程组，便于整组信号。
func configureProcAttr(cmd *exec.Cmd) {
	cmd.SysProcAttr = &syscall.SysProcAttr{Setpgid: true}
}

// terminateGroup 向整个进程组发 SIGTERM。
func terminateGroup(cmd *exec.Cmd) error {
	if cmd.Process == nil {
		return os.ErrProcessDone
	}
	err := syscall.Kill(-cmd.Process.Pid, syscall.SIGTERM)
	if errors.Is(err, syscall.ESRCH) {
		return os.ErrProcessDone
	}
	return err
}

// killGroup 向整个进程组发 SIGKILL；进程早已退出视为成功。
func killGroup(cmd *exec.Cmd) error {
	err := syscall.Kill(-cmd.Process.Pid, syscall.SIGKILL)
	if errors.Is(err, syscall.ESRCH) {
		return nil
	}
	return err
}
```

**Windows 差异**要单独处理：Windows 没有 POSIX 信号和进程组，`SysProcAttr` 用 `CreationFlags: syscall.CREATE_NEW_PROCESS_GROUP` 建新控制台进程组，但“优雅停止”需要 `GenerateConsoleCtrlEvent` 发 CTRL_BREAK，很多项目直接退化为 `taskkill /T /F /PID`。写跨平台进程管理器时，把“发停止信号”抽象成平台函数，不要在业务代码里散落 `runtime.GOOS` 分支。

## 端到端演示与真实输出

用一个每秒输出一行的脚本验证“采集 + 停止”全流程（`newRecorder` 是把行追加到切片并转发到通道的测试助手）：

```go
rec := newRecorder()
p := New("sh", "-c", `i=0; while true; do echo "tick $i"; i=$((i+1)); sleep 1; done`).
	WithStdout(func(line string) { rec.add(line) }).
	WithStopTimeout(2 * time.Second)
if err := p.Start(); err != nil {
	t.Fatal(err)
}
<-rec.ch                            // 等到第一行输出
time.Sleep(2500 * time.Millisecond) // 攒出更多行
if err := p.Stop(); err != nil {
	t.Fatal(err)
}
err := p.Wait()
```

真实输出：

```text
=== RUN   TestCaptureOutputAndStop
    SIGTERM 后 1.041459ms 内退出
    Wait 返回: signal: terminated
    ExitCode=-1（-1 表示被信号终止）
    共采集 3 行:
      tick 0
      tick 1
      tick 2
```

整组杀进程树的验证：启动 `sh -c 'sleep 300 & echo "child $!"; wait'`，脚本会打印后台 sleep 的 PID。Stop 之后用 `kill(pid, 0)` 探测，子进程已经不在了：

```text
=== RUN   TestStopKillsProcessGroup
    子进程信息: "child 73544"
    停止后探测子进程 73544: no such process
```

最顽固的情况：父进程和子进程都 `trap "" TERM` 无视 SIGTERM，并且子进程抱着 stdout。此时 SIGTERM 无效，`WaitDelay` 先杀掉主进程，Stop 在 `timeout+1s` 处强杀整个进程组：

```text
=== RUN   TestStopEscalatesToKill
    顽固子进程: 73552
    Stop 耗时: 2.001459833s（timeout=1s）
    强杀后的 Wait 返回: signal: killed
    强杀后探测子进程 73552: no such process
```

如果不加处理、只等管道 EOF，这个测试会一直挂住——子进程不退，管道不关，读取协程读不到 EOF，管理器永远不认为进程结束。

## 测试与 vet

完整跑一遍：

```bash
go vet ./...
go test -v ./...
```

除了上面的用例，还有两个容易被忽略的场景被测试覆盖：不设回调时 2 万行输出 180ms 内跑完（证明管道被读空，没有写阻塞），以及启动不存在的可执行文件时 `Start` 返回 `executable file not found`、重复 `Start` 返回错误：

```text
=== RUN   TestNilCallbacksStillDrain
    2 万行输出、无回调，179.7765ms 内跑完
=== RUN   TestStartErrorsAndDoubleStart
    启动不存在的程序: exec: "definitely-not-a-real-binary-xyz": executable file not found in $PATH
```

`go test -race ./...` 也通过，`Process` 的字段没有数据竞争。

## 踩坑与边界

最后把容易踩的边界集中列一下：

1. **管道不读就会卡死**。`StdoutPipe` 拿到的管道缓冲区（Linux 通常 64KB）写满后子进程阻塞在 write，`Wait` 永远等不到退出。任何情况下都要读空，不关心内容就 `io.Copy(io.Discard, r)`。
2. **回调为 nil 必须防御**。调用 nil 函数值会直接 panic，而 panic 发生在采集 goroutine 里，会带走整个进程。
3. **先 Wait 后读是错的**。`Wait` 会关闭父进程侧的管道读端，进程退出前残留在缓冲区的输出会丢；反过来，读取必须一直进行到 EOF 才能调用 `Wait`。
4. **子进程可能留下孙进程**。`sh -c 'daemon &'` 这种写法，shell 立刻退出，但后台进程继承着 stdout，读取协程迟迟等不到 EOF。这就是必须 `Setpgid` + 整组信号的原因；对 `setsid` 逃出进程组的守护进程，只能靠它自己的 PID 文件管理。
5. **停止后的错误是正常错误**。SIGTERM 结束的进程返回 `*exec.ExitError`、`ExitCode() == -1`；把它当成崩溃上报会制造假警报，建议在调用 `Stop` 的地方约定好这个语义。
6. **取消和已退出存在竞态**。进程刚好在 `cancel()` 之后、发信号之前退出时，`kill` 会返回 `ESRCH`；把 `ESRCH` 映射成 `os.ErrProcessDone`，避免 exec 记一条无意义的“canceling Cmd”错误。
7. **Windows 没有 SIGTERM**。`Kill` 是 TerminateProcess（不给清理机会），想优雅退出要基于控制台事件或 `taskkill`，跨平台代码应把信号逻辑隔离到平台文件。

## 总结

- `Start` 只负责启动，收尾放到后台，让“异步”名副其实；
- 输出采集跟着进程的生命周期走：读空管道、处理 nil 回调、检查 Scanner 上限；
- `Stop` 有明确的升级阶梯：SIGTERM 整组 → `WaitDelay` 兜底 → SIGKILL 整组，每一步都有超时；
- 错误语义透明：`Wait` 返回真实退出状态，`Stop` 的杀死与自然退出可区分，回调与字段访问都受锁保护。

这套骨架不到 200 行，可以按需替换成 JSON 行输出解析、环形缓冲日志之类的采集逻辑，但时序和信号处理不要改——那是一类“看起来能跑、出事找不到原因”的代码。
