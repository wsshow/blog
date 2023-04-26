---
title: js中通用方法
date: 2022-12-13 22:00:30
author: ws
description: 多条件过滤/函数防抖
categories: js
tags: [js, utils]
cover: 
---

```javascript
// 多条件数据过滤
const filterDataSource = (condition: any, data: any) => {
  return data.filter((item: any) => {
    return Object.keys(condition).every((key) => {
      return String(item[key])
        .toLowerCase()
        .includes(String(condition[key]).trim().toLowerCase())
    })
  })
}
```

```javascript
// 函数防抖
const debounce = () => {
  let timeoutId: any = null
  return (callback: any, wait = 1500) => {
    if (timeoutId) {
      window.clearTimeout(timeoutId)
    }
    timeoutId = window.setTimeout(callback, wait)
  }
}
```