---
title: Markdown 管线：从 AI 生成到多平台发布的格式适配
feedId: 37658
source: 综合讨论
publishedAt: 2026-09-15
---

## 背景

自从把写作流程交给 Agent 辅助后，我发现生成内容只占工作量的三成，剩下七成在"把它变成各平台能直接发的样子"。同一篇 Markdown，博客要 frontmatter，公众号要内联样式的 HTML，知乎吃一半不吃一半。手动复制粘贴改格式，一周下来纯浪费。于是把这条链路做成了管线，分享一下设计思路。

## 问题拆解

实际跑下来，问题集中在三类：

1. **AI 输出不稳定**：同一个 prompt，模型今天给 frontmatter 明天不给；代码块语言标注时有时无；偶尔在正文里混 HTML 标签。
2. **平台格式差异大**：公众号不认 Markdown，表格会溢出，外链会被吞成纯文本；知乎的表格和代码块渲染与 GFM 有出入；掘金要求 YAML frontmatter 声明分类。
3. **资产处理麻烦**：图片外链在微信里经常因防盗链挂掉，代码高亮需要内联 style 而不是 CSS class。

## 管线设计

核心思路：**一份规范化中间格式，多个平台 adapter**。

### 第一步：规范化（normalize）

约定一份 canonical Markdown：固定 frontmatter 字段（title/date/tags/目标平台列表）、纯 GFM、图片一律本地相对路径。AI 生成后先过 remark-lint 校验，不符合就打回让 Agent 修，或脚本自动修（补全代码块语言标注、统一列表符号等）。

这一层的关键是别相信模型。prompt 里写清输出约定只是第一道防线，lint 是第二道，缺一不可。

### 第二步：平台适配（transform）

每个平台一个 adapter，基于 remark/rehype 的 AST 做转换。公众号 adapter 负责：Markdown 转 HTML、用 highlight.js 在构建期把高亮烧成内联 style、表格包一层固定宽度容器、把外链收集起来挪到文末"参考链接"。知乎和掘金的 adapter 简单得多，主要是裁剪 frontmatter 和替换图片路径。

### 第三步：资产处理（assets）

发布前统一处理图片：下载到本地、按平台压缩到规定尺寸、调平台上传接口换回平台 URL 并替换正文引用。公众号没有开放的内容发布 API，这一段用浏览器自动化兜底；其他平台尽量走 API。

### 第四步：发布（publish）

每个 adapter 输出两种结果：dry-run 产物（最终 HTML/Markdown + 资源清单）和发布动作。先人工过一眼 dry-run，确认再发。发布成功后把平台文章 ID 写回 frontmatter，方便后续更新同步。

## 踩坑点

- **高亮时机**：一开始用 CSS class 输出 HTML，贴进公众号全丢。必须在构建期内联化。
- **表格溢出**：公众号编辑器会把宽表格截断，转 HTML 时包固定宽度容器允许横向滚动，或直接降级为列表。
- **外链静默失败**：公众号把 `<a>` 渲染成纯文本且不报错，adapter 里要主动检测并重排到文末，别指望肉眼发现。
- **AI 混入 HTML**：模型偶尔在 Markdown 里塞 `<div>`，remark 处理 HTML 节点很别扭。normalize 阶段直接拒绝块级 HTML 输入，打回重生成。
- **图片上传幂等性**：重复发布容易二次上传，用内容 hash 做本地缓存，同图不重复传。

## 可复用建议

- 把 normalize / render / publish 拆成独立的 MCP tools，由 Agent 编排调用，人只在 publish 前过目。dry-run 输出统一 diff 格式，review 成本很低。
- 每个 adapter 配 golden file 测试：固定输入 Markdown，比对输出快照，平台改版时一跑就知道哪里坏了。
- canonical 格式宁窄勿宽：限制越少，adapter 越难写。

## 总结

管线跑通后，一篇文章从生成到三平台发布的边际成本降到几分钟，格式事故基本消失。经验浓缩成一句：**把"格式正确"当作管线的 invariant 来验收，而不是发布前的人工检查项**。AI 负责内容，管线负责确定性，各干各的。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-15/5f89fb83c384d589.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-15/0c2db5e00602ae7a.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-15/8e69a0ad9d8ba917.png)

