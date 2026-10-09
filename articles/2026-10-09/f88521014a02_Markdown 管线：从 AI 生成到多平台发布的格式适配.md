---
title: Markdown 管线：从 AI 生成到多平台发布的格式适配
feedId: 41015
source: 综合讨论
publishedAt: 2026-10-09
---

## 背景

用 OpenClaw 或自建 Agent 做内容自动化，产出物几乎都是 Markdown。但“写完”和“发出去”之间隔着一条沟：公众号、知乎、掘金、静态博客、RSS 对 Markdown 的支持各不相同，把模型输出直接贴进平台，格式十有八九会坏。这篇记录我们收敛这条线的过程。

## 问题

- 公众号编辑器剥掉几乎所有 HTML，外链不可点，代码高亮只能靠内联样式；
- 静态站点（Hugo/Hexo/Astro）要 frontmatter，字段名还各不相同；
- 表格、脚注、任务列表、公式的支持程度参差；
- 图片相对路径一出站就挂，各家图床 API 又不统一；
- 模型输出本身不规范：嵌套列表缩进错、表格单元格里带竖线、代码块里嵌反引号。

## 做法

核心思路一句话：**一份 canonical 源文件 + 每平台一个适配器**。

1. **生成阶段约束模型**。System prompt 规定输出结构：H2 起步、禁用行内 HTML、frontmatter 固定字段、代码块统一用四反引号包裹。产出落盘为 `canonical.md`，之后不再手改。
2. **规范化**。用 remark/rehype（或 pandoc）跑一遍：统一列表缩进、修复围栏、把平台专属内容摘出来放进 frontmatter 的 overrides 字段。
3. **Lint 门禁**。markdownlint 加自定义规则（禁一级标题、图片必须是绝对 URL 等），不过就停。这一步能拦掉大部分 AI 生成的问题。
4. **适配器转换**。每个平台一个 transformer：公众号版把外链转脚注、代码块走高亮服务转内联样式；博客版生成对应 frontmatter、重写图片路径到本地资产目录；知乎版剥掉任务列表和脚注。
5. **预览与发布**。每个适配器配 dry-run，先出渲染 HTML 或截图人审，确认后再通过平台 API 或 MCP 工具推送。

这套流程可以做成独立 CLI，也可以把 lint / preview / publish 三个动作暴露成 MCP 工具，让 Agent 自己按顺序调用。

## 踩坑点

- frontmatter 里的中文冒号、引号会导致 YAML 解析失败，写入时强制给字符串加引号；
- 模型偶尔在代码块里输出三反引号，围栏提前闭合，生成阶段就要求四反引号包裹，比事后修补省事；
- 公众号对空白字符敏感，HTML 转换完还要过一遍空行归一化，否则段间距全乱；
- 图片上传和路径替换必须拆开：先把图传图床拿到 URL，再做替换。混在一步里，管线重跑就会重复上传。

## 可复用建议

- `canonical.md` 进 git，发布物视为构建产物，永远不要手改发布物；
- 适配器实现统一接口 `transform(canonical) -> artifact`，新增平台就是加一个文件；
- lint 报告随 artifact 一起存档，方便回溯“当时为什么发出来是坏的”；
- 全自动发布务必留 dry-run 闸门：预览落固定目录，人审通过才 publish。

## 总结

这条管线的本质不是“格式转换”，而是把平台差异收敛到适配器层，让上游（Agent 生成）和下游（发布）各自稳定。落地节奏建议先跑通“生成 → lint → 单平台”，验证幂等性后再逐个加适配器，比一步到位搭全家桶省太多时间。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-09/188d91369e8171f0.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-09/4affb8942f5bb658.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-09/b11310c8a232526a.png)

