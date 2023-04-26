---
title: golang通用代码
date: 2023-01-03 20:13:30
author: ws
description: 通用代码
categories: utils
tags: [golang, utils]
cover:
---

```go
package utils

import (
	"crypto/md5"
	"encoding/hex"
	"hash/crc32"
	"os"
)

func MD5(v string) string {
	d := []byte(v)
	m := md5.New()
	m.Write(d)
	return hex.EncodeToString(m.Sum(nil))
}

func HashCode(s string) uint32 {
	return crc32.ChecksumIEEE([]byte(s))
}

func IsPathExist(filePath string) bool {
	_, err := os.Stat(filePath)
	return err == nil
}

func CreatDir(dirPath string) error {
	err := os.MkdirAll(dirPath, os.ModePerm)
	if err != nil {
		return err
	}
	err = os.Chmod(dirPath, 0777)
	if err != nil {
		return err
	}
	return nil
}

func NotExistToMkdir(dirPath string) {
	if !IsPathExist(dirPath) {
		CreatDir(dirPath)
	}
}

func SuitableDisplaySize(size int64) string {
	if size > (1 << 30) {
		return strconv.FormatInt(size>>30, 10) + "GB"
	} else if size > (1 << 20) {
		return strconv.FormatInt(size>>20, 10) + "MB"
	} else if size > (1 << 10) {
		return strconv.FormatInt(size>>10, 10) + "KB"
	} else {
		return strconv.FormatInt(size, 10) + "B"
	}
}

func Cmd(cmd string) (string, error) {
	var result []byte
	var err error
	curOS := runtime.GOOS
	if curOS == "linux" {
		result, err = exec.Command("/bin/sh", "-c", cmd).Output()
	} else if curOS == "windows" {
		result, err = exec.Command("powershell", "/c", cmd).Output()
	}
	if err != nil {
		return "", err
	}
	return strings.TrimSpace(string(result)), nil
}

func ContainEx(a interface{}, f func(predicate interface{}) bool) bool {
	src := reflect.ValueOf(a)
	switch src.Kind() {
	case reflect.Slice, reflect.Array:
		count := src.Len()
		for i := 0; i < count; i++ {
			e := src.Index(i).Interface()
			if f(e) {
				return true
			}
		}
	default:
	}
	return false
}

func ReadFileToSlice(filePath string, handleFunc func(string) (string, bool)) ([]string, error) {
	if !IsPathExist(filePath) {
		return nil, errors.New(filePath + " not found")
	}
	f, err := os.Open(filePath)
	if err != nil {
		return nil, err
	}
	defer func(f *os.File) {
		err := f.Close()
		if err != nil {
			return
		}
	}(f)
	var fileList []string
	var s string
	fileScanner := bufio.NewScanner(f)
	for fileScanner.Scan() {
		s = fileScanner.Text()
		if s, bOK := handleFunc(s); bOK {
			fileList = append(fileList, s)
		}
	}
	return fileList, nil
}

func WriteInfoToFile(filePath string, content string) error {
	file, err := os.OpenFile(filePath, os.O_CREATE|os.O_WRONLY, 0666)
	if err != nil {
		return err
	}
	defer file.Close()
	_, err = io.WriteString(file, content)
	if err != nil {
		return err
	}
	return nil
}

func WriteSliceToFile(filePath string, contents []string) error {
	s := strings.Join(contents, "\n")
	return WriteInfoToFile(filePath, s)
}
```

