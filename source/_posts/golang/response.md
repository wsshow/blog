---
title: golang服务响应封装
date: 2022-01-03 21:13:30
author: ws
description: 通用代码
categories: utils
tags: [golang, utils]
cover:
---

```go
package utils

type Response struct {
	Code int         `json:"code"`
	Desc string      `json:"desc"`
	Data interface{} `json:"data"`
}

func (r Response) Success(data interface{}) Response {
	return Response{
		Code: 0,
		Desc: "success",
		Data: data,
	}
}

func (r Response) Failure() Response {
	return Response{
		Code: 1,
		Desc: "failure",
		Data: nil,
	}
}

func (r Response) WithDesc(desc string) Response {
	r.Desc = desc
	return r
}
```

