---
title: JavaScript 代码混淆实战：javascript-obfuscator 配置与边界
date: 2024-01-01 21:00:30
updated: 2026-10-09
author: ws
description: 混淆能防到什么程度，哪些配置值得开、代价是什么
categories: ["JavaScript"]
tags: ["JavaScript", "安全"]
cover:
---

## 引言

前端代码只要发送到浏览器，对方就拥有全部字节，“加密”在前端是个伪命题。javascript-obfuscator 是同类工具里最常用的一个，但很多人开启它时并不清楚每个配置的代价：体积涨十倍、热路径慢三十倍、线上报错定位不到源码。本文用 5.9.0 版本实际跑一遍混淆、性能基准和 source map 调试，并回答一个更实际的问题——什么值得混淆，什么应该干脆放到后端。适合准备给线上产物加混淆、或正在评估混淆收益的开发者。

## 先澄清：混淆不是加密

混淆只做一件事：把代码改写成人类难以阅读的形式，同时保证运行结果不变。浏览器必须能执行它，就意味着字符串、分支、常量在运行时都会还原。攻击者不需要“破解”，只要让代码跑起来，就能在运行时拿到原始信息。

```text
源码
  │  javascript-obfuscator
  ▼
不可读代码（标识符重命名、字符串搬家、控制流打乱）
  │  浏览器加载执行
  ▼
解码函数在运行时还原字符串/常量
  │
  ▼
内存里仍是明文数据（可直接调试、HOOK、抓包）
```

所以三个东西不要放进前端：“核心算法”、“密钥/令牌的生成规则”、“决定权限的校验逻辑”。混淆能提高的是阅读和二次修改的成本，挡不住决心足够的人在运行时把逻辑摸出来。

## 安装与最小用法

```bash
npm i javascript-obfuscator    # 本文实测版本 5.9.0
```

命令行方式：

```bash
npx javascript-obfuscator src/calc.js --output dist/calc.obf.js
```

Node API 方式（构建脚本里更常用）：

```js
const JavaScriptObfuscator = require('javascript-obfuscator')

const result = JavaScriptObfuscator.obfuscate(source, {
  compact: true, // 默认就是 true，输出压缩到单行
})

const code = result.getObfuscatedCode()
```

注意 `obfuscate()` 接收的是源码字符串，不是文件路径；`stringArray` 默认开启，所以即使不写任何配置，字符串也已经被搬进数组。

## 先看一份“开关全开”的脚本

一个常见的误区是“把选项全打开就更安全”，于是混淆脚本往往写成这样：

```javascript
const JavaScriptObfuscator = require('javascript-obfuscator');
const fs = require('fs');

let inputContent = '';
try {
  console.log('read file...');
  inputContent = fs.readFileSync('./index.js', 'utf8');
  console.log('read file success');
} catch (err) {
  console.error(err);
  process.exit(1);
}

console.log('obfuscating...');
const obfuscationResult = JavaScriptObfuscator.obfuscate(inputContent,
                                                         {
                                                           compact: true,
                                                           controlFlowFlattening: true,
                                                           controlFlowFlatteningThreshold: 1,
                                                           numbersToExpressions: true,
                                                           simplify: true,
                                                           stringArrayShuffle: true,
                                                           splitStrings: true,
                                                           stringArrayThreshold: 1,
                                                           log: false,
                                                           debugProtection: true,
                                                           disableConsoleOutput: true
                                                         }
                                                        );
console.log('obfuscating success');

console.log('writing file...');
const outContent = obfuscationResult.getObfuscatedCode();
fs.writeFile('./index-d.js', outContent, err => {
  if (err) {
    console.error(err);
    return
  }
  console.log('file written successfully');
});
```

每个配置项的含义：

| 配置项 | 作用 | 默认值 |
| :----- | :--- | :----- |
| `compact` | 压缩空白，输出单行 | `true` |
| `controlFlowFlattening` + `Threshold: 1` | 控制流扁平化，把分支改写成状态机；`1` 表示所有函数都处理 | `true` + `0.75` |
| `numbersToExpressions` | 数字改写成等价表达式 | `false` |
| `simplify` | 等价简化（如 `true` → `!![]`） | `true` |
| `stringArrayShuffle` | 字符串数组顺序打乱 | `true` |
| `splitStrings` | 字符串切成小块拼接 | `false` |
| `stringArrayThreshold: 1` | 所有字符串都进数组 | `0.75` |
| `log: false` | 不打印内部日志 | `false` |
| `debugProtection` | 反调试：检测 devtools 后不断触发 `debugger` | `false` |
| `disableConsoleOutput` | 屏蔽全局 `console` 方法 | `false` |

这段脚本能跑，但埋着六个隐患：

1. **输入输出写死**，只处理 `./index.js`，多文件项目要么手改路径要么循环调用；
2. **没开 `stringArrayEncoding`**，字符串数组里是明文（下一节有实证），防的是“懒得读”而不是“读不到”；
3. **`debugProtection` 是浏览器场景的反调试**，和构建产物兼容性、线上稳定性都要额外评估，不适合无脑开；
4. **`disableConsoleOutput` 会全局屏蔽 `console`**，线上错误收集和埋点日志一起遭殃（也有实证）；
5. **没有 `seed`**，每次构建产物都不同，无法复现、无法对比 diff；
6. **没有 source map**，线上报错只能看到 `bug.obf.js:1:1222`，排错成本极高。

## 推荐配置：一份可复现的构建脚本

推荐的实现（本文实际使用的脚本），支持三种预设：

```js
// 用法：node obfuscate.js <输入文件> <输出文件> [preset]
// preset: mild | balanced | naive，默认 balanced
const JavaScriptObfuscator = require('javascript-obfuscator')
const fs = require('fs')
const path = require('path')

const [, , input = './src/calc.js', output = './dist/calc.obf.js', preset = 'balanced'] =
  process.argv

const presets = {
  // 温和：只做字符串数组 + 压缩，体积和性能损失都小
  mild: {
    compact: true,
    simplify: true,
    stringArray: true,
    stringArrayThreshold: 1,
    stringArrayShuffle: true,
    rotateStringArray: true,
    stringArrayEncoding: ['base64'],
  },
  // 常见生产配置：加控制流扁平化、死代码、自保护
  balanced: {
    compact: true,
    simplify: true,
    stringArray: true,
    stringArrayThreshold: 1,
    stringArrayShuffle: true,
    rotateStringArray: true,
    stringArrayEncoding: ['base64'],
    splitStrings: true,
    splitStringsChunkLength: 10,
    controlFlowFlattening: true,
    controlFlowFlatteningThreshold: 0.75,
    deadCodeInjection: true,
    deadCodeInjectionThreshold: 0.3,
    numbersToExpressions: true,
    selfDefending: true,
    seed: 20240101, // 相同 seed + 相同源码 => 相同产物，便于复现构建
    sourceMap: false,
  },
  // 不假思索的“全开”配置：用于观察 debugProtection/disableConsoleOutput 的副作用
  naive: {
    compact: true,
    controlFlowFlattening: true,
    controlFlowFlatteningThreshold: 1,
    numbersToExpressions: true,
    simplify: true,
    stringArrayShuffle: true,
    splitStrings: true,
    stringArrayThreshold: 1,
    log: false,
    debugProtection: true,
    disableConsoleOutput: true,
  },
}

const config = presets[preset]
if (!config) throw new Error(`未知 preset: ${preset}（可选 ${Object.keys(presets).join(' / ')}）`)

const source = fs.readFileSync(input, 'utf8')
const result = JavaScriptObfuscator.obfuscate(source, config)

fs.mkdirSync(path.dirname(output), { recursive: true })
fs.writeFileSync(output, result.getObfuscatedCode())

const inSize = Buffer.byteLength(source)
const outSize = Buffer.byteLength(result.getObfuscatedCode())
console.log(
  `[${preset}] ${input} -> ${output}  ${inSize}B => ${outSize}B (${(outSize / inSize).toFixed(1)}x)`,
)
```

推荐脚本补充了几个关键项，逐条说明：

- `stringArrayEncoding: ['base64']`：字符串进数组后额外编码。可选 `'base64'`、`'rc4'`；`rc4` 更难读、更慢、体积更大（实测数据见下）。不开则是明文数组。
- `rotateStringArray: true`（默认值）：构建时把字符串数组旋转若干位，运行时先自转还原，防止直接读数组下标。
- `deadCodeInjection` + `Threshold: 0.3`：注入永不执行的垃圾分支，阈值控制比例，代价是体积。
- `selfDefending`：产物被格式化/二次压缩时自毁（要求 `compact: true`）。副作用是调试和生产改代码都要小心。
- `seed`：固定后相同输入得到相同产物；不写则默认每次随机（实测验证见下）。
- `sourceMap` 系列：下一节单独讲。

## 真实跑一遍

### 演示源码

用一段带字符串和分支的电商价格计算函数，最贴近真实业务：

```javascript
/**
 * 电商价格计算：有字符串常量、有分支、有循环，
 * 贴近真实业务函数，适合观察混淆前后的差别。
 */
function calculatePrice(user, items) {
  let total = 0
  for (let i = 0; i < items.length; i++) {
    const item = items[i]
    let price = item.price
    if (item.type === 'digital') {
      price = price * 0.9
    } else if (item.type === 'book') {
      price = price * 0.85
    } else if (item.type === 'food') {
      price = price * 0.95
    }
    total += price * item.quantity
  }
  if (user.level === 'vip') total = total * 0.95
  if (user.coupon === 'WELCOME2024') total = total - 20
  if (total < 0) total = 0
  return Math.round(total * 100) / 100
}

function formatPrice(price) {
  return '¥' + price.toFixed(2)
}

module.exports = { calculatePrice, formatPrice }
```

用最小配置（只开字符串数组）混淆一个更小的函数来观察结构：

```javascript
function greet(name) {
  if (name === 'admin') {
    return '你好，管理员'
  }
  return '你好，' + name
}
console.log(greet('admin'))
```

### 混淆前后对比

产物是**一整行**（`compact: true`），下面是截取的关键片段。混淆前的 `greet` 函数变成：

```javascript
function greet(_0x1f563c){var _0x278d0a=_0x298c;if(_0x1f563c===_0x278d0a(0xf9))return _0x278d0a(0xfa);return _0x278d0a(0xf0)+_0x1f563c;}
```

`console.log` 和调用参数也被拆掉：

```javascript
console[_0x1e5a02(0xee)](greet(_0x1e5a02(0xf9)));
```

所有字符串被搬进一个数组，由 `_0x298c` 这类“解码函数”按索引取：

```javascript
var _0x4b7cb1=['log','67455BQCreU','你好，','2048940YeJeHo','2cLoLwJ','27408aROuhk','1258569MARlCE','247952LpdTqU','731275ySpjpI','679rwUJmM','1988904YbEnOP','admin','你好，管理员'];
```

看清了：**默认的字符串数组只是“搬家”，没有加密**——`'你好，管理员'` 还在数组里躺着。中间那些 `67455BQCreU` 之类的乱码是数组旋转用的干扰项。要真正挡住一眼可见，需要加编码选项：

```javascript
// stringArrayEncoding: ['base64'] 之后，同一个字符串变成：
'5l2G5Aw977Ym566H55cg5zgy'
```

这个 base64 用的是**大小写互换过的字母表**，不是标准 base64；按标准解码会失败。把字符映射回标准字母表再解码就能还原（实测）：

```text
映射回标准 base64: 5L2g5aW977yM566h55CG5ZGY
解码结果: 你好，管理员
```

产物里同时注入了对应的解码函数，运行时把字符串还原后使用。也就是说：**想还原只需要跑一遍代码**，这再次说明混淆的上限。

### 体积代价

同一份 823 字节的源码，三种预设的真实产物大小：

| 预设 | 产物大小 | 相对源码 | gzip 后 |
| :--- | :------- | :------- | :------ |
| 原始源码 | 823B | 1.0x | 512B |
| mild（字符串数组 + base64） | 3224B | 3.9x | 1365B |
| balanced（推荐配置） | 8490B | 10.3x | 3475B |
| naive（全开配置） | 10273B | 12.5x | 3596B |

gzip 能抹平一部分膨胀（乱码字符串压缩率很高），但下载体积仍会膨胀到源码的 2.7~7 倍。对首屏敏感的项目，这一项要先算账。

## 性能代价：每项配置值多少钱

基准方法：固定数据调用 `calculatePrice` 50 万次，预热 5000 次后计时，每个变体单独进程运行；四个变体的校验和完全一致（`498635000.00`），说明混淆没有改变语义。源码原始函数基准约 7.5ms。

| 变体 | 50 万次耗时 | 相对原始 | 体积 |
| :--- | :---------- | :------- | :--- |
| 原始源码 | 7.3~7.6ms | 1x | 823B |
| 只加明文字符串数组 | 12.1ms | 1.6x | 2092B |
| 只加控制流扁平化（阈值 1） | 27.0ms | 3.6x | 3213B |
| 只加 base64 字符串编码 | 66.1ms | 8.8x | 3157B |
| 只加 rc4 字符串编码 | 100.1ms | 13.3x | 4698B |
| mild（base64 组合） | ~68ms | 9x | 3224B |
| naive（全开配置） | ~37ms | 5x | 10273B |
| balanced（推荐配置） | ~245ms | 33x | 8490B |

三条结论：

1. **字符串编码是最大单项开销**。base64 每次访问字符串都要解码，rc4 更慢；如果热路径代码字符串访问频繁，这两项要谨慎。
2. **控制流扁平化其次**，阈值 1（所有函数）约 3.6 倍；生产上把阈值降到 0.5~0.75 性价比更高。
3. **组合配置的开销会放大**。balanced 里 base64 + 扁平化 + 死代码 + selfDefending 叠加后达到 33 倍；而 naive 反而“只”有 5 倍，因为它没开字符串编码，也说明了开销大头在哪里。

这些数字来自函数级微基准，真实项目里耗时往往被网络和渲染掩盖；但任何被高频调用的纯计算函数（排序、解析、加解密）都应该用实际数据先跑基准再决定。

### seed 可复现性

固定 `seed` 后连续构建两次，产物哈希一致；不写 `seed` 则两次不同：

```text
固定 seed 两次: e5731397d1cca40c e5731397d1cca40c 一致
不写 seed 两次: 612ce0be306fc6e7 d9753f1b9c6da628 不一致
```

需要产物对比、增量发布、缓存命中的项目务必固定 `seed`。

## 调试与排错

### disableConsoleOutput 的真实副作用

实测一份打开了 `disableConsoleOutput` 的产物，在 Node 下运行：

```text
$ node check-naive.js
result=125.11（这行来自 process.stdout.write）

# 脚本里的 console.log('这行来自 console.log，还看得到吗？') 没有任何输出
```

`console.log` 被全局替换成空函数，`process.stdout.write` 不受影响。危害在于：错误监控 SDK 如果走 `console.error` 上报，会直接失联。两个应对方式：

- 不要开 `disableConsoleOutput`，只混淆代码本身；
- 或者在混淆入口之外的独立模块里注册全局错误处理：

```javascript
// 在混淆范围之外的入口文件注册，保证上报通道不被屏蔽
window.addEventListener('error', (event) => {
  navigator.sendBeacon(
    '/api/error',
    JSON.stringify({ message: event.message, stack: event.error?.stack, at: Date.now() }),
  )
})
```

### source map：能不能定位线上错误

开启方式：

```js
const result = JavaScriptObfuscator.obfuscate(source, {
  compact: true,
  stringArray: true,
  stringArrayThreshold: 1,
  sourceMap: true,
  sourceMapMode: 'separate',
  // sources 字段要求手动指定原始文件名（Node API 下）
  sourceMapSourcesMode: 'sources',
  inputFileName: '../src/bug.js',
})

// separate 模式不会自动追加 sourceMappingURL 注释，需要手动写
const codeWithMap = result.getObfuscatedCode() + '\n//# sourceMappingURL=bug.obf.js.map\n'
fs.writeFileSync('./dist/bug.obf.js', codeWithMap)
fs.writeFileSync('./dist/bug.obf.js.map', result.getSourceMap())
```

两个容易踩的点：Node API 的 `separate` 模式**不会自动在产物末尾追加** `sourceMappingURL` 注释（CLI 会），要自己拼；而 `sources` 字段想显示真实文件名，必须设置 `sourceMapSourcesMode: 'sources'` 并给出 `inputFileName`。

实测一个会抛错的函数，对比堆栈：

```text
# 不开 source map
Error: input is required
    at Object.buggy (.../dist/bug.obf.js:1:1222)

# node --enable-source-maps 加载带 map 的产物
Error: input is required
    at Object.buggy (.../src/bug.js:3:11)
```

堆栈从“混淆文件第 1 行第 1222 列”精确回到“源码第 3 行”。取舍也很明确：**.map 文件绝不能公开部署**，它等于把源码送回去；正确做法是构建时上传到错误监控平台（Sentry 等），线上只发布 js。inline 模式会把 map 塞进 js 里，等于公开，禁用。

## 与 Terser 压缩的区别

| 维度 | Terser | javascript-obfuscator |
| :--- | :----- | :-------------------- |
| 目的 | 减小体积 | 降低可读性 |
| 标识符 | 重命名为短名 | 重命名为无意义十六进制名 |
| 字符串 | 原样保留 | 搬进数组，可编码/拆分 |
| 控制流 | 不动 | 可扁平化成状态机 |
| 产物体积 | 显著变小 | 显著变大（实测 4~12 倍） |
| 性能 | 基本无损，有时更快 | 明显变慢（热路径可达数十倍） |
| 报错定位 | 开 map 即可 | 需要 map + 额外配置 |
| 反格式化 | 无 | `selfDefending` 可自毁 |

两者不是替代关系：现代构建工具默认已经用 Terser/SWC 压缩，混淆是在压缩之后额外加的一层。想省体积就停在这层，想加阅读成本再往上叠。

## 什么时候别用

- **开源项目**：来源可读是基本礼仪，混淆只会劝退贡献者；
- **纯工具/内部系统**：攻击者拿不到产物，混淆只有维护成本；
- **性能敏感路径**：高频计算、动画主循环上应用了混淆可能直接掉帧；
- **需要清晰错误堆栈的调试期**：源码映射链再补一层，排查成本翻倍；
- **小团队无人维护构建脚本**：锁不住版本、每次升级都要回归验证，收益远小于成本。

## 更靠谱的保护策略

按投入产出排序：

1. **核心逻辑后端化**：算法、定价、权限判断放服务器，前端只提交参数和展示结果；
2. **服务端校验一切**：前端校验只做体验，权限、频控、金额、签名必须在服务端二次验证；
3. **敏感数据不落地**：密钥、盐值、token 生成规则从前端代码里彻底拿走；
4. **WASM 提高门槛**：计算密集且确实要放前端的逻辑编译成 WASM，反编译成本高于 JS，但不是不可读；
5. **License 校验 + 服务端验证**：客户端只做“提示未授权”，真正的功能开关由服务端下发的授权验证决定；
6. **埋点与风控**：监控异常的批量请求、爬虫特征，用运行时的行为对抗运行时的问题。

混淆是这些策略里最外层的一件“防锈漆”，可以刷，但不要指望它承重。

## 总结

- 混淆不是加密：它把字符串和分支搬到运行时还原，攻击者跑一遍就能拿到明文；核心机密必须放后端。
- “开关全开”的脚本有六个隐患：路径写死、字符串明文、`debugProtection` 副作用、`disableConsoleOutput` 屏蔽日志、无 `seed`、无 source map，每一项都要单独权衡。
- 字符串编码（base64 8.8 倍 / rc4 13.3 倍）是最大单项性能开销，控制流扁平化其次是 3.6 倍，全开组合可达 33 倍；资源敏感场景把 `controlFlowFlatteningThreshold` 降下来、不要 base64。
- 体积实测膨胀 3.9~12.5 倍（gzip 后 2.7~7 倍），首屏敏感项目要提前评估。
- `seed` 保证可复现构建；source map 让线上堆栈回到源码行，但 `.map` 绝不能公开部署。
- 该不该混淆取决于威胁模型：内部系统与开源项目通常不值得，需要保护商业逻辑时，后端化 + 服务端校验永远比混淆可靠。
