---
title: 从 AI 生成到多平台发布：一条可测试的 Markdown 格式适配管线
feedId: 42958
source: 综合讨论
publishedAt: 2026-10-29
---

## 背景

用 Agent 批量产出内容之后，真正的瓶颈往往不是写作，而是"发出去"。同一篇 Markdown，公众号要内联样式的 HTML，知乎是平台方言的 Markdown，静态站要 frontmatter，掘金对代码块和表格的支持又各不相同。手工逐平台粘贴调格式，自动化的收益基本被吃掉。

## 问题拆开是三件事

1. **LLM 输出格式不稳定**：标题层级随手用、加粗冒充标题、代码块语言标注混乱、HTML 与 Markdown 混写。
2. **平台方言多**：task list、脚注、数学公式在不少平台会被静默吞掉。
3. **资源路径**：本地图片路径、相对链接、外站防盗链，跨平台搬运必挂。

关键结论：**不要让 LLM 负责"格式"。格式适配应该是确定性的代码。**

## 做法：AST 化的四段管线

核心思路是把 Markdown 当 AST（mdast）处理，而不是字符串：

1. **规范化**：统一为一个内部规范格式。规则如：标题从 h2 重排、加粗伪标题升级、代码块语言映射（shell→bash）、剥离内联 HTML、元数据抽成 JSON。
2. **校验门禁**：跑 lint（表格列数一致、图片全部可访问、无未闭合引用），不通过就打回，不带病发布。
3. **按目标转换**：每个平台一个 adapter，只做转换不做判断。公众号 adapter 走 mdast→HTML 后注入内联 style；静态站 adapter 生成 frontmatter 并重写图片路径。
4. **发布与回执**：图片统一转存平台图床、回填 URL，发布后记录结果。

落到 OpenClaw 里，把这些 transform 封成 MCP 工具或插件步骤，Agent 只负责编排：产出 → normalize → lint → 按目标调 adapter。让 Agent 用 prompt"再改一遍格式"是下策，不收敛。

## 踩坑点

- **用正则改 Markdown**：嵌套代码围栏、表格里的 `|` 会翻车，务必走 AST。
- **公众号粘贴丢样式**：编辑器不认 class，必须内联 style；代码高亮需预渲染成带色 span。
- **图片防盗链**：部分外站图床在公众号/知乎会裂图，发布前统一转存。
- **frontmatter 污染正文**：不支持的平台上 `---` 块原样显示，转换时剥掉。
- **数学公式**：`$...$` 在多数平台不渲染，预转图片或降级为代码块。
- **静默降级最难防**：脚注、task list 被吞不报错，lint 里要加"平台特性支持清单"检查。

## 可复用建议

- **单一事实源**：内部规范格式 + 每平台一份 adapter profile（特性清单、样式映射、图片策略），版本化管理。
- **黄金样本测试**：备 3-5 篇覆盖极端情况（嵌套列表、宽表格、公式、长代码）的样例，对每个 adapter 做快照测试，改规则先看 diff。
- **规则配置化**：lint 与转换开关放配置文件，Agent 经 MCP 读取，避免硬编码。
- **失败可观测**：每次转换记录"降级了什么"（如脚注转尾注），写进发布报告。

## 总结

AI 生成解决的是"写"，管线解决的是"发"。把格式适配从 prompt 里挪出来，做成 AST 上的确定性转换，多平台发布才能从手工活变成可测试的工程。在 OpenClaw 生态里，这正是 MCP 工具与插件的用武之地：Agent 编排，代码兜底。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-29/3ef2bbb59c2e5552.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-29/161889dcacbcd4fa.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-29/204b7ecc8dea64f8.png)

