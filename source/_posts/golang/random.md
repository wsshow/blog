---
title: Go 高效生成随机字符串：位掩码技巧、并发陷阱与 crypto/rand
date: 2023-01-03 21:13:30
updated: 2026-10-09
author: ws
description: 拆解随机字符串生成中的抽样算法，分清性能与安全的边界
categories: ["Go"]
tags: ["Go", "随机数"]
cover:
---

## 引言

这篇文章从一段很常见的 Go 随机字符串代码讲起：它用位掩码加拒绝采样从 `math/rand` 里高效抽字母，还用 `unsafe` 把 `[]byte` 零拷贝转成 `string`。技巧本身没问题，但代码里埋着一个致命 bug——包级共享的 `rand.Source` 不是并发安全的，线上并发调用会直接产生数据竞争。

适合已经会写 Go、正在写工具库或服务的读者。读完你会掌握：位掩码抽样的原理与均匀性证明、`unsafe` 转换的适用前提、用 `go test -race` 复现并定位竞争的具体方法，以及什么时候必须换成 `crypto/rand`。

## 常见写法与位掩码抽样

先看这段代码：

```go
package utils

import (
	"math/rand"
	"time"
	"unsafe"
)

const letters = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ"

var src = rand.NewSource(time.Now().UnixNano())

const (
	letterIdBits = 6
	letterIdMask = 1<<letterIdBits - 1
	letterIdMax  = 63 / letterIdBits
)

func RandStr(n int) string {
	b := make([]byte, n)
	for i, cache, remain := n-1, src.Int63(), letterIdMax; i >= 0; {
		if remain == 0 {
			cache, remain = src.Int63(), letterIdMax
		}
		if idx := int(cache & letterIdMask); idx < len(letters) {
			b[i] = letters[idx]
			i--
		}
		cache >>= letterIdBits
		remain--
	}
	return *(*string)(unsafe.Pointer(&b))
}
```

核心是"挑随机数、筛掉超范围的"这个采样过程。`src.Int63()` 返回一个 63 位随机数，代码每次从最低位扣 6 位当作一个 0~63 的下标，用完右移 6 位继续，直到调用者要的 n 个字符都填满：

```text
一次 Int63() 得到的 cache（63 位 = 10 组 6 位 + 最高 3 位）：

  [ 6 bits ⑩ ][ 6 bits ⑨ ] ... [ 6 bits ② ][ 6 bits ① ]
                                            ↑
                                     先取最低 6 位

  ① ~ ⑩ 从低位到高位依次被取出；剩下的最高 3 位直接丢弃

每 6 位是一个 0~63 的均匀随机下标：
  0~51  -> 命中 letters，接受
  52~63 -> 超出字母表长度 52，丢弃，从下一组 6 位重试
```

三个常量各司其职：

- `letterIdBits = 6`：因为 `2^6 = 64` 能覆盖 52 个字母；
- `letterIdMask = 63`：`cache & 63` 取出最低 6 位；
- `letterIdMax = 63/6 = 10`：一次 `Int63()` 最多提供 10 个候选，用满就重新取值。

**为什么丢弃超范围的下标能保证均匀？** 每个 6 位块在 0~63 上等概率（`Int63()` 本身均匀，低 6 位也均匀），其中 0~51 被接受。条件概率 `P(k | k<52) = (1/64) / (52/64) = 1/52`，与 k 无关，所以 52 个字母严格等概率。接受率 52/64 = 81.25%，平均每个字符消耗约 1.23 个候选块，摊到每字符不到 0.13 次 `Int63()`。

字母表里没有数字，是有意的取舍：只出纯字母串时，同样的位掩码参数最简单。要注意它的熵：每个字符 `log2(52) ≈ 5.7 bit`，16 位随机串约 91 bit 随机性，做普通分享码够用。如果哪天给 `letters` 补上数字，字母表变成 62 个字符，代码依然正确（拒绝率略降）；但字母表一旦超过 64 个字符，`idx` 永远落在 0~63，新增的字符永远不会被生成，而且不会有任何报错——这个边界必须留意。顺带一个优雅的观察：**字母表恰好 64 个字符时，6 位掩码零拒绝**，例如 `base64url` 字符表 `A-Za-z0-9-_`，`v & 63` 直接查表，连拒绝采样都省了。

## unsafe：切片头到字符串头的转换

Go 里 `[]byte` 转 `string` 通常会复制一份（保证字符串不可变）。`*(*string)(unsafe.Pointer(&b))` 做的是重新解释内存布局，实现零拷贝：

```text
slice header:  +----------+----------+----------+
               | ptr      | len      | cap      |
               +----------+----------+----------+
string header: +----------+----------+
               | ptr      | len      |
               +----------+----------+
```

切片的头三个机器字是 `{数据指针, len, cap}`，字符串的头两个字是 `{数据指针, len}`。把切片头的地址转成 `*string` 再解引用，就是按字符串头的前两个字段去读：指针和长度完全对上，`cap` 被忽略。前提只有一个——**得到的字符串存活期间，`b` 的字节不能再被修改**。好在这里 `b` 是函数内新分配、写完就不再动的，所以安全。

Go 1.20 之后有更清晰的官方写法，不需要自己转换类型：

```go
return unsafe.String(unsafe.SliceData(b), len(b))
```

`unsafe.SliceData(b)` 拿到 `*byte`，`unsafe.String` 按给定长度构造字符串。语义和上面完全一致，但意图明确，也不会被误读成任意类型转换。两者都要求"禁止再修改 `b`"；另外字符串持有底层数组的指针，GC 会把它保活，不用担心 `b` 被回收。

## 并发陷阱：共享的 rand.Source 会打架

这段代码把 `src` 放在包级，所有调用者共享。`math/rand` 的文档写得很清楚：`rand.NewSource` 返回的 Source **不是并发安全的**，需要同步才能跨 goroutine 共享（对比之下，`math/rand` 的顶层函数如 `rand.Int63()` 自带锁，是安全的）。

写个测试并发调用，用竞态检测器抓现行：

```go
func TestConcurrentRandStr(t *testing.T) {
	var wg sync.WaitGroup
	for g := 0; g < 8; g++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for i := 0; i < 3000; i++ {
				if got := RandStr(16); len(got) != 16 {
					t.Errorf("长度错误: %d", len(got))
				}
			}
		}()
	}
	wg.Wait()
}
```

```bash
go test -race -run TestConcurrentRandStr ./bad/
```

真实输出（截取）：

```text
==================
WARNING: DATA RACE
Read at 0x00c000138000 by goroutine 11:
  math/rand.(*rngSource).Uint64()
  math/rand.(*rngSource).Int63()
  blogdemo/randstr/bad.RandStr()
      .../bad/randstr.go:22

Previous write at 0x00c000138000 by goroutine 13:
  math/rand.(*rngSource).Uint64()
  math/rand.(*rngSource).Int63()
  blogdemo/randstr/bad.RandStr()
      .../bad/randstr.go:22
==================
FAIL	blogdemo/randstr/bad	0.418s
FAIL
```

竞争点的后果不只是"随机性变差"：`rngSource` 的内部状态被交错读写，可能产生重复序列；更现实的是，任何开了 `-race` 的测试或 CI 都会直接失败。除了加锁，还有三种干净的写法。

**写法一：用 `math/rand/v2` 的顶层函数。** 顶层函数并发安全、自动播种，连 `time.Now` 都不用写：

```go
package good

import (
	"math/rand/v2" // 包路径是 v2，包名依然是 rand
	"unsafe"
)

const letters = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ"

// sample 是抽取算法本体：把 next 提供的 63 位随机数按 6 位一组切开，
// 落在字母表范围内就接受，否则整组丢弃。
func sample(n int, next func() int64) string {
	b := make([]byte, n)
	for i, cache, remain := n-1, next(), 63/6; i >= 0; {
		if remain == 0 {
			cache, remain = next(), 63/6
		}
		if idx := int(cache & 63); idx < len(letters) {
			b[i] = letters[idx]
			i--
		}
		cache >>= 6
		remain--
	}
	return unsafe.String(unsafe.SliceData(b), len(b))
}

// RandStrV2 用 math/rand/v2 的顶层函数：自动播种、并发安全、无需自己管 Source。
func RandStrV2(n int) string {
	return sample(n, rand.Int64)
}
```

**写法二：`sync.Pool` 复用独占的 `*rand.Rand`。** 适合还在用传统 `math/rand` 且要调其他方法的项目（下面是 v1 API，用别名 `mrand` 与 v2 区分）：

```go
import mrand "math/rand"

var pool = sync.Pool{
	New: func() any {
		return mrand.New(mrand.NewSource(mrand.Int63()))
	},
}

func RandStrPool(n int) string {
	r := pool.Get().(*mrand.Rand)
	defer pool.Put(r)
	return sample(n, r.Int63)
}
```

每个调用借到的是独占实例，用完归还，既没有锁竞争也不会串数据。注意 `pool.New` 里用 `math/rand` 顶层函数 `mrand.Int63()` 取种子——顶层函数自带锁，是并发安全的。

**写法三：每个 goroutine 持有自己的 `*rand.Rand`（v2）。** 用 `rand.NewPCG(seed1, seed2)` 创建独立随机源，适合高频调用路径：goroutine 只创建一次，之后一直用。下面这个是"每次调用新建"的反面教材，虽然安全但多两次分配：

```go
func RandStrPCGPerCall(n int) string {
	r := rand.New(rand.NewPCG(rand.Uint64(), rand.Uint64()))
	return sample(n, func() int64 { return int64(r.Uint64() >> 1) })
}
```

换成并发安全的写法之后，同样的并发测试 `go test -race ./good/` 输出干净：

```text
ok  	blogdemo/randstr/good	1.527s
```

## 安全场景：crypto/rand

`math/rand` 的输出是可预测的——知道种子和调用序列就能推出后续所有值，绝对不能用于 token、验证码、会话 ID、密码重置链接。安全场景用 `crypto/rand`，它是操作系统的密码学随机源。

`crypto/rand.Read` 的文档说：填充 `b`，**永远不返回错误**（除非遗留 Linux 系统调用失败，Go 会直接崩掉程序而不是返回错误），所以在正常路径上不用担心错误处理。用 64 个字符的字母表把拒绝采样降到零：

```go
package secure

import "crypto/rand"

// alphabet 64 个字符：正好占满 6 bit，抽样时零拒绝、零偏差。
const alphabet = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789-_"

// Token 用 crypto/rand 生成 n 个字符的 URL 安全 token。
func Token(n int) string {
	b := make([]byte, n)
	buf := make([]byte, 64)
	for i := 0; i < n; {
		if _, err := rand.Read(buf); err != nil {
			panic(err) // 文档承诺不会失败，这里只是保持习惯
		}
		for _, v := range buf {
			b[i] = alphabet[v&63]
			i++
			if i == n {
				break
			}
		}
	}
	return string(b)
}
```

一次读 64 字节，每字节取低 6 位查表，全部命中、零浪费，比逐字节调用系统接口高效得多。

如果只需要一个现成的安全 token，Go 1.24 起标准库直接提供 `crypto/rand.Text()`：

```go
s := rand.Text() // 例：2HN3BV5MU772BZM6NGVLOXWEQO
```

官方文档说明：返回 RFC 4648 base32 字母表的随机字符串，至少含 128 位熵，足够抵抗暴力猜测且碰撞概率可忽略。实测长度固定 26 个字符，样例如下：

```text
crypto/rand.Text() 三个样例:
  2HN3BV5MU772BZM6NGVLOXWEQO (长度 26, 全部命中 base32 字母表: true)
  6G6D2J6XZYSCPM25LJ4G3GNH4D (长度 26, 全部命中 base32 字母表: true)
  O6XLJIEGEJUSK7DSLGEVSQA6PJ (长度 26, 全部命中 base32 字母表: true)
```

生成 32 字符 token 和 `Text()`，用 benchmark 实测性能（`go test -bench=. -benchmem -count=3 ./bench/`，Apple M1 Pro）：

| 基准 | 每次耗时 | 内存 | 分配次数 | 并发安全 |
| --- | --- | --- | --- | --- |
| `bad.RandStr`（math/rand） | 109 ns | 32 B | 1 | 否，有竞争 |
| `good.RandStrV2`（rand/v2 顶层） | 125 ns | 32 B | 1 | 是 |
| `good.RandStrPool`（sync.Pool） | 130 ns | 32 B | 1 | 是 |
| `good.RandStrPCGPerCall` | 160 ns | 48 B | 2 | 是（每次新建源） |
| `crypto/rand.Read`（32 字节） | 282 ns | 0 B | 0 | 是 |
| `secure.Token`（crypto/rand） | 323 ns | 32 B | 1 | 是 |
| `secure.Text`（crypto/rand.Text） | 305 ns | 32 B | 1 | 是 |

这组数字最重要的结论是：**`crypto/rand` 只比 `math/rand` 慢约 2.5~3 倍，绝不是数量级差距**。Go 1.24 起在 macOS 上 `crypto/rand` 直接调用系统的 `arc4random_buf`（内核维护的 CSPRNG，永远不返回错误），每次 32 字节读取只要约 280 ns。为了省不到 200 纳秒把 token 换成可预测的伪随机数，是最典型的捡芝麻丢西瓜。

另外我用竞态检测器把基准也跑了一遍（`go test -race -bench='BenchmarkV2Global|BenchmarkSecureToken' -benchtime=200000x ./bench/`），确认基准代码本身在并发检测下没有问题：

```text
BenchmarkV2Global-8       200000    671.2 ns/op    32 B/op    1 allocs/op
BenchmarkSecureToken-8    200000     1005 ns/op    32 B/op    1 allocs/op
PASS
```

`-race` 下的数字带检测开销（本次约 3~5 倍），只用来验证正确性，不要拿来比较性能。

## 常见误区

1. **用时间戳做种子**。`rand.NewSource(time.Now().UnixNano())` 的问题有两个：种子可预测，攻击者能枚举出附近的时间戳复现序列；在快速创建的多个实例里，同一纳秒可能拿到相同的种子，产生完全相同的"随机"序列。`math/rand/v2` 干脆移除了种子 API，顶层函数自动播种，也就不存在这个问题；自己建源则显式传 `NewPCG(seed1, seed2)`。

2. **取模偏差**。`randomByte % 52` 看似均匀，其实 256 不是 52 的整数倍：`256 = 4×52 + 48`，所以余数 0~47 有 5 个原像，48~51 只有 4 个。取 100 万个均匀字节实测：

   ```text
   残差    次数      实测频率   理论频率
      0     19494    1.9494%   5/256 = 1.9531%
      1     19654    1.9654%   5/256 = 1.9531%
      2     19676    1.9676%   5/256 = 1.9531%
      3     19273    1.9273%   5/256 = 1.9531%
     48     15803    1.5803%   4/256 = 1.5625%
     49     15738    1.5738%   4/256 = 1.5625%
     50     15440    1.5440%   4/256 = 1.5625%
     51     15576    1.5576%   4/256 = 1.5625%
   ```

   前 48 个字符出现概率比后 4 个高 25%。解法就是本文的拒绝采样，或者用 `rand.IntN(52)`（标准库内部做了无偏处理），安全场景则选 64 字符表直接消掉整除了问题。

3. **把 `math/rand` 用在安全用途**。验证码、token、密钥、抽奖顺序，只要对手能观察或影响结果，就必须 `crypto/rand`。性能差距只有 2.5~3 倍，没有任何理由冒险。

4. **`unsafe` 转换后继续改 `b`**。`unsafe.String` 得到的字符串和 `b` 共享内存，之后 `b[0] = 'x'` 会改掉字符串内容，违反字符串不可变契约，行为未定义。转换后把 `b` 当作只读，或者干脆复制。

5. **字母表变化导致掩码失效**。`v & 63` 只产生 0~63；字母表一旦超过 64 个字符，后面的字符永远不会出现在结果里，而且不报错。改字母表时要么保证长度不超过掩码范围，要么同步调整位数和拒绝逻辑。

## 总结

- 位掩码抽样的本质是"6 位对 64，筛掉 52~63"的拒绝采样，被接受的字母严格均匀；一次 `Int63()` 提供 10 个候选。字母表恰好 64 个字符时零拒绝。
- `unsafe` 零拷贝转换利用的是切片头和字符串头前两个字段布局相同；Go 1.20+ 用 `unsafe.String(unsafe.SliceData(b), len(b))` 表达更清楚，前提是之后不再修改 `b`。
- 这种写法最大的坑是包级共享 `rand.Source`，`go test -race` 能稳定复现数据竞争，8 个 goroutine 各 3000 次调用就会触发。替代写法任选：`math/rand/v2` 顶层函数、`sync.Pool`、每 goroutine 独立 `*rand.Rand`。
- 安全场景一律 `crypto/rand`：`rand.Read` 文档保证不返回错误，Go 1.24+ 的 `rand.Text()` 直接给出至少 128 位熵的 base32 串。实测每次生成 32 字符 token 约 323 ns，只比伪随机方案慢 2~3 倍。
- 避开五个经典错误：时间戳种子、取模偏差、安全场景用 `math/rand`、改共享字节后继续用 `unsafe` 字符串、字母表超长导致掩码失效。
