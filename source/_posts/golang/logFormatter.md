---
title: 格式化请求日志
date: 2023-01-03 23:22:30
author: ws
description: Gin框架格式化请求日志
categories: gin
tags: [golang, gin]
cover:
---

```go
package middleware

import (
	"acc/log"
	"fmt"
	"time"

	"github.com/gin-gonic/gin"
)

func logFormatter(param gin.LogFormatterParams) string {
	var statusColor, methodColor, resetColor string
	if param.IsOutputColor() {
		statusColor = param.StatusCodeColor()
		methodColor = param.MethodColor()
		resetColor = param.ResetColor()
	}

	if param.Latency > time.Minute {
		// Truncate in a golang < 1.8 safe way
		param.Latency = param.Latency - param.Latency%time.Second
	}

	if len(param.ErrorMessage) > 0 {
		log.Error("%s,%s,%s", param.ClientIP, param.Method, param.ErrorMessage)
	}

	return fmt.Sprintf("%v\t[INFO]\tmiddleware/recorder.go:28\t[%s %3d %s| %13v | %15s |%s %-7s %s %#v]\n",
		param.TimeStamp.Format("2006-01-02 15:04:05.000000"),
		statusColor, param.StatusCode, resetColor,
		param.Latency,
		param.ClientIP,
		methodColor, param.Method, resetColor,
		param.Path,
	)
}

func Recorder() gin.HandlerFunc {
	return gin.LoggerWithFormatter(logFormatter)
}
```

