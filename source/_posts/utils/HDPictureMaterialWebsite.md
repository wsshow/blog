---
title: 免费高清图片素材站合集：授权差异与选用指南
date: 2022-12-07 22:20:30
updated: 2026-10-09
author: ws
description: 6 个常用免费图库的授权、适用场景与使用技巧
categories: ["资源"]
tags: ["图片素材", "资源"]
cover: false
---

## 引言

写博客、做 PPT、给产品配图，绕不开找图。免费图库不少，但"免费"和"能商用"是两回事，各家的授权条款细节也经常不一样。本文整理 6 个常用图库，逐站核对官方授权页，整理成一张可以直接查的表，再讲清楚搜索技巧和 API 批量取图的规矩。所有授权描述均以 2026-10 官方页面为准，来源链接都附在文中。

## 六个图库速查表

| 图库 | 授权协议 | 需要署名 | 可商用 | 中文检索 | 最适合 |
| --- | --- | --- | --- | --- | --- |
| [Unsplash](https://unsplash.com/) | Unsplash License（自定义） | 否（推荐） | 可以 | 一般（英文关键词效果最好） | 高质量摄影感配图、生活方式、产品场景 |
| [Pixabay](https://pixabay.com/) | Pixabay Content License（2019 年前内容为 CC0） | 否（推荐） | 可以 | 好 | 中文检索、插画/矢量/视频/音效全品类 |
| [PicJumbo](https://picjumbo.com/) | 站点自有条款 | 否（推荐） | 可以 | 无中文界面 | 网页背景、UI 素材、自然风光、4K 壁纸 |
| [Foodiesfeed](https://www.foodiesfeed.com/) | 等效 CC0 | 否（推荐） | 可以 | 有中文界面 | 食物摄影、食谱、餐饮内容 |
| [Pexels](https://www.pexels.com/zh-cn/) | Pexels License（自定义） | 否（推荐） | 可以 | 好 | 通用商业配图、营销物料、视频素材 |
| [Hippopx](https://www.hippopx.com/zh) | 宣称 CC0 | 否 | 可以（需逐图确认） | 好 | 中文检索、壁纸、PPT 背景 |

需要先泼一盆冷水：表中"可商用"的正式含义是"图库的授权条款允许商用"，它不覆盖图中人物/商标/私人物业的第三方权利。这一层后文会展开。

## 先分清授权类型：CC0、自定义 License 和自有条款

这三类东西经常被混着说，实际约束强度不同：

- **CC0（Creative Commons Zero）**：作者主动放弃版权，把作品贡献给公共领域。CC0 官方文本明确："您可以复制、修改、发行和表演本作品，甚至可用于商业性目的，都无需要求同意。"同时也提醒：**不得暗示作者或声明人认可你的使用**，且形象权、隐私权等不受 CC0 影响。参考 [CC0 1.0 中文摘要](https://creativecommons.org/publicdomain/zero/1.0/deed.zh-hans)。
- **自定义 License（Unsplash / Pexels / Pixabay）**：图库自己写的授权文本，底子都是"免费商用、无需署名"，但每家都加了自己的限制条款（比如不能原样转售、不能在竞品图库再分发）。**不要笼统当成 CC0 用**。
- **自有条款（PicJumbo）**：以 FAQ/Terms 形式给出，没有标准协议的名字，同样约束于具体条款。

一个历史细节值得知道：Pixabay 在 **2019 年 1 月 9 日**从 CC0 切换到了自有的 Pixabay License，2019 年之前上传的内容仍按 CC0（该内容在其条款中称为 CC0 Content）。所以"Pixabay 是 CC0 图库"的说法，对 2019 年后的图片不成立。

### 看懂英文授权文本里的关键词

各家条款都是英文写的，下面这些词决定了"能不能用"的边界，遇到时不要凭感觉翻译：

| 英文术语 | 含义 | 为什么重要 |
| --- | --- | --- |
| Standalone basis | 保持原样、没有创作加工 | Pixabay 明确禁止以 standalone 形式出售/分发，即"不能原样卖" |
| Redistribute / Re-distribution | 再分发 | 把图片传到别的图库、壁纸站、打包进素材库都算，普遍禁止 |
| Endorsement | 背书、认可 | 不能让画面暗示图中人物/品牌支持你的产品或观点 |
| Model release / Property release | 模特肖像授权 / 物权授权 | 图库授权不覆盖这两类第三方权利，商业使用需自行确认 |
| Derivative work | 演绎作品 | "可以修改"意味着允许创作衍生作品，但原样转售不算 |
| Hotlink | 热链 | 直接引用图床 URL，部分站点（如 PicJumbo）明确禁止 |
| Attribution | 署名 | 多数站点"不强制但欢迎"，API 场景往往变成强制 |
| Royalty-free | 免版税 | 只表示无需按次付费，**不等于无版权**，条款限制依然有效 |
| Editorial use | 编辑用途（新闻、评论） | 有些限制只对广告/商业用途生效，编辑用途的门槛更低 |

其中最容易被误解的是 **Royalty-free**：它不是"随便用"，而是"付费模式按次"的反义词。免费图库里的绝大多数图片版权仍归作者所有，只是通过 License 授权给你使用。

## 逐站介绍

### Unsplash：摄影感最强的那个

Unsplash 的图偏"作品感"，适合需要氛围、光影、场景叙事的配图。官方授权页（[unsplash.com/license](https://unsplash.com/license)，本次自动化访问被其反爬系统拦截，文本经官方帮助中心与页面存档核实）的核心内容：

> All images can be downloaded and used for free. Commercial and non-commercial purposes. No permission needed (though attribution is appreciated!)

不允许的两件事：

- **不能未经显著修改地售卖图片**（"Images cannot be sold without significant modification."）；
- **不能把 Unsplash 图片汇编成与 Unsplash 类似或竞争的服务**（"Compiling images from Unsplash to replicate a similar or competing service."）。

长篇授权条款进一步说明：这是一份"irrevocable, nonexclusive, worldwide copyright license"，可自由下载、修改、分发，包括商用，无需署名。两个实践提醒：

1. 官方帮助中心的 [Releases and Trademarks](https://help.unsplash.com/en/articles/2612329-releases-and-trademarks) 页提醒：图片中出现可识别人物、私人场所、品牌 logo 时，可能涉及肖像权/物权/商标权，图库无法替你保证；商用前建议向作者确认。
2. 想印 T 恤、做周边、卖打印品，Unsplash 明确"不包括把图片印在商品上出售的授权"，需要联系作者单独授权。

### Pixabay：全品类 + 中文友好

图片、插画、矢量、视频、音效、3D 模型都有，中文界面和中文标签检索体验在免费图库里是第一梯队。官方 [Content License Summary](https://pixabay.com/service/license-summary/) 的要点：

允许：免费使用；**无需署名**（"Use Content without having to attribute the author"）；可以修改、改编。

不允许（官方称为 Prohibited Uses）：

- 不能以 **Standalone** 形式出售或分发内容——"Standalone"指没有创作加工、基本保持原样，包括作为图片/视频/音频文件、NFT、壁纸、海报、商品印花等；
- 内容中含有可识别商标、logo、品牌时，不能用于商品和服务的商业场景，尤其不能印在商品上出售；
- 不能用于不道德或非法用途（尤其是含可识别人物的内容），不能用于误导或欺骗性场景；
- 不能把内容用作商标、设计标志、商号、企业名或服务标志。

另外两点：Pixabay 上现在有 AI 生成内容，页面会标注来源；使用前要自行判断是否需要图片中人物的额外许可。

### PicJumbo：个人摄影师的小而美图库

由捷克设计师/摄影师 Viktor Hanacek 运营，风格偏明亮的网页素材（背景、办公、自然、旅行）。授权细节在 [FAQ and Terms](https://picjumbo.com/faq-and-terms/)：

- 可以商用，包括给客户做项目；在 HTML/PSD/PPT 模板中使用也可以；
- **不能转售图片本身**；把图片上传到其他免费图库/壁纸站供人下载同样不允许；
- 把图片用在网站上时，下载按钮必须指向 picjumbo 的图片页，**不允许热链**；
- 如果要在网站构建器/SaaS/插件里内置图片库给用户用，需要购买 Photo Redistribution 计划；
- 署名非强制，但作者很欢迎（"It's not necessary but we're very happy for every mention"）。

适合：网页背景、hero 图、UI 展示图、4K 壁纸。没有中文界面，英文关键词搜索。

### FoodiesFeed：专做食物

食物主题的垂直图库，官方 [License 页](https://www.foodiesfeed.com/license/) 写明：图片按**等效 CC0** 的许可发布——可以复制、修改、分发，个人和商业项目都免费，无需署名。限制与其他图库一致：不能不加创作地原样出售（单独打印、海报、文件）、不能在别的图库平台再分发、不能暗示图中人物/品牌的认可。站内 AI 生成食物图遵循同样规则。

两个实际情况需要注意：

- 该站首页当前（2026-10 实测）挂着横幅 **"Foodiesfeed is for sale"**，网站正在出售中。服务本身目前正常，但长期存在不确定性，建议把常用的食物图尽早下载留档。
- 食物摄影里经常出现品牌包装、餐厅 logo，商用前留意画面内容。

### Pexels：授权写得最清楚，中文站可用

[Pexels License](https://www.pexels.com/license/) 的表述非常直白，允许清单：

- 所有图片和视频免费使用，可修改、可商用；
- 署名不是必须的，但"not necessary but always appreciated"；
- 官方还举了使用场景：网站/博客/电商/邮件/电子书/演示模板、广告和营销活动、印刷物料（传单、杂志、CD 封面等）、社交媒体。

禁止清单（全部来自官方条款）：

- 可识别人物不能以负面、冒犯的方式出现；
- 不能"未经修改地原样出售照片或视频"（比如当海报、印刷品、印在实体商品上）；
- 不能暗示图片中的人物或品牌为你的产品背书；
- 不能在其他图库/壁纸平台再分发或出售；
- 不能把图片用作商标、设计标志、商号、企业名或服务标志。

Pexels 有 [中文站](https://www.pexels.com/zh-cn/)，界面和检索都做了汉化；视频素材也是免费图库中质量较高的一档。API 注册简单（见后文），是"程序取图"的首选。

### Hippopx：基于 CC0 的中文图库，但当前访问受限

Hippopx 长期定位是"基于 CC0 协议的免版权图库"，提供中文界面、中文标签检索，20 万+ 张高清图，覆盖风景、建筑、美食、人物等。

但本次实测（2026-10-09）需要如实说明：**该站在自动化访问下会被 Cloudflare 的"人机验证"页面拦截**——`curl` 返回 403；用浏览器访问会进入"正在进行安全验证"页，本次测试未能通过该验证，因此本文无法直接展示其授权页的完整条款。可以确认的事实：DNS 正常解析到 Cloudflare 节点，TLS 证书由 Google Trust Services 于 2026-09-26 签发（有效期至 2026-12-25），说明站点仍在运营，只是防护策略不接受自动化流量。正常浏览器访问一般可以通过验证。

使用建议：按 CC0 图库使用，但**下载前逐图页确认授权信息**，并留意站点公告——CC0 图库变更许可或关站的情况在大图库圈子里发生过（参考 Pixabay 2019 年切换许可）。

## 所有平台的共同禁区

把六家的条款放一起，可以看到四条共同红线：

1. **原样转售/再分发**：把图片存成文件打包卖、印成海报卖、上传到别的图库——几乎所有平台都禁止。
2. **暗示背书**：图片来源无论哪家，都不能让画面给人"图中人物/品牌推荐了你的产品"的错觉。
3. **敏感、负面、违法用途**：含可识别人物的图片用于这些场景属于明确禁区。
4. **第三方权利**：肖像权、商标、地标建筑、私人财产不会因为图库授权而消失。Unsplash 帮助中心说得很直接："We cannot make any guarantees about the scope of permitted uses."

关于署名：已知条款的五家都是"不强制、推荐"（Hippopx 授权页内容未能核对，按 CC0 类图库惯例通常也不强制署名）。推荐做法是顺手写上，成本很低——标准格式示例："Photo by Jane Doe on Unsplash"（Unsplash），"by Contributor via Pixabay"（Pixabay），"Photo by John Doe on Pexels"（Pexels）。**注意 Pexels 的 API 使用场景是例外，见下文。**

## 四个常见的误用案例

条款读起来枯燥，落到具体场景就清楚了。下面四种做法都是真实世界里经常踩的坑：

1. **把 Unsplash 的风景图印成明信片在网店卖**。违反"不能未经显著修改地售卖图片"和"不提供把图片印在商品上出售的授权"。正确做法：换成允许印刷的素材站付费授权，或联系摄影师单独获取授权。
2. **把 Pixabay 的图片打包成"壁纸精选"App / 网站**。这同时踩了"standalone 再分发"和"复刻图库核心功能"两条，属于平台重点打击的行为。正确做法：用官方 API 且遵守规范，做有自身创作价值的产品，而不是图片搬运。
3. **用一张可识别人物的图配医美广告，写"用了某某项目，效果肉眼可见"**。人物形象 + 效果承诺，同时涉及"敏感/负面用途"和"暗示背书"。正确做法：找有明确模特授权（model release）的商业素材，或改用没有可识别面孔的画面。
4. **商业站点直接热链图库的图片 URL**。PicJumbo 条款明确要求下载按钮指向其图片页、禁止热链；其他站点的图床链接也可能变动或防盗链，页面出现裂图。正确做法：下载到本地（或自己的对象存储）再引用，同时保留来源记录。

这四类问题的共同点是：技术上都能"跑起来"，但法律上站不住。图库的免费是"授权免费"，不是"没有规则"。

## 搜索技巧

- **优先英文关键词**。除 Pixabay、Hippopx 的中文检索做得不错外，其他站点的标签体系以英文为主。找不到时换同义词：`coffee shop` → `cafe` → `coffeehouse`；`程序员` → `developer` / `laptop code`。
- **善用筛选器**。方向（横图 landscape / 竖图 portrait / 方图 square）、颜色、人数、尺寸是各站通用的筛选维度。写博客配图优先横图，手机壁纸优先竖图；电商 banner 按颜色筛能快速统一视觉风格。
- **从一张好图出发找相似**。图片详情页一般有作者主页和相关推荐；跨平台找同风格图，可以用 Google Lens、必应可视搜索以图搜图。
- **建立素材库**。下载时按"主题-来源-授权"命名保存（如 `coffee-pexels-license2026.png`），并把来源页面 URL、作者、授权页链接记在笔记里。商用项目被问来源时，这套留档能省很多事。
- **下载到本地，不要热链**。热链（直接引用图床 URL）在其他站点的条款里可能被明确禁止（如 PicJumbo），而且链接随时可能失效。

常用主题的英文关键词参考：

| 场景 | 建议关键词 | 备注 |
| --- | --- | --- |
| 程序员 / 开发 | `developer workspace`、`laptop code`、`programming` | 比 `programmer` 的可用素材多 |
| 商务 / 会议 | `business meeting`、`team collaboration` | `handshake` 已被滥用，慎选 |
| 科技感背景 | `abstract technology`、`gradient background`、`circuit board` | 海报底图优先选抽象类 |
| 中式 / 东方元素 | `chinese food`、`lantern`、`tea ceremony`、`ink painting` | 英文词的命中率高于中文翻译 |
| 生活 / 家居 | `cozy home`、`morning coffee`、`flat lay` | `flat lay` 适合俯拍风格的排版配图 |
| 自然 / 旅行 | `landscape`、`mountain sunrise`、`ocean waves` | 叠加时间词（sunrise/sunset）出图更好看 |

### 下载与留档清单

素材进项目前，建议过一遍这份清单：

1. 图片 ID / 页面 URL 记入项目文档或素材表；
2. 作者名字与主页链接（署名和后续联系都用得上）；
3. 授权页链接和查看日期（条款会变，日期是关键证据）；
4. 是否涉及可识别人物、商标、私人场所，需要额外授权就换图；
5. 下载原始尺寸到本地或自有存储，不依赖第三方图床。

## 用 API 批量取图：规则比技术重要

如果要把图库接进自己的工具（批量下载、素材管理、自动配图），Pexels 和 Unsplash 都提供官方 API。它们的技术文档很薄，真正要遵守的是使用规范。

### Pexels API

注册账号即可[申请 key](https://www.pexels.com/api/)，通过 `Authorization` 请求头认证。官方文档中的示例：

```bash
curl -H "Authorization: YOUR_API_KEY" \
  "https://api.pexels.com/v1/search?query=nature&per_page=1"
```

搜索接口支持 `orientation`（横竖方）、`color`（颜色或十六进制）、`size`（最小分辨率）、`locale`（含 `zh-CN`）等参数。限额默认 200 次/小时、20000 次/月，响应头会给 `X-Ratelimit-Remaining` 等额度信息。使用规范（来自官方 API 文档）：

- 每个 API 请求都应展示**显著的 Pexels 链接**（文本或 logo 均可）；
- 尽可能给摄影师署名，例如 "Photo by John Doe on Pexels" 并附图片页链接；
- 不得复制或复刻 Pexels 的核心功能（比如做成壁纸应用）；
- 不得滥用限额。

一个细节：Pexels 的图片资源自带 `src.large`/`medium`/`small` 等多种尺寸，做响应式页面时直接用对应字段，比拉原图再压缩更省事。

批量取图的落地建议：

- 把 API 返回的 `id`、`photographer`、`url` 落库，不要只存图片文件——署名和后续核查都依赖这些字段；
- 展示层使用 API 返回的最新 URL，而不是把某个链接写死在代码里（Unsplash 的 hotlink 要求本质上就是这个意思）；
- 遵守限额，给请求加缓存和重试退避，不要为"多拉一点"而绕过限制。

### Unsplash API

需要在 [Unsplash Developers](https://unsplash.com/developers) 注册应用，拿到 Access Key 和 Secret Key。官方 [API Guidelines](https://help.unsplash.com/en/articles/2511245-unsplash-api-guidelines)（2026-10 核对）要求比 Pexels 细：

1. 必须使用 API 返回的 **hotlinked 图片 URL**（`photo.urls` 系列字段），API 之外的直接下载路径不算；
2. 当应用行为类似"下载"时（用户选了图放进博客、设为头图等），必须请求 `photo.links.download_location` 上报下载；
3. 展示图片时必须署名 Unsplash 和摄影师，并链接回其主页，链接带 `?utm_source=你的应用名&utm_medium=referral`；
4. Access/Secret Key 必须保密，客户端直连需要加代理；
5. 不能把 API 用于出售原样照片、不能复刻 Unsplash 核心体验（非官方客户端、壁纸应用等）、要面向"非自动化、高质量"的使用场景。

一句话总结：**API 是用来做产品的，不是用来爬图的**。想批量囤图的，两家条款都不会支持，用爬虫绕过限额还会直接封号。

实操补充两点：Access Key 泄露会被滥用甚至封禁，客户端应用建议走自己的后端代理；Unsplash 对"自动化、低质量"的使用很敏感，靠脚本批量申请限额基本会被拒，用 API 前先想清楚产品形态。

## 链接可用性实测（2026-10-09）

| 链接 | curl 状态 | 说明 |
| --- | --- | --- |
| [unsplash.com](https://unsplash.com/) | 401 + 跳转验证页 | 站点正常，BotStopper 反自动化系统拦截脚本访问，浏览器访问不受影响 |
| [pixabay.com](https://pixabay.com/) | 403 | Cloudflare 拦截脚本访问；浏览器正常，官方授权页内容已核实 |
| [picjumbo.com](https://picjumbo.com/) | 200 | 可直接访问，FAQ 与 Terms 页正常 |
| [foodiesfeed.com](https://www.foodiesfeed.com/) | 403（默认 UA）/ 200（浏览器 UA） | 可访问，首页横幅显示网站正在出售 |
| [pexels.com/zh-cn](https://www.pexels.com/zh-cn/) | 403（curl） | 站点正常，License 页内容完整抓取核实 |
| [hippopx.com/zh](https://www.hippopx.com/zh) | 403 | Cloudflare 人机验证未通过；证书有效、站点仍在运营，授权细节未能直接核对 |

结论：六个站点目前都在线，其中三个（Unsplash、Pixabay、Hippopx）对自动化访问有较严的反爬策略，这属于防护行为，不代表站点关停。Hippopx 是唯一一个本次无法核对授权页内容的站点。

## 选图清单

按用途直接抄作业：

- **商用宣传 / 广告物料**：Pexels（官方明说可做广告和营销活动）> Pixabay（避开含商标的画面）> Unsplash（不要印在商品上卖）。三者都建议下载留档并记录作者。
- **博客 / 公众号配图**：Unsplash 出质感，Pexels 出场景，中文内容可补 Pixabay；同一篇文章尽量统一风格（都选同一站点、同一种色调）。
- **食物 / 食谱 / 餐饮**：FoodiesFeed 首选（图站定位最垂直，且是等效 CC0）；不够时用 Pixabay、Pexels 补。
- **中文界面 / 中文关键词检索**：Pixabay、Pexels 中文站、Hippopx（访问需通过人机验证）；Hippopx 使用前逐图确认授权。
- **需要 API 自动化取图**：Pexels 起步最省事（注册即用、限额明确）；Unsplash 素材质量高但规范要求多，适合认真做产品的团队。

## 总结

- 六家图库都允许免费商用、都不强制署名，但授权文件各不相同：Unsplash/Pexels/Pixabay 是自定义 License，FoodiesFeed 是等效 CC0，PicJumbo 是自有条款，Hippopx 宣称 CC0 且当前无法核对授权条款。
- 通用红线只有四条：不原样转售、不在图库平台再分发、不暗示背书、不用于敏感负面场景；在此之上还要自行排查肖像权、商标与物权。
- 搜索时优先英文关键词 + 方向/颜色筛选，下载后按"主题-来源-授权"留档，不要热链。
- 用 API 前先读使用规范：Pexels 要求显著署名链接，Unsplash 要求热链、下载上报与 UTM 署名——规范没做到，谈技术实现没有意义。
