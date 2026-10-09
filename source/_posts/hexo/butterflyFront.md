---
title: Hexo Butterfly 主题 Front Matter 完全指南
date: 2022-12-06 23:20:30
updated: 2026-10-09
author: ws
description: 逐字段讲清文章头部配置，附常用组合示例与踩坑
categories: ["Hexo"]
tags: ["Hexo", "Butterfly"]
cover: false
---

## 引言

每篇 Hexo 文章的头部都有一段 `---` 包裹的配置，它决定文章显示什么日期、有没有封面、要不要目录和评论。字段看起来不少，但实际上 80% 的文章只需要 5 个。本文按“极简模板 → 全字段 → 逐字段解释 → 常见错误”的顺序，把 Butterfly 主题下 Front Matter 的行为讲清楚；所有结论都对照本仓库主题源码和实际构建产物验证过，并标注了本站当前配置下的真实表现。

## Front Matter 是什么

Front Matter 是写在文章文件最开头、由两行 `---` 包裹的 YAML 元数据：

```yaml
---
title: 文章标题
date: 2022-12-06 23:20:30
tags: [Hexo]
---
```

Hexo 在渲染前会先用 YAML 解析器把它解析成一个对象，赋给 `page` 变量；主题的 pug 模板再从 `page.xxx` 取值决定怎么渲染。所以：

- 语法必须是合法 YAML，标点、缩进、引号都会直接影响解析结果；
- 字段能不能用、怎么用，取决于主题模板是否读取它——写了一个主题不认识的字段不会报错，但也不会有任何效果。

## 日常极简模板

新建文章时直接用这个就够，覆盖 80% 场景：

```yaml
---
title: 文章标题
date: 2022-12-06 23:20:30
tags: [Hexo, Butterfly]
categories: [Hexo]
cover:
---
```

`cover:` 留空表示“用主题的默认随机封面”；不想要封面可以写 `cover: false`，两者区别见下文。

## 全字段示例

需要精细控制时，可以参考这份全量模板（字段含义逐一在下一节解释）：

```yaml
---
title: "Hexo + Butterfly：那些容易踩的坑"
date: 2022-12-06 23:20:30
updated: 2023-01-15 10:00:00
tags: [Hexo, Butterfly, 教程]
categories: [Hexo, 进阶]
keywords: [hexo, butterfly, front-matter]
description: 讲讲 Front Matter 里最容易写错的几个字段
top_img: /img/1.jpg
cover: /img/0.jpg
comments: false
toc: true
toc_number: false
toc_style_simple: true
toc_expand: true
copyright: true
copyright_author: ws
copyright_author_href: https://github.com/wsshow
copyright_url: https://wsshow.github.io/blog
copyright_info: 转载请保留出处
mathjax: true
katex: false
aplayer: false
highlight_shrink: true
aside: false
---
```

## 字段总览

下面是 Butterfly 支持的文章级字段总表（含 `toc_expand`）：

| 关键字                | 解释                                                     | 重要性     |
| :-------------------- | :------------------------------------------------------- | ---------- |
| title                 | 文章标题                                                 | **`必须`** |
| date                  | 文章创建日期                                             | **`必须`** |
| updated               | 文章更新日期                                             | *可选*     |
| tags                  | 文章标签                                                 | *可选*     |
| categories            | 文章分类                                                 | *可选*     |
| keywords              | 文章关键字                                               | *可选*     |
| description           | 文章描述                                                 | *可选*     |
| top_img               | 文章顶部图片                                             | *可选*     |
| cover                 | 文章缩略图（首页封面 / 分享图）                          | *可选*     |
| comments              | 显示文章评论模块                                         | *可选*     |
| toc                   | 显示文章 TOC（默认为设置中 toc 的 enable 配置）          | *可选*     |
| toc_number            | 显示 TOC 序号                                            | *可选*     |
| toc_style_simple      | 显示 TOC 简洁模式                                        | *可选*     |
| toc_expand            | TOC 默认展开                                             | *可选*     |
| copyright             | 显示文章版权模块（默认为设置中 post_copyright 的 enable） | *可选*     |
| copyright_author      | 文章版权模块的文章作者                                   | *可选*     |
| copyright_author_href | 文章版权模块的文章作者链接                               | *可选*     |
| copyright_url         | 文章版权模块的文章链接                                   | *可选*     |
| copyright_info        | 文章版权模块的版权声明                                   | *可选*     |
| mathjax               | 显示 mathjax（默认为 false）                             | *可选*     |
| katex                 | 显示 katex（默认为 false）                               | *可选*     |
| aplayer               | 在需要的页面加载 aplayer 的 js 和 css                    | *可选*     |
| highlight_shrink      | 配置代码框是否展开（默认为设置中 highlight_shrink 的值） | *可选*     |
| aside                 | 显示侧边栏（默认为 true）                                | *可选*     |

## 逐字段详解

### title：含冒号必须加引号

`title` 是全站最关键的字段：它同时用于文章页标题、浏览器 tab、首页列表和 SEO 标题。唯一的坑是 YAML 语法：

```yaml
# 错误：冒号会被当成 YAML 的键值分隔符，解析报错
title: Hexo 踩坑记录: 第一部分

# 正确：整体加引号
title: "Hexo 踩坑记录: 第一部分"
```

### date 与 updated：格式与时区

本站 `_config.yml` 里配置了：

```yaml
timezone: 'Asia/Shanghai'
date_format: YYYY-MM-DD
time_format: HH:mm:ss
updated_option: 'mtime'
```

`date` 推荐写 `YYYY-MM-DD HH:mm:ss`（如 `2022-12-06 23:20:30`），Hexo 会按站点时区 `Asia/Shanghai` 解析。可以只写日期 `2022-12-06`，时间默认 `00:00:00`。

`updated` 不写时受 `updated_option` 控制：本站是 `mtime`，也就是**取文件最后修改时间**。构建产物里能直接看到这条规则的效果——下面是没写 `updated` 的文章生成的 HTML：

```text
datetime="2022-12-06T15:20:30.000Z"          <!-- date: 23:20:30 +08:00 -->
datetime="2026-10-09T02:11:10.197Z"          <!-- updated: 文件 mtime 10:11:10 +08:00 -->
```

注意两个时间都被转成 UTC 存储（+08:00 减 8 小时）。如果不想让“随便碰了一下文件”就刷新文章的更新日期，每篇显式写 `updated`；如果压根不想显示更新日期，可以把 `post_meta.post.date_type` 从 `both` 改成 `created`（只显示创建时间）。

### tags 与 categories

两者的区别：**tags 是文章的关键词，categories 是文章的归属**。一篇文章可以有多个标签，但分类通常是一个路径。写法上：

```yaml
# 行内数组：适合数量少
tags: [Hexo, Butterfly]

# 块状数组：更清晰，推荐
tags:
  - Hexo
  - Butterfly

# 多层级分类（嵌套数组），表示 Hexo / 进阶
categories:
  - Hexo
  - 进阶

# 多分类（并列数组），文章同时属于两个一级分类
categories:
  - [Hexo, 教程]
  - [前端工具]
```

tags 不支持嵌套层级，别写多级标签；分类页面和标签页面的链接都是根据这些字段自动生成的。

### top_img 与 cover：最容易混淆的一组

先用一句话记住：**top_img 是文章页顶部的大图，cover 是首页/归档列表里的封面缩略图**。主题对文章页的取值逻辑在 `themes/butterfly/layout/includes/header/index.pug`：

```pug
//- 文章页：top_img 优先，其次 cover，最后主题默认图
- var top_img = page.top_img || page.cover || theme.default_top_img
```

几个关键行为：

| 写法            | 效果                                                             |
| :-------------- | :--------------------------------------------------------------- |
| `top_img: /img/1.jpg` | 文章页顶部显示该图（前提是主题未禁用顶部图）                |
| `top_img: false`      | 该文章彻底不显示顶部图，回退链也不再生效                    |
| 不写 top_img          | 用 `cover`；`cover` 也没有时用主题的 `default_top_img`      |
| `cover: /img/0.jpg`   | 首页封面用该图；文章页顶部图没配时也拿它当顶部图            |
| `cover: false`        | 不显示封面，也不会分配随机封面                              |
| 不写 cover（`cover:`）| 构建时从 `cover.default_cover` 列表里随机分配一张            |

“随机分配”是主题的 `before_post_render` 过滤器（`themes/butterfly/scripts/filters/random_cover.js`）实现的：`cover` 为空则从 `default_cover` 随机取一张，`cover: false` 则直接跳过。所以 `cover:` 和 `cover: false` 差一个词，行为完全不同。

本站当前配置下的真实表现（`_config.butterfly.yml`）：

```yaml
disable_top_img: true        # 全站禁用顶部图
cover:
  index_enable: false        # 首页不显示封面
  aside_enable: false        # 侧边栏不显示封面
  archives_enable: false     # 归档页不显示封面
```

也就是说，**目前在本站写 `top_img` 或 `cover` 都不会看到图片**，因为 `disable_top_img: true` 直接把顶部图全部关掉了（生成 HTML 里 header 的类是 `not-top-img`）。想让图片重新出现，先把 `disable_top_img` 改成 `false`、按需打开 `cover.index_enable`。

图片路径的三种写法：

```yaml
top_img: /img/1.jpg                          # 放在 source/img/ 下，站点根路径引用
top_img: https://example.com/banner.jpg      # 完整外链（注意防盗链和失效风险）
top_img: banner.jpg                          # 仅当开启 post_asset_folder 时可用（本站未开启）
```

### toc / toc_number / toc_style_simple

目录相关字段只影响“这篇文章”，全局默认值在 `_config.butterfly.yml` 的 `toc` 配置里。本站当前是：

```yaml
toc:
  post: true          # 文章页默认显示 TOC
  page: false
  number: true        # 默认显示序号
  expand: false       # 默认折叠
  style_simple: false
```

文章级覆盖写法：

```yaml
toc: false                # 这篇不显示目录
toc_number: false         # 这篇的目录不带序号
toc_expand: true          # 这篇的目录默认展开
toc_style_simple: true    # 简洁模式：侧边栏只保留 TOC 卡片
```

三个前提条件（模板 `config_site.pug`、`widget/index.pug` 里的判断）：

1. 侧边栏必须启用且没有关掉：`theme.aside.enable: true` 且文章没有写 `aside: false`；
2. 文章内容里要有能被识别的标题；
3. `toc: false` 的优先级最高，写了一切都不显示。

`toc_style_simple: true` 的效果是“隐藏作者卡片、公告等其它侧边栏模块，只留目录”，适合长文。

### comments

`comments: false` 用于单篇关闭评论区，模板判断是 `page.comments !== false`。但要注意：本站 `_config.butterfly.yml` 里 `comments.use` 是空的（没有接入任何评论系统），所以无论 Front Matter 怎么写，现在页面上都没有评论区。等配置了 Valine、Twikoo、Giscus 之类的系统后，这个字段才会真正起作用。

### copyright 系列

本站开启了全局版权模块：

```yaml
post_copyright:
  enable: true
  author_href: ws
  license: CC BY-NC-SA 4.0
```

文章级字段的作用是**覆盖或关闭**：

```yaml
copyright: false                          # 这篇不显示版权模块
copyright_author: 某位朋友                  # 覆盖默认作者
copyright_author_href: https://example.com # 覆盖作者链接
copyright_url: https://example.com/post    # 覆盖版权模块中的文章链接
copyright_info: 本文禁止转载                 # 覆盖默认版权声明
```

模板逻辑（`post-copyright.pug`）是 `theme.post_copyright.enable && page.copyright !== false`，所以 `copyright: true` 不是“开启”而是“不关闭”。

### mathjax / katex：按页开启有前提

这是最容易误解的一组字段。本站配置是：

```yaml
mathjax:
  enable: false
  per_page: false
katex:
  enable: false
  per_page: false
```

主题模板 `third-party/math/index.pug` 的判断顺序是：

```pug
if theme.mathjax && theme.mathjax.enable    // 全局开关必须为 true
  if theme.mathjax.per_page                 // 每页都加载
    if is_post() || is_page()
      include ./mathjax.pug
  else
    if page.mathjax                         // 按 Front Matter 加载
      include ./mathjax.pug
```

结论：**`enable: false` 时，文章里写 `mathjax: true` 没有任何作用**。我实际建了一篇带 `mathjax: true`、`katex: true` 的文章重新构建，生成页面里连一个相关 `<script>` 都没有。

要用按页开启，正确步骤是：

1. 在 `_config.butterfly.yml` 里设 `mathjax.enable: true`、`per_page: false`；
2. 在需要公式的文章 Front Matter 里写 `mathjax: true`。

如果 `per_page: true`，则是所有文章页都加载，Front Matter 字段就无关紧要了。mathjax 和 katex 二选一，不要同时开（公式渲染性能会明显变差）。

### description、keywords 与 SEO

`description` 是最值得写的一个字段，它有两个去处：

1. **首页摘要**：本站 `index_post_content.method: 1`，首页列表直接显示 `description`；
2. **`<meta name="description">`**：构建产物里实测输出为：

```text
$ grep -o '<meta name="description"[^>]*>' public/2022/12/06/hexo/butterflyFront/index.html
<meta name="description" content="butterfly主题下设置文章信息">
```

搜索引擎和社交平台会读取它。建议每篇文章都写，长度控制在 80~150 字以内。

`keywords` 则是一个**写了没用的字段**：实测生成的 HTML 里没有任何 `<meta name="keywords">`。Hexo 的 Open Graph 逻辑只会把 `tags` 输出成 `article:tag`，不读 `page.keywords`。所以做 SEO 就把关键词写进 `tags`，`keywords` 字段可以忽略。

本站配置里还有一处容易踩的坑：`_config.butterfly.yml` 里 `Open_Graph_meta: true` 写成布尔值是不生效的，主题模板读的是 `theme.Open_Graph_meta.enable`，这样写会导致 Open Graph 系列标签（`og:title`、`og:image` 等）**一个都没有输出**（构建产物里 `og:` 相关标签数量为 0）。正确的写法是对象形式：

```yaml
Open_Graph_meta:
  enable: true
  option:
```

改成对象形式之后，`og:image` 的取值规则是：`cover` 被识别为图片路径时用 `cover`，否则退回头像（`avatar.img`）；注意 `cover: false` 会退回头像，留空则由随机封面兜底。

### 其余字段

```yaml
aplayer: true            # 只在这篇文章加载 APlayer 资源
highlight_shrink: true   # 这篇的代码块默认折叠
aside: false             # 这篇隐藏侧边栏
```

- `aplayer`：主题 `additional-js.pug` 里判断 `page.aplayer`，但只有 `aplayerInject.enable: true` 时才处理；本站没开，写了无效。
- `highlight_shrink`：覆盖主题的代码块折叠设置；本站主题值是 `false`（展开），写 `true` 可让单篇折叠。
- `aside: false`：隐藏侧边栏，长表格、宽图文章常用；它同时会让 TOC 消失（因为 TOC 在侧边栏里）。

## 常见错误

1. **title 含冒号不加引号**。`title: Hexo: 入门` 直接导致构建报 YAML 错误，写成 `title: "Hexo: 入门"` 即可。
2. **tags/categories 写成逗号字符串**。`tags: Hexo, Butterfly` 会被解析成**一个**字符串标签 `"Hexo, Butterfly"`，标签页里会多出一个诡异标签。必须用数组：`[Hexo, Butterfly]`。
3. **date 格式写错**。`date: 2022/12/06` 这类写法 moment 也能解析，但容易在不同环境产生时区差异；统一写 `YYYY-MM-DD HH:mm:ss` 最稳。另外本站 `future: true`，写未来时间也能生成。
4. **外链封面失效**。图片站防盗链、链接过期都会让封面变空白，主题有 `error_img` 兜底占位但体验不好；长期用的图建议下载到 `source/img/` 本地引用。
5. **cover 和 top_img 混淆**。只配 `cover` 却期待文章顶部出现大图——只有 `top_img` 未配置时 `cover` 才作为回退；而本站还叠加了 `disable_top_img: true`，两个都不显示。
6. **`cover:` 与 `cover: false` 分不清**。留空会触发随机封面（构建时分配 `default_cover` 中的一张），`false` 才是真的不要封面；两种写法让文章行为和分享图都会不同。

## 80% 的文章只需要 5 个字段

最后回到实用主义：绝大多数文章只需要 `title`、`date`、`tags`、`categories`、`cover` 五个字段。其余字段都是“有特殊需求再查表”的开关。判断标准很简单：

- 有公式 → 先改主题 `mathjax.enable`，再加 `mathjax: true`；
- 不想上目录 → `toc: false`；
- 转载/分享相关 → 看 `copyright` 系列；
- 图片展示 → 分清 `top_img`（文章顶部）和 `cover`（列表封面）。

## 总结

- Front Matter 是 YAML，写错语法会直接构建失败；title 含冒号必须加引号，tags/categories 必须是数组。
- `date` 按 `Asia/Shanghai` 解析并转 UTC 存储；本站 `updated_option: mtime`，不写 `updated` 就用文件修改时间。
- `top_img` 是文章页大图，`cover` 是列表封面且是 top_img 的回退；`false` 禁用、留空随机。本站在 `disable_top_img: true` 下两者都不显示。
- `toc: false` 能单篇关目录，但前提是侧边栏没被 `aside: false` 关掉；`toc_style_simple: true` 只留目录卡片。
- `mathjax/katex` 的按页开关以主题 `enable: true` + `per_page: false` 为前提，本站当前两个 `enable` 都是 false，写了无效（已实测）。
- `description` 影响首页摘要和 meta description，值得认真写；`keywords` 字段不参与渲染，SEO 用 `tags`。
- 站点配置里 `Open_Graph_meta` 必须写成 `enable: true` 的对象形式，分享标签才会生效；写成布尔值不生效。
