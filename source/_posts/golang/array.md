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

## 测试代码

```go
package array

import (
	"testing"

	"github.com/stretchr/testify/assert"
)

func TestArray_Add(t *testing.T) {
	type args struct {
		elems []any
	}
	tests := []struct {
		name     string
		a        *Array
		args     args
		expected []any
	}{
		{name: "int", a: &Array{data: []any{1, 2}}, args: args{elems: []any{3}}, expected: []any{1, 2, 3}},
		{name: "string", a: &Array{data: []any{"1"}}, args: args{elems: []any{"2", "3"}}, expected: []any{"1", "2", "3"}},
	}
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			tt.a.Add(tt.args.elems...)
			assert.Equal(t, tt.expected, tt.a.data, "they should be equal")
		})
	}
}

func TestArray_Remove(t *testing.T) {
	type args struct {
		e any
	}
	tests := []struct {
		name     string
		a        *Array
		args     args
		expected []any
	}{
		{name: "int", a: &Array{data: []any{1, 2, 3}}, args: args{e: 2}, expected: []any{1, 3}},
		{name: "string", a: &Array{data: []any{"1", "2", "3"}}, args: args{e: "1"}, expected: []any{"2", "3"}},
	}
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			tt.a.Remove(tt.args.e)
			assert.Equal(t, tt.expected, tt.a.data, "they should be equal")
		})
	}
}

func TestArray_RemoveAll(t *testing.T) {
	type args struct {
		e interface{}
	}
	tests := []struct {
		name     string
		a        *Array
		args     args
		expected []any
	}{
		{name: "int", a: &Array{data: []any{1, 2, 3, 3, 3}}, args: args{e: 3}, expected: []any{1, 2}},
		{name: "string", a: &Array{data: []any{"1", "2", "2", "3"}}, args: args{e: "2"}, expected: []any{"1", "3"}},
	}
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			tt.a.RemoveAll(tt.args.e)
			assert.Equal(t, tt.expected, tt.a.data, "they should be equal")
		})
	}
}
```