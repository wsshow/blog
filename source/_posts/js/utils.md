---
title: 前端常用工具函数：多条件过滤、防抖与节流
date: 2022-12-13 22:00:30
updated: 2026-10-09
author: ws
description: 用 TypeScript 写对防抖，并说清它和节流的区别
categories: ["JavaScript"]
tags: ["JavaScript", "TypeScript"]
cover:
---

## 引言

多条件过滤、防抖、节流是前端出现频率最高的三个工具函数，但随手写出来的版本里藏着不少坑：条件为 `undefined` 时会误匹配、防抖函数每次调用都要重新传 callback、`this` 和参数在异步里丢失。本文先把这种常见写法的问题用测试用例固定下来，再给出可直接使用的 TypeScript 实现，并用 Node 实际跑一遍验证行为——包括“防抖和节流在同一场景下分别执行几次”的真实时间线。

## 先看一种常见的写法

不假思索地实现这两个工具函数，代码往往长这样（后面列出的问题都针对这两段）：

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

这两段代码能跑，但分别埋着这些隐患：

**过滤函数：**

1. `String(undefined)` 会变成字符串 `"undefined"`。如果某个条件的值是 `undefined`（表单里没填的字段很常见），它会去匹配“字段值恰好也是 undefined”的行；
2. 条件值是空字符串时，`includes('')` 恒为 `true`，等于没过滤；
3. 数字条件按子串匹配：`{ id: 12 }` 会命中 `id: 123`；
4. 强制 `toLowerCase()`，做不到大小写敏感匹配；
5. 全部 `any`，TypeScript 形同虚设。

**防抖函数：**

1. API 反人类：防抖的目的就是“把回调包起来”，这种写法却要求每次触发都重新传 callback；
2. callback 被裸调用，`this` 和事件参数都会丢失；
3. 没有 `cancel`/`flush`：组件卸载或需要立即执行（表单提交）时无法控制；
4. 依赖 `window`，在 Node、SSR、Worker 里直接报错；
5. 状态挂在工厂闭包里，一旦写成 `debounce()(callback)` 这种内联调用，每次都会新建闭包，防抖完全失效。

下面逐个给出更稳妥的实现。

## 多条件过滤：可预测的 TypeScript 实现

设计目标很明确：空条件跳过、数字和布尔按值比较、字符串比较方式可配置、类型安全。完整实现如下：

```typescript
export type MatchMode = 'contains' | 'equals' | 'startsWith'

export interface FilterOptions {
  /** 字符串匹配方式，默认 contains（包含） */
  mode?: MatchMode
  /** 是否区分大小写，默认 false */
  caseSensitive?: boolean
  /** 条件值为空（undefined / null / 空白字符串）时是否跳过，默认 true */
  skipEmpty?: boolean
}

/** 判断条件值是否为空：表单里未填的字段传进来通常就是这三种 */
function isBlank(value: unknown): boolean {
  return (
    value === undefined ||
    value === null ||
    (typeof value === 'string' && value.trim() === '')
  )
}

function matchValue(
  field: unknown,
  cond: unknown,
  options: Required<FilterOptions>,
): boolean {
  // 数字条件按“值相等”比较：12 不应该命中 123
  if (typeof cond === 'number') {
    if (typeof field === 'number') return field === cond
    if (typeof field === 'string' && field.trim() !== '') {
      return Number(field) === cond
    }
    return false
  }
  // 布尔条件同样按值相等比较
  if (typeof cond === 'boolean') {
    if (typeof field === 'boolean') return field === cond
    if (typeof field === 'string') {
      return field.trim().toLowerCase() === String(cond)
    }
    return false
  }

  const needle = options.caseSensitive
    ? String(cond).trim()
    : String(cond).trim().toLowerCase()
  const haystack = options.caseSensitive
    ? String(field)
    : String(field).toLowerCase()

  switch (options.mode) {
    case 'equals':
      return haystack === needle
    case 'startsWith':
      return haystack.startsWith(needle)
    case 'contains':
    default:
      return haystack.includes(needle)
  }
}

export function filterDataSource<T extends object>(
  condition: Partial<Record<keyof T, unknown>>,
  data: readonly T[],
  options: FilterOptions = {},
): T[] {
  const opts: Required<FilterOptions> = {
    mode: options.mode ?? 'contains',
    caseSensitive: options.caseSensitive ?? false,
    skipEmpty: options.skipEmpty ?? true,
  }

  // 只保留“有效”条件，避免 undefined / 空串把结果全部误命中
  const entries = Object.entries(condition).filter(
    ([, value]) => !(opts.skipEmpty && isBlank(value)),
  )

  // 没有任何有效条件时返回全部数据
  if (entries.length === 0) return [...data]

  return data.filter((item) =>
    entries.every(([key, cond]) =>
      matchValue((item as Record<string, unknown>)[key], cond, opts),
    ),
  )
}
```

几个关键设计：

- `skipEmpty` 默认 `true`，把 `undefined`/`null`/空白字符串条件直接剔除，从根上避免 `"undefined"` 误匹配；
- 条件值是数字或布尔时走**严格按值比较**分支，数字字符串（`"12"`）会被 `Number()` 转换后比较，兼容表单数据的常见形态；
- 字符串比较的 `mode`/`caseSensitive` 可以按场景配置：用户搜索用默认的 `contains` + 忽略大小写，筛选枚举状态用 `equals` 更准确；
- 泛型 `T extends object` 让 `condition` 的 key 必须是数据的字段，写错字段名会直接编译报错。

### 用测试用例固定这些边界行为

测试数据与教学用的错误实现：

```typescript
const rows = [
  { id: 1, name: 'Alice', city: 'Shenzhen', vip: true },
  { id: 12, name: 'Bob', city: 'Beijing', vip: false },
  { id: 123, name: 'Cindy' },
]

// 错误示范：故意保留这些缺陷，用来和正确实现做对照
function naiveFilter(
  condition: Record<string, unknown>,
  data: readonly object[],
): object[] {
  return data.filter((item: any) =>
    Object.keys(condition).every((key) =>
      String(item[key])
        .toLowerCase()
        .includes(String(condition[key]).trim().toLowerCase()),
    ),
  )
}
```

对应测试断言：

```typescript
test('naive_condition_undefined', () => {
  // 只有第三条数据没有 city 字段：String(undefined) === 'undefined'，正好被条件命中
  assert.equal(naiveFilter({ city: undefined }, rows).length, 1)
  // 推荐实现把空条件当作「不筛选」
  assert.equal(filterDataSource({ city: undefined }, rows).length, 3)
})

test('naive_numeric_substring', () => {
  assert.equal(naiveFilter({ id: 12 }, rows).length, 2)
  assert.equal(filterDataSource({ id: 12 }, rows).length, 1)
})
```

其中 `naive_` 开头的两条用例专门断言错误示范的行为，再和推荐实现的结果形成对照。用 Node 内置测试运行器执行（Node 25）：

```bash
node --test utils.test.ts
```

实际输出（节选）：

```text
✔ naive_condition_undefined (0.9655ms)
✔ naive_numeric_substring (0.194625ms)
✔ filter_case_sensitive (0.651125ms)
✔ filter_match_mode (0.088291ms)
...
ℹ tests 14
ℹ pass 14
ℹ fail 0
```

注意：`node --test` 直接跑 `.ts` 依赖 Node 23.6+ 的类型擦除能力；项目 `package.json` 里需要有 `"type": "module"`（或者把文件命名成 `.mts`），否则会按 CommonJS 解析报错。

## 防抖：真正可用的实现

先明确目标行为：

- 接收一个函数 `fn`，返回包装后的防抖函数（而不是每次传 callback）；
- 调用时的 `this` 和参数原样透传给 `fn`，取“最后一次调用”的参数；
- 支持 `leading`（首次立即执行）和 `trailing`（结束后执行最后一次）；
- 提供 `cancel()` 和 `flush()`。

实现如下：

```typescript
export interface DebounceOptions {
  /** 是否在第一次调用时立即执行，默认 false */
  leading?: boolean
  /** 是否在等待结束后执行最后一次调用，默认 true */
  trailing?: boolean
}

export interface Debounced<F extends (this: any, ...args: any[]) => any> {
  (this: ThisParameterType<F>, ...args: Parameters<F>): void
  /** 取消还未执行的调用 */
  cancel(): void
  /** 立即执行挂起的调用并返回其结果 */
  flush(): ReturnType<F> | undefined
}

export function debounce<F extends (this: any, ...args: any[]) => any>(
  fn: F,
  wait = 300,
  options: DebounceOptions = {},
): Debounced<F> {
  const { leading = false, trailing = true } = options
  let timer: ReturnType<typeof setTimeout> | null = null
  let lastArgs: Parameters<F> | null = null
  let lastThis: ThisParameterType<F> | undefined
  let result: ReturnType<F> | undefined

  // 真正执行 fn，并把 this 和参数原样透传
  const invoke = (): ReturnType<F> | undefined => {
    if (lastArgs) {
      result = fn.apply(lastThis as ThisParameterType<F>, lastArgs)
      lastArgs = null
      lastThis = undefined
    }
    return result
  }

  const debounced = function (
    this: ThisParameterType<F>,
    ...args: Parameters<F>
  ): void {
    lastArgs = args
    lastThis = this

    const isFirstCall = timer === null
    if (timer !== null) clearTimeout(timer)

    // 首次调用且开启 leading 时立即执行
    if (leading && isFirstCall) invoke()

    timer = setTimeout(() => {
      timer = null
      if (trailing) invoke()
    }, wait)
  } as Debounced<F>

  debounced.cancel = (): void => {
    if (timer !== null) clearTimeout(timer)
    timer = null
    lastArgs = null
    lastThis = undefined
  }

  debounced.flush = (): ReturnType<F> | undefined => {
    if (timer !== null) {
      clearTimeout(timer)
      timer = null
    }
    return invoke()
  }

  return debounced
}
```

实现细节说明：

- **`lastThis`/`lastArgs` 缓存调用现场**，定时器触发时用 `fn.apply()` 执行，`this` 和参数不再丢失；
- **`invoke()` 执行后清空 `lastArgs`**，这是 `leading + trailing` 语义的关键：首次立即执行后，如果等待期内没有新调用，结束时不会重复执行；有新调用才会补执行最后一次；
- **`cancel()` 清定时器和缓存**，适合组件卸载；**`flush()` 立即执行挂起调用并返回结果**，适合“用户点了搜索按钮，不等防抖结束”的场景；
- 定时器用 `setTimeout`/`clearTimeout` 而不是 `window.setTimeout`，浏览器、Node、Worker 都能跑。

### leading / trailing 行为矩阵

| 配置                        | 首次调用 | 停止触发 wait 后           | 典型场景             |
| :-------------------------- | :------- | :------------------------- | :------------------- |
| `trailing`（默认）          | 不执行   | 执行最后一次               | 搜索输入、自动保存   |
| `leading: true, trailing: false` | 立即执行 | 不执行                | 按钮防连点           |
| `leading: true, trailing: true`  | 立即执行 | 等待期内有新调用则补一次 | 高频操作 + 收尾动作 |

### 行为验证

`this` 透传和参数取值的测试最能说明问题：对一个带 `value` 的对象包裹防抖方法，连续调用 `add(2)`、`add(3)`，等待后断言 `value === 3`——`this` 指向源对象，参数取最后一次调用。

用 `node --test` 跑完整套防抖用例，真实输出：

```text
✔ debounce：连续触发只执行最后一次（trailing） (121.333125ms)
✔ debounce：leading 立即执行第一次调用 (123.484958ms)
✔ debounce：leading + trailing 执行首尾各一次 (121.715083ms)
✔ debounce：cancel 丢弃挂起的调用 (121.250792ms)
✔ debounce：flush 立即执行并返回结果 (0.296709ms)
✔ debounce：this 与参数原样透传 (80.795625ms)
```

### 防抖、节流的真实时间线

同一场景对比：150ms 内每 30ms 触发一次、共 5 次，防抖和节流的等待窗口都是 100ms：

```text
throttle-时间戳 执行 @  0ms
throttle-定时器 执行 @101ms
throttle-时间戳 执行 @126ms
debounce        执行 @227ms
throttle-定时器 执行 @227ms
```

数字解读：时间戳版在 `0ms` 立即执行第一次，第二次因为窗口已过 100ms，在 `126ms` 又执行了一次；定时器版把第一次延迟到 `101ms`，最后一轮触发后又排了一次、`227ms` 执行；防抖只在最后一次调用（约 `120ms`）之后安静 100ms，`227ms` 执行一次——整个 750ms 的场景里总共只执行 1 次。

## 节流：两种经典实现

节流的目标是“固定频率执行”。两种常见实现各有取舍。

### 时间戳版：立即执行

```typescript
/** 时间戳版：首次立即执行，窗口内的后续调用直接丢弃 */
export function throttleTimestamp<
  F extends (this: any, ...args: any[]) => any,
>(fn: F, wait = 200): F {
  let lastTime = 0
  return function (this: ThisParameterType<F>, ...args: Parameters<F>) {
    const now = Date.now()
    if (now - lastTime >= wait) {
      lastTime = now
      return fn.apply(this, args)
    }
    return undefined
  } as F
}
```

优点是首次调用立即生效，适合点击这类需要即时反馈的场景；缺点是**窗口内的调用被完全丢弃**，如果事件流到最后突然停止，最后一次状态不会被执行。

### 定时器版：保证窗口内有一次

```typescript
/** 定时器版：首次延迟执行，窗口内只保证执行一次（用第一次的参数） */
export function throttleTimer<F extends (this: any, ...args: any[]) => any>(
  fn: F,
  wait = 200,
): F {
  let timer: ReturnType<typeof setTimeout> | null = null
  return function (this: ThisParameterType<F>, ...args: Parameters<F>) {
    if (timer !== null) return
    timer = setTimeout(() => {
      timer = null
      fn.apply(this, args)
    }, wait)
  } as F
}
```

优点是“窗口结束必有一次执行”（适合滚动位置上报、自动保存），缺点是首次执行要等一个窗口，且执行时用的是窗口内**第一次**触发的参数，不是最后一次。

两种实现的对比：

| 维度           | 时间戳版                 | 定时器版                     |
| :------------- | :----------------------- | :--------------------------- |
| 首次调用       | 立即执行                 | 延迟 wait 后执行             |
| 窗口内调用     | 丢弃                     | 丢弃                         |
| 执行参数       | 触发窗口的那次调用       | 窗口内第一次调用             |
| 是否有 trailing | 否                      | 是（延迟执行即收尾）         |
| 典型场景       | 点击、mousemove 采样     | 滚动上报、保证最终状态同步   |

如果需要“首次立即执行 + 结尾补一次 + 保证最大间隔”，即 lodash 的 `throttle` 语义，它的实现其实是 `debounce` 加 `maxWait` 选项。追求行为完备可以直接用 lodash，或者给本文的 `debounce` 扩展一个 `maxWait`。

## 什么时候用哪个

| 场景                     | 推荐方案                                      | 理由                             |
| :----------------------- | :-------------------------------------------- | :------------------------------- |
| 搜索框输入联想           | 防抖（trailing，300ms 左右）                  | 停止输入再请求，次数最少         |
| 滚动加载、resize 布局    | 节流（100~200ms）或 `requestAnimationFrame`   | 需要过程中的持续反馈             |
| 按钮防连点               | 防抖（leading）或点击后禁用按钮               | 立即响应且只响应一次             |
| 拖拽、动画跟随           | `requestAnimationFrame`                       | 与渲染帧对齐，天然节流           |
| 表单自动保存             | 防抖 + 失焦时 `flush()`                       | 停顿才保存，离开时立即落盘       |
| 窗口关闭前上报           | 不用防抖/节流，直接 `sendBeacon`              | 延迟执行可能来不及               |

一个高频错误：给**每个**输入框、每个搜索实例共用一个防抖函数。防抖状态绑定在返回的函数实例上，多个组件共用会互相取消。正确做法是每个组件实例各自持有一个（下面的 React Hook 就是按实例创建的）。

## React 中的写法

### useDebouncedValue：延迟一个值

最常见的需求是“输入框值变化后延迟请求”，用 Hook 返回延迟后的值：

```tsx
import { useEffect, useState } from 'react'

/** 返回延迟后的值：输入框搜索推荐场景 */
export function useDebouncedValue<T>(value: T, delay = 300): T {
  const [debounced, setDebounced] = useState(value)

  useEffect(() => {
    const timer = setTimeout(() => setDebounced(value), delay)
    // value/delay 变化时清掉旧定时器，等价于重新计时
    return () => clearTimeout(timer)
  }, [value, delay])

  return debounced
}
```

使用方式：`const debouncedKeyword = useDebouncedValue(keyword, 300)`，再让请求逻辑依赖 `debouncedKeyword` 触发。

### useDebouncedCallback：延迟一个回调

需要自己控制执行时机时，用 `useMemo` 创建防抖函数、`useRef` 保存最新回调、卸载时 `cancel`：

```tsx
import { useEffect, useMemo, useRef } from 'react'
import { debounce } from './utils.ts'
import type { Debounced } from './utils.ts'

/** 返回防抖后的回调：适合按钮、提交、需要自己控制时机的场景 */
export function useDebouncedCallback<F extends (this: any, ...args: any[]) => any>(
  fn: F,
  delay = 300,
): Debounced<F> {
  const fnRef = useRef(fn)

  // 每次渲染后更新引用，保证防抖函数内拿到的永远是最新回调
  useEffect(() => {
    fnRef.current = fn
  }, [fn])

  const debounced = useMemo(
    () => debounce(((...args: Parameters<F>) => fnRef.current(...args)) as F, delay),
    [delay],
  )

  // 组件卸载时取消挂起的调用，避免在已卸载组件上 setState
  useEffect(() => () => debounced.cancel(), [debounced])

  return debounced
}
```

这两段代码的要点：

- `useMemo` 只在 `delay` 变化时重建实例，避免每次渲染都创建新的防抖函数导致状态丢失；
- `fnRef` 解决闭包陈旧问题：防抖函数持有的是稳定的包装函数，真正执行时才读取最新的 `fn`；
- 卸载时的 `cancel()` 必须写，否则定时器会在组件卸载后触发 `setState`。

两个 Hook 均通过 `tsc --noEmit --strict` 严格类型检查（需要 `react` 与 `@types/react`）。

## 总结

- 过滤函数的三条设计底线：空条件跳过（别让 `"undefined"` 参与匹配）、数字/布尔按值比较、大小写和匹配方式可配置；泛型约束能把字段名写错挡在编译期。
- 防抖的 API 应该是 `const d = debounce(fn, wait)`，返回函数自带 `cancel`/`flush`，`this` 与参数在 `invoke` 里用 `apply` 透传。
- `leading` 管“立即执行一次”，`trailing` 管“停下来补最后一次”；搜索用默认 trailing，防连点用 leading。
- 节流的时间戳版立即执行但会丢尾，定时器版有尾但首次延迟；选型标准只有一个——想清楚**触发时机**：“停下来才做”用防抖，“过程中按频率做”用节流，“跟渲染帧同步”用 rAF。
- React 里每个组件实例各自持有防抖函数，配合 `fnRef` 和卸载时 `cancel`；需要立即落盘时调用 `flush()`。
