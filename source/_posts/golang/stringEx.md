---
title: Go 字符串处理：先搞懂 UTF-8 和 rune，再谈链式封装
date: 2022-04-29 23:18:30
updated: 2026-10-09
author: ws
description: 通过一个 String 包装类型，讲清 Go 字符串的字节与字符
categories: ["Go"]
tags: ["Go", "字符串"]
cover:
---

## 引言

这篇文章从一个看起来很顺手的 `String` 包装类型出发：它封装了常用的字符串操作，还支持链式调用，但 `Length()` 会骗你、`Index()` 会给你半个汉字、链式 `ReplaceAll` 会悄悄改掉别人手里的值。问题不在于封装本身，而在于把 Go 的 `string` 当成了"字符数组"。搞懂 UTF-8 编码和 rune 之后，这些问题都会变得一眼可见。

适合写过一些 Go、但对 `len(s)` 和 `for range` 的区别只有一个模糊印象的读者。读完你会掌握：string 的字节模型、rune 与 byte 的取舍、字符串拼接的性能边界，以及一个能正确处理中文的封装该长什么样。

## string 是只读的字节序列

Go 的 `string` 在内存里就是一段只读的字节：一个数据指针加一个长度，没有任何"字符"的概念。所谓字符，是你用 UTF-8 解码这段字节之后才有的东西。UTF-8 是变长编码：ASCII 字符 1 字节，汉字通常 3 字节，emoji 通常是 4 字节。

写个程序把这些事实打出来：

```go
package main

import (
	"fmt"
	"strings"
	"unicode/utf8"
)

func main() {
	s := "中国"
	fmt.Println("len(\"中国\") =", len(s))
	fmt.Println("utf8.RuneCountInString(\"中国\") =", utf8.RuneCountInString(s))
	for i, r := range s {
		fmt.Printf("字节下标 %d -> %c (U+%04X)\n", i, r, r)
	}
	fmt.Printf("\"中国\"[0] = %#x\n", s[0])
	r, size := utf8.DecodeRuneInString(s)
	fmt.Printf("DecodeRuneInString: %c, 宽度 %d 字节\n", r, size)

	text := "你好，世界"
	idx := strings.Index(text, "世界")
	half := text[:idx+1]
	fmt.Printf("\nstrings.Index(\"你好，世界\", \"世界\") = %d (字节)\n", idx)
	fmt.Printf("切到半个字符: %q\n", half)
	fmt.Println("utf8.ValidString =", utf8.ValidString(half))
}
```

真实输出（Go 1.25.9）：

```text
len("中国") = 6
utf8.RuneCountInString("中国") = 2
字节下标 0 -> 中 (U+4E2D)
字节下标 3 -> 国 (U+56FD)
"中国"[0] = 0xe4
DecodeRuneInString: 中, 宽度 3 字节

strings.Index("你好，世界", "世界") = 9 (字节)
切到半个字符: "你好，\xe4"
utf8.ValidString = false
```

三个要点：

1. `len("中国")` 是 6，因为"中"是 `E4 B8 AD`、"国"是 `E5 9B BD`，各 3 字节。想要"字符数"用 `utf8.RuneCountInString`，返回 2。
2. `for i, r := range s` 会边走边按 UTF-8 解码：`i` 是当前 rune 的**字节**起始下标（0 和 3），`r` 才是解码后的 rune（U+4E2D、U+56FD）。遇到非法字节，`range` 会解出 `U+FFFD` 并前进 1 字节，不会崩。
3. 按字节随便切片会切出不合法的 UTF-8：`"你好，世界"[:10]` 把"世"的 3 个字节砍掉一半，得到 `"你好，\xe4"`，`utf8.ValidString` 为 `false`。这种字符串打印出来是乱码，写进 JSON 或数据库还可能报错。

常用的 `unicode/utf8` API 就这几个：

| 函数 | 作用 |
| --- | --- |
| `utf8.RuneCountInString(s)` | 字符（rune）个数，O(n) |
| `utf8.DecodeRuneInString(s)` | 解出首字节开始的第一个 rune 及宽度 |
| `utf8.DecodeLastRuneInString(s)` | 从尾部解码一个 rune |
| `utf8.ValidString(s)` | 是否是合法 UTF-8 |
| `utf8.RuneLen(r)` | rune 编码成 UTF-8 后的字节数 |
| `utf8.RuneStart(b)` | 字节 b 是否是某个 rune 的首字节 |

byte 和 rune 的关系可以这样记：`byte` 是 `uint8` 的别名，面向编码；`rune` 是 `int32` 的别名，面向"一个 Unicode 码点"。`[]byte` 面向字节，可以直接交给 I/O 接口使用，`[]rune` 适合按字符下标操作，但两者之间的转换都是 O(n) 且会分配内存。

## 朴素实现：String 包装类型

先看这个包装类型的完整实现：

```go
package stringEx

import (
	"strconv"
	"strings"
)

type String struct {
	str string
}

func NewString(s string) *String {
	return &String{str: s}
}

func (s *String) Contain(substr string) bool {
	return strings.Contains(s.str, substr)
}

func (s *String) Index(substr string) int {
	return strings.Index(s.str, substr)
}

func (s *String) LastIndex(substr string) int {
	return strings.LastIndex(s.str, substr)
}

func (s *String) Split(sep string) []string {
	return strings.Split(s.str, sep)
}

func (s *String) Length() int {
	return len(s.str)
}

func (s *String) ReplaceAll(old, new string) *String {
	s.str = strings.ReplaceAll(s.str, old, new)
	return s
}

func (s *String) ToString() string {
	return s.str
}

func (s *String) ToInt() (int, error) {
	return strconv.Atoi(s.str)
}
```

方法和字段都很简单，`ReplaceAll` 返回 `*String` 是为了支持链式调用。配套的测试写起来也很短：

```go
var str = NewString("123qwe...")

func TestString_Contain(t *testing.T) {
	log.Println(str.Contain("123"), str.Length())
	log.Println(str.ReplaceAll("123", "789").ToString())
	log.Println(str.ReplaceAll("123", "789").Contain("789"))
}
```

跑出来是这样：

```text
2026/10/09 10:34:16 true 9
2026/10/09 10:34:16 789qwe...
2026/10/09 10:34:16 true
```

输出一切正常，测试"通过"，但每一行都埋着问题。下面逐个拆开看。

## 三个典型问题

### 问题一：Length() 返回的是字节数

`Length()` 直接返回 `len(s.str)`，对全 ASCII 的 `"123qwe..."` 恰好等于字符数 9，测试自然看不出问题。换成中文立刻穿帮：

```go
s := stringEx.NewString("中国")
s.Length() // 6，不是 2
```

如果这个长度被用来分配缓冲区、做分页、校验昵称长度，中文用户就会拿到错误的结果——一个 6 个汉字的昵称会被算成 18。

### 问题二：Index 返回字节下标，拿去切片会出事

`strings.Index` 返回的永远是字节下标，但方法名 `Index` 很容易让人以为是字符位置。来看两个具体的后果：

```go
text := stringEx.NewString("你好，世界")
idx := text.Index("世界")     // 9（字节）
half := text.ToString()[:idx+1]
// half = "你好，\xe4"，utf8.ValidString(half) == false
```

截出了半个"世"。如果反过来把字节下标当成字符下标用，就不只是乱码，而是直接越界：

```go
s := stringEx.NewString("中文abc")
i := s.Index("abc") // 字节下标 6
rs := []rune(s.ToString())
_ = rs[i:]          // panic: slice bounds out of range [6:5]
```

"中文abc"只有 5 个 rune，用 6 去切 `[]rune` 必然 panic。这类 bug 的恶心之处在于：**全英文数据永远正常，上线遇到中文才开始崩**。

### 问题三：原地修改 + 链式调用，共享指针一起变

`ReplaceAll` 把结果写回 `s.str` 再返回 `s`。这意味着所有指向同一个 `*String` 的变量共享同一块状态，谁调用谁就是"全局修改"：

```go
a := stringEx.NewString("123qwe...")
b := a.ReplaceAll("123", "789")
// a == b 为 true
// a = "789qwe...", b = "789qwe..."
```

回到上面的测试：第一行打印后 `str` 还是 `"123qwe..."`；第二行把 `str` 就地改成 `"789qwe..."`；第三行再次 `ReplaceAll("123", "789")` 时，`"123"` 已经被上一行吃掉了，这次调用是空操作，而 `Contain("789")` 返回 `true` 只是因为第二行的残留。**如果测试是独立的、顺序打乱的，或者第二行没有先跑，第三行的结果就会完全不同（`Contain("789")` 会是 false）**。这是一个典型的测试相互污染：测试通过不代表行为正确。

顺带说清封装本身的问题：`*String` 保存的是 `string` 字段，`string` 在 Go 里本来就是不可变值，把它包起来再手动模拟可变对象，等于放弃语言给你的安全保证，还额外带来一次堆分配。

## 拼接性能：+=、Builder、Join、bytes.Buffer

字符串拼接是"包装类型"最容易加错功能的地方，先把基准数据摆出来。测试在循环里拼 1000 个片段，每段是 14 字节的 `"hello, 世界 "`：

```go
const parts = 1000

var piece = "hello, 世界 "

func BenchmarkConcatPlus(b *testing.B) {
	for i := 0; i < b.N; i++ {
		s := ""
		for j := 0; j < parts; j++ {
			s += piece
		}
		_ = s
	}
}

func BenchmarkBuilderGrow(b *testing.B) {
	for i := 0; i < b.N; i++ {
		var sb strings.Builder
		sb.Grow(parts * len(piece))
		for j := 0; j < parts; j++ {
			sb.WriteString(piece)
		}
		_ = sb.String()
	}
}
```

`go test -bench=. -benchmem -count=3 ./bench/`，Apple M1 Pro 上的代表值：

```text
BenchmarkConcatPlus-8      1629    845107 ns/op   7395239 B/op   999 allocs/op
BenchmarkBuilder-8       101385     12403 ns/op     62960 B/op    16 allocs/op
BenchmarkBuilderGrow-8   283813      4393 ns/op     14336 B/op     1 allocs/op
BenchmarkJoin-8          130690      9751 ns/op     14336 B/op     1 allocs/op
BenchmarkBuffer-8        117597      9172 ns/op     47040 B/op    10 allocs/op
```

| 写法 | 每次操作耗时 | 内存分配 | 适用场景 |
| --- | --- | --- | --- |
| `s += piece` | 约 845 µs | 7.4 MB，999 次 | 只拼两三个短串；循环里禁用 |
| `strings.Builder` | 约 12 µs | 63 KB，16 次 | 通用循环拼接 |
| `strings.Builder` + `Grow` | 约 4.4 µs | 14 KB，1 次 | 能预估总长度时最优 |
| `strings.Join` | 约 9.8 µs | 14 KB，1 次 | 已经有 `[]string` 时 |
| `bytes.Buffer` | 约 9.2 µs | 47 KB，10 次 | 混合二进制/文本，或实现 `io.Writer` |

`+=` 比 `Builder` + `Grow` 慢约 190 倍，原因很直白：每拼一次都要分配一个新字符串并拷贝旧内容，总拷贝量是 O(n²)，1000 个片段就是约 7 MB 的搬运。`Builder.Grow` 一次把容量申请够，之后只是往 `[]byte` 尾部追加，所以只有 1 次分配。

选择建议：**循环里用 `strings.Builder`，能估准总长就先 `Grow`；手里已经是切片就 `Join`；要写文件、网络或实现 `io.Writer` 接口用 `bytes.Buffer`**。`+=` 只在拼接次数是常数、长度很小时用。

## strings / strconv 常用 API 速查

大部分"String 扩展"需求，标准库已经有对应的函数。先查表再动手：

| `strings` API | 作用 |
| --- | --- |
| `Contains(s, sub)` / `ContainsAny` / `ContainsRune` | 子串/字符包含判断 |
| `Index(s, sub)` / `LastIndex` | 字节下标，找不到返回 -1 |
| `IndexRune(s, r)` | 按 rune 找字节下标 |
| `Split(s, sep)` / `SplitN` | 切分成 `[]string` |
| `SplitSeq(s, sep)` | Go 1.24+ 的迭代器版切分，不构造切片 |
| `Fields(s)` / `FieldsSeq` | 按连续空白切分 |
| `Join(elems, sep)` | 用分隔符连接 |
| `HasPrefix` / `HasSuffix` | 前后缀判断 |
| `TrimSpace` / `TrimPrefix` / `TrimSuffix` / `Trim` / `TrimFunc` | 去空白/去前后缀 |
| `Replace` / `ReplaceAll` | 替换（`Replace` 可限次数） |
| `ToLower` / `ToUpper` | 大小写转换（Unicode 感知） |
| `Count(s, sub)` | 子串出现次数 |
| `Repeat(s, n)` | 重复拼接 |
| `Map(f, s)` | 按 rune 映射，比如自定义过滤 |
| `Builder` | 高效增量拼接 |

| `strconv` API | 作用 |
| --- | --- |
| `Atoi(s)` / `Itoa(n)` | `string` 与 `int` 互转 |
| `ParseInt(s, base, bitSize)` / `ParseFloat` / `ParseBool` | 带进制/位宽的解析 |
| `FormatInt(n, base)` / `FormatFloat(f, fmt, prec, bits)` | 数字转字符串 |
| `Quote(s)` / `Unquote(s)` | 加/去 Go 转义引号 |

真要用 `[]byte` 处理中文时，记住一个原则：**只要涉及"第几个字符"，就先明确自己说的是字节还是 rune**。标准库所有 `Index` 系列函数都返回字节下标，这个约定是一致且不可更改的。

## 推荐实现：不可变风格 + rune 正确性

实现思路有两条：每次"修改"都返回新对象（保持不可变），长度类 API 明确区分字节和字符。完整代码如下：

```go
// Package good 是 String 包装类型的推荐实现：不可变风格 + 正确处理 rune。
package good

import (
	"strconv"
	"strings"
	"unicode/utf8"
)

// String 是不可变的字符串包装，所有"修改"方法返回新实例。
type String struct {
	str string
}

func New(s string) *String {
	return &String{str: s}
}

func (s *String) ToString() string { return s.str }

// Length 返回字符（rune）数，而不是字节数。
func (s *String) Length() int { return utf8.RuneCountInString(s.str) }

// ByteLen 返回字节数，等价于 len(s.str)。
func (s *String) ByteLen() int { return len(s.str) }

func (s *String) Contain(substr string) bool { return strings.Contains(s.str, substr) }

// Index 返回子串的字节下标，找不到返回 -1。
func (s *String) Index(substr string) int { return strings.Index(s.str, substr) }

// LastIndex 返回子串最后一次出现的字节下标。
func (s *String) LastIndex(substr string) int { return strings.LastIndex(s.str, substr) }

// ReplaceAll 返回替换后的新 String，不修改原值。
func (s *String) ReplaceAll(old, new string) *String {
	return &String{str: strings.ReplaceAll(s.str, old, new)}
}

// Split 按分隔符切分，返回新的切片。
func (s *String) Split(sep string) []string { return strings.Split(s.str, sep) }

// Runes 返回字符切片，适合按下标精确操作。
func (s *String) Runes() []rune { return []rune(s.str) }

// Substr 按字符下标截取 [start, end)，越界自动截断。
func (s *String) Substr(start, end int) string {
	rs := []rune(s.str)
	if start < 0 {
		start = 0
	}
	if end > len(rs) {
		end = len(rs)
	}
	if start >= end {
		return ""
	}
	return string(rs[start:end])
}

func (s *String) ToInt() (int, error) { return strconv.Atoi(s.str) }
```

用一段程序验证行为：

```go
s := good.New("中国abc")
fmt.Printf("Length=%d ByteLen=%d Substr(0,2)=%q Index(abc)=%d\n",
	s.Length(), s.ByteLen(), s.Substr(0, 2), s.Index("abc"))

base := good.New("123qwe...")
chained := base.ReplaceAll("123", "789").ReplaceAll("qwe", "xxx")
fmt.Printf("chained=%q base=%q\n", chained.ToString(), base.ToString())
```

真实输出：

```text
Length=5 ByteLen=9 Substr(0,2)="中国" Index(abc)=6
chained="789xxx..." base="123qwe..."
```

结果一目了然：`Length` 是 5（字符数），`ByteLen` 是 9（字节数）；`Substr` 按字符下标截取，不会切碎汉字；链式 `ReplaceAll` 每一步返回新对象，调用方手里的 `base` 始终保持 `"123qwe..."`；`Index` 仍然返回字节下标，但方法注释写清楚了，调用方要按字符切就用 `Substr` 或 `Runes`。

## 踩坑与边界

1. **rune 数不等于肉眼可见的"字"**。emoji 组合（如 `👨‍👩‍👧`）、带变音符号的拉丁字母由多个码点组成，`RuneCountInString` 会数多。真要按用户感知切字，得引入字素簇（grapheme cluster）库，标准库不做这件事。
2. **`[]rune(s)` 是 O(n) 复制**。大文本上反复 `[]rune` 转换是性能杀手；能按字节做的事（`Contains`、`Index`、`Split`）就按字节做。
3. **ASCII 假设会潜伏很久**。`len` 当字符数、`s[i]` 当第 i 个字符，在全英文数据上跑一年都没事，换中文就爆。代码评审时看到 `len(s)` 出现在"长度校验"语境里就要警觉。
4. **拆分和索引的 API 语义要统一**。`strings` 包全部返回字节下标；自己封装时要么沿用这个约定并在命名上体现（`ByteIndex`），要么提供 `RuneIndex` 并且文档写明白，最忌讳名字含糊、两种语义混用。
5. **封装要克制**。这个 `String` 提供的所有能力，标准库 `strings` + `strconv` 都能直接做，包一层只带来学习成本和一次堆分配。只有当确实要挂业务语义（比如统一长度校验规则、绑定默认 locale）时，封装才划算。

## 总结

- `string` 是不可变的字节序列；`len` 数字节，`utf8.RuneCountInString` 数字符，`range` 按 UTF-8 解码并给出字节下标和 rune。所有 `strings` 的 `Index` 系列返回的都是字节下标。
- 朴素实现的三个问题是连锁的：`Length` 返回字节数；`Index` 的字节下标被当作字符下标使用，切出非法 UTF-8 甚至 panic；`ReplaceAll` 原地修改并返回自身，让共享同一指针的所有变量互相干扰，连测试都在悄悄污染全局状态。
- 拼接 1000 个片段，`+=` 约 845 µs、999 次分配；`strings.Builder` 约 12 µs，加 `Grow` 预分配后约 4.4 µs 且只有 1 次分配。循环拼接一律用 Builder。
- 推荐实现用不可变风格（每次返回新对象）+ rune 感知的 `Length`/`Substr`，链式调用不再互相干扰。但更推荐的做法是：**直接用 `strings` 包，按需封装，不为了"链式"而链式**。
