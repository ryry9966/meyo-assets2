---
title: Markdown 管线：AI 生成内容到多平台发布的格式适配实践
feedId: 41617
source: 综合讨论
publishedAt: 2026-10-15
---

## 背景

我们团队的内容初稿现在基本都由 Agent 生成，产出是标准 Markdown。但发布端不止一个：博客和掘金吃标准 Markdown，公众号只接受内联样式的 HTML，知乎编辑器介于两者之间。最早靠人肉复制粘贴加手动调格式，一篇稿子发三个平台要花半小时以上，改一次稿就全部重来。

## 问题在哪

格式适配的本质不是“转换”，而是**降级 + 重排**：各平台支持的特性集合不同，转换器必须决定丢什么、留什么、怎么兜底。最怕的是静默丢内容——脚注没了、表格碎了，你都不知道。一次性脚本解决不了这个问题，平台行为一变就失效。

## 管线设计：五步

**1. 固定单一事实源。** 所有 AI 产出先过 markdownlint（单 H1、标题不跳级、表格前留空行），不合规让 Agent 自己修，人不管格式。

**2. 动态内容前置静态化。** Mermaid 图在源头用 mermaid-cli 渲成 PNG，数学公式转图片或确认目标平台支持再保留。这步越早做，后面 adapter 越简单。

**3. AST 层转换，不用正则。** 用 unified/remark 把 Markdown 解析成 mdast，按节点类型写 visitor：footnote 在受限平台降级为括号内联，task list 转普通列表加符号。正则改不动 blockquote 套列表套代码块这种嵌套结构。

**4. 每个平台一个 profile + adapter。** 共享同一份 mdast，输出差异只来自配置：

```yaml
# profiles/wechat.yaml
code_fence: inline-style-html
footnote: inline
math: image
image_host: required
table: degrade-to-list-if-wide
```

公众号走 remark-rehype 后再做 CSS 内联——class 会被剥掉，必须 inline 到每个标签上。

**5. 发布前 diff 校验。** 渲染产物存档并与上一版 diff，防止静默丢内容。整条链路包成 OpenClaw 的 skill / MCP 工具，Agent 写完稿自己调 publish，人只做最后确认。

## 踩坑记录

- **公众号剥 style 标签和 class**，highlight.js 的 class 高亮直接失效，只能把代码块预渲染成带内联颜色的 HTML；
- **单换行行为不统一**，有的平台当 `<br>`，有的当空格，要在源头把换行策略定死，别指望下游；
- **图片相对路径发布必挂**，adapter 里强制走图床并校验 URL 可达，SVG 在部分平台会被当附件下载；
- **宽表格在各端支持都很差**，超过四列直接降级为列表，别硬撑。

## 可复用的几点

- adapter 只依赖 mdast 和 profile，不依赖任何平台 SDK，新增平台 = 新 profile + 少量 visitor；
- profile 里显式声明“不支持列表”，缺省降级行为集中写在文档里，别散落在代码各处；
- canonical 源和各平台渲染产物一起进版本库，回滚和审计都靠它；
- pandoc 可以覆盖一部分场景，但涉及内联样式和特性降级时，AST 方案可控性明显更好。

## 总结

这套管线的核心不是某个转换器，而是“单一事实源 + 特性矩阵 + AST 适配层”的分层结构。AI 把内容生产成本压下来之后，格式适配才是真正吃时间的环节——它值得被当成一个正经子系统来维护，而不是每次发布前的手工劳动。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-15/3a193e4f23a51fc7.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-15/c54da823e2c204ad.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-15/da6442d082aa534f.png)

