---
title: Agent Skills 完全指南：从 SKILL.md 到 OpenCode / Claude Code 实战
date: 2026-10-09 11:20:00
author: ws
description: 讲清 Agent Skills 的开放标准与渐进式披露机制，并用一个完整示例演示如何给 AI agent 编写可复用的技能
categories: ["AI"]
tags: ["Agent", "Skill", "OpenCode", "Claude Code", "MCP"]
cover:
---

## 引子：每个 agent 用户迟早都会遇到的循环

用 AI agent 处理稍复杂一点的工作，很快就会陷入一个循环：

```text
新会话 → 把流程重新讲一遍 → 干得不错 → 下次新会话 → 再讲一遍……
```

比如“准备发布”这件事，你要告诉它：先跑测试，再按 git 历史写 release notes，版本号按 SemVer 规则升，破坏性变更要单独标出来。讲完一遍，它做得很好；换个会话，这些上下文全部归零。把流程写进 `CLAUDE.md` / `AGENTS.md` 能解决“忘”，但新的问题来了：这些文件是**常驻上下文**的，写得越多，agent 每轮对话都要背着它们，真正关键的信息反而被稀释。

Agent Skills 就是为这类问题设计的：**把流程、领域知识和可执行脚本打包成一个文件夹，agent 需要时才加载**。它最早由 Anthropic 在 2025 年 10 月 16 日发布，同年 12 月 18 日成为开放标准；到 2026 年，agentskills.io 的客户端展示页已经列出 40 多个支持产品，包括 Claude Code、ChatGPT & Codex、GitHub Copilot、VS Code、Cursor、Gemini CLI、OpenCode 等。

这篇文章会讲清楚四件事：skill 是什么、渐进式披露是怎么工作的、为什么它对 agent 特别重要，以及如何从零写一个真正能被触发、可靠执行的 skill。文中所有示例都在本机实际运行过，命令和输出一并给出。

## 什么是 Agent Skills

Anthropic 的定义很朴素：**skill 是一个装着指令、脚本和资源的文件夹，agent 可以发现并动态加载它，从而更好地完成特定任务**。官方的比喻是“给新员工准备入职指南”——新员工（agent）有通用的工作能力，但不知道你们团队怎么做发布、怎么填那张 PDF 表单，入职指南把这些补上。

一个 skill 的目录结构如下：

```text
release-notes/
├── SKILL.md          # 必需：元数据 + 指令
├── scripts/          # 可选：可执行代码
├── references/       # 可选：按需加载的文档
├── assets/           # 可选：模板、图片等静态资源
└── ...               # 其他任何文件或目录
```

`SKILL.md` 由两部分组成：开头的 YAML frontmatter（元数据）和后面的 Markdown 正文（指令）。frontmatter 至少要写 `name` 和 `description`：

```markdown
---
name: release-notes
description: 根据 git 提交历史生成面向用户的发布说明，并给出语义化版本号建议。当用户提到发布说明、release notes、CHANGELOG、版本号、打 tag 时使用。
---

# 发布说明生成

## 工作流程

1. 运行 `python3 scripts/collect_commits.py` 收集提交。
2. 按 `references/style-guide.md` 的规则润色条目。
3. 输出草稿，等用户确认后再改文件。
```

整个格式由 [agentskills.io](https://agentskills.io/specification) 上的开放规范定义，必填字段和约束如下：

| 字段 | 必需 | 约束 |
| --- | --- | --- |
| `name` | 是 | 1–64 字符；只能用小写字母、数字和连字符；不能以连字符开头或结尾，不能有连续连字符；必须与所在目录同名 |
| `description` | 是 | 1–1024 字符；同时说清“做什么”和“什么时候用” |
| `license` | 否 | 许可证名称，或指向目录内的许可证文件 |
| `compatibility` | 否 | 最长 500 字符，声明环境要求，比如依赖的命令、网络访问 |
| `metadata` | 否 | 自定义键值对（字符串到字符串） |
| `allowed-tools` | 否 | 预授权工具列表（实验性字段，各客户端支持程度不同） |

注意 `name` 与目录同名这一条：目录叫 `release-notes`，frontmatter 里的 `name` 就必须是 `release-notes`，不能写 “Release Notes”。这是为了让所有客户端都能用同一套规则定位和引用 skill。

## 核心机制：渐进式披露

Skill 能在实践中站住脚，靠的不是格式简单，而是 **Progressive Disclosure（渐进式披露）**——信息分三层，按需加载：

| 层级 | 加载内容 | 加载时机 | 成本 |
| --- | --- | --- | --- |
| 第 1 层：目录 | 每个 skill 的 `name` + `description` | 会话启动时 | 约 50–100 token/skill |
| 第 2 层：指令 | 完整 `SKILL.md` 正文 | 任务匹配、skill 被激活时 | 建议 <5000 token（约 500 行） |
| 第 3 层：资源 | `scripts/`、`references/`、`assets/` 里的文件 | 指令显式引用时 | 按需，多少都行 |

一次典型的触发过程，上下文窗口是这样变化的：

```text
会话开始
┌─────────────────────────────────────────────┐
│ 系统提示 + 用户消息                          │
│ + available_skills 目录：                    │
│   release-notes: 根据 git 历史生成发布说明…  │  ← 只到这里，约 100 token
│   pdf-processing: 提取 PDF 文本、填表单…     │
│   data-analysis: 分析数据集、生成图表…       │
└─────────────────────────────────────────────┘
                 │ 用户："帮我写这个版本的 release notes"
                 ▼
┌─────────────────────────────────────────────┐
│ 上述内容 + release-notes/SKILL.md 全文       │  ← 第 2 层：指令进上下文
│ （frontmatter 通常被剥掉，只有正文）          │
└─────────────────────────────────────────────┘
                 │ 指令说"按 references/style-guide.md 润色"
                 ▼
┌─────────────────────────────────────────────┐
│ 上述内容 + style-guide.md（只有需要时才读）   │  ← 第 3 层：资源按需加载
└─────────────────────────────────────────────┘
```

三个数字值得记住：

- **目录很便宜**：每个 skill 约 50–100 token 的元数据，装 20 个 skill 启动成本只有一两千 token，agent 却“知道”自己有 20 项技能。
- **正文有建议上限**：规范建议 `SKILL.md` 控制在 500 行、5000 token 以内——这是每次触发都要支付的固定成本。
- **资源几乎无上限**：被引用的文件只在真正需要时读取。Anthropic 的原话是，“能打包进一个 skill 的上下文，规模实际上是无限的”。

对比一下“把一切都写进系统提示”的方案：N 个流程的常驻成本是 O(N)，而且随着你积累的经验越来越多，提示词只会越来越臃肿。渐进式披露把启动成本降到 O(N × 一行描述)，把执行成本降到 O(本次用到的那几个)——这就是它成为整个标准核心设计的原因。

## 为什么 skill 对 agent 很重要

### 1. 上下文是 agent 最稀缺的资源

Agent 的能力上限不只取决于模型，还取决于它每一轮能看到什么。系统提示、工具定义、历史对话、检索结果都在抢同一扇窗口。Skill 把“完整的操作手册”从常驻改为按需，收益在真实场景里非常可观：社区项目 [hello-agents](https://github.com/datawhalechina/hello-agents) 记录过一个案例——直接挂载一个工具定义庞大的 MCP 服务，启动占用约 16000 token；套一层 skill 作为“网关”，只在目录里描述能力、真正需要时再调用底层 MCP，启动开销降到约 500 token。这个数字来自社区实践、不是官方基准，但它说明了机制的价值。

### 2. 程序性知识第一次有了标准载体

模型权重里有的是通用知识，缺的是“我们这里怎么做”。你的发布流程、数据表的坑、审查清单、接口约定，这些属于**程序性知识**（procedural knowledge），过去只有两个去处：

- 每次在对话里重新讲（不可复用、依赖记忆）；
- 写进提示词或微调进权重（要么常驻占上下文，要么成本高、更新慢）。

Skill 提供了第三个去处：外置的、可读的、可改的文件。更新一个流程只需要改一个 Markdown 文件，下一次会话立即生效，不需要训练、不需要发布新版本。

### 3. 指令和代码可以放在同一个包里

有些操作让模型“用 token 生成”既慢又不稳定：排序、解析 PDF 表单、批量重命名、生成统计图。Skill 允许把这些写成 `scripts/` 下的普通脚本，agent 直接执行：

- 结果确定、可重复，不受采样随机性影响；
- 脚本本身不用进上下文，只有它的输出会进；
- 脚本可以带完整错误信息，方便 agent 自我纠错。

这也是 Anthropic 在发布文章里强调的点：LLM 擅长很多事，但排序这种事跑一段代码就是比生成 token 更便宜、更可靠。

### 4. 一次编写，跨产品复用

2025 年 12 月 18 日，Agent Skills 被发布为开放标准。同一份 skill 目录，在 Claude Code、ChatGPT & Codex、GitHub Copilot、VS Code、Cursor、Gemini CLI、OpenCode 等兼容客户端里都能用——和 MCP 解决“工具互操作”是同一个思路：**标准把 M×N 的适配问题变成了 M+N**。

跨客户端还有一个具体约定：`~/.agents/skills/` 和 `<project>/.agents/skills/` 正在成为跨客户端共享 skill 的目录，很多客户端会同时扫描它们和自家目录。

### 5. 让团队知识变成可版本化的资产

Skill 就是普通文件，这意味着它可以获得软件工程的一切基础设施：

- 进 git，随代码仓库分发，code review 时一起审；
- 项目级 skill 提交到仓库后，所有人（以及所有兼容的 agent）自动获得同一套流程；
- 团队可以用插件、市场、HTTP 目录等方式集中分发；企业可以统一管理；
- 有官方校验工具（`skills-ref`）保证格式正确。

过去的经验散落在文档、聊天记录和老员工脑子里；现在它可以像代码一样被维护。

### 6. 可以评测、可以迭代

Skill 是明确的产物：写完之后可以拿真实任务测——哪些该触发没触发（漏报）、哪些不该触发乱触发（误报）、哪一步 agent 走偏了。根据执行轨迹（而不只是最终输出）修改指令，再测，形成闭环。这种可评测性让“给 agent 加能力”从玄学变成工程。

## 辨析：Skill 与其他几种机制的关系

初次接触很容易把 skill 和 MCP、和提示词、和斜杠命令搞混。它们解决的是不同层次的问题：

| 机制 | 解决的问题 | 与 skill 的关系 |
| --- | --- | --- |
| MCP | 连接外部工具和数据源 | 接口层；skill 是“怎么用这些工具干活”的操作层，二者互补 |
| 系统提示 / `CLAUDE.md` / `AGENTS.md` | 常驻的事实与规则 | 常驻 vs 按需：事实放常驻文件，流程放 skill |
| 斜杠命令 | 用户手动触发的固定入口 | Claude Code 已把自定义命令合并进 skill；skill 额外支持自动触发、附属文件、权限控制 |
| 子代理（subagent） | 用独立上下文执行复杂任务 | 部分客户端支持 skill 在子代理里运行（如 Claude Code 的 `context: fork`） |
| RAG | 检索事实性知识 | 路径不同：RAG 走向量检索，skill 走文件系统按需读取 |
| 微调 | 把行为写进模型权重 | skill 外置、可读、可改、跨模型、即时生效 |

理解它们分工的一个好例子是“skill 网关”：你有一个工具很多、定义很大的 MCP 服务，直接挂着很贵；写一个 skill 描述“什么时候应该用这个服务、第一步查什么、常见参数怎么传”，agent 先读 skill，真正要调用时再用 MCP 工具。外部能力还是 MCP 提供的，调用流程由 skill 提供。

## 实战：从零写一个 release-notes skill

下面用一个真实可用的例子走完整流程。目标：让 agent 在“准备发布”时自动按团队规则生成发布说明，并给出 SemVer 版本号建议。

### 第 1 步：从真实任务里提炼

规范里的第一条最佳实践是：**不要凭空让模型编 skill**，要先在真实任务中获得经验，再把它提取出来。具体观察四样东西：

1. 哪些步骤是必须的（跑脚本收集提交 → 按规则润色 → 校验版本号）；
2. 你在哪些地方纠正过 agent（“不规范的提交不要臆测类型”“先给草稿不要直接改文件”）；
3. 输入输出长什么样（Conventional Commits → 分组的 Markdown 清单）；
4. 你补充了哪些它不知道的上下文（团队的写作风格、版本号规则）。

这些纠正是 skill 里最值钱的部分——它们会成为正文的流程和“注意事项”。

### 第 2 步：搭目录

```bash
mkdir -p ~/.agents/skills/release-notes/{scripts,references}
```

这里直接使用跨客户端的 `.agents/skills` 约定。目录结构：

```text
release-notes/
├── SKILL.md
├── scripts/
│   └── collect_commits.py
└── references/
    └── style-guide.md
```

### 第 3 步：写 SKILL.md

````markdown
---
name: release-notes
description: 根据 git 提交历史生成面向用户的发布说明，并给出语义化版本号（SemVer）建议。当用户提到发布说明、release notes、CHANGELOG、版本号、打 tag 或准备发布时使用。
---

# 发布说明生成

## 工作流程

1. 收集提交：运行 `python3 scripts/collect_commits.py --from <上一个 tag>`。
   不传 `--from` 时脚本自动取最近一个 tag。
2. 阅读 `references/style-guide.md`，按其中的规则润色条目。
3. 输出草稿：先给出**版本号建议**和**发布说明草稿**，等待用户确认。
4. 用户确认后，才更新 `CHANGELOG.md`、执行 `git tag` 等写操作。

## 输出格式

```markdown
## v1.4.0 建议版本号：minor

### 新功能
- api: 支持按标签过滤订单

### 问题修复
- web: 修复 Safari 下表头错位
```

## 注意事项

- 提交信息不规范（不符合 Conventional Commits）时，不要臆测类型，放进"其他"一节。
- 破坏性变更必须单独成节并置顶，让用户先看到。
- 发布说明面向使用者，不写"重构了 XX 函数"这类内部实现细节；确有必要时说明它带来的行为变化。
- 不要在未确认版本号前修改任何文件或打 tag。
````

几个写作要点：

- **`description` 是触发器**。模型就是靠它判断“这个任务要不要加载这个 skill”，所以要同时写清能力（生成发布说明、建议版本号）和触发场景（提到 release notes、CHANGELOG、打 tag……）。用户怎么说，就写进描述里。
- **流程分步、可检查**。四步里有两处明确的门槛（等确认、才写文件），避免 agent 自作主张。
- **约束写在正文里**。像“不规范的提交不要臆测类型”这种纠正，来自真实执行中踩过的坑，是比通用建议价值高得多的内容。
- **正文保持精简**。规范建议 500 行以内，这里只有几十行——详细的写作规则被放进了 `references/`。

### 第 4 步：把确定性工作交给脚本

`scripts/collect_commits.py` 负责最机械的部分：取提交、按类型分组、输出 Markdown。参数解析和输出都尽量简单，让 agent 容易调用、容易看懂结果：

```python
#!/usr/bin/env python3
"""收集版本区间内的提交，按 Conventional Commits 类型归类输出 Markdown。

用法:
    python3 collect_commits.py                 # 最近一个 tag 到 HEAD
    python3 collect_commits.py --from v1.2.0   # 指定起点（不含）
    python3 collect_commits.py --from v1.2.0 --to v1.3.0
"""
import argparse
import re
import subprocess
import sys

PATTERN = re.compile(
    r"^(?P<type>[a-z]+)(?:\((?P<scope>[^)]+)\))?(?P<breaking>!)?:\s*(?P<subject>.+)$"
)

GROUPS = [
    ("feat", "新功能"),
    ("fix", "问题修复"),
    ("perf", "性能优化"),
]


def run_git(*args):
    result = subprocess.run(["git", *args], capture_output=True, text=True)
    if result.returncode != 0:
        sys.exit("git %s 失败: %s" % (" ".join(args), result.stderr.strip()))
    return result.stdout


def latest_tag():
    tags = run_git("tag", "--sort=-creatordate").split()
    return tags[0] if tags else None


def main():
    parser = argparse.ArgumentParser(description=__doc__)
    parser.add_argument("--from", dest="start", help="起始版本（不含），默认最近一个 tag")
    parser.add_argument("--to", dest="end", default="HEAD", help="结束版本，默认 HEAD")
    args = parser.parse_args()

    start = args.start or latest_tag()
    revision_range = "%s..%s" % (start, args.end) if start else args.end

    buckets = {key: [] for key, _ in GROUPS}
    breaking = []
    others = []

    for line in run_git("log", "--pretty=format:%s", revision_range).splitlines():
        line = line.strip()
        if not line:
            continue
        match = PATTERN.match(line)
        if not match:
            others.append("- " + line)
            continue
        scope = "**%s**: " % match["scope"] if match["scope"] else ""
        entry = "- %s%s" % (scope, match["subject"].strip())
        if match["breaking"]:
            breaking.append(entry)
        if match["type"] in buckets:
            buckets[match["type"]].append(entry)
        else:
            others.append(entry)

    print("# 发布说明草稿（%s → %s）\n" % (start or "仓库开始", args.end))
    if breaking:
        print("## 破坏性变更\n")
        print("\n".join(breaking))
        print()
    for key, title in GROUPS:
        if buckets[key]:
            print("## %s\n" % title)
            print("\n".join(buckets[key]))
            print()
    if others:
        print("## 其他\n")
        print("\n".join(others))
        print()


if __name__ == "__main__":
    main()
```

写脚本时有三条经验：

1. **输出就是给 agent 看的**，所以格式要稳定、语义要直白（分组标题直接写“新功能/问题修复”）。
2. **错误信息要可读**。`git` 失败时把 stderr 原样带出来，agent 能据此判断是“没有 tag”还是“当前目录不是 git 仓库”，而不是面对一个空结果发呆。
3. **只做确定性的事**。脚本不做“润色”，也不决定版本号——那些需要判断，交给模型；需要精确的部分交给代码。

### 第 5 步：把长篇知识放进 references

写作规则、版本号对照表这类内容不需要每次触发都进上下文，放进 `references/style-guide.md`：

```markdown
# 发布说明写作规则

## 归类

- `feat` → 新功能；`fix` → 问题修复；`perf` → 性能优化；`docs`、`chore`、`refactor`、`test` 等默认不进发布说明，除非直接改变用户可见行为。
- 带 `!` 或 `BREAKING CHANGE:` 的提交同时在"破坏性变更"一节列出。

## 语气与粒度

- 一条提交对应一条说明，动词开头，使用祈使句（"支持按标签过滤订单"，而不是"支持了/增加了"）。
- 面向用户描述行为变化，不写文件名、函数名、内部模块名。
- 同一模块的多个小修复可以合并为一条，例如"修复若干表单校验问题"。

## 版本号建议

按语义化版本（SemVer）规则：

| 变化 | 版本位 | 示例 |
| --- | --- | --- |
| 有破坏性变更 | major | 1.4.2 → 2.0.0 |
| 有 `feat` | minor | 1.4.2 → 1.5.0 |
| 只有 `fix`/`perf` | patch | 1.4.2 → 1.4.3 |

首个正式版本之前（0.x.y）可放宽：破坏性变更只升 minor。
```

引用文件时要告诉 agent **什么时候读**。`SKILL.md` 里写的是“阅读 `references/style-guide.md`，按其中的规则润色条目“，并放在流程的第二步——比笼统地写”详见 references 目录“有效得多，后者经常导致 agent 该读的时候不读、不该读的时候乱读。规范还建议引用保持一层深，不要 `a.md` 指向 `b.md`、`b.md` 再指向 `c.md`，层层套娃既浪费上下文，也容易迷路。

### 第 6 步：跑通并校验

准备一个最小的仓库验证脚本行为（真实项目里就是一个有 tag、有提交历史的仓库）：

```bash
cd demo-repo
git log --oneline --decorate
# 31f7dd0 (HEAD -> main) 更新一下配置文件
# 9f61e5c docs: 补充部署说明
# 34bf752 perf(db): 批量写入提速 3 倍
# 2551a22 fix(export): 导出 CSV 时转义逗号
# 0f7aa6a feat(api)!: 订单查询接口改为游标分页
# ca7fe08 (tag: v1.3.0) fix(web): 修复 Safari 下表头错位
# 85fbd0e feat(api): 支持按标签过滤订单
```

最近一个 tag 是 `v1.3.0`，脚本默认从这里开始收集。

```bash
python3 ~/.agents/skills/release-notes/scripts/collect_commits.py
```

真实输出：

```text
# 发布说明草稿（v1.3.0 → HEAD）

## 破坏性变更

- **api**: 订单查询接口改为游标分页

## 新功能

- **api**: 订单查询接口改为游标分页

## 问题修复

- **export**: 导出 CSV 时转义逗号

## 性能优化

- **db**: 批量写入提速 3 倍

## 其他

- 更新一下配置文件
- 补充部署说明
```

脚本把机械分组做完了；接下来模型会按 `style-guide.md` 润色、合并条目、给出 `major` 版本号建议（因为有破坏性变更），并在改动任何文件之前等待确认。

再用官方参考库校验一下格式。`skills-ref` 可以直接用 `uvx` 运行：

```bash
uvx --from skills-ref agentskills validate ./release-notes
# Valid skill: release-notes
```

`validate` 通过意味着 frontmatter 合法、命名符合规范。它还有一个我很喜欢的子命令 `to-prompt`，直接把 skill 生成 agent 系统提示里要用的目录 XML：

```bash
uvx --from skills-ref agentskills to-prompt ./release-notes
```

```xml
<available_skills>
<skill>
<name>
release-notes
</name>
<description>
根据 git 提交历史生成面向用户的发布说明，并给出语义化版本号（SemVer）建议。当用户提到发布说明、release notes、CHANGELOG、版本号、打 tag 或准备发布时使用。
</description>
<location>
/path/to/release-notes/SKILL.md
</location>
</skill>
</available_skills>
```

（输出里的绝对路径做了省略。）这份 XML 就是第 1 层“目录”的真实形态——每个 skill 只花几十个 token，agent 却能在任务匹配时知道去读哪个文件。

### 第 7 步：用真实执行迭代

第一版能跑不等于好用。把 release-notes skill 放进真实发布流程，观察 agent 的执行轨迹，常见的修正包括：

- 它把不规范的提交也强行归了类？→ 在“注意事项”里写死“不确定就放其他”；
- 它总是忘记读 style guide？→ 把引用移到流程第 2 步，并用条件句写明何时读；
- 它一上来就改 CHANGELOG？→ 把“先给草稿、等确认”设为不可跳过的步骤，并解释原因（发布说明是给人看的，措辞需要人拍板）；
- 某个环节卡住了？→ 把纠正写进 skill，下一次就不会再犯。

官方把这种“执行 → 观察 → 修订”的循环放在最佳实践的第一位，甚至建议让 Claude 在真实任务里帮你总结成功的步骤和踩过的坑，直接沉淀回 skill。

## 在两个客户端里跑起来

格式是标准的，加载方式各客户端有自己的实现，但都围绕“发现 → 目录 → 激活”三步。下面看两个最常见的。

### OpenCode V2

把 skill 放进 `.opencode/skills/<id>/SKILL.md`，OpenCode 会自动发现并把 `description` 展示给模型；模型用 `skill` 工具按 ID 加载，用户也可以在提示里写 `@skill-id` 手动加载。

发现位置（项目级会从当前目录一直向上找到项目根）：

| 范围 | 路径 |
| --- | --- |
| 全局 | `~/.config/opencode/skills/` |
| 全局兼容 | `~/.claude/skills/`、`~/.agents/skills/` |
| 项目 | `.opencode/skills/` |
| 项目兼容 | `.claude/skills/`、`.agents/skills/` |

几个容易踩的点：

- **ID 由路径决定，frontmatter 的 `name` 只是显示名**。`.opencode/skills/release/SKILL.md` 的 ID 是 `release`，不是 “Git Release”。
- **同名按来源优先级覆盖**：后注册的来源赢，项目级优先于全局级；同一个 ID 出现在两个地方时，记得确认用的是哪一个。
- **加载的细节**：OpenCode 只把 ID、name、description 放进每一步的上下文；加载时才把 `SKILL.md` 正文（去掉 frontmatter）注入对话，同时附上 skill 的基础目录和最多 10 个附属文件的路径示例——附属文件的内容不会自动加载，由 agent 按指令去读。
- **`description` 缺失的 skill 不会被广播**给模型，等于装了也不会被自动触发。

配置里还可以给 skill 加来源（本地目录、`~` 路径、HTTP 目录都行）：

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "skills": [
    "./team-skills",
    "~/shared/opencode-skills",
    "https://example.com/opencode/skills/"
  ]
}
```

HTTP 目录是一个带 `index.json` 的静态站点，团队可以把 skill 集中托管，客户端按版本号刷新缓存：

```json
{
  "skills": [
    {
      "name": "git-release",
      "version": "3",
      "files": ["git-release.md", "references/release-policy.md"]
    }
  ]
}
```

权限也值得注意：OpenCode 用 `skill` 这个 action 控制加载，`allow` 直接放行、`ask` 加载前询问、`deny` 对模型隐藏且拒绝加载：

```jsonc
{
  "permissions": [
    { "action": "skill", "resource": "*", "effect": "allow" },
    { "action": "skill", "resource": "internal-*", "effect": "deny" },
    { "action": "skill", "resource": "experimental-*", "effect": "ask" }
  ]
}
```

如果你想让某个 skill 只能被用户手动 `@`、不允许模型自动触发，在 frontmatter 里写 `metadata: { opencode/autoinvoke: false }`（或 `disable-model-invocation: true`）。

### Claude Code

Claude Code 遵循同一个开放标准，但在发现位置和 frontmatter 上做了扩展。skill 可以放在多个层级：

| 位置 | 路径 | 生效范围 |
| --- | --- | --- |
| 企业 | 受管设置目录下的 `.claude/skills/<name>/SKILL.md` | 组织内所有用户 |
| 个人 | `~/.claude/skills/<name>/SKILL.md` | 你这台机器的所有项目 |
| 项目 | `.claude/skills/<name>/SKILL.md` | 当前仓库，提交后团队共享 |
| 嵌套 | 子目录里的 `.claude/skills/...` | 工作到该子目录时加载（monorepo 友好） |
| 插件 | 插件包内的 `skills/` | 启用该插件的地方 |
| claude.ai 同步 | 账号里启用的 skill | Cowork / 云会话等 |

几个和标准有关的差异点：

- **自定义命令已经合并进 skill**。`.claude/commands/deploy.md` 和 `.claude/skills/deploy/SKILL.md` 都会创建 `/deploy`，老文件仍然兼容，但新写的推荐用 skill——因为 skill 还能带附属文件、控制谁可以调用。
- **frontmatter 扩展**（这些字段不是开放标准的一部分，只在 Claude Code 里生效）：

  | 字段 | 作用 |
  | --- | --- |
  | `disable-model-invocation: true` | 禁止模型自动触发，只能用户 `/name` 手动调用 |
  | `user-invocable: false` | 反过来：只允许模型用，用户菜单里隐藏 |
  | `allowed-tools` / `disallowed-tools` | 本次调用预授权/禁用某些工具 |
  | `context: fork` | 在独立的子代理上下文里运行这个 skill |
  | `paths` | 只有操作匹配的文件时才自动激活 |
  | `model` / `effort` | 调用时临时指定模型或思考强度 |
  | `hooks` | 调用时注册钩子 |

- **动态上下文注入**。正文里写 `` !`git diff HEAD` ``，Claude Code 会在把内容交给模型之前先执行命令、把结果内联进去。适合“总是需要最新数据”的流程。
- **内置 skill**。Claude Code 自带 `/code-review`、`/debug`、`/doctor`、`/claude-api` 等，用法和你自己写的一样，也可以被 `disableBundledSkills` 关掉。
- **会话内热加载**。修改已有 skill 目录会被实时检测，不用重启；新建的顶层 skills 目录需要 `/reload-skills`。

两个客户端的对照：

| 维度 | OpenCode V2 | Claude Code |
| --- | --- | --- |
| 原生目录 | `.opencode/skills/` | `.claude/skills/` |
| 目录约定 | 原生目录 + `.claude/`、`.agents/` 兼容 | 原生多级目录 + 插件 + 账号同步 |
| 手动触发 | `@skill-id` | `/skill-name` |
| 控制自动触发 | `opencode/autoinvoke` | `disable-model-invocation` |
| 附属文件 | 加载时给出路径示例，按需读取 | 支持 `@` 引用和动态注入 |
| 权限 | `skill` action 的 allow/ask/deny | `allowed-tools` 走常规权限流程 |
| 远程分发 | `skills` 配置 + HTTP 目录 | 插件、市场、claude.ai 账号同步 |

如果只记一句话：**先按开放标准写（`name` + `description` + 正文），再把客户端特有的能力当作可选增强**，这样 skill 才能跨工具复用。

## 怎么写出好用的 skill

官方最佳实践加上实际使用经验，可以浓缩成十条：

1. **从真实执行中提炼，不要凭空生成**。让模型凭空写，产出的多半是“妥善处理错误”“遵循最佳实践”这类正确的废话。真正有价值的是你在实践中纠正过它的那些具体点。
2. **`description` 决定生死**。它既是模型判断是否加载的依据，也是用户搜索的入口。把“做什么”和“什么时候用”都写进去，用用户嘴里会说的词（发布说明、CHANGELOG、打 tag……）。
3. **只写 agent 不知道的**。PDF 是什么、HTTP 怎么工作，它比你清楚；你们的接口约定、某个表必须带 `WHERE deleted_at IS NULL`，才是 skill 该写的内容。标准问法：没有这条指令，它会不会做错？不会就删掉。
4. **粒度像函数**。太窄会导致完成一件事要连加载好几个 skill，太宽又会触发不准。判断标准：它是否封装了一个内聚的工作单元。
5. **控制强度匹配任务脆弱度**。多种做法都行时，给出原理和自由度；操作脆弱、顺序固定时（比如数据库迁移），写死命令并明确“不要加参数”。
6. **给默认值，不给菜单**。列出 pypdf、pdfplumber、PyMuPDF 让模型自己选，不如写“用 pdfplumber；扫描件用 pdf2image + pytesseract”。
7. **教流程，不教答案**。“把 orders 表 join customers 表，sum amount” 只解决一次；“先读 schema，按 `_id` 约定 join，按用户条件过滤，再聚合“可以解决一类问题。
8. **高价值区：gotchas**。环境里反直觉的事实——软删除、同一个 ID 在三套系统里的不同命名、健康检查接口的假阳性——这些是 agent 最容易踩、也最值得写进正文的坑。每次纠正都往这里加一条。
9. **用结构减少不确定性**：需要固定输出时给模板；多步骤流程给 checklist；关键操作设计“验证循环”（改完跑校验脚本，失败就修，通过才继续）；批量或破坏性操作走“计划 → 校验 → 执行”三段式。
10. **观察轨迹、持续迭代**。看 agent 怎么执行（不只是看结果），注意它是否浪费时间反复试探、是否被不适用的指令带偏。一次“执行—修订”就能明显改善，复杂领域需要多轮。

还有个成本提醒：`SKILL.md` 正文一旦加载就会留在上下文里，像所有上下文一样按 token 计费、并且参与注意力竞争。保持精简，长内容拆去 `references/`，并写明“什么时候读哪个文件”。

## 安全：skill 是可以带代码的“文档”

Skill 的本质上是一段会被 agent 信任并执行的指令，还可以附带可执行脚本、引用外部依赖、访问网络。这带来一个必须正视的攻击面：

- 一个恶意 skill 可以教 agent 泄露数据、执行危险命令，或者在脚本里做手脚；
- 项目级 skill 尤其危险：克隆一个陌生仓库、打开编辑器，仓库里的 `.claude/skills`、`.opencode/skills` 就可能把指令注入 agent 的上下文；
- skill 不是加密容器，写在里面的密钥、token 会原样躺在文件系统里。

防护建议：

1. **只从可信来源安装 skill**；来源不可信时，像审代码一样逐文件审计，重点看命令、依赖和联网行为；
2. **项目级 skill 要过信任门控**。官方给客户端实现者的建议就是如此：只有用户标记为信任的项目，才加载它的 skill；
3. **用权限系统兜底**。OpenCode 可以对 skill 设置 allow/ask/deny，对敏感 skill 用 `ask`；Claude Code 的 `allowed-tools` 走正常权限流程，同步来的 skill 也会做内容净化；
4. **不要把密钥写进 skill**，需要凭据时让脚本从环境变量或密钥管理服务读取；
5. **定期审查**：skill 是活的文件，依赖和指向的地址会变，别装完就不管。

一句话：把 skill 当供应链的一部分对待——它是你能给 agent 的最强输入之一，也是值得用同样严格标准审查的输入。

## 生态现状与时间线

| 时间 | 事件 |
| --- | --- |
| 2025-10-16 | Anthropic 发布 Agent Skills，首发支持 Claude 应用、Claude Code、Agent SDK、Developer Platform |
| 2025-12-18 | 发布为开放标准，规范、参考实现与校验库上线 agentskills.io（代码 Apache-2.0，文档 CC-BY-4.0） |
| 2026 年 | 客户端展示页列出 40 多个支持产品：Claude Code、Claude、ChatGPT & Codex、GitHub Copilot、VS Code、Cursor、Gemini CLI、JetBrains Junie、Kiro、Mistral Vibe、OpenCode、Goose、OpenHands、Amp、Qodo、Snowflake Cortex Code…… |
| 正在发生 | 官方与社区的 skill 市场、团队目录、HTTP 分发源；客户端把 skill 与命令、子代理、插件打通 |

Anthropic 在发布文章里给未来的方向是：让 agent 能够自己创建、编辑和评估 skill，把实践中形成的模式固化成可复用的能力。结合已经出现的“skill 生成 + eval 迭代”工作流，这个方向并不难想象——**人类负责总结流程，agent 负责执行与反馈，流程本身变成一种可以被版本控制的软件资产**。

## 总结

- **Skill 是什么**：一个装着 `SKILL.md`（元数据 + 指令）和可选脚本、参考文档、资源的文件夹；格式由 agentskills.io 的开放标准定义，`name` + `description` 是必填项。
- **怎么工作**：渐进式披露分三层——启动时只加载每个 skill 约 50–100 token 的目录，任务匹配时加载完整指令（建议 <5000 token），资源在引用时才读。这让“打包进 skill 的上下文”实际上可以无限大。
- **为什么重要**：它用外置文件承载程序性知识，解决了上下文稀缺、团队经验复用、跨工具移植、确定性执行和知识版本化这几件事，而且可以评测、可以迭代。
- **怎么写好**：从真实任务提炼；把 `description` 当作触发器精雕细琢；只写 agent 不知道的；用 gotchas、模板、checklist、验证循环控制质量；保持正文精简，长内容拆到 `references/`。
- **怎么落地**：OpenCode 用 `.opencode/skills/`（兼容 `.claude/`、`.agents/`），Claude Code 用 `.claude/skills/` 并支持 `disable-model-invocation`、`allowed-tools`、`context: fork` 等扩展；先写标准格式，再加客户端特性。
- **安全底线**：skill 是能带代码的指令，按供应链风险对待——只装可信来源，项目级 skill 做信任门控，用权限系统限制敏感操作。
