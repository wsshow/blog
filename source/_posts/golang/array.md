---
title: 用 Go 泛型实现动态数组：从 []any 到类型安全
date: 2022-04-29 23:18:30
updated: 2026-10-09
author: ws
description: 以动态数组为例，讲清 Go 切片底层原理与泛型容器的取舍
categories: ["Go"]
tags: ["Go", "数据结构", "泛型"]
cover:
---

## 引言

很多从 Java、C# 转过来的开发者会习惯性写一个 `Array` 容器类，用 `any` 存元素，再配一套 `Add`/`Remove`/`Filter` 方法。这篇文章就从这样一种常见的朴素写法讲起：先讲清 Go 切片本身就是动态数组，再逐个拆开它最容易踩的三个坑，最后给出类型安全的泛型实现 `Array[T]`，并用基准测试回答一个更根本的问题——日常到底该不该自己写容器。

适合有基础 Go 语法、写过几天项目、想搞清楚切片和泛型的读者。读完你会得到：切片的内存模型与扩容规律、`any` 容器的具体坑、泛型方法的边界（为什么 `Map` 不能改元素类型），以及一组可复现的实测数字。

## 切片就是 Go 内置的动态数组

一个切片变量在内存里是三部分：指向底层数组的指针、长度、容量。

```text
slice header                backing array (底层数组)
+----------+                +----+----+----+----+----+
| ptr      |--------------->| 10 | 20 | 30 | 40 | 50 |
+----------+                +----+----+----+----+----+
| len = 3  |                  ↑         ↑
+----------+                 可读写范围   cap 终点
| cap = 5  |
+----------+
```

- `len` 是当前元素个数，`append` 在这里加元素；
- `cap` 是底层数组能容纳的元素总数，超出就要重新分配、拷贝；
- 切片只是"视图"，多个切片可能共享同一个底层数组，这是后文一个 bug 的根源。

当容量不够时，`append` 会调用运行时的扩容逻辑（`runtime.nextslicecap`）。Go 1.25 的规则是：需要的容量超过旧容量两倍时直接按需分配；否则旧容量小于 256 就翻倍，达到 256 之后按约 1.25 倍增长，最后再按运行时内存尺寸类（size class）向上取整。不用背具体数字，写个程序观察即可：

```go
package main

import "fmt"

func main() {
	s := make([]int, 0, 256)
	prev := cap(s)
	for i := 0; i < 20000; i++ {
		s = append(s, i)
		if cap(s) != prev {
			fmt.Printf("len=%5d cap: %d -> %d\n", len(s), prev, cap(s))
			prev = cap(s)
		}
	}
}
```

运行 `go run .`，本机（Go 1.25.9，darwin/arm64）输出：

```text
len=  257 cap: 256 -> 512
len=  513 cap: 512 -> 848
len=  849 cap: 848 -> 1280
len= 1281 cap: 1280 -> 1792
len= 1793 cap: 1792 -> 2560
len= 2561 cap: 2560 -> 3408
len= 3409 cap: 3408 -> 5120
len= 5121 cap: 5120 -> 7168
len= 7169 cap: 7168 -> 9216
len= 9217 cap: 9216 -> 12288
len=12289 cap: 12288 -> 16384
len=16385 cap: 16384 -> 21504
```

可以看到 256 → 512 还是翻倍，512 → 848 这一步涨到约 1.66 倍（1.25 倍公式估算出 832，又被尺寸类抬到 848），之后涨幅在 1.3~1.5 倍之间波动——公式趋于 1.25 倍，波动来自内存尺寸类的向上取整。这些行为不需要自己实现——**切片加 `slices` 标准库已经覆盖了动态数组的全部刚需**：查找用 `slices.Contains`/`slices.Index`，删除用 `slices.Delete`，排序用 `slices.SortFunc`，克隆用 `slices.Clone`。

那为什么还要看 `[]any` 的写法？因为它能集中暴露几个新手常踩的坑，值得逐个拆开看。

## 朴素实现：基于 []any 的动态数组

先看完整实现，结构体和方法的命名都很直白：

```go
package bad

import "sort"

type Array struct {
	data []any
}

func NewArray() *Array {
	return new(Array)
}

func (a *Array) Add(elems ...any) {
	a.data = append(a.data, elems...)
}

func (a *Array) Remove(e any) {
	d := a.data
	for i, cnt := 0, len(d); i < cnt; i++ {
		if d[i] == e {
			d = append(d[:i], d[i+1:]...)
			break
		}
	}
	a.data = d
}

func (a *Array) RemoveAll(e any) {
	d := a.data
	for i := 0; i < len(d); {
		if d[i] == e {
			d = append(d[:i], d[i+1:]...)
		} else {
			i++
		}
	}
	a.data = d
}

func (a *Array) Contain(e any) bool {
	for _, v := range a.data {
		if v == e {
			return true
		}
	}
	return false
}

func (a *Array) Count() int {
	return len(a.data)
}

func (a *Array) ForEach(f func(any)) {
	for _, v := range a.data {
		f(v)
	}
}

func (a *Array) Clear() {
	a.data = nil
}

func (a *Array) Data() []any {
	return a.data
}

func (a *Array) Sort(less func(i, j int) bool) {
	sort.Slice(a.data, less)
}

func (a *Array) Filter(f func(any) bool) *Array {
	na := NewArray()
	for _, v := range a.data {
		if f(v) {
			na.Add(f(v))
		}
	}
	return na
}

func (a *Array) Map(f func(any) any) *Array {
	na := NewArray()
	for _, v := range a.data {
		na.Add(f(v))
	}
	return na
}
```

这段代码的问题藏得比较深：如果测试只覆盖 `Add`、`Remove`、`RemoveAll`，数据又全是 `int`、`string` 这类可以用 `==` 正确比较的类型，所有断言都会通过，隐患也就一直留了下来。

## 三个典型问题

### 问题一：Contain 遇到 slice/map/func 会 panic

`Contain` 直接对两个 `any` 做 `==`。Go 比较 interface 值时，先看动态类型是否相同，相同再比较动态值；而**切片、map、函数这三种动态类型不可比较**，一旦类型命中，运行时直接 panic。如果测试数据只用可比较类型，这个隐患可以潜伏很久：只有真的往容器里塞了切片，才会在运行时炸出来。

写个最小复现：

```go
a := bad.NewArray()
a.Add([]int{1, 2})
a.Add([]int{3, 4})
fmt.Println(a.Contain([]int{1, 2}))
```

真实输出：

```text
panic: runtime error: comparing uncomparable type []int

goroutine 1 [running]:
blogdemo/array/bad.(*Array).Contain(...)
	.../bad/bad.go:45
main.main()
	.../bad/cmd/main.go:26 +0x654
exit status 2
```

补充一个容易忽略的细节：如果传入的值动态类型不同（比如容器里是 `[]int`，传进去的是 `[]string`），`==` 会直接判定为不相等并返回 `false`，不会 panic。panic 只在"动态类型相同且不可比较"时发生——这反而更难排查，因为它在大多数输入下工作正常。

### 问题二：Filter 把判断结果当成了元素

看 `Filter` 的这一行：

```go
if f(v) {
	na.Add(f(v)) // 加了 bool，而不是 v
}
```

`f(v)` 返回 `bool`，判断通过后塞进新数组的却不是 `v` 本身。对 `[1,2,3,4]` 过滤偶数，期望 `[2 4]`，实际得到两个 `true`：

```text
Filter 期望 [2 4]，实际: [true true]
```

正确的写法只差一个字：`na.Add(v)`——判断结果只用来决定是否追加，追加的应该是元素本身。

### 问题三：删除后底层数组残留旧引用

`Remove` 用 `append(d[:i], d[i+1:]...)` 左移元素、缩短长度，但底层数组最后一个槽位没有被清空，仍然指向被删除的元素。如果元素是大对象或带 finalizer 的资源，它不会被 GC 回收，等于悄悄泄漏。

用 `Data()` 返回的切片反查底层数组就能看到：

```go
b := bad.NewArray()
b.Add("a", "b", "c")
b.Remove("b")
fmt.Println("可见元素:", b.Data(), "底层数组:", b.Data()[:cap(b.Data())])
```

```text
可见元素: [a c] 底层数组: [a c c]
```

`Data()` 之外还看不到那个 `c`，但它确实活着。标准库的 `slices.Delete` 解决了这个问题，它在左移之后显式 `clear` 掉末尾的废弃位置（Go 1.22 起），方便 GC 回收。

顺带一提，`RemoveAll` 在循环里反复 `append` 左移是 O(n²) 的，删除一半元素时开销非常可观；批量删除的正确做法是用一次 `slices.DeleteFunc`（见后文「踩坑与边界」）。

## 用泛型实现：Array[T any]

泛型版的目标很明确：类型安全、不装箱、不 panic、复用 `slices` 包的成熟实现。先看数据结构和增删查部分：

```go
package good

import "slices"

// Array 是类型安全的动态数组，底层直接复用切片的扩容策略。
type Array[T any] struct {
	data []T
}

// New 创建一个空数组。
func New[T any]() *Array[T] {
	return &Array[T]{}
}

// Of 用给定元素创建一个数组，会复制一份底层数据。
func Of[T any](elems ...T) *Array[T] {
	return &Array[T]{data: slices.Clone(elems)}
}

// Add 追加一个或多个元素。
func (a *Array[T]) Add(elems ...T) {
	a.data = append(a.data, elems...)
}

// Len 返回元素个数。
func (a *Array[T]) Len() int { return len(a.data) }

// Cap 返回当前容量，便于观察扩容行为。
func (a *Array[T]) Cap() int { return cap(a.data) }

// At 返回下标 i 处的元素，越界会 panic。
func (a *Array[T]) At(i int) T { return a.data[i] }

// Set 修改下标 i 处的元素。
func (a *Array[T]) Set(i int, v T) { a.data[i] = v }

// RemoveAt 删除下标 i 处的元素并返回它。
func (a *Array[T]) RemoveAt(i int) T {
	v := a.data[i]
	a.data = slices.Delete(a.data, i, i+1)
	return v
}

// Remove 删除第一个满足 eq 的元素，返回是否真的删除了。
func (a *Array[T]) Remove(eq func(T) bool) bool {
	i := slices.IndexFunc(a.data, eq)
	if i < 0 {
		return false
	}
	a.data = slices.Delete(a.data, i, i+1)
	return true
}

// Contain 判断是否存在满足 eq 的元素。
func (a *Array[T]) Contain(eq func(T) bool) bool {
	return slices.ContainsFunc(a.data, eq)
}
```

这里有一个设计取舍需要解释：`T any` 意味着 `T` 可能是不可比较的切片、map，所以 `Remove`/`Contain` 不能直接用 `==`，改成让调用方传一个判断函数 `eq func(T) bool`。另一个选择是把约束收紧成 `Array[T comparable]`，这样 `Contain(v T)` 可以直接用 `==`，调用方写起来更短；代价是数组不能再存 `[]string`、`map`、函数等类型。

标准库 `slices` 包自己也是这么分裂的：`slices.Contains`、`slices.Index` 要求 `E comparable`；`slices.ContainsFunc`、`slices.IndexFunc` 接受任意 `E`，代价是多一层函数调用。我的建议是：**只有当元素的相等性可以用 `==` 表达时才考虑 `comparable` 约束，否则用谓词版本**；如果你确实不需要存不可比较类型，用 `comparable` 约束更省心。后文基准测试会看到，`slices.Contains` 与 `slices.ContainsFunc` 的开销几乎一致，谓词版本不用担心性能。

接下来是遍历、变换和排序：

```go
// ForEach 按顺序遍历所有元素。
func (a *Array[T]) ForEach(f func(T)) {
	for _, v := range a.data {
		f(v)
	}
}

// Filter 返回由满足 f 的元素组成的新数组。
func (a *Array[T]) Filter(f func(T) bool) *Array[T] {
	out := &Array[T]{data: make([]T, 0, len(a.data))}
	for _, v := range a.data {
		if f(v) {
			out.data = append(out.data, v)
		}
	}
	return out
}

// Map 对每个元素做同类型变换，返回新数组。
// Go 的方法不能引入新的类型参数，想换成另一种元素类型要用包级函数 MapTo。
func (a *Array[T]) Map(f func(T) T) *Array[T] {
	out := &Array[T]{data: make([]T, 0, len(a.data))}
	for _, v := range a.data {
		out.data = append(out.data, f(v))
	}
	return out
}

// Sort 用给定比较函数原地排序。
// 对可排序的 T，直接传 cmp.Compare[T] 即可。
func (a *Array[T]) Sort(cmp func(T, T) int) {
	slices.SortFunc(a.data, cmp)
}

// Clear 清空所有元素并保留容量，同时把元素置零让 GC 能回收引用。
func (a *Array[T]) Clear() {
	clear(a.data)
	a.data = a.data[:0]
}
```

`Filter` 的写法有一个关键细节：满足条件时追加的是 `v` 本身，而不是判断结果。`Map` 是个值得注意的限制——**Go 的方法不允许引入新的类型参数**，所以方法版 `Map` 只能做 `T → T` 的变换。想从 `[]int` 映射出 `[]string`，得用包级泛型函数：

```go
// MapTo 把数组元素映射成另一种类型，弥补方法无法新增类型参数的缺口。
func MapTo[T, U any](a *Array[T], f func(T) U) *Array[U] {
	out := &Array[U]{data: make([]U, 0, len(a.data))}
	for _, v := range a.data {
		out.data = append(out.data, f(v))
	}
	return out
}
```

调用示例（对应测试代码，已实际运行）：

```go
a := good.Of(3, 1, 2)
a.Add(5, 4)
a.Sort(cmp.Compare[int])                  // [1 2 3 4 5]
even := a.Filter(func(v int) bool { return v%2 == 0 }) // [2 4]
sq := even.Map(func(v int) int { return v * v })       // [4 16]
strs := good.MapTo(sq, func(v int) string { return fmt.Sprint(v) }) // ["4" "16"]
```

前面朴素实现的 `Sort` 用的是需要交换下标的 `sort.Slice`；泛型版改用 `slices.SortFunc` 加比较函数，配合 `cmp.Compare`、`cmp.Or` 可以很自然地组合出多字段排序，比如先按年龄再按名字：

```go
a.Sort(func(x, y user) int {
	return cmp.Or(cmp.Compare(x.age, y.age), cmp.Compare(x.name, y.name))
})
```

## 实测：Array[T] 到底比 []T 慢多少

跑 `go test -bench=. -benchmem -count=3 ./good/`，机器是 Apple M1 Pro、Go 1.25.9。取三次中的代表值：

```text
BenchmarkAddIntSlice-8            2813064    395.3 ns/op   2040 B/op   8 allocs/op
BenchmarkAddIntArray-8            2310624    530.5 ns/op   2040 B/op   8 allocs/op
BenchmarkAddIntAnyArray-8         1000000     1171 ns/op   4464 B/op   8 allocs/op
BenchmarkContainsSlice-8          3637586    355.2 ns/op      0 B/op   0 allocs/op
BenchmarkContainsSliceFunc-8      3526300    340.3 ns/op      0 B/op   0 allocs/op
BenchmarkContainsGenericArray-8   3586576    335.1 ns/op      0 B/op   0 allocs/op
BenchmarkContainsAnyArray-8        464932     2602 ns/op      0 B/op   0 allocs/op
```

基准代码就是同一个操作分别用三种载体实现，每个 op 向 `int` 数组追加 100 个元素，或者在前 1000 个元素的数组里查找 `999`：

```go
const benchN = 100 // 每个 op 追加/查找的规模

func BenchmarkAddIntSlice(b *testing.B) {
	for i := 0; i < b.N; i++ {
		var s []int
		for j := 0; j < benchN; j++ {
			s = append(s, j)
		}
		_ = s
	}
}

func BenchmarkAddIntArray(b *testing.B) {
	for i := 0; i < b.N; i++ {
		var a good.Array[int]
		for j := 0; j < benchN; j++ {
			a.Add(j)
		}
	}
}

func BenchmarkAddIntAnyArray(b *testing.B) {
	for i := 0; i < b.N; i++ {
		a := bad.NewArray()
		for j := 0; j < benchN; j++ {
			a.Add(j)
		}
	}
}
```

三个结论都能从数字里直接读出来：

1. **泛型版几乎免费**。`Array[int].Add` 每次追加约 5.3 ns，裸切片约 4.0 ns，多出来的部分主要是方法调用的内联边界；两者内存分配完全一样（2040 B/op，8 次）。
2. **`[]any` 版贵一倍以上**。每次追加约 12 ns，内存 4464 B/op——每个元素在切片里占一个 16 字节的 interface（类型指针 + 数据指针两个机器字），体积是 `int` 的两倍，比较时还要走接口比较。
3. **查找的差距更夸张**。泛型版 335 ns 与 `slices.ContainsFunc`（340 ns）、`slices.Contains`（355 ns）在同一水平；`[]any` 版 2602 ns，慢了约 7.8 倍，因为每次比较都是接口比较，且无法被编译器特化。

顺带回答前面的悬念：`slices.Contains` 和 `slices.ContainsFunc` 的差距在噪声范围内（355 vs 340 ns），所以谓词版本并不用担心"多一层函数调用"的开销，`ContainsFunc` 的闭包在测试循环里也能被内联处理。

## 踩坑与边界

1. **`Data()` 会把内部切片暴露出去**。它和数组共享底层数组，调用方 `append` 或改元素会直接影响容器。安全的做法是提供 `Clone()` 返回副本：

   ```go
   // Clone 返回内部切片的副本，适合对外暴露。
   func (a *Array[T]) Clone() []T { return slices.Clone(a.data) }
   ```

2. **`slices.Delete` 是原地修改，不是拷贝**。它会改写传入切片的底层数组，并返回缩短后的切片。如果别的地方还持有旧切片，会看到"元素被搬走但长度没变"的中间状态。删除前想保留原数据，先 `slices.Clone`。

3. **`Clear()` 有两种语义**。朴素写法 `a.data = nil`：引用全部释放、内存归还，但下次 `Add` 从零扩容；泛型版用 `clear` + `a.data[:0]`：元素置零（引用可回收）但保留容量，适合反复复用的池化场景。按用途选，不要无脑只认一种。

4. **别用循环逐个 `Remove`**。每次删除都要左移，n 个元素最坏 O(n²)。批量删除用一次遍历构造结果，或 `slices.DeleteFunc`。

5. **`comparable` 约束会改变 API 形状**。`Array[T comparable]` 写起来短，但没法存 `[]byte`、map、函数；`Array[T any]` 灵活，但相等性要由调用方提供。这不是谁对谁错，而是容器的使用场景决定约束。

## 总结

- Go 切片本身就是动态数组：`ptr + len + cap`，容量不足时按"小于 256 翻倍、之后约 1.25 倍并向上取尺寸类"的规则扩容，日常开发直接用 `[]T` 加 `slices` 包即可，不需要自造容器。
- `[]any` 写法有三个典型问题：`Contain` 对 slice/map/func 做 `==` 会 panic；`Filter` 把 `bool` 塞进了结果；删除不清空底层数组尾部，造成引用残留。前两个是纯 bug，第三个是容易埋很久的内存问题。
- 泛型版 `Array[T any]` 用 `slices.ContainsFunc`/`IndexFunc` 接受任意类型，用 `slices.SortFunc` + `cmp` 组合排序，用 `slices.Delete` 保证删除时清空废弃槽位；需要跨类型映射时用包级函数 `MapTo`，因为方法不能引入新的类型参数。
- 实测数据说明：`[]any` 的装箱成本让追加比泛型版慢约 2.2 倍、查找慢约 7.8 倍；泛型 `Array[T]` 与裸切片相差约 34%，性能不是选择它的理由，类型安全和 API 组织才是。
