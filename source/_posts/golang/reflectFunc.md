---
title: Go 反射调用函数：reflect.Value.Call 的用法与坑
date: 2022-12-13 22:00:30
updated: 2026-10-09
author: ws
description: 从动态调用讲到可变参数、nil 处理与性能代价
categories: ["Go"]
tags: ["Go", "反射"]
cover:
---

## 引言

插件式设计、RPC 框架、依赖注入容器、按配置分发命令——这些场景都有同一个需求：**在运行期按名字调用一个函数或方法**。Go 的 `reflect.Value.Call` 能做到，但它和普通调用是两套规则：参数要逐个装箱、错误藏在 panic 里、可变参数有额外语义、性能差几百倍。这篇文章先看一种常见的错误写法——它看起来四平八稳，却会在好几类输入上直接 panic；然后一步步给出先校验再调用的完整实现，最后用真实 benchmark 说明反射的代价和替代方案。适合写过 Go、想在框架层用反射但不想被它坑的读者。

## 反射三件套：Type、Value、Kind

反射的核心是三个概念，先分清它们，后面的 API 才不会用错：

```text
接口值（any）
+-------------------+
| 动态类型 type     | -- reflect.TypeOf -->  Type：具体类型。方法集、字段、Kind 都在这
+-------------------+
| 数据 data         | -- reflect.ValueOf --> Value：数据本身。读到值、改值靠它
+-------------------+
                                 |
                                 +-- Value.Kind() --> Kind：底层类别（Ptr、Struct、Int、Slice ...）
```

拿一个自定义类型举例，下面这段代码实际跑出来的结果是：

```go
type Celsius float64

c := Celsius(36.5)
fmt.Println("TypeOf:", reflect.TypeOf(c))           // main.Celsius
fmt.Println("ValueOf:", reflect.ValueOf(c))         // 36.5
fmt.Println("Kind:", reflect.ValueOf(c).Kind())     // float64
fmt.Println("Type.Name:", reflect.TypeOf(c).Name()) // Celsius
fmt.Println("Type.Kind:", reflect.TypeOf(c).Kind()) // float64
```

`Type` 回答“它是什么类型”，`Value` 装着“它的数据”，`Kind` 是“它属于哪一类”。写反射代码时，判断分支用 `Kind`（`reflect.Int`、`reflect.String`……），做调用、取字段、查方法用 `Type`。

## 反射常用 API 速查

下面这张表把常用的类型与值方法集中列出，“注意点”一列是实际写代码时最容易踩坑的地方：

| 方法/函数 | 描述 | 注意点 |
| :--- | :--- | :--- |
| `TypeOf(i any) Type` | 返回值的动态类型 | `i` 为 nil 接口时返回 nil Type |
| `ValueOf(i any) Value` | 返回值的反射值 | `i` 为 nil 时返回无效 Value（`IsValid()==false`） |
| `New(t Type) Value` | 返回**指向** t 零值的指针 Value | 结果是 `*T`，要零值本身用 `reflect.Zero(t)` 或 `.Elem()` |
| `Value.Elem() Value` | 指针/接口解引用后的值 | nil 指针得到无效 Value |
| `Value.Kind() Kind` | 底层种类，不是具体类型 | `type Celsius float64` 的 Kind 是 `float64` |
| `Value.Type() Type` | 具体类型 | 和 `Kind()` 配合使用 |
| `Value.Interface() any` | 转回接口值 | 来自未导出字段的 Value 调用会 panic |
| `Value.IsValid() bool` | 值是否有效 | 零值 Value（如 `ValueOf(nil)`）返回 false |
| `Value.NumField() int` | 结构体字段数 | 仅 Struct 可用 |
| `Value.Field(i int) Value` | 第 i 个字段 | 未导出字段可读但不可 Set、不可 Interface |
| `Value.MethodByName(name string) Value` | 取绑定好接收者的方法值 | **只有一个返回值**，找不到时返回无效 Value；只看导出方法 |
| `Value.Call(args []Value) []Value` | 调用函数 | 参数不匹配直接 panic；变参可传散开的多个值 |
| `Value.CanSet() bool` | 值是否可修改 | 要求可寻址且可导出 |
| `Value.Set(value Value)` | 修改值内容 | 类型必须可赋值给目标 |
| `Type.NumMethod() int` | 类型的方法数 | 非接口类型只统计导出方法 |
| `Type.Method(i int) Method` | 第 i 个方法 | 非接口类型只含导出方法；`Method.Func` 第一个参数是接收者 |
| `Type.MethodByName(name string) (Method, bool)` | 按名字找方法 | 返回的是描述信息，`Func` 含接收者 |
| `Type.Field(i int) StructField` | 第 i 个字段 | 字段描述含 Tag、Offset |
| `Type.NumField() int` | 字段数 | 仅 Struct 可用 |
| `Type.Kind() Kind` | 底层种类 | 同 `Value.Kind()` |
| `Type.Name() string` | 类型名 | `*T`、`[]int` 等非定义类型返回空串 |
| `Type.PkgPath() string` | 包路径 | 预声明类型（int、string）返回空串 |
| `Type.String() string` | 类型字符串 | 可能用短包名，不等价于类型身份 |
| `FuncOf(in, out []Type, variadic bool) Type` | 构造函数类型 | 用于和 `MakeFunc` 配合动态造函数 |

两条边界值得单独记住：`TypeOf(nil) == nil`，而 `ValueOf(nil).IsValid() == false`，实测都为真。

## 常见的错误写法：四类会 panic 的输入

下面这段 `CallFunc` 的思路没错：拿类型、装参数、调用、拆返回值，四步齐全。问题出在“装参数”这一步完全没有校验：

```go
func CallFunc(fn interface{}, arguments ...interface{}) ([]interface{}, error) {
	var args []reflect.Value
	rv := reflect.ValueOf(fn)
	if rv.Kind() != reflect.Func {
		return nil, fmt.Errorf("not a function, kind: %s", rv.Kind())
	}
	for numArgs, rt, i := len(arguments), rv.Type(), 0; i < numArgs; i++ {
		if arguments[i] == nil {
			args = append(args, reflect.New(rt.In(i)).Elem())
		} else {
			args = append(args, reflect.ValueOf(arguments[i]))
		}
	}
	res := rv.Call(args)
	interfaces := make([]interface{}, len(res))
	for i, v := range res {
		interfaces[i] = v.Interface()
	}
	return interfaces, nil
}
```

逐段看：`rv.Kind() != reflect.Func` 拦住了非函数；`reflect.ValueOf(arguments[i])` 做了装箱；nil 走 `reflect.New(rt.In(i)).Elem()`。把这类写法放进一个可运行的小程序，逐种输入跑一遍，结果是：

```text
CallFunc(Sum, 1, 2, 3)              => [6], err=<nil>            ← 散开的变参其实能跑通
CallFunc(Sum, 1, nil)               => panic: runtime error: index out of range [1] with length 1
CallFunc(Sum, []int{1, 2, 3})       => panic: reflect: cannot use []int as type int in Call
CallFunc(Sum, "1")                  => panic: reflect: cannot use string as type int in Call
AnyAdd(int64(1))                    => 0, err=param invalid, index: 0
AnyAdd([]string{"a"})               => panic: reflect: call of reflect.Value.Int on string Value
CallFuncByName(nil, "Do")           => panic: reflect: call of reflect.Value.MethodByName on zero Value
CallFunc(nilFunc, 1)                => panic: reflect.Value.Call: call of nil function
```

这里有一个反直觉的事实：**可变参数函数用 `Call` 传散开的多个参数并不会 panic**——`reflect` 会自动把它们打包成变参切片，这也是上面第一行能返回 6 的原因。真正会炸的是两类变参场景：

1. **变参位置出现 nil**：循环里用 `rt.In(i)` 取形参类型，但 `func(...int)` 的 `NumIn()` 是 1，`i=1` 时直接越界 panic；
2. **把切片当成变参整体传**：`func(...int)` 对应的单个实参类型是 `int`，传给 `Call` 的却是 `[]int`，类型校验失败。

第二类场景想要“整片传参”，用的是另一个方法 `CallSlice`：

```go
f := reflect.ValueOf(Sum) // func(...int) int
// CallSlice：最后一个参数必须是变参切片本身
fmt.Println(f.CallSlice([]reflect.Value{reflect.ValueOf([]int{1, 2, 3})})[0].Interface())
fmt.Println(f.Call([]reflect.Value{reflect.ValueOf(1), reflect.ValueOf(2), reflect.ValueOf(3)})[0].Interface())
```

这段代码两条输出都是 6：`CallSlice` 整片传参，`Call` 散开传参。判断函数是否变参用 `f.Type().IsVariadic()`；变参形参的“单个实参类型”是 `In(NumIn-1).Elem()`，不是 `In(NumIn-1)` 本身。另外三颗雷：参数个数/类型不匹配时 `Call` 直接 panic（不是返回 error）；未初始化的接口调 `MethodByName` 会 panic；函数变量本身为 nil 时 `Call` 也会 panic。推荐的实现会在调用前把它们全部转成普通 error。

## 推荐的实现：先校验、再调用

设计目标：参数错误在调用前拦下并返回 error；支持可变参数；nil 有明确语义；同时保留一层 recover 兜底，但**只把 reflect 自己抛的 panic 转成 error**，被调函数内部的 panic 继续向上抛。

先写两个入口函数，结构一目了然：参数交给 `buildArgs` 校验，调用交给 `invoke` 执行：

```go
// CallFunc 动态调用任意函数 fn，args 按位置对应形参。
// 可变参数 fn(...T)：args 尾部多出的值会作为 T 传入。
// 实参为 nil 时按对应形参类型的零值传入（*T 得到 nil 指针，int 得到 0）。
func CallFunc(fn any, args ...any) ([]any, error) {
	rv := reflect.ValueOf(fn)
	if rv.Kind() != reflect.Func {
		return nil, fmt.Errorf("CallFunc: %T 不是函数（Kind=%s）", fn, rv.Kind())
	}
	argVals, err := buildArgs(rv.Type(), args)
	if err != nil {
		return nil, err
	}
	return invoke(rv, argVals)
}

// CallFuncByName 调用 receiver 上名为 name 的导出方法，参数规则同 CallFunc。
// receiver 必须提供方法的实际接收者：指针方法要传指针。
func CallFuncByName(receiver any, name string, args ...any) ([]any, error) {
	rv := reflect.ValueOf(receiver)
	if !rv.IsValid() {
		return nil, errors.New("CallFuncByName: receiver 为 nil（未初始化的接口）")
	}
	m := rv.MethodByName(name)
	if !m.IsValid() {
		return nil, fmt.Errorf("CallFuncByName: %T 上不存在可导出的方法 %q", receiver, name)
	}
	argVals, err := buildArgs(m.Type(), args)
	if err != nil {
		return nil, fmt.Errorf("CallFuncByName: 方法 %s %w", name, err)
	}
	return invoke(m, argVals)
}
```

参数构造是整套实现的核心。`buildArgs` 做三件事：校验个数、处理 nil、校验类型（可赋值优先，可转换兜底，兼容 `type MyInt int` 这类命名类型）：

```go
// buildArgs 在真正调用之前校验参数个数和类型，
// 把 reflect.Call 本来会 panic 的输入转换成普通 error。
func buildArgs(ft reflect.Type, args []any) ([]reflect.Value, error) {
	arity := ft.NumIn()
	if ft.IsVariadic() {
		arity-- // 可变参数是一个切片形参，但调用方可以传 0..n 个元素
		if len(args) < arity {
			return nil, fmt.Errorf("参数个数不足：至少需要 %d 个，实际 %d 个", arity, len(args))
		}
	} else if len(args) != arity {
		return nil, fmt.Errorf("参数个数不匹配：需要 %d 个，实际 %d 个", arity, len(args))
	}

	vals := make([]reflect.Value, 0, len(args))
	for i, arg := range args {
		want := paramTypeAt(ft, i)
		if arg == nil {
			// reflect.ValueOf(nil) 得到的是无效 Value（Kind=Invalid），
			// 塞给 Call 会 panic；这里用形参类型的零值代替。
			// reflect.Zero(want) 与 reflect.New(want).Elem() 对调用等价，
			// 区别只是后者可寻址、前者不可寻址。
			vals = append(vals, reflect.Zero(want))
			continue
		}
		v := reflect.ValueOf(arg)
		if !v.Type().AssignableTo(want) {
			if v.Type().ConvertibleTo(want) {
				v = v.Convert(want) // 兼容命名类型，如 type MyInt int
			} else {
				return nil, fmt.Errorf("第 %d 个参数类型不匹配：需要 %s，得到 %s", i, want, v.Type())
			}
		}
		vals = append(vals, v)
	}
	return vals, nil
}

// paramTypeAt 返回第 i 个形参的“单个实参”类型。
// 可变参数位置要取切片元素类型：f(...int) 的 In(0) 是 []int，
// 而调用方在位置 0 上传的每个值都是 int。
func paramTypeAt(ft reflect.Type, i int) reflect.Type {
	if ft.IsVariadic() && i >= ft.NumIn()-1 {
		return ft.In(ft.NumIn() - 1).Elem()
	}
	return ft.In(i)
}
```

注意 `paramTypeAt`：它同时消除了错误写法里“变参里有 nil 就越界”的 panic，也保证类型校验针对的是变参元素类型而不是切片类型。这里选择“把变参展开成一个个 `reflect.Value`”而不是用 `CallSlice`，因为调用方的 API 是 `args ...any`（0 到 n 个散值），展开后 `Call` 天然支持；如果你的 API 需要“传一个空切片”和“传一个 nil 元素”的区分，再考虑 `CallSlice`。

关于 nil：为什么用 `reflect.Zero(want)`（而不是 `reflect.ValueOf(nil)`）？因为 `reflect.ValueOf(nil)` 得到的是一个无效 Value，`Kind()` 是 `Invalid`，交给 `Call` 必然 panic。`reflect.Zero(t)` 给的是“类型 t 的零值”：指针形参得到 nil，int 形参得到 0。语义上要明确：**调用方传 nil，对 `int` 形参来说就是传 0**，这未必总是你想要的，框架层最好在文档里写清楚。

最后是执行与兜底。区分两种 panic 是关键：

```go
// invoke 执行反射调用，并兜底把 reflect 自己抛出的 panic 转成 error。
// 被调函数内部的 panic 会继续向上抛，不在这里吞掉。
func invoke(fn reflect.Value, args []reflect.Value) (ret []any, err error) {
	defer func() {
		if r := recover(); r != nil {
			if isReflectPanic(r) {
				err = fmt.Errorf("反射调用被拒绝：%v", r)
				return
			}
			panic(r)
		}
	}()
	results := fn.Call(args)
	ret = make([]any, len(results))
	for i, v := range results {
		ret[i] = v.Interface()
	}
	return ret, nil
}

// isReflectPanic 区分“reflect 包自己拒绝调用”和“被调函数内部 panic”。
// reflect 的 panic 有几种形态：*reflect.ValueError、
// "reflect: ..."（参数不匹配）以及 "reflect.Value.Call: call of nil function"（函数值为 nil）。
func isReflectPanic(r any) bool {
	if _, ok := r.(*reflect.ValueError); ok {
		return true
	}
	s, ok := r.(string)
	if !ok {
		return false
	}
	return strings.HasPrefix(s, "reflect:") || strings.HasPrefix(s, "reflect.")
}
```

为什么不全盘 recover？因为那样会把被调函数自己的 panic（比如除零、nil 解引用）也变成“调用失败”，掩盖真正的 bug。`go run` 实测：调用 `Div(1, 0)` 时 `main` 里能 recover 到 `runtime error: integer divide by zero`，说明被调函数的 panic 被继续向上传递了。

## CallFuncByName 的四个边界

1. **未导出方法找不到**。`Account.hidden` 是小写方法，`MethodByName(“hidden“)` 返回无效 Value，推荐的实现会返回 error：`*main.Account 上不存在可导出的方法 ”hidden”`。
2. **方法集决定成败**。`Deposit` 是指针接收者方法，`(*Account).MethodByName` 找得到，`Account.MethodByName` 找不到。对 `Account` 值调用报：`main.Account 上不存在可导出的方法 "Deposit"`。传 receiver 时要和方法的接收者类型一致，而且改状态本来也要传指针。
3. **未初始化接口**（`var empty any`）会走 `!rv.IsValid()` 分支，返回 `receiver 为 nil（未初始化的接口）`。不做任何校验的写法在这里会 panic：`reflect: call of reflect.Value.MethodByName on zero Value`。
4. **带类型的 nil 指针**是另一回事：`var acc *Account` 时方法找得到，调用时在 `Deposit` 内部 nil 解引用 panic。这是被调函数自身的 bug，`invoke` 不会吞，测试里用 recover 验证过。另外 `m := rv.MethodByName(name)` 返回的已经是**绑定好接收者**的方法值，`m.Type()` 不含 receiver，所以 `buildArgs(m.Type(), args)` 的参数个数和调用方给的一致，不需要手动处理 receiver。

## 返回值里的 error 要自己检查

反射调用成功只代表“调用发生了”，不代表业务成功。`Withdraw` 返回 `(int, error)` 时，结果切片是 `[]any{150, error}`：

```go
	res, err := CallFuncByName(acc, "Withdraw", 500)
	if err != nil { // 反射层面的失败（方法不存在、参数不匹配）才走这里
	}
	if bizErr, ok := res[len(res)-1].(error); ok && bizErr != nil {
		fmt.Println("业务失败:", bizErr)
	}
```

实测输出是 `Withdraw(500) => [150 余额不足：当前 150，尝试取出 500]`：余额没变，错误躺在第二个返回值里。框架里最好封装一个“把最后一个 error 返回值拆出来”的辅助函数，否则调用方很容易只看到 `err == nil` 就以为成功。

## AnyAdd：反射处理任意类型

先看一个更典型的错误示范。`AnyAdd` 想用反射把任意类型的参数相加，但这段实现有两个 bug：`Int64`/`Float64` 类型落到 `default` 报错，而 `[]string` 这类切片会在 `val.Index(i).Int()` 处 panic（对 string 元素调 `Int()`）。下面这段实现把数值类型分门别类，切片先检查元素类型，不支持的输入返回 error：

```go
// AnyAdd 把任意个数字相加，支持 int/float64/string/[]int 等：
// int 系按整型取，浮点按浮点取，string 解析成数字，切片/数组逐元素累加。
// nil 和不支持的类型的返回 error，而不是 panic。
func AnyAdd(args ...any) (float64, error) {
	var sum float64
	for i, arg := range args {
		v := reflect.ValueOf(arg)
		if !v.IsValid() {
			return 0, fmt.Errorf("第 %d 个参数为 nil", i)
		}
		switch v.Kind() {
		case reflect.Int, reflect.Int8, reflect.Int16, reflect.Int32, reflect.Int64:
			sum += float64(v.Int())
		case reflect.Uint, reflect.Uint8, reflect.Uint16, reflect.Uint32, reflect.Uint64:
			sum += float64(v.Uint())
		case reflect.Float32, reflect.Float64:
			sum += v.Float()
		case reflect.String:
			f, err := strconv.ParseFloat(strings.TrimSpace(v.String()), 64)
			if err != nil {
				return 0, fmt.Errorf("第 %d 个参数 %q 不是数字: %w", i, v.String(), err)
			}
			sum += f
		case reflect.Slice, reflect.Array:
			if v.Type().Elem().Kind() != reflect.Int {
				return 0, fmt.Errorf("第 %d 个参数是 %s，只支持元素为 int 的切片/数组", i, v.Type())
			}
			for j := 0; j < v.Len(); j++ {
				sum += float64(v.Index(j).Int())
			}
		default:
			return 0, fmt.Errorf("第 %d 个参数类型不支持: %s", i, v.Type())
		}
	}
	return sum, nil
}
```

实测：`AnyAdd(1, int64(2), 3.5, “4“, []int{5, 6})` 得到 `21.5`；`AnyAdd([]string{”a”})` 返回清晰 error 而不是 panic。但说实话，这个函数用 type switch 写更合适——反射在这里没有带来任何表达力，只带来了开销。

## 完整程序与真实输出

把上面的实现代码按顺序拼起来，加上入口函数，就是完整的 `main.go`（实现部分前文已给全，下面只补 `show` 和 `main`，完整约 300 行）：

```go
// show 把 (返回值, 错误) 打印成一行，方便对照本文输出。
func show(name string, res []any, err error) {
	if err != nil {
		fmt.Printf("%-46s => error: %v\n", name, err)
		return
	}
	fmt.Printf("%-46s => %v\n", name, res)
}

func main() {
	fmt.Println("== CallFunc：动态调用函数 ==")
	res, err := CallFunc(Sum, 1, 2, 3, 4)
	show("CallFunc(Sum, 1, 2, 3, 4)", res, err)
	res, err = CallFunc(fmt.Sprintf, "%s-%d", "gopher", 8)
	show(`CallFunc(fmt.Sprintf, "%s-%d", "gopher", 8)`, res, err)
	res, err = CallFunc(Join, "root", nil)
	show(`CallFunc(Join, "root", nil)`, res, err)
	res, err = CallFunc(Add, 1)
	show("CallFunc(Add, 1)", res, err)
	res, err = CallFunc(Add, 1, "2")
	show(`CallFunc(Add, 1, "2")`, res, err)
	fmt.Println()
	fmt.Println("== CallFuncByName：按方法名调用 ==")
	acc := &Account{Owner: "ws", Balance: 100}
	res, err = CallFuncByName(acc, "Deposit", 50)
	show("Deposit(50) on *Account", res, err)
	res, err = CallFuncByName(acc, "Withdraw", 500)
	show("Withdraw(500)", res, err)
	res, err = CallFuncByName(acc, "hidden")
	show("hidden（未导出方法）", res, err)
	res, err = CallFuncByName(Account{Owner: "ws"}, "Deposit", 1)
	show("Deposit on Account 值（指针接收者）", res, err)
	var empty any
	res, err = CallFuncByName(empty, "Deposit")
	show("Deposit on nil 接口", res, err)
	fmt.Println()
	fmt.Println("== AnyAdd：反射处理任意数字 ==")
	sum, err := AnyAdd(1, int64(2), 3.5, "4", []int{5, 6})
	fmt.Printf("AnyAdd(1, int64(2), 3.5, \"4\", []int{5,6}) = %v, err=%v\n", sum, err)
	if _, err := AnyAdd([]string{"a"}); err != nil {
		fmt.Println("AnyAdd([]string{\"a\"}) error:", err)
	}
	sum2, _ := AnyAddSwitch(1, int64(2), 3.5, "4", []int{5, 6})
	fmt.Println("AnyAddSwitch 同样输入 =", sum2)
	fmt.Println()
	fmt.Println("== 被调函数内部 panic：继续向上抛 ==")
	func() {
		defer func() { fmt.Println("main 里 recover 到:", recover()) }()
		_, _ = CallFunc(Div, 1, 0)
	}()
}
```

`go run .` 的真实输出：

```text
== CallFunc：动态调用函数 ==
CallFunc(Sum, 1, 2, 3, 4)                      => [10]
CallFunc(fmt.Sprintf, "%s-%d", "gopher", 8)    => [gopher-8]
CallFunc(Join, "root", nil)                    => [root:(nil)]
CallFunc(Add, 1)                               => error: 参数个数不匹配：需要 2 个，实际 1 个
CallFunc(Add, 1, "2")                          => error: 第 1 个参数类型不匹配：需要 int，得到 string

== CallFuncByName：按方法名调用 ==
Deposit(50) on *Account                        => [150]
Withdraw(500)                                  => [150 余额不足：当前 150，尝试取出 500]
hidden（未导出方法）                                  => error: CallFuncByName: *main.Account 上不存在可导出的方法 "hidden"
Deposit on Account 值（指针接收者）                    => error: CallFuncByName: main.Account 上不存在可导出的方法 "Deposit"
Deposit on nil 接口                              => error: CallFuncByName: receiver 为 nil（未初始化的接口）

== AnyAdd：反射处理任意数字 ==
AnyAdd(1, int64(2), 3.5, "4", []int{5,6}) = 21.5, err=<nil>
AnyAdd([]string{"a"}) error: 第 0 个参数是 []string，只支持元素为 int 的切片/数组
AnyAddSwitch 同样输入 = 21.5

== 被调函数内部 panic：继续向上抛 ==
main 里 recover 到: runtime error: integer divide by zero
```

`go vet ./...` 无输出（通过）；`go test -v .` 共 16 个用例全部通过，摘录如下：

```text
=== RUN   TestCallFuncVariadic
--- PASS: TestCallFuncVariadic (0.00s)
...
PASS
ok  	reflectcall	0.469s
```

## 性能实测：benchmark 对比

测试文件里放了三个关键基准：直接调用、只测 `Call` 本身（复用参数槽）、以及包含参数校验和装箱的完整 `CallFunc`（`BenchmarkCallFuncWrapper`，就是 `CallFunc(Add, i, i+1)`）：

```go
func BenchmarkDirectAdd(b *testing.B) {
	for i := 0; i < b.N; i++ {
		_ = Add(i, i+1)
	}
}

func BenchmarkReflectCall(b *testing.B) {
	fn := reflect.ValueOf(Add)
	a, c := 0, 1
	args := []reflect.Value{reflect.ValueOf(&a).Elem(), reflect.ValueOf(&c).Elem()} // 复用参数槽，只测 Call 本身
	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		a, c = i, i+1
		_ = fn.Call(args)
	}
}
```

`go test -bench 'DirectAdd|ReflectCall|CallFuncWrapper|AnyAdd' -benchmem -run '^$' .` 的真实结果（取一轮，本机 Go 1.25 / Apple Silicon）：

```text
BenchmarkDirectAdd-8         1000000000    0.3357 ns/op     0 B/op    0 allocs/op
BenchmarkReflectCall-8        8597887      141.5 ns/op     32 B/op    2 allocs/op
BenchmarkCallFuncWrapper-8    5098282      234.8 ns/op    111 B/op    5 allocs/op
BenchmarkAnyAddReflect-8     33076682       35.42 ns/op      0 B/op    0 allocs/op
BenchmarkAnyAddSwitch-8      51477106       23.27 ns/op      0 B/op    0 allocs/op
```

几个结论：

- 直接调用被编译器内联，0.3 ns/op 这个量级基本是循环的下界；`reflect.Call` 约 141 ns/op，**慢约 440 倍**，每次还有 32 字节、2 次堆分配（`[]reflect.Value` 和参数装箱）。
- 加上参数校验和 `[]any` 转换的 `CallFunc` 约 235 ns/op，**慢约 740 倍**，5 次分配。也就是说真正的业务函数如果只有几纳秒，反射的固定开销是它的几百倍。
- 同一段逻辑换成 type switch（`AnyAddSwitch`）只要 23 ns，比反射写法快约 50%，而且零分配、可读性更好。

## 什么时候不该用反射

**type switch**：类型集合封闭、逻辑简单时首选。上面 `AnyAddSwitch` 就是；新增类型加一个 `case`，编译器帮你检查拼写。
**泛型**（Go 1.18+）：类型集合能用约束表达时用它，性能和直接调用一样。比如批量求和：

```go
type Number interface{ ~int | ~int64 | ~float64 }

func SumG[T Number](nums ...T) T {
	var total T
	for _, n := range nums {
		total += n
	}
	return total
}
```

**代码生成**：类型集合开放、但调用点固定时（ORM 扫描、序列化、命令注册），用 `go generate` 在编译期生成类型断言代码，运行时没有反射开销。`encoding/json` 的 `Marshaler` 接口、`sql.Rows.Scan` 的调用方都是这个思路。
反射的合理位置是**装配期和低频路径**：注册路由、解析配置、构造容器、CLI 命令分发。这些路径每次进程只跑几十次，几百纳秒无所谓；而每请求、每行数据的热路径，一律避开 `Call`。

## 总结

- `Type`/`Value`/`Kind` 分别回答“是什么类型/装了什么数据/属于哪一类”，判断分支用 `Kind`，调用和取字段用 `Type`/`Value`。
- `reflect.Call` 的错误处理是 panic 而不是 error：参数个数、类型、变参形状、nil 函数、nil 接口都会炸。推荐方案的顺序是“提前校验（返回 error）+ recover 只兜 reflect 自身的 panic”。
- 变参三件事：`IsVariadic()` 判断；单个实参类型取 `In(NumIn-1).Elem()`；散开传参用 `Call`（reflect 自动打包），整片传参用 `CallSlice`。
- nil 实参的语义是“形参类型的零值”，`reflect.Zero(t)` 与 `reflect.New(t).Elem()` 对调用等价；`ValueOf(nil)` 是无效值，不能直接用。
- 返回值里的 `error` 不会自动变成函数的第二个返回值，调用方必须自己检查。
- 性能上反射比直接调用慢几百倍且带堆分配；能用 type switch、泛型或代码生成的地方，就别用反射。
