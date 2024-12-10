---
title: js文件加密
date: 2024-01-01 21:00:30
author: ws
description: 使用Obfuscator.js对js文件进行混淆加密
categories: js
tags: [js, 文件加密]
cover:  
---

## 加密脚本

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

## tips

使用前需要修改脚本输入和输出位置