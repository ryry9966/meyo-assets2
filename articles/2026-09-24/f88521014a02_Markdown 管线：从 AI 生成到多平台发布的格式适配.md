---
title: Markdown 管线：从 AI 生成到多平台发布的格式适配
feedId: 38716
source: 综合讨论
publishedAt: 2026-09-24
---

## 背景

Agent 产出的内容默认都是 Markdown，但发布目标——公众号、知乎、掘金、自建博客、Notion——的格式约束各不相同。单篇手动排版还能忍，当 Agent 批量产出时，格式适配就成了纯工程问题。这篇帖记录我们把「AI 生成 → 多平台发布」整理成一条 Markdown 管线的过程。

## 问题

1. **AI 输出不稳定**：同一套 prompt，有时给纯 GFM，有时夹带内联 HTML、脚注、mermaid，标题还会跳级。
2. **平台差异大**：公众号会剥离 class 且图片防盗链严格；知乎对 HTML 白名单管控；Notion 是块模型，只能走 API 转换。
3. **图片是重灾区**：本地图片、外链图、平台转存接口，三种情况处理逻辑完全不同。

## 做法

整体结构是「一次归一化，边缘适配」，分四步：

**第一步：输出契约。** 在 Agent 的 system prompt 里约束：只输出标准 GFM、图片用本地相对路径、禁止内联 HTML。这是成本最低的一道防线。

**第二步：归一化层。** 用 unified/remark 把 Markdown 解析成 mdast，再跑规范化 transform：修复标题跳级、脚注降级为括号注释、mermaid 块替换为预渲染 SVG、剥离 frontmatter。注意是 AST 操作，不是正则。

**第三步：平台适配器。** 每个平台一个 adapter 插件，输入同一个 AST：

- 公众号 adapter：遍历 AST 生成内联 style 的 HTML（平台会丢弃 class），图片先走转存接口换成平台 CDN 链接；
- 知乎 / 掘金 adapter：输出受限 Markdown，代码块语言标识映射到平台支持的高亮名；
- 博客 adapter：直出 GFM + frontmatter。

**第四步：校验门禁。** 发布前跑一次 lint：检查残留 HTML 标签、失效图片引用、超长代码行的移动端折行风险。dry-run 模式产出预览 HTML，确认后再真实推送。

整条管线以 MCP tool 暴露给 Agent：`normalize_markdown`、`preview`、`publish(platform)` 三个工具，Agent 只面对统一接口。

## 踩坑点

- 早期用正则做转换，嵌套代码块里的示例 Markdown 被误伤，换 AST 后这类问题归零。
- 公众号外链图会被替换成灰色占位，图片转存没有捷径，别指望平台宽容。
- 高亮语言标识不统一：`shell` / `bash` / `console` 指向不同效果，adapter 里需要一张映射表。
- AI 偶尔无视输出契约仍然吐 HTML，所以 lint 门禁必须常驻，prompt 约束只能降频，不能替代校验。
- 数学公式有 KaTeX / MathJax / 平台原生三套语法，统一在归一化层转换，不要把差异留给 adapter。

## 可复用建议

1. 把 AI 输出当不可信输入，永远先归一化再消费。
2. 中间表示选 AST 而不是字符串，后续扩展成本低一个数量级。
3. 每个平台 adapter 配一组 golden file 快照用例，改渲染逻辑时立刻暴露回归。
4. 发布动作默认 dry-run，真实推送需要显式参数，避免 Agent 误触发。
5. adapter 做成独立插件文件，新平台接入只新增一个模块，不动主干。

## 总结

这条管线的价值不在某个具体工具，而在结构：归一化层收敛 AI 输出的不确定性，适配器吸收平台差异，门禁兜住最后的意外。做完之后，新增一个发布平台的工作量从「一次重构」降到「写一个插件」，Agent 侧的发布调用也稳定了下来。如果你也在维护多平台内容自动化，建议先把归一化层立起来，剩下的都是增量。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/3bfc38718c516b1c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/acfc3b854fdf030f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/ab187f700b33a6da.png)

