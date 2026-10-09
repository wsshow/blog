---
title: Git 提交规范实战：Conventional Commits 与自动校验
date: 2022-12-07 22:20:30
updated: 2026-10-09
author: ws
description: 不只列 type 表格，讲清格式、示例与 commitlint/husky 落地
categories: ["Git"]
tags: ["Git", "工程化"]
cover: false
---

## 引言

`git log --oneline` 是排查问题、定位回归、生成 CHANGELOG 时最常用的入口，但很多仓库的提交历史根本没法看：`update`、`fix bug`、`111` 混在一起，想知道“这个功能是哪次提交加的”只能一条条点开 diff。本文讲清楚 Conventional Commits 的完整格式（不只是一张 type 表），并用 commitlint + husky 在真实仓库里跑通自动校验，附上完整的命令和输出。适合想在团队里落地提交规范、但又不想一开始就搞得太重的开发者。

## 混乱的提交历史长什么样

先看一段真实生成的提交历史（空提交模拟日常随手的提交）：

```text
$ git log --oneline
99d8edd Merge branch 'dev'
7e79b9c 做完首页嘻嘻
ea7895a 111
0df504f fix bug
974fb89 update
```

这段历史的问题：

- 看不出每次提交改了什么范围、属于功能还是修复；
- 无法用脚本筛选出“所有新功能”或“所有修复”，CHANGELOG 只能手写；
- 回滚时 `git revert 111` 这种命令自己都不好意思敲。

规范提交信息的核心不是“好看”，而是让机器能读懂每次变更的类型和范围，从而自动做三件事：生成 CHANGELOG、决定版本号（feat → minor，fix → patch，BREAKING CHANGE → major）、在 review 时快速判断影响面。

## Conventional Commits 格式拆解

Conventional Commits 1.0.0 的完整格式如下：

```text
<type>(<scope>): <subject>

<body>

<footer>
```

逐段解释：

- **type**：必填，说明提交的类别，只能是约定枚举里的值（见下表）。
- **scope**：可选，说明影响范围，比如 `blog`、`parser`、`api`，团队自行约定。
- **subject**：必填，一句话描述，祈使句、不加句号。
- **body**：可选，解释“为什么这么改”，与 subject 之间空一行。
- **footer**：可选，放 BREAKING CHANGE 说明或 issue 关联，与 body 之间空一行。

### type 枚举与真实示例

| 关键字   | 描述                                     | 示例                                             |
| :------- | :--------------------------------------- | :----------------------------------------------- |
| feat     | 功能新增                                 | `feat(search): 支持按标签过滤文章`               |
| fix      | BUG 修复                                 | `fix: 修复暗色模式切换后代码高亮丢失`            |
| docs     | 文档更新                                 | `docs(readme): 补充本地调试步骤`                 |
| style    | 不影响程序逻辑的代码修改（格式、空格）   | `style: 统一缩进为 2 空格`                       |
| refactor | 重构（既不新增功能，也不修复 BUG）       | `refactor(utils): 抽离防抖逻辑为独立模块`        |
| perf     | 性能、体验优化                           | `perf(list): 用虚拟滚动替代全量渲染`             |
| test     | 新增测试用例或更新现有测试               | `test(filter): 补齐空条件过滤用例`               |
| build    | 修改项目构建系统配置（Makefile、npm 等） | `build: 升级 vite 到 7.x`                        |
| ci       | 项目自动构建流程相关的提交               | `ci: 增加 PR 提交信息校验`                       |
| chore    | 构建过程或辅助工具的变动                 | `chore: 升级依赖并更新 lockfile`                 |
| revert   | 回滚到早前提交                           | `revert: feat(search): 支持按标签过滤文章`       |

`revert` 的 subject 按规范应该写“被回滚提交的头部”，本文最后一节会单独讲这个类型的坑。

### scope 写什么

scope 用来缩小范围，常见约定：

- 按模块：`feat(auth): ...`、`fix(cart): ...`；
- 按包：monorepo 里写包名 `feat(ui): ...`；
- 不确定就不写，不要为了写而写一个 `feat(all): ...`。

团队可以把允许的 scope 写进 commitlint 的 `scope-enum` 规则，但建议起步阶段不要限制 scope，否则维护成本比重构还高。

## body 与 footer

subject 回答“改了什么”，body 回答“为什么改”。body 没有格式要求，但有几个习惯值得遵守：与 subject 空一行；每行控制在 72 字符以内（`git log` 默认排版不会乱）；不要复述 diff，diff 本身已经能看到改动。

footer 主要放两类信息：

```text
feat(api)!: 移除 v1 用户接口

BREAKING CHANGE: /api/v1/user 已下线，请迁移到 /api/v2/user
```

BREAKING CHANGE 有**两种写法**，含义等价：

1. 在 type 或 scope 后加 `!`，如 `feat(api)!: ...`；
2. 在 footer 里写 `BREAKING CHANGE: 说明`（必须大写，`BREAKING-CHANGE` 是等价写法）。

区别在于：`!` 适合“光看头部就知道破坏性”的场景，工具能直接识别；footer 形式可以写较长的迁移说明。两者可以同时出现，但没必要。

关联 issue 也放 footer：

```text
fix(login): 修复 token 过期后死循环刷新

Refs: #128
```

## subject 怎么写

三条实用规则：

1. **祈使句**：读起来要像“应用这个提交后会发生什么”。中文写“修复登录页白屏”，不写“修复了登录页白屏”；英文写 `fix login crash`，不写过去式或第三人称单数（如 `fixes`）。
2. **长度**：50 个字符以内最舒服，超过 72 就会在 `git log` 里被截断。commitlint 的 `header-max-length` 默认 100，可以按团队习惯调小。
3. **结尾不加句号**。commitlint 有专门的 `subject-full-stop` 规则拦这个。

中英文选择上没有技术优劣，只有一致性要求：**一个仓库内统一**。中文团队写全中文完全没问题，commitlint 实测对中文 subject 放行；只有英文 subject 要注意 `config-conventional` 默认的 `subject-case` 规则会拒绝 `Add dark mode` 这种首字母大写写法（下一节会给实测输出）。

一个信息完整的提交长这样：

```text
feat(search): 支持按标签过滤文章

标签数据来自 /tags 页面同源的索引，过滤逻辑放在客户端，
避免每次输入都请求后端。

Closes: #42
```

## 自动校验：commitlint + husky

规范写成文档没人看，最有效的办法是把校验放进提交流程。commitlint 负责判断“消息是否合规”，husky 负责在 `git commit` 时触发 commitlint。下面所有命令都在真实仓库里跑通过（Node 25 + npm 11）。

### 安装

```bash
npm i -D @commitlint/cli @commitlint/config-conventional husky
npx husky init
```

本次实测安装到：`@commitlint/cli@21.2.3`、`@commitlint/config-conventional@21.2.3`、`husky@9.1.7`。

`npx husky init` 会做两件事：在 package.json 写入 `"prepare": "husky"`，并生成 `.husky/pre-commit`（内容默认是 `npm test`）。**注意**：如果你的项目没有 test 脚本或还没配好，这个默认钩子会让每次提交都失败，先删掉或改掉它。

### 配置文件

package.json 里需要出现的部分：

```json
{
  "scripts": {
    "prepare": "husky"
  },
  "devDependencies": {
    "@commitlint/cli": "^21.2.3",
    "@commitlint/config-conventional": "^21.2.3",
    "husky": "^9.1.7"
  }
}
```

`commitlint.config.js`（注意：package.json 里如果声明了 `"type": "module"`，这里要用 ESM 写法）：

```js
module.exports = {
  extends: ['@commitlint/config-conventional'],
  rules: {
    // 中文不受大小写规则影响；英文全小写太苛刻，直接关闭该规则
    'subject-case': [0],
    // 头部最大长度，默认 100，写出来方便团队按需调整
    'header-max-length': [2, 'always', 100],
  },
}
```

`.husky/commit-msg`（husky v9 的钩子文件不需要 shebang，也不需要可执行权限）：

```bash
npx --no -- commitlint --edit "$1"
```

### 实测校验输出

先看一条完全不规范的消息：

```text
$ echo "update" | npx commitlint
⧗   --- input ---
update
✖   subject may not be empty [subject-empty]
✖   type may not be empty [type-empty]

✖   found 2 problems, 0 warnings
ⓘ   Get help: https://github.com/conventional-changelog/commitlint/#what-is-commitlint
```

规范的消息则直接通过，退出码为 0：

```text
$ echo "feat(blog): 支持暗色模式切换" | npx commitlint && echo "(规范通过)"
(规范通过)
```

这里有一个容易踩的坑：`config-conventional` 默认带 `subject-case` 规则，**首字母大写的英文 subject 会被拒绝**：

```text
$ echo "feat: Add dark mode" | npx commitlint
⧗   --- input ---
feat: Add dark mode
✖   subject must not be sentence-case [subject-case]

✖   found 1 problems, 0 warnings
```

所以配置里把 `subject-case` 关掉了；如果你坚持英文首字母大写，这是常见的调整方式。

### 钩子端到端验证

配置好 `.husky/commit-msg` 后，直接用 `git commit` 测试：

```text
$ git commit --allow-empty -m "update"
⧗   --- input ---
update
✖   subject may not be empty [subject-empty]
✖   type may not be empty [type-empty]

✖   found 2 problems, 0 warnings
ⓘ   Get help: https://github.com/conventional-changelog/commitlint/#what-is-commitlint

husky - commit-msg script failed (code 1)
```

不合规的提交被拦下；换一条规范消息就能正常提交：

```text
$ git commit --allow-empty -m "feat(blog): 支持暗色模式切换"
[main 2e3b00e] feat(blog): 支持暗色模式切换
```

### CI 兜底

本地钩子可以被 `--no-verify` 跳过，也可以在没装依赖的机器上完全不生效，所以 CI 里要再校验一遍。对 PR 场景，只校验 PR 范围内的提交即可（全量校验历史大概率会失败，实测对上面那段混乱历史执行 `commitlint --from` 的退出码是 1）：

```yaml
# .github/workflows/commitlint.yml
name: commitlint
on: [pull_request]
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0 # commitlint 需要拿到范围内的提交
      - uses: actions/setup-node@v4
        with:
          node-version: 22
      - run: npm ci
      - run: npx commitlint --from ${{ github.event.pull_request.base.sha }} --to ${{ github.event.pull_request.head.sha }}
```

### commitizen：记不住格式就用交互式提交

commitizen 会把提交过程变成问答：选 type、填 scope、写 subject，最后自动拼出规范消息。

```bash
npm i -D commitizen cz-conventional-changelog
npx commitizen init cz-conventional-changelog --save-dev --save-exact
```

之后通过 `npx cz` 或配置好的 `npm run commit` 提交。package.json 里的关键配置：

```json
{
  "scripts": {
    "commit": "cz"
  },
  "config": {
    "commitizen": {
      "path": "cz-conventional-changelog"
    }
  }
}
```

适合刚开始推规范、或者有大量不熟悉命令行的成员的团队。它和 commitlint 不冲突：commitizen 负责“写得对”，commitlint 负责“兜底检查”。

## 用提交历史生成 CHANGELOG

规范提交的直接收益是自动化版本管理。工具链会解析提交历史里的 type，决定版本号（`feat` → minor，`fix` → patch，`BREAKING CHANGE` → major），并生成 CHANGELOG。

几个工具的选择：

- **release-please**（Google 维护）：在 CI 里根据提交自动创建 release PR，更新 CHANGELOG 和版本号，当前版本 17.x，推荐新项目使用。
- **commit-and-tag-version**：`standard-version` 的社区维护分支，当前 13.x，用法几乎一样，适合本地跑流程。
- **standard-version**：很多老教程还在推荐它，但官方 README 已经写明 `standard-version is deprecated`，并建议改用 release-please。新项目不要再选它。

以 commit-and-tag-version 为例，本地发布流程是：

```bash
npx commit-and-tag-version --release-as minor
```

它会读取自上次 tag 以来的提交，自动写 CHANGELOG、升级 package.json 版本号并打 tag。注意，这一切成立的前提正是提交信息符合 Conventional Commits——`update` 这种消息在解析器眼里不存在。

## revert 提交的格式与坑

`revert` 类型的规范写法是：subject 写被回滚提交的头部，body 说明回滚目标：

```text
revert: feat: add beta feature

This reverts commit b9a992a.
```

但 `git revert` 自动生成的并不长这样。实测（Git 2.54）：

```text
$ git revert --no-edit HEAD
[main 19b9e64] Revert "feat: add beta feature"
```

生成了 `Revert "feat: add beta feature"`。这里有两件事值得知道：

1. commitlint 默认**忽略**以 `Revert `、`Merge `、`fixup!`、semver 版本号等开头的消息（源码里叫 default ignores）。实测 `echo 'Revert "anything"' | npx commitlint` 退出码为 0，不会被拦。
2. 更彻底一点：在测试的 Git 版本下，`git revert` 的自动提交**不会触发 commit-msg 钩子**（换成不依赖 husky 的原生钩子验证过，同样不执行）。

也就是说，靠本地钩子管不住 revert 的提交信息。想要规范覆盖到它，两个办法：`git revert --no-commit <hash>` 后手工 `git commit`，或者干脆依赖 CI 的范围校验。这也是“本地钩子必须配 CI 兜底”的一个具体理由。

## 小团队落地建议

- **第一步只规范 feat/fix**。这两个类型覆盖了 80% 的日常提交，先把它们强制起来，其他类型允许自由使用；scope 可选、长度放宽到 100。
- **不要一上来就全量强制**。`header-max-length` 设成 50、限制 `scope-enum`、打开 `subject-case`，都会让成员在第一周就产生“规范好麻烦”的印象。规则是迭代出来的。
- **配置文件进仓库**。`commitlint.config.js` 和 `.husky/` 一起提交，团队共享同一套规则；本地钩子 + CI 范围校验双保险。
- **中文 subject 放心用**。实测 `feat(blog): 支持暗色模式切换` 直接通过；中英混写时统一专有名词的大小写即可。
- **定期回看收益**。规范跑上一两个月后，`git log --oneline` 可以直接当变更清单用，配合 release-please 还能自动积累 CHANGELOG。

## 总结

- Conventional Commits 的格式是 `<type>(<scope>): <subject>` + 可选 body/footer，type 是枚举，subject 用祈使句、不加句号。
- BREAKING CHANGE 有两种等价写法：`!` 和 footer 里的 `BREAKING CHANGE:`，后者可以附带迁移说明。
- commitlint + husky 的落地是三步：装依赖、写 `commitlint.config.js`、把 `npx --no -- commitlint --edit "$1"` 写进 `.husky/commit-msg`。实测坏消息被拦、好消息通过。
- `config-conventional` 默认拒绝首字母大写的英文 subject，中文不受影响；关闭 `subject-case` 即可兼容两种习惯。
- 本地钩子管不住 revert（git 不触发钩子）也防不了 `--no-verify`，CI 范围校验是必要兜底。
- standard-version 已被官方标记 deprecated，新项目用 release-please 或 commit-and-tag-version。
