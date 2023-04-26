---
title: golang自定义动态数组
date: 2022-04-29 23:18:30
author: ws
description: 自定义动态数组
categories: 数据结构
tags: [golang, 数据结构]
cover:
---

# golang自定义动态数组

## 完整代码

```go
package array

import (
	"sort"
)

type Array struct {
	data []interface{}
}

func NewArray() *Array {
	return new(Array)
}

func (a *Array) Add(elems ...interface{}) {
	a.data = append(a.data, elems...)
}

func (a *Array) Remove(e interface{}) {
	d := a.data
	cnt := len(d)
	for i := 0; i < cnt; i++ {
		if d[i] == e {
			d = append(d[:i], d[i+1:]...)
			break
		}
	}
	a.data = d
}

func (a *Array) RemoveAll(e interface{}) {
	d := a.data
	cnt := len(d)
	for i := 0; i < cnt; i++ {
		if d[i] == e {
			d = append(d[:i], d[i+1:]...)
			continue
		}
	}
	a.data = d
}

func (a *Array) Contain(e interface{}) bool {
	d := a.data
	cnt := len(d)
	for i := 0; i < cnt; i++ {
		if d[i] == e {
			return true
		}
	}
	return false
}

func (a *Array) Count() int {
	return len(a.data)
}

func (a *Array) ForEach(f func(e interface{})) {
	d := a.data
	cnt := len(d)
	for i := 0; i < cnt; i++ {
		f(d[i])
	}
}

func (a *Array) Clear() {
	a.data = nil
}

func (a *Array) Data() []interface{} {
	return a.data
}

func (a *Array) Sort(less func(i, j int) bool) {
	sort.Slice(a.data, less)
}

func (a *Array) Filter(f func(e interface{}) bool) *Array {
	newArr := NewArray()
	d := a.data
	cnt := len(d)
	for i := 0; i < cnt; i++ {
		if f(d[i]) {
			newArr.Add(d[i])
		}
	}
	return newArr
}
```

## 测试代码

```go
package array

import (
	"log"
	"testing"
)

var arr = NewArray()

func TestAdd(t *testing.T) {
	arr.Add(1, 2, 3, 4, 5)
	if arr.Count() != 5 {
		t.Fatal("arr count should equal 5")
	}
	arr.ForEach(func(e interface{}) { e = e.(int) * 2; log.Println(e) })
}

func TestRemove(t *testing.T) {
	arr.Add(1, 2, 3, 4, 5)
	arr.Remove(2)
	arr.Remove(1)
	if arr.Count() != 3 {
		t.Fatal("arr count should equal 3")
	}
	arr.ForEach(func(e interface{}) { log.Println(e) })
}

func TestClear(t *testing.T) {
	arr.Add(1, 2, 3, 4, 5)
	arr.Clear()
	if arr.Count() != 0 {
		t.Fatal("arr count should equal 0")
	}
	arr.ForEach(func(e interface{}) { log.Println(e) })
}

func TestSort(t *testing.T) {
	arr.Add(5, 3, 1, 2, 4)
	arr2 := NewArray()
	arr2.Add(1, 2, 3, 4, 5)
	arr.Sort(func(i, j int) bool {
		return arr.Data()[i].(int) < arr.Data()[j].(int)
	})
	d1 := arr.Data()
	d2 := arr2.Data()
	for i := 0; i < 5; i++ {
		if d1[i] != d2[i] {
			t.FailNow()
		}
	}
}

func TestFilter(t *testing.T) {
	arr.Add(1, 2, 3, 4, 5)
	a2 := arr.Filter(func(e interface{}) bool {
		return e.(int) > 3
	}).Filter(func(e interface{}) bool {
		return e.(int) > 4
	})
	if a2.data[0] != 5 {
		t.FailNow()
	}
}
```