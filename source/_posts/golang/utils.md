---
title: Go 工具函数的正确写法：文件、哈希与命令执行避坑
date: 2023-01-03 20:13:30
updated: 2026-10-09
author: ws
description: 逐个审查常用工具函数，修掉权限、截断、注入等隐藏问题
categories: ["Go"]
tags: ["Go", "工具函数"]
cover:
---

`utils` 目录几乎是每个 Go 项目里最早堆积代码的地方：哈希、判断文件存在、创建目录、执行命令、读写文件，看着都是“一行就能写对”的小函数。但这些函数里藏着不少基础设施级的坑——命令注入、覆盖写入残留旧内容、Scanner 默默丢掉超长行、权限错误被当成“文件不存在”。

这篇文章挑出工具库中最常见的八类问题，每类都配一段可复现的最小代码，用真实的测试输出说明问题出在哪，再给出推荐的实现。适合已经会写 Go、想把工具函数写得更稳的读者。读完你可以对照检查自己的项目，也能明白这些坑为什么危险。

## 问题清单

先把这批函数里最值得警惕的缺陷列出来，后文逐条展开：

| 函数 | 问题 | 后果 |
| --- | --- | --- |
| `MD5` | 拿 MD5 当通用摘要，容易顺手用在口令上 | 口令可被离线暴力破解 |
| `IsPathExist` | `err == nil` 就算存在 | 权限不足时误报“不存在” |
| `CreatDir` | `ModePerm`/`Chmod(0777)` 无视 umask | 目录对所有用户可写 |
| `SuitableDisplaySize` | 边界用 `>`、右移截断 | `1024` 显示成 `1024B`，`1536` 显示成 `1KB` |
| `Cmd` | 整串命令交给 shell / powershell | 参数里的用户输入变成命令注入 |
| `WriteInfoToFile` | 缺少 `O_TRUNC` | 覆盖写短内容时残留旧内容 |
| `ReadFileToSlice` | 默认 Scanner 缓冲且不检查 `Err()` | 超过 64KB 的行被静默丢弃 |
| `NotExistToMkdir` | 吞掉 `CreatDir` 的错误 | 创建失败无人知晓 |

## MD5 与 CRC32：先分清用途

不假思索的写法往往是这样两个函数：

```go
func MD5(v string) string {
	d := []byte(v)
	m := md5.New()
	m.Write(d)
	return hex.EncodeToString(m.Sum(nil))
}

func HashCode(s string) uint32 {
	return crc32.ChecksumIEEE([]byte(s))
}
```

代码本身没有 bug，问题在用途。MD5 已经不适合任何安全场景：口令、签名、防篡改都不行。口令应该用 bcrypt、scrypt 或 argon2id 这类自带盐和慢哈希的函数。MD5 只适合“没有对手”的场景，比如缓存键、内容去重；即便如此，新代码也可以优先选 SHA-256，统一心智负担。

推荐的实现从命名上就把用途讲清楚：字符串摘要叫 `MD5String`，文件校验用流式的 `FileMD5`，内存占用固定几十 KB，1GB 的文件也能算：

```go
// MD5String 只用于缓存键、去重等非安全场景。
func MD5String(v string) string {
	sum := md5.Sum([]byte(v))
	return hex.EncodeToString(sum[:])
}

// FileMD5 流式计算文件摘要，内存占用恒定。
func FileMD5(path string) (string, error) {
	f, err := os.Open(path)
	if err != nil {
		return "", err
	}
	defer f.Close()

	h := md5.New()
	if _, err := io.Copy(h, f); err != nil {
		return "", err
	}
	return hex.EncodeToString(h.Sum(nil)), nil
}

// HashPassword 生成 bcrypt 摘要，cost 用默认值（当前为 10）。
func HashPassword(plain string) (string, error) {
	b, err := bcrypt.GenerateFromPassword([]byte(plain), bcrypt.DefaultCost)
	return string(b), err
}

// VerifyPassword 校验口令；错误一律返回 false，不区分原因。
func VerifyPassword(hash, plain string) bool {
	return bcrypt.CompareHashAndPassword([]byte(hash), []byte(plain)) == nil
}
```

CRC32 是校验和，不是加密哈希：它能快速发现传输中的随机错误，但挡不住故意构造的碰撞。合理用途是哈希分片、哈希表索引这类内部场景，例如 `HashCode(key)%16` 选一个缓存分片。**不要**用它判断“两段内容是否相同”——攻击者可以轻易构造相同 CRC32 的输入。

跑一下测试，bcrypt 的摘要里不会包含明文，错误口令也过不了：

```text
=== RUN   TestBcrypt
    utils_test.go:38: bcrypt hash = $2a$10$JUH6qA/I1d26CAZ9PAIaTuqfiH3fWntA6XwIl/Iox59OkSL6Nb2yG
--- PASS: TestBcrypt (0.22s)
=== RUN   TestFileMD5
    utils_test.go:24: FileMD5 = 5eb63bbbe01eeed093cb22bb8f5acdc3
```

## IsPathExist：权限错误不是“不存在”

这个判断最直觉的写法只有一行：

```go
func IsPathExist(filePath string) bool {
	_, err := os.Stat(filePath)
	return err == nil
}
```

`os.Stat` 返回的错误并不只有“不存在”一种。如果文件在某个没有执行权限的目录里，Linux/macOS 会返回 `EACCES`，这个函数照样返回 `false`——调用方会误以为文件不存在，从而执行创建逻辑，或报出误导性的“文件未找到”。

用等价代码复现，目录 `chmod 000` 之后去 stat 里面的文件：

```text
=== RUN   TestNaivePathCheckMisjudgesPermission
    os.Stat 的真实错误: stat .../secret.txt: permission denied
    朴素写法 IsPathExist(".../secret.txt") = false（文件其实存在，只是目录没权限）
```

更稳的写法是区分三种结果：存在、确定不存在、其它错误。`errors.Is(err, fs.ErrNotExist)` 是现在的标准写法，它比 `os.IsNotExist` 更通用，能识别经过 `fmt.Errorf("...: %w", err)` 包装过的错误：

```go
// PathExists 区分“不存在”和“访问受限/其它错误”。
func PathExists(path string) (bool, error) {
	_, err := os.Stat(path)
	switch {
	case err == nil:
		return true, nil
	case errors.Is(err, fs.ErrNotExist):
		return false, nil
	default:
		return false, err
	}
}
```

签名从 `bool` 变成 `(bool, error)` 是关键：调用方现在无法忽略“第三种情况”。测试里 `chmod 000` 的目录返回了明确的 `permission denied`，而不是伪装成 `false, nil`。

## CreatDir：权限位不是越高越好

先看一种常见的目录创建写法：`MkdirAll(dirPath, os.ModePerm)` 之后再补一个 `Chmod(dirPath, 0777)`。这里有两层问题：

1. `MkdirAll` 的权限参数只是“上限”，实际权限会被进程的 umask 削减。在 `umask=022` 的机器上，`0777` 实际得到 `0755`；但换一台 `umask=000` 的机器或容器，就会得到真正的 `0777`——行为随环境漂移。
2. `Chmod(0777)` 直接绕过 umask，把目录改成**所有用户可写**，等于对自己屏蔽了这台机器上的权限模型。

实测同一台机器上两种写法：

```text
=== RUN   TestEnsureDirUmask
    EnsureDir(0777) 在 umask=022 下实际权限: -rwxr-xr-x
    EnsureDir(0755) 在 umask=022 下实际权限: -rwxr-xr-x
```

在 `umask=022` 下看不出区别，但 `0755` 的语义是明确的：这是我的上限，umask 只能让它更严；而 `0777` 在宽松的环境里会直接暴露。除非程序确实要让所有用户共享写入（例如 `/tmp` 下的交换目录），否则用 `0755`。

推荐的做法只保留一次 `MkdirAll`，权限由调用方显式传入：

```go
// EnsureDir 创建目录树。perm 只是“上限”，实际权限还会被 umask 削掉。
func EnsureDir(dir string, perm fs.FileMode) error {
	return os.MkdirAll(dir, perm)
}
```

不少工具库还会提供一个 `NotExistToMkdir`：先判断存在再创建，并且吞掉了 `CreatDir` 的错误。`MkdirAll` 本身是幂等的：目录已存在时直接返回 nil，不需要先判断；真要封装，应该把 error 返回出去，而不是 `func NotExistToMkdir(dirPath string)` 这样静默失败。

## SuitableDisplaySize：边界、截断与进制

再来看一个典型的格式化函数，用右移做除法：

```go
func SuitableDisplaySize(size int64) string {
	if size > (1 << 30) {
		return strconv.FormatInt(size>>30, 10) + "GB"
	} else if size > (1 << 20) {
		return strconv.FormatInt(size>>20, 10) + "MB"
	} else if size > (1 << 10) {
		return strconv.FormatInt(size>>10, 10) + "KB"
	} else {
		return strconv.FormatInt(size, 10) + "B"
	}
}
```

三个问题都能用测试固定下来：

```text
naiveSuitableDisplaySize(1024) = 1024B          // 边界用 >，恰好 1KB 掉回 B
naiveSuitableDisplaySize(1048576) = 1024KB      // 恰好 1MB 显示成 1024KB
naiveSuitableDisplaySize(1536) = 1KB            // 右移截断，1.5KB 变 1KB
naiveSuitableDisplaySize(1945600) = 1MB         // 1.86MB 变 1MB
naiveSuitableDisplaySize(1073741823) = 1023MB
naiveSuitableDisplaySize(1073741824) = 1024MB   // 1GB 显示成 1024MB
```

边界应该用 `>=`；右移等价于向下取整，想要可读性就得用浮点和格式化。推荐的实现按 1024 进制、保留两位小数，并顺手处理了单位选择：

```go
// HumanSize 按 1024 进制输出人类可读大小，保留两位小数。
func HumanSize(size int64) string {
	const unit = 1024
	if size < unit {
		return strconv.FormatInt(size, 10) + "B"
	}
	div, exp := int64(unit), 0
	for n := size / unit; n >= unit; n /= unit {
		div *= unit
		exp++
	}
	return fmt.Sprintf("%.2f%c", float64(size)/float64(div), "KMGTPE"[exp])
}
```

关键输出：

```text
HumanSize(1024) = 1.00K
HumanSize(1536) = 1.50K
HumanSize(1945600) = 1.86M
HumanSize(500000000000) = 465.66G
```

最后一行值得单独提醒：**1024 进制和厂商标注的 1000 进制不是一回事**。一块标称 500GB 的硬盘有 500,000,000,000 字节，按 1024 进制算出来是 465.66G。界面写“465.66GB”没有错，但用户刚买回来可能会觉得“少了 35GB”。严格的做法是用 `GiB` 标注 1024 进制，或者按产品需求统一用 1000 进制。

注意第一个判断是 `size < unit`，等价于 `size >= 1024` 就进入 K 分支，边界问题自然消失。

## Cmd：命令注入与错误详情

先看一段常见的命令执行封装：根据系统选择 `/bin/sh -c` 或 `powershell /c`，把一整串命令交给 shell。问题不在平台判断，而在**字符串拼接**：只要命令里混入了用户输入，shell 就会把它当语法执行。

用一个不带恶意的例子复现：

```text
=== RUN   TestNaiveCmdGoesThroughShell
    拼接进 shell 的字符串 "echo hello; id -un" 输出 = "hello\nwangshang"
    拆成参数后输出 = "hello; id -un"
```

`;` 之后的内容被 shell 执行了。真实场景里，输入可能来自 HTTP 参数、文件名、配置项，攻击者只要控制其中一个片段，就能执行任意命令。

更稳妥的方案分两层。默认情况下，能拆分就拆分，用 `exec.Command(name, args...)`，参数不经过任何 shell 解释：

```go
// RunCmd 直接执行可执行文件，参数不经过 shell 解释。
// 返回的输出已经包含 stderr（CombinedOutput）。
func RunCmd(name string, args ...string) (string, error) {
	cmd := exec.Command(name, args...)
	out, err := cmd.CombinedOutput()
	if err != nil {
		return string(out), fmt.Errorf("run %s: %w", name, err)
	}
	return strings.TrimSpace(string(out)), nil
}
```

`CombinedOutput` 同时收 stdout 和 stderr，所以 `ExitError` 的 stderr 已经在返回值里，排障时不会丢掉关键信息；`%w` 保留原始错误，调用方可以用 `errors.As` 取出退出码：

```text
=== RUN   TestRunCmdExitError
    exit code = 7
    返回的 error = run sh: exit status 7
```

确实需要 shell 语义时（管道、重定向、变量展开、通配符），才用 `RunShell`，并且必须满足两个前提：`script` 是代码里写死的、或者所有插值都经过严格校验；同时用 `context` 控制超时，避免挂死的脚本拖垮服务：

```go
// RunShell 只在确实需要管道、重定向等 shell 语义时使用；
// script 必须是可信输入，否则就是命令注入。
func RunShell(ctx context.Context, script string) (string, error) {
	cmd := exec.CommandContext(ctx, "/bin/sh", "-c", script)
	out, err := cmd.CombinedOutput()
	if err != nil {
		return string(out), fmt.Errorf("run shell script: %w", err)
	}
	return strings.TrimSpace(string(out)), nil
}
```

如果确实要把用户输入拼进 shell 脚本，那不是“换个函数”能解决的，需要在业务层做白名单校验。**跨平台**也要注意：`/bin/sh` 在 Windows 上不存在，需要单独适配 `cmd.exe /c` 或 PowerShell，不要在 `runtime.GOOS` 的判断里硬编码一套只有在 CI 上才跑过的分支。

## WriteInfoToFile：被忽略的 O_TRUNC

这种写法打开文件时只传了 `O_CREATE|O_WRONLY`：

```go
file, err := os.OpenFile(filePath, os.O_CREATE|os.O_WRONLY, 0666)
```

这会从文件开头开始写，但**不截断旧内容**。写入内容变短时，旧文件尾巴就会露出来。用两次覆盖写复现：

```text
=== RUN   TestNaiveWriteResidue
    朴素写法两次写入 "hello world" 再写 "hi"，读回: "hillo world"
```

如果这是配置文件或状态文件，读回来就是一段语法都不对的残留内容，而且排查时很难想到是写入函数的锅。推荐实现直接用 `os.WriteFile`，它内部是 `O_WRONLY|O_CREATE|O_TRUNC`：

```go
// WriteFile 覆盖写：os.WriteFile 自带 O_TRUNC，不会残留旧内容。
func WriteFile(path, content string, perm fs.FileMode) error {
	return os.WriteFile(path, []byte(content), perm)
}

// WriteLines 每行一条记录，末尾补一个换行符。
func WriteLines(path string, lines []string, perm fs.FileMode) error {
	var b strings.Builder
	for _, line := range lines {
		b.WriteString(line)
		b.WriteByte('\n')
	}
	return os.WriteFile(path, []byte(b.String()), perm)
}
```

另一种常见写法是 `WriteSliceToFile` 式的实现：用 `strings.Join(contents, "\n")` 拼接，文件末尾没有换行。很多按行处理的工具（包括不少 CLI、编辑器、`git diff` 提示）都默认最后一行有换行符；末尾不补 `\n`，文件会被贴上“No newline at end of file”的标记，逐行 diff 也会变难看。推荐的实现逐行写并统一补换行：

```text
=== RUN   TestWriteLinesTrailingNewline
    文件内容 = "alpha\nbeta\ngamma\n"
```

## ReadFileToSlice：64KB 行上限与静默错误

用 `bufio.Scanner` 逐行读，思路本身没错，但有两个坑：

```go
fileScanner := bufio.NewScanner(f)
for fileScanner.Scan() {
	// ...
}
return fileList, nil // 没有检查 fileScanner.Err()
```

`Scanner` 的默认单行上限是 `bufio.MaxScanTokenSize`，也就是 64KB。超长行会让 `Scan()` 返回 false，错误存在 `scanner.Err()` 里——而这段代码直接返回了 `nil` 错误，调用方看到的是“读到了几行、一切正常”。

复现：造一个 70KB 的单行文件，用上面的逻辑读：

```text
=== RUN   TestNaiveScannerTooLong
    朴素写法读到 1 行，返回错误 = <nil>
    被忽略的 scanner.Err() = bufio.Scanner: token too long
```

第一行正常读出来了，超长行被丢掉，错误被吞掉。日志文件、导出的 CSV、用户上传的 JSON 都容易出现超长行，这类“少了几行但没人知道”的问题非常难查。

推荐实现做三件事：用 `scanner.Buffer` 提高上限、检查 `scanner.Err()` 并返回、把打开文件失败的错误原样传出：

```go
// ReadFileLines 逐行读取，单行上限 1MB，并且把读取错误返回给调用方。
// transform 为 nil 时保留所有行；否则只有返回 true 的行会被保留。
func ReadFileLines(path string, transform func(string) (string, bool)) ([]string, error) {
	f, err := os.Open(path)
	if err != nil {
		return nil, err
	}
	defer f.Close()

	scanner := bufio.NewScanner(f)
	scanner.Buffer(make([]byte, 0, 64*1024), 1024*1024)

	var out []string
	for scanner.Scan() {
		line := scanner.Text()
		if transform == nil {
			out = append(out, line)
			continue
		}
		if v, ok := transform(line); ok {
			out = append(out, v)
		}
	}
	if err := scanner.Err(); err != nil {
		return nil, fmt.Errorf("read %s: %w", path, err)
	}
	return out, nil
}
```

注意 `Buffer` 的第一个参数是初始缓冲，`make([]byte, 0, 64*1024)` 让常见情况不额外分配，第二个参数 1MB 才是硬上限；仍然超限时会返回 `ErrTooLong`，所以**必须**检查 `Err()`。

如果确定文件不大、也不介意一次性读进内存，其实可以不折腾 Scanner：`os.ReadFile` + `strings.Split` + 处理末尾空串，代码更短。但只要文件可能很大，逐行 + 明确上限就是更稳的选择。

```text
=== RUN   TestReadFileLines
    过滤后行数 = 3，第三行长度 = 71680
```

70KB 的超长行被完整读到，同时过滤逻辑照常工作。

## ContainEx：反射版与泛型写法

没有泛型时，一种常见写法是先反射判断容器类型，再对每个元素调用回调：

```go
func ContainEx(a interface{}, f func(predicate interface{}) bool) bool {
	src := reflect.ValueOf(a)
	switch src.Kind() {
	case reflect.Slice, reflect.Array:
		count := src.Len()
		for i := 0; i < count; i++ {
			e := src.Index(i).Interface()
			if f(e) {
				return true
			}
		}
	}
	return false
}
```

额外代价是每次取值都要经过 `Interface()` 装箱，回调还得自己断言类型，写起来费劲、运行时更慢。Go 1.21 起 `slices` 包已经提供了 `slices.ContainsFunc`，自己再包一层泛型函数即可：

```go
// Contains 用回调判断切片中是否存在满足条件的元素；判断“等于某个值”时可直接用 slices.Contains。
func Contains[T any](s []T, pred func(T) bool) bool {
	return slices.ContainsFunc(s, pred)
}
```

调用时类型由编译器推导，回调参数就是元素类型：

```go
nums := []int{1, 2, 3, 4, 5}
Contains(nums, func(n int) bool { return n > 4 }) // true
```

如果判断条件只是“等于某个值”，`slices.Contains(nums, 4)` 连回调都不用写。唯一需要注意的场景是元素数量极少、并且类型在编译期未知——那种情况反射版也帮不上忙，应该重新设计接口。

## 验证方式

文中的推荐实现和错误示范的复现代码都放在一个临时 module 里，直接跑：

```bash
go vet ./...
go test -v ./...
```

本机 Go 1.25.9 下的结果：

```text
--- PASS: TestFileMD5
    FileMD5 = 5eb63bbbe01eeed093cb22bb8f5acdc3
--- PASS: TestBcrypt
    bcrypt hash = $2a$10$JUH6qA/I1d26CAZ9PAIaTuqfiH3fWntA6XwIl/Iox59OkSL6Nb2yG
--- PASS: TestPathExists
    存在的文件: ok=true err=<nil>
    不存在的文件: ok=false err=<nil>
--- PASS: TestPathExistsPermissionError
    目录 chmod 000 后: ok=false err=stat ...: permission denied
--- PASS: TestEnsureDirUmask
    EnsureDir(0777) 在 umask=022 下实际权限: -rwxr-xr-x
    EnsureDir(0755) 在 umask=022 下实际权限: -rwxr-xr-x
--- PASS: TestHumanSize
    HumanSize(1536) = 1.50K
    HumanSize(500000000000) = 465.66G
--- PASS: TestRunCmdKeepsArgsLiteral
    RunCmd(echo, "hello; whoami") = "hello; whoami"
--- PASS: TestRunCmdExitError
    exit code = 7
--- PASS: TestWriteFileTruncates
    两次覆盖写之后 = "hi"
--- PASS: TestWriteLinesTrailingNewline
    文件内容 = "alpha\nbeta\ngamma\n"
--- PASS: TestReadFileLines
    过滤后行数 = 3，第三行长度 = 71680
--- PASS: TestContains
    Contains 在 [1 2 3 4 5] 上工作正常
ok  	example.com/goutils	0.703s
```

另外 `go test -race ./...` 也没有报告数据竞争。

## 总结

这批工具函数里，真正写错的代码不多，危险的是**语义模糊的返回值**：`bool` 表达不了“未知”，忽略的 `error` 让问题消失，`interface{}` 换不来类型安全。写这类工具函数时，可以按下面的顺序检查：

1. 返回值能不能表达失败？路径判断返回 `(bool, error)`；创建目录的封装必须把 error 返回出去，而不是静默失败。
2. 写文件有没有明确覆盖语义？`os.WriteFile`（`O_TRUNC`）和“必须不覆盖时用 `O_EXCL`”应该是两个显式的函数。
3. 读取有没有上限？Scanner 默认 64KB、`Buffer` 提高上限之后，`Err()` 仍然必须检查。
4. 命令是否存在字符串拼接？默认走 `exec.Command(name, args...)`，shell 只留给可信脚本，并且带 context 超时。
5. 摘要函数的用途是什么？口令走 bcrypt/argon2id，文件校验流式计算，分片才用 CRC32。

这些点单看都不大，但每一个都能消掉一类难排查的生产问题。
