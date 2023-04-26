---
title: 外部可执行程序管理
date: 2023-01-02 23:22:30
author: ws
description: 运行/停止外部可执行程序/接入控制台输出
categories: process
tags: [golang, process]
cover:
---

```go
package utils

import (
	"bufio"
	"context"
	"io"
	"os/exec"
)

type Process struct {
	exit   chan struct{}
	err    error
	stdout func(*bufio.Reader)
	stderr func(*bufio.Reader)
}

func NewProcess() *Process {
	return &Process{
		exit: make(chan struct{}),
	}
}

func (p *Process) WithStdOut(fn func(reader *bufio.Reader)) *Process {
	p.stdout = fn
	return p
}

func (p *Process) WithStdErr(fn func(reader *bufio.Reader)) *Process {
	p.stderr = fn
	return p
}

func (p *Process) Run(userCmd string) *Process {
	ctx, cancelFunc := context.WithCancel(context.Background())
	defer cancelFunc()

	go func() {

		var (
			err    error
			stdout io.ReadCloser
			stderr io.ReadCloser
		)
		defer func() {
			if err != nil {
				p.exit <- struct{}{}
				p.err = err
			}
		}()

		cmd := exec.CommandContext(ctx, "/bin/sh", "-c", userCmd)

		if stdout, err = cmd.StdoutPipe(); err != nil {
			return
		} else {
			go p.stdout(bufio.NewReader(stdout))
		}

		if stderr, err = cmd.StderrPipe(); err != nil {
			return
		} else {
			go p.stderr(bufio.NewReader(stderr))
		}

		if err = cmd.Start(); err != nil {
			return
		}

		if err = cmd.Wait(); err != nil {
			return
		}

		p.exit <- struct{}{}
	}()

	<-p.exit
	return p
}

func (p *Process) Stop() {
	p.exit <- struct{}{}
}

func (p *Process) Error() error {
	return p.err
}
```

