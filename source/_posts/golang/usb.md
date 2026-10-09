---
title: Linux USB 热插拔监听：用 Go 解析 netlink uevent
date: 2023-01-03 21:22:30
updated: 2026-10-09
author: ws
description: 从 udev/uevent 原理到 netlink 套接字，实现 U 盘插入自动挂载
categories: ["Go"]
tags: ["Go", "Linux", "系统编程"]
cover:
---

## 引言

插上 U 盘自动挂载、拔掉自动卸载，是典型的 Linux 系统编程需求。本文从内核 uevent 和 udev 的分工讲起，用 Go 直接监听 netlink 套接字拿到原始插拔事件，再调用 `blkid` 与 `mount(2)` 完成自动挂载。中间会先展示一段“看起来能跑、实际语义错误”的常见写法，把收不到事件、解析错位、消息截断、数据竞争这些坑逐个讲透，再给出可独立编译的完整实现。代码只在 Linux 上运行，在 macOS 上只能交叉编译，这属于正常现象。

## 内核 uevent 与 udev：谁负责什么

内核里任何 kobject 状态变化（设备注册、分区扫描、模块加载）都会通过 `kobject_uevent()` 发一条 uevent 给用户空间。发送方式是 netlink 多播，协议号 `NETLINK_KOBJECT_UEVENT`，多播组固定为 1：

```text
内核 kobject_uevent
        |  add@/devices/.../block/sda/sda1
        |  ACTION=add SUBSYSTEM=block DEVNAME=sda1 ...
        v
NETLINK_KOBJECT_UEVENT 多播组 1
        |---------------------------> systemd-udevd（订阅组 1）
        |                                | 建节点、改权限、跑规则、解析属性
        |                                v
        |                          组 2：udev 处理完的合成事件
        |---------------------------> 我们的程序（也订阅组 1）
```

关键点：**uevent 是内核主动发的，谁订阅谁就能收到**，udev 只是其中一个订阅者。所以直接监听 netlink 不需要 udev 参与，代价是拿到的是原始字段——如果你想要 udev 解析出来的 `ID_FS_UUID` 之类属性，得自己调 `blkid`（本文做法），或者用 libudev 订阅组 2。另外，块设备的 `DEVNAME`、`DEVTYPE`、`MAJOR/MINOR` 都是内核直接放在 uevent 里的，不需要 udev 补充。

## 先看常见错误写法：五个容易踩的坑

下面这段写法跑起来“能用”，但每一处都值得推敲：

1. **组号写错命名空间**。这段代码在 `NETLINK_KOBJECT_UEVENT` 的 socket 上用了 `syscall.RTNLGRP_LINK | syscall.RTNLGRP_IPV4_IFADDR | syscall.RTNLGRP_IPV6_IFADDR`。这些常量属于 `NETLINK_ROUTE` 协议，而且它们是**组序号**不是位掩码（Go 里 `LINK=0x1`、`IPV4_IFADDR=0x5`、`IPV6_IFADDR=0x9`，相或等于 13，二进制 `1101`）。能在 `NETLINK_KOBJECT_UEVENT` 上收到事件，纯粹因为 `LINK=1` 恰好把 bit 0 设上了，而内核 uevent 用的正是 bit 0。改用一个不含 bit 0 的组就什么都收不到：

```text
-- groups=1（内核 uevent 组）:          收到 add@/devices/virtual/net/gtestY ...
-- groups=13（三个常量相或的实际值）:   收到（bit 0 被 RTNLGRP_LINK=1 设上了）
-- groups=4（不含 bit 0 的对照组）:     什么都收不到
```

2. **消息按空格和固定下标解析**。`generateUFD` 用 `strings.Split(line, " ")` 拆 blkid 输出，再对每个字段取 `s[:6]`、`s[7:len(s)-1]`。这假设了引号一定存在、字段顺序固定、位置固定；卷标带空格的 U 盘直接解析错乱，字段缺失时还可能切片越界。正确做法是按 `KEY=VALUE` 解析。
3. **2048 字节缓冲 + 只 Read 一次 + recover 吞错**。内核侧 uevent 的属性缓冲是 2048 字节（`UEVENT_BUFFER_SIZE`），加上 `action@devpath` 头部和 `SEQNUM`，整条消息可能超过 2048 字节，长路径或属性多时会被截断；libudev 用的接收缓冲是 8192 字节。`FindDevice` 里 `recover()` 不记录任何信息，`Read` 一次就返回，被截断的消息会被当成完整的用。
4. **卸载和大小都不可靠**。`umount /dev/sda1` 是按设备名卸载，只有 utab/fstab 有记录时才有效，本文这种直接用 `mount(2)` 挂载的场景必须按挂载点卸载；`df -h | grep | awk` 解析大小既脆弱又容易误匹配。
5. **设备表并发不安全**。`Add` 先调无锁的 `Contain` 再加锁，`Contain`/`Get` 完全不持锁；监听 goroutine 和其他读设备列表的 goroutine 之间是数据竞争。另外，这种错误写法还依赖 `acc/global`、`acc/log` 等与主题无关的包，无法独立编译；下面给出的完整实现只依赖标准库。

## 消息格式：NUL 分隔的键值对，不是一行文本

用接收缓冲把一条真实消息按字节打印出来（`%q`），可以看到 uevent 的实际样子：

```text
n=217 raw="add@/devices/virtual/block/loop0/loop0p1\x00ACTION=add\x00DEVPATH=/devices/virtual/block/loop0/loop0p1\x00SUBSYSTEM=block\x00MAJOR=259\x00MINOR=1\x00DEVNAME=loop0p1\x00DEVTYPE=partition\x00DISKSEQ=15\x00PARTN=1\x00PARTUUID=4e4ad3e9-01\x00SEQNUM=1085\x00"
```

要点：

- 第一段是 `action@devpath`，例如 `add@/devices/...`；之后每段是 `KEY=VALUE`，全部用 `\0` 分隔，末尾也是 `\0`。值里可以带空格，甚至带 `=`，所以切分时只能按第一个 `=` 切。
- 一个 netlink datagram 就是一条 uevent，不会粘包；但一次插拔会产生**一串**事件：盘 `change`、分区 `add`、文件系统模块 `add module`、处理完 `remove` 等，监听端必须按 `SUBSYSTEM/DEVTYPE/DEVNAME` 过滤，不能假设“第一条就是我要的”。
- 缓冲不够时内核静默截断并在 `recvmsg` 的 `flags` 里置 `MSG_TRUNC`，必须检查，否则拿到的就是半条消息。

## 解析消息（uevent.go）

这个文件与操作系统无关，解析逻辑完全靠 `bytes` 和 `strings`，可以在 macOS 上直接跑单元测试。用 `bytes.Split(msg, []byte{0})` 切出所有字段，`strings.Cut` 只切第一个 `=`；同时拒绝空消息、没有 NUL 的消息、以及 udev 合成事件（消息头是 `libudev`）等非内核格式。

```go
// Package main —— uevent 解析（与操作系统无关，可在任何平台编译测试）。
package main

import (
	"bytes"
	"fmt"
	"strings"
)

// UEvent 是一条内核 uevent。消息不是文本行，而是一串 '\0' 分隔的字符串：
// 第一段是 "add@/devices/.../block/sda1"，之后每段是 "KEY=VALUE"。
type UEvent struct {
	Action  string            // add / remove / change / bind / unbind
	DevPath string            // 内核设备路径
	Env     map[string]string // 全部 KEY=VALUE
}

func (e *UEvent) Key(key string) string { return e.Env[key] }

// ParseUEvent 解析一条 netlink uevent 消息（纯 payload，不含 netlink 消息头）。
func ParseUEvent(msg []byte) (*UEvent, error) {
	if len(msg) == 0 || !bytes.ContainsRune(msg, 0) {
		return nil, fmt.Errorf("uevent: 空消息或缺少 NUL 分隔符")
	}
	fields := bytes.Split(msg, []byte{0})
	action, devPath, ok := strings.Cut(string(fields[0]), "@")
	if !ok || action == "" || !strings.HasPrefix(devPath, "/") {
		// udev 合成消息的头是 "libudev"，格式不同；这里只处理内核事件。
		return nil, fmt.Errorf("uevent: 无法解析消息头 %q", fields[0])
	}

	env := make(map[string]string, len(fields)-1)
	for _, field := range fields[1:] {
		if len(field) == 0 {
			continue // 末尾 NUL 会切出空段
		}
		key, value, ok := strings.Cut(string(field), "=")
		if ok && key != "" {
			env[key] = value
		}
	}
	return &UEvent{Action: action, DevPath: devPath, Env: env}, nil
}
```

## 设备表：并发安全的读写（device.go）

监听 goroutine 写设备表，程序其他部分（HTTP 接口、状态查询）随时可能读，所以读写都要持锁。一种常见的写法是 `Contain`/`Get` 不加锁、`Add` 的“查重”也在锁外，用 `-race` 一跑就是数据竞争；下面用 `sync.RWMutex`：`Add`/`Remove` 用写锁，`Get` 用读锁。

```go
// Package main —— 并发安全的设备表（与操作系统无关）。
package main

import (
	"sync"
	"time"
)

// Device 记录一个已挂载的设备。
type Device struct {
	Path      string    `json:"path"`       // /dev/sda1
	Name      string    `json:"name"`       // 卷标，可能为空
	UUID      string    `json:"uuid"`       // 文件系统 UUID
	FSType    string    `json:"fstype"`     // vfat / exfat / ext4 ...
	MountPath string    `json:"mount_path"` // 挂载点
	AddedAt   time.Time `json:"added_at"`
}

// DeviceTable 并发安全：监听 goroutine 写，其他 goroutine 随时读。
// Add/Remove 用写锁，Get 用读锁；监听 goroutine 写、其他 goroutine 读都不能绕开锁。
type DeviceTable struct {
	mu sync.RWMutex
	m  map[string]*Device
}

func NewDeviceTable() *DeviceTable { return &DeviceTable{m: make(map[string]*Device)} }

// Add 插入或覆盖同路径的设备。
func (t *DeviceTable) Add(d *Device) {
	t.mu.Lock()
	defer t.mu.Unlock()
	t.m[d.Path] = d
}

// Get 按设备路径查询。
func (t *DeviceTable) Get(path string) (*Device, bool) {
	t.mu.RLock()
	defer t.mu.RUnlock()
	d, ok := t.m[path]
	return d, ok
}

// Remove 删除设备并返回旧记录。
func (t *DeviceTable) Remove(path string) (*Device, bool) {
	t.mu.Lock()
	defer t.mu.Unlock()
	d, ok := t.m[path]
	if ok {
		delete(t.m, path)
	}
	return d, ok
}
```

## 监听器与挂载（monitor_linux.go）

监听器实现和 `main_linux.go` 一样只在 Linux 上编译（`//go:build linux`）。下面按“建立 socket → 接收循环 → 事件处理与挂载”的顺序说明，最后给出完整代码。

**建立 socket**：组号就是内核固定的 1（作为位掩码的 bit 0）。`syscall.Socket` 创建的 fd 不会被 Go 运行时登记到 netpoller，所以顺手设置 `CloseOnExec`，避免泄漏给 `exec` 出来的子进程；读接口用阻塞式即可，阻塞期间运行时会调度其他 goroutine。

**接收循环**：`receive` 每次读一条 datagram，检查截断标志，被信号打断（`EINTR`）就重试；`Run` 里遇到 `ENOBUFS`（接收队列溢出，内核已丢事件）只告警不退出。缓冲在整个循环里复用，不为每条事件分配。

**事件处理与挂载**：`handle` 只处理 `SUBSYSTEM=block` 且带 `DEVNAME` 的 `add`/`remove`；`DEVTYPE` 允许 `partition` 和 `disk`（整盘一个文件系统的 superfloppy 是 `disk`）。`addDevice` 先查重，再用 `blkid` 读文件系统类型与卷标，然后挂载并记录；`removeDevice` 按挂载点卸载并清理目录。`mountDirName` 会把卷标里路径不安全的字符替换掉（卷标是外部数据，可能带空格、斜杠甚至 `..`）。

`blkid` 的输出格式有个小坑：默认输出带引号，`-o export` 虽然稳定成 `KEY=VALUE`，但会对空格、反斜杠、引号做转义——卷标 `My Disk` 会输出成 `LABEL=My\ Disk`，还得再写反转义。所以这里直接用 `blkid -o value -s <tag>`，一次取一个字段、输出不作引号包裹与转义；没有卷标时输出为空且退出码为 0。

```go
//go:build linux

// Package main —— netlink uevent 监听与挂载（仅 Linux）。
package main

import (
	"errors"
	"fmt"
	"os"
	"os/exec"
	"path/filepath"
	"strings"
	"syscall"
	"time"
)

const (
	// 内核把 kobject uevent 多播到 NETLINK_KOBJECT_UEVENT 的 1 号组，
	// 与 syscall.RTNLGRP_*（NETLINK_ROUTE 的组）是两套命名空间；
	// RTNLGRP_LINK 恰好也是 1，所以写错常量会“看起来能跑”。
	ueventGroupKernel = 1

	// 内核侧属性缓冲是 2048 字节（UEVENT_BUFFER_SIZE），加上 "action@devpath"
	// 头和 SEQNUM，整条消息可能略超 2048；接收缓冲用 2048 会被截断，
	// 这里与 libudev 保持一致，用 8192 字节。
	ueventBufSize = 8192
)

// UEventMonitor 封装订阅了内核 uevent 的 netlink 套接字。
type UEventMonitor struct {
	fd int
}

// NewUEventMonitor 创建 NETLINK_KOBJECT_UEVENT 套接字并订阅 1 号组。
// 内核把事件发给所有订阅者，udev 只是其中之一，所以直接监听也能拿到插拔事件。
func NewUEventMonitor() (*UEventMonitor, error) {
	fd, err := syscall.Socket(syscall.AF_NETLINK, syscall.SOCK_DGRAM, syscall.NETLINK_KOBJECT_UEVENT)
	if err != nil {
		return nil, fmt.Errorf("socket(NETLINK_KOBJECT_UEVENT): %w", err)
	}
	syscall.CloseOnExec(fd) // 防止 fd 泄漏给子进程
	addr := &syscall.SockaddrNetlink{Family: syscall.AF_NETLINK, Groups: ueventGroupKernel}
	if err := syscall.Bind(fd, addr); err != nil {
		syscall.Close(fd)
		return nil, fmt.Errorf("bind(组 %d): %w", ueventGroupKernel, err)
	}
	return &UEventMonitor{fd: fd}, nil
}

func (m *UEventMonitor) Close() error { return syscall.Close(m.fd) }

// receive 读一条事件。一条 datagram 就是一条 uevent，不会粘包；
// 缓冲不够时内核静默截断并置 MSG_TRUNC，必须检查这个标志。
func (m *UEventMonitor) receive(buf []byte) (*UEvent, error) {
	for {
		n, _, flags, _, err := syscall.Recvmsg(m.fd, buf, nil, 0)
		if err != nil {
			if errors.Is(err, syscall.EINTR) {
				continue // 被信号打断，重试
			}
			return nil, fmt.Errorf("recvmsg: %w", err)
		}
		if flags&syscall.MSG_TRUNC != 0 {
			return nil, fmt.Errorf("uevent 超过 %d 字节被截断", len(buf))
		}
		if n == 0 {
			continue
		}
		return ParseUEvent(buf[:n])
	}
}

// Run 阻塞读取事件直到不可恢复的错误；verbose 会打印所有事件。
func (m *UEventMonitor) Run(table *DeviceTable, mountRoot string, verbose bool) error {
	buf := make([]byte, ueventBufSize) // 复用缓冲
	for {
		e, err := m.receive(buf)
		if err != nil {
			if errors.Is(err, syscall.ENOBUFS) {
				// 队列溢出时内核已经丢了一条事件，无法补读，继续听后面的。
				fmt.Fprintln(os.Stderr, "[warn] 接收队列溢出，有事件丢失")
				continue
			}
			return err
		}
		if verbose {
			fmt.Printf("[uevent] %-6s %-8s %s\n", e.Action, e.Key("SUBSYSTEM"), e.DevPath)
		}
		m.handle(e, table, mountRoot)
	}
}

// handle 只关心带 DEVNAME 的块设备 add/remove。
// 分区是 DEVTYPE=partition；整盘一个文件系统（superfloppy）是 disk。
func (m *UEventMonitor) handle(e *UEvent, table *DeviceTable, mountRoot string) {
	if e.Key("SUBSYSTEM") != "block" || e.Key("DEVNAME") == "" {
		return
	}
	if dt := e.Key("DEVTYPE"); dt != "partition" && dt != "disk" {
		return
	}
	switch devPath := filepath.Join("/dev", e.Key("DEVNAME")); e.Action {
	case "add":
		m.addDevice(devPath, table, mountRoot)
	case "remove":
		m.removeDevice(devPath, table)
	}
}

func (m *UEventMonitor) addDevice(devPath string, table *DeviceTable, mountRoot string) {
	if _, ok := table.Get(devPath); ok {
		return // 重复的 add 事件
	}
	fstype, label, uuid, err := ProbeFilesystem(devPath)
	if err != nil || fstype == "" {
		fmt.Printf("[skip] %s 没有可挂载的文件系统: %v\n", devPath, err)
		return
	}
	target := filepath.Join(mountRoot, mountDirName(devPath, label, uuid))
	if err := MountDevice(devPath, target, fstype); err != nil {
		fmt.Printf("[fail] 挂载 %s 失败: %v\n", devPath, err)
		return
	}
	table.Add(&Device{Path: devPath, Name: label, UUID: uuid, FSType: fstype, MountPath: target, AddedAt: time.Now()})
	fmt.Printf("[add]  %s name=%q uuid=%s type=%s mount=%s\n", devPath, label, uuid, fstype, target)
}

func (m *UEventMonitor) removeDevice(devPath string, table *DeviceTable) {
	d, ok := table.Remove(devPath)
	if !ok {
		return
	}
	if d.MountPath != "" {
		if err := syscall.Unmount(d.MountPath, 0); err != nil {
			fmt.Printf("[fail] 卸载 %s 失败: %v\n", d.MountPath, err)
		}
		if err := os.Remove(d.MountPath); err != nil && !os.IsNotExist(err) {
			fmt.Printf("[warn] 目录 %s 未清理: %v\n", d.MountPath, err)
		}
	}
	fmt.Printf("[remove] %s 已处理\n", devPath)
}

// ProbeFilesystem 用 blkid 读取文件系统信息。
// 用 `blkid -o value -s <tag>`：输出不作引号包裹与转义。`-o export` 虽然也是
// KEY=VALUE，但会把空格、反斜杠、引号转义（LABEL=My\ Disk），还得再反转义。
func ProbeFilesystem(devPath string) (fstype, label, uuid string, err error) {
	fstype, err = blkidValue(devPath, "TYPE")
	if err != nil || fstype == "" {
		return "", "", "", err // 没识别到文件系统（例如整盘只有分区表）
	}
	label, _ = blkidValue(devPath, "LABEL") // 没有卷标时输出为空、退出码 0
	uuid, _ = blkidValue(devPath, "UUID")
	return fstype, label, uuid, nil
}

func blkidValue(devPath, tag string) (string, error) {
	out, err := exec.Command("blkid", "-o", "value", "-s", tag, devPath).Output()
	if err != nil {
		return "", fmt.Errorf("blkid %s %s: %w", tag, devPath, err)
	}
	return strings.TrimSpace(string(out)), nil
}

// mountDirName 生成挂载目录名，并把卷标里路径不安全的字符过滤掉。
func mountDirName(devPath, label, uuid string) string {
	base := devPath
	switch {
	case label != "" && uuid != "":
		base = label + "-" + uuid
	case label != "":
		base = label
	case uuid != "":
		base = uuid
	}
	base = strings.TrimPrefix(base, "/dev/")
	return strings.Map(func(r rune) rune {
		switch {
		case r >= 'a' && r <= 'z', r >= 'A' && r <= 'Z', r >= '0' && r <= '9', r == '.', r == '_', r == '-':
			return r
		default:
			return '_'
		}
	}, base)
}

// MountDevice 把设备挂到挂载点，需要 root（CAP_SYS_ADMIN）。
// 普通用户请用 udisksctl mount -b <dev>，或用 udev + systemd 方案。
func MountDevice(devPath, target, fstype string) error {
	if err := os.MkdirAll(target, 0o755); err != nil {
		return fmt.Errorf("mkdir %s: %w", target, err)
	}
	const flags = syscall.MS_NOSUID | syscall.MS_NODEV // 可移动介质按不可信内容处理
	if err := syscall.Mount(devPath, target, fstype, flags, ""); err != nil {
		return fmt.Errorf("mount %s %s (%s): %w", devPath, target, fstype, err)
	}
	return nil
}
```

## main：只做接线（main_linux.go）

`main` 解析两个参数，检查 root 权限，创建挂载根目录后进入事件循环；每个设备的挂载点由 `MountDevice` 按需创建。

```go
//go:build linux

// ueventmon：监听内核 uevent，U 盘/移动硬盘插入时自动挂载（需要 root）。
// 构建：GOOS=linux go build -o ueventmon . ；运行：sudo ./ueventmon -v
package main

import (
	"flag"
	"fmt"
	"os"
)

func main() {
	mountRoot := flag.String("mount-root", "/media/ueventmon", "自动挂载的根目录")
	verbose := flag.Bool("v", false, "打印每一条收到的 uevent")
	flag.Parse()

	if os.Geteuid() != 0 {
		fmt.Fprintln(os.Stderr, "挂载需要 root：sudo 运行；只观察事件可以用 udevadm monitor")
		os.Exit(1)
	}
	if err := os.MkdirAll(*mountRoot, 0o755); err != nil {
		fmt.Fprintln(os.Stderr, "创建挂载根目录失败:", err)
		os.Exit(1)
	}

	mon, err := NewUEventMonitor()
	if err != nil {
		fmt.Fprintln(os.Stderr, "创建 uevent 监听失败:", err)
		os.Exit(1)
	}
	defer mon.Close()
	fmt.Printf("监听 NETLINK_KOBJECT_UEVENT（内核组 %d），挂载根目录 %s\n", ueventGroupKernel, *mountRoot)
	if err := mon.Run(NewDeviceTable(), *mountRoot, *verbose); err != nil {
		fmt.Fprintln(os.Stderr, "监听失败:", err)
		os.Exit(1)
	}
}
```

## 在 Linux 上验证

macOS 没有 AF_NETLINK，第一步是在 macOS 上做交叉编译和静态检查；真实运行放到 Linux（虚拟机、物理机，或 `docker run --privileged` 的容器）：

```bash
$ GOOS=linux GOARCH=amd64 go vet ./...          # 无输出即通过
$ GOOS=linux GOARCH=amd64 go build -o ueventmon-linux-amd64 .
$ file ueventmon-linux-amd64
ueventmon-linux-amd64: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), statically linked, ...
$ GOOS=linux GOARCH=arm64 go build -o ueventmon-linux-arm64 .   # 树莓派等 ARM 平台同理
```

可移植的解析与设备表逻辑直接在 macOS 上跑单元测试（含 `-race`；下面只摘录 PASS 行）：

```text
$ go test -v -race -count=1 .
--- PASS: TestDeviceTableBasic (0.00s)
--- PASS: TestDeviceTableConcurrent (0.01s)
--- PASS: TestParseUEventKernelAdd (0.00s)
--- PASS: TestParseUEventNoTrailingNul (0.00s)
--- PASS: TestParseUEventValueContainsEquals (0.00s)
--- PASS: TestParseUEventValueContainsSpace (0.00s)
--- PASS: TestParseUEventReject (0.00s)
ok  	ueventmon	1.291s
```

在特权容器里创建一个 veth 网卡，监听器会收到内核发来的一整串事件（节选）：

```text
[uevent] add    net      /devices/virtual/net/vtest1
[uevent] add    queues   /devices/virtual/net/vtest1/queues/rx-0
...（中间还有几十条 net/queues 事件）...
[uevent] remove net      /devices/virtual/net/vtest1
```

用 loop 设备模拟 U 盘（镜像里建一个 ext4 分区）做端到端挂载，插入和移除的真实日志：

```text
[uevent] add    block    /devices/virtual/block/loop0/loop0p1
[add]  /dev/loop0p1 name="TESTVOL" uuid=1e93d4f7-e346-4aef-9e16-24ace66a9afa type=ext4 mount=/media/ueventmon/TESTVOL-1e93d4f7-e346-4aef-9e16-24ace66a9afa
/dev/loop0p1 /media/ueventmon/TESTVOL-1e93d4f7-e346-4aef-9e16-24ace66a9afa ext4 rw,nosuid,nodev,relatime 0 0
[uevent] remove block    /devices/virtual/block/loop0/loop0p1
[remove] /dev/loop0p1 已处理
```

上面的 `[add]` 行来自程序，紧接着的 `ext4 rw,nosuid,nodev,relatime` 行来自 `/proc/mounts`；触发 `remove` 后挂载消失、挂载目录被清理。在没有 udev 的容器里，内核的 uevent 照样能收到，但 `/dev` 下的设备节点不会自动出现（devtmpfs 没挂全），本文用 `mount -t devtmpfs` + 符号链接模拟了 udev 建节点的过程；真实发行版上由 devtmpfs 与 udev 负责。真机验证可以对照现成工具：`udevadm monitor --kernel --property --subsystem-match=block` 看到的内核事件应该和本文程序打印的 `DEVNAME` 一致。

## 踩坑与边界

- **只在 Linux 上运行**。`AF_NETLINK`、`syscall.Mount`、`MSG_TRUNC` 都是 Linux 专有；macOS 上 `GOOS=linux go build` 能编译，但直接 `go run` 不行，也不需要给非 Linux 平台做兼容——用构建标签把它隔离掉即可。
- **内核事件 ≠ udev 事件**。本文订阅的是组 1 的原始事件；`/run/udev/control` 存在时 udevd 才会在组 2 上发处理过的事件，容器里通常没有 udevd，组 2 永远收不到东西。需要 udev 解析后的属性（`ID_FS_*`、`ID_SERIAL` 等）时，要么自己调 `blkid`，要么改用 libudev/sd-device 的 API。
- **挂载点被占用时卸载会失败**。拔盘瞬间如果有进程还在读写挂载点，`syscall.Unmount` 返回 `EBUSY`，程序会打印失败；`MNT_DETACH`（lazy unmount）可以避免卡住，但可能掩盖数据未落盘的问题，按场景取舍。清理挂载目录也可能因为目录被占用而失败，代码里只告警不中断。
- **blkid 缓存与转义**。`blkid` 会使用缓存，刚写完分区表的设备可能读不到；必要时用 `blkid -p` 强制低层探测。`-o export` 的转义问题见正文（`LABEL=My\ Disk`），逐字段用 `-o value -s` 更省事。
- **权限与属主**。`mount(2)` 需要 root；桌面环境优先用 `udisksctl mount -b /dev/sdb1`（普通用户即可，自动挂到 `/media/$USER`），或者写 udev 规则调用 `systemd-mount`：`ACTION=="add", SUBSYSTEM=="block", ENV{DEVTYPE}=="partition", ENV{ID_FS_TYPE}!="", RUN{program}+="/usr/bin/systemd-mount --no-block --collect $devnode"`。另外 `mount(2)` 直接挂时文件默认属于 root，需要普通用户读写就在 `data` 参数里传 `uid=1000,gid=1000`。

## 总结

- 内核通过 `NETLINK_KOBJECT_UEVENT` 多播组 1 发送原始 uevent，udev 只是订阅者之一；直接监听 netlink 就能拿到插拔事件，且块设备的 `DEVNAME`/`DEVTYPE` 内核已经写好。
- `syscall.RTNLGRP_*` 是 `NETLINK_ROUTE` 的**组序号**，不是 KOBJECT 协议的位掩码；错误写法能收到事件只是因为 `RTNLGRP_LINK=1` 碰巧占用了 bit 0，换成别的组合就收不到。正确的组号就是 1。
- uevent 不是一行文本，而是 `action@devpath` 开头、NUL 分隔的 `KEY=VALUE` 序列；解析必须按第一个 `=` 切分，过滤必须按字段而不是固定下标。
- 接收端要处理三件事：8 KiB 缓冲与 `MSG_TRUNC` 截断检查、`EINTR` 重试、`ENOBUFS` 告警；设备表读写都要加锁。
- 挂载用 `blkid` 探测 + `syscall.Mount`（root），卸载按挂载点；普通用户改用 `udisksctl` 或 udev 规则，macOS 上只能交叉编译。
