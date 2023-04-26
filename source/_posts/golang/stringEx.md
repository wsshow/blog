---
title: golang扩展string
date: 2022-04-29 23:18:30
author: ws
description: golang扩展string
categories: utils
tags: [golang]
cover:
---

# golang扩展string

## 完整代码

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

## 测试代码

```go
package stringEx

import (
	"log"
	"testing"
)

var str = NewString("123qwe...")

func TestString_Contain(t *testing.T) {
	log.Println(str.Contain("123"), str.Length())
	log.Println(str.ReplaceAll("123", "789").ToString())
	log.Println(str.ReplaceAll("123", "789").Contain("789"))
}
```