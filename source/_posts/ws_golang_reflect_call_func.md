---
title: golang中使用反射调用函数
date: 2022-12-13 22:00:30
author: ws
description: golang中使用reflect包动态调用函数
categories: golang
tags: [golang, reflect]
cover: false
---

## reflect包中的方法/函数介绍

| 方法/函数                                       | 描述                                                         |
| :---------------------------------------------- | :----------------------------------------------------------- |
| `TypeOf(i interface{}) Type`                    | 返回一个值的类型的反射值                                     |
| `ValueOf(i interface{}) Value`                  | 返回一个值的反射值                                           |
| `New(t Type) Value`                             | 返回类型 t 的零值的反射值                                    |
| `Value.Elem() Value`                            | 返回指针或接口指向的值的反射值                               |
| `Value.Kind() Kind`                             | 返回一个值的类型                                             |
| `Value.Type() Type`                             | 返回一个值的类型                                             |
| `Value.Interface() interface{}`                 | 返回一个值的接口类型                                         |
| `Value.IsValid() bool`                          | 判断一个值是否合法                                           |
| `Value.NumField() int`                          | 返回一个结构体类型的字段数                                   |
| `Value.Field(i int) Value`                      | 返回结构体类型的第 i 个字段的反射值                          |
| `Value.MethodByName(name string) (Value, bool)` | 返回一个方法的反射值，根据方法名查找                         |
| `Value.Call(args []Value) []Value`              | 调用函数，args 为函数的参数，返回 Value 的切片               |
| `Value.CanSet() bool`                           | 判断一个值是否可以修改                                       |
| `Value.Set(value Value)`                        | 修改一个值的内容                                             |
| `Type.NumMethod() int`                          | 返回一个类型的方法数                                         |
| `Type.Method(i int) Method`                     | 返回一个类型的第 i 个方法的反射值                            |
| `Type.MethodByName(name string) (Method, bool)` | 返回一个类型的方法的反射值，根据方法名查找                   |
| `Type.Field(i int) StructField`                 | 返回一个结构体类型的第 i 个字段的反射值                      |
| `Type.NumField() int`                           | 返回一个结构体类型的字段数                                   |
| `Type.Kind() Kind`                              | 返回一个类型的种类                                           |
| `Type.Name() string`                            | 返回一个类型的名称                                           |
| `Type.PkgPath() string`                         | 返回一个类型的包路径                                         |
| `Type.String() string`                          | 返回一个类型的字符串表示                                     |
| `FuncOf(in, out []Type, variadic bool) Type`    | 返回一个函数类型的反射值，参数 in 是入参类型，out 是出参类型 |

## 动态调用函数

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

## 根据方法名调用函数

```go
func CallFuncByName(obj interface{}, methodName interface{}, arguments ...interface{}) ([]interface{}, error) {
	var args []reflect.Value
	f := reflect.ValueOf(obj).MethodByName(methodName.(string))
	if !f.IsValid() {
		return nil, fmt.Errorf("method not found, current: %s", methodName)
	}
	for numArgs, rt, i := len(arguments), f.Type(), 0; i < numArgs; i++ {
		if arguments[i] == nil {
			args = append(args, reflect.New(rt.In(i)).Elem())
		} else {
			args = append(args, reflect.ValueOf(arguments[i]))
		}
	}
	res := f.Call(args)
	interfaces := make([]interface{}, len(res))
	for i, v := range res {
		interfaces[i] = v.Interface()
	}
	return interfaces, nil
}
```

## 完整代码

```go
package main

import (
	"fmt"
	"reflect"
	"strconv"
)

type DynamicFunc struct {
	Name string
	Args []interface{}
}

func (DynamicFunc) AddFunc(args ...int) (res int) {
	for _, v := range args {
		res += v
	}
	return
}

func CallFuncByName(obj interface{}, methodName interface{}, arguments ...interface{}) ([]interface{}, error) {
	var args []reflect.Value
	f := reflect.ValueOf(obj).MethodByName(methodName.(string))
	if !f.IsValid() {
		return nil, fmt.Errorf("method not found, current: %s", methodName)
	}
	for numArgs, rt, i := len(arguments), f.Type(), 0; i < numArgs; i++ {
		if arguments[i] == nil {
			args = append(args, reflect.New(rt.In(i)).Elem())
		} else {
			args = append(args, reflect.ValueOf(arguments[i]))
		}
	}
	res := f.Call(args)
	interfaces := make([]interface{}, len(res))
	for i, v := range res {
		interfaces[i] = v.Interface()
	}
	return interfaces, nil
}

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

func AnyAdd(args ...interface{}) (res int, err error) {
	for i, param := range args {
		val := reflect.ValueOf(param)
		switch val.Kind() {
		case reflect.String:
			iVal, err := strconv.Atoi(val.String())
			if err != nil {
				return 0, err
			}
			res += iVal
		case reflect.Int:
			res += int(val.Int())
		case reflect.Array, reflect.Slice:
			cnt := val.Len()
			for i := 0; i < cnt; i++ {
				res += int(val.Index(i).Int())
			}
		default:
			return 0, fmt.Errorf("param invalid, index: %d", i)
		}
	}
	return
}

func TestCallFuncByName() {
	// 不存在的函数调用的检验
	df := DynamicFunc{Name: "FmtFunc"}
	_, err := CallFuncByName(df, df.Name, df.Args...)
	fmt.Println("CallFuncByName FmtFunc Expect:", "method not found, current: FmtFunc")
	fmt.Println("CallFuncByName FmtFunc Actual:", err)
	fmt.Println()
	// 通过名称调用函数的检验
	df = DynamicFunc{Name: "AddFunc", Args: []interface{}{1, 2, 3, 4, 5}}
	fmt.Println("CallFuncByName AddFunc Args:", df.Args)
	res, err := CallFuncByName(df, df.Name, df.Args...)
	if err != nil {
		fmt.Println(err)
        return
	}
	fmt.Println("CallFuncByName AddFunc Expect:", 15)
	fmt.Println("CallFuncByName AddFunc Actual:", res, "error:", err)
	fmt.Println()
}

func TestCallFunc() {
	// 动态调用函数的检验
	res, err := CallFunc(AnyAdd, 1, "2", []int{3, 6, 9})
	fmt.Println("CallFunc AnyAdd Args:", 1, "2", []int{3, 6, 9})
	if err != nil {
		fmt.Println(err)
        return
	}
	fmt.Println("CallFunc AnyAdd Expect:", 21)
	fmt.Println("CallFunc AnyAdd Actual:", res, "error:", err)
}

func main() {
	TestCallFuncByName()
	TestCallFunc()
}
```

## 输出结果

> CallFuncByName FmtFunc Expect: method not found, current: FmtFunc
> CallFuncByName FmtFunc Actual: method not found, current: FmtFunc
>
> CallFuncByName AddFunc Args: [1 2 3 4 5]
> CallFuncByName AddFunc Expect: 15
> CallFuncByName AddFunc Actual: [15] error: <nil>
>
> CallFunc AnyAdd Args: 1 2 [3 6 9]
> CallFunc AnyAdd Expect: 21
> CallFunc AnyAdd Actual: [21 <nil>] error: <nil>