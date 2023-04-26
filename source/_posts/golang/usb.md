---
title: linux下的usb接入监控
date: 2023-01-03 21:22:30
author: ws
description: linux下利用netlink进行usb接入监控
categories: usb
tags: [golang, usb]
cover:
---

## usb-usb.go

```go
package usb

import (
	"acc/global"
	"acc/log"
	"acc/storage"
	"acc/utils"
	"fmt"
	"strings"
	"syscall"
	"time"
)

type ufd struct {
	Path string
	Name string
	UUID string
	Type string
}

type NetlinkListener struct {
	fd int
	sa *syscall.SockaddrNetlink
}

func ListenNetlink() (*NetlinkListener, error) {
	groups := syscall.RTNLGRP_LINK |
		syscall.RTNLGRP_IPV4_IFADDR |
		syscall.RTNLGRP_IPV6_IFADDR

	s, err := syscall.Socket(syscall.AF_NETLINK, syscall.SOCK_DGRAM,
		syscall.NETLINK_KOBJECT_UEVENT)
	if err != nil {
		return nil, fmt.Errorf("socket: %s", err)
	}

	saddr := &syscall.SockaddrNetlink{
		Family: syscall.AF_NETLINK,
		Pid:    uint32(0),
		Groups: uint32(groups),
	}
	err = syscall.Bind(s, saddr)
	if err != nil {
		return nil, fmt.Errorf("bind: %s", err)
	}
	return &NetlinkListener{fd: s, sa: saddr}, nil
}

func generateUFD(dev string) *ufd {
	u := &ufd{
		Name: "unknown",
	}
	ss := strings.Split(strings.TrimSpace(dev), " ")
	u.Path = strings.TrimSuffix(ss[0], ":")
	for _, s := range ss[1:] {
		switch s[:6] {
		case "LABEL=":
			u.Name = s[7 : len(s)-1]
		case "UUID=\"":
			u.UUID = s[6 : len(s)-1]
		case "TYPE=\"":
			u.Type = s[6 : len(s)-1]
		}
	}
	return u
}

func (l *NetlinkListener) FindDevice() {
	defer func() {
		recover()
	}()
	pkt := make([]byte, 2048)
	_, err := syscall.Read(l.fd, pkt)
	if err != nil {
		log.Error(err)
		return
	}
	outMsg := string(pkt)
	if find := strings.Contains(outMsg, "DEVTYPE=partition") && strings.Contains(outMsg, "SUBSYSTEM=block"); find {
		go opDevices(outMsg)
	}
}

func opDevices(outMsg string) {
	action := strings.Split(outMsg, "@")[0]
	tmp := strings.Split(outMsg, "ACTION")[0]
	name := strings.Split(tmp, "/")[len(strings.Split(tmp, "/"))-1]
	name = utils.TrimNul(name)
	bindPath := fmt.Sprint("/dev/", name)
	switch action {
	case "add":
		blkidRet, _ := utils.Cmd("blkid")
		devList := strings.Split(blkidRet, "\n")
		for _, dev := range devList {
			if !strings.Contains(dev, strings.TrimSpace(bindPath)) {
				continue
			}
			u := generateUFD(dev)
			mountDir := fmt.Sprintf("./devices/%s", fmt.Sprintf("%s-%s", u.Name, u.UUID))
			if !utils.IsPathExist(mountDir) {
				utils.CreatDir(mountDir)
			}
			utils.Cmd(fmt.Sprintf("mount %s %s", bindPath, mountDir))
			size, _ := utils.Cmd(fmt.Sprintf("df -h | grep \"%s\" | awk '{print $2}'", u.Path))
			addUSBDevice := &storage.USBScanResult{
				Path: u.Path, Name: u.Name, UUID: u.UUID, Type: u.Type,
				Size: size, MountPath: mountDir, StartTime: time.Now().Format("2006-01-02 15:04:05"),
			}
			global.USBDeviceList.Add(addUSBDevice)
			log.Debug("add device: ", bindPath)
			log.Debug(addUSBDevice.String())
		}
	case "remove":
		log.Debug("remove device: ", bindPath)
		utils.Cmd("umount " + bindPath)
		usr := global.USBDeviceList.Get(bindPath)
		if usr == nil {
			return
		}
		if utils.IsPathExist(usr.MountPath) {
			utils.RemoveAll(usr.MountPath)
		}
		global.USBDeviceList.Remove(bindPath)
	}
}

func Monitor() {
	go func() {
		l, _ := ListenNetlink()
		log.Debug("device monitor start")
		for {
			l.FindDevice()
		}
	}()
}
```

## storage-usb.go

```go
package storage

import (
	"fmt"
	"sync"
)

type USBScanResult struct {
	Path      string `json:"path,omitempty"`
	Name      string `json:"name,omitempty"`
	UUID      string `json:"uuid,omitempty"`
	Type      string `json:"type,omitempty"`
	Size      string `json:"size,omitempty"`
	MountPath string `json:"mount_path,omitempty"`
	StartTime string `json:"start_time,omitempty"`
}

func (u *USBScanResult) String() string {
	return fmt.Sprintf("\nPath:%s\nName:%s\nUUID:%s\nType:%s\nSize:%s\nMountPath:%s\nStartTime:%s\n", u.Path, u.Name, u.UUID, u.Type, u.Size, u.MountPath, u.StartTime)
}

type USBList struct {
	USRL []*USBScanResult `json:"usrl,omitempty"`
	Mu   sync.Mutex       `json:"-"`
}

func NewUSBList() *USBList {
	return &USBList{
		USRL: make([]*USBScanResult, 0),
	}
}

func (ul *USBList) Add(u *USBScanResult) {
	if ul.Contain(u.Path) {
		return
	}
	ul.Mu.Lock()
	defer ul.Mu.Unlock()
	ul.USRL = append(ul.USRL, u)
}

func (ul *USBList) Remove(up string) {
	ul.Mu.Lock()
	defer ul.Mu.Unlock()
	d := ul.USRL
	for i, cnt := 0, len(d); i < cnt; i++ {
		if d[i].Path == up {
			d = append(d[:i], d[i+1:]...)
			break
		}
	}
	ul.USRL = d
}

func (ul *USBList) Contain(up string) bool {
	d := ul.USRL
	for i, cnt := 0, len(d); i < cnt; i++ {
		if d[i].Path == up {
			return true
		}
	}
	return false
}

func (ul *USBList) Get(up string) *USBScanResult {
	d := ul.USRL
	for i, cnt := 0, len(d); i < cnt; i++ {
		if d[i].Path == up {
			return d[i]
		}
	}
	return nil
}
```

## global-global.go

```go
package global

import "acc/storage"

var USBDeviceList = storage.NewUSBList()
```

