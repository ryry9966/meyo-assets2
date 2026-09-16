---
title: Markdown 管线：从 AI 生成到多平台发布的格式适配
feedId: 37832
source: 综合讨论
publishedAt: 2026-09-16
---

## 背景

Agent 工作流里，LLM 的交付物几乎默认是 Markdown。但真正的下游往往不止一个出口：公众号、掘金、静态博客、RSS 订阅，各平台对 Markdown 的支持差异很大。公众号只认内联样式的 HTML；部分平台不支持脚注、mermaid、复杂表格；图片外链会被防盗链掐断。在社区里看到的实际情况是，格式适配吃掉了发布链路里相当比例的调试时间。

## 问题

问题分两层：

1. **LLM 生成的 Markdown 并不标准。** 常见症状：代码块缩进混用、无谓转义（`\*`）、标题层级跳跃、正文里夹带 HTML 片段。
2. **平台差异是持续变化的。** 用 sed/regex 写死的转换脚本，撑不过两次平台改版。

## 做法

我把管线拆成四层：

**1. 源头约束。** system prompt 里限定严格 GFM：禁行内 HTML、代码块一律 fenced、标题从 H2 起步。这一步能在源头消掉一半问题。

**2. 归一化层。** 落盘后先跑 remark + 自定义 lint：修列表缩进、剥多余转义、检查标题层级，同时统一中文排版（中西文间距、全半角标点，pangu 类规则包成 remark 插件即可）。产出 canonical Markdown + frontmatter，frontmatter 里声明目标平台列表。

**3. 适配层。** 基于 mdast 写转换插件，每个平台一个 adapter，输入 canonical AST，输出平台方言。公众号走 HTML + 内联样式（样式表用 JSON 主题描述，方便换肤）；静态站做图片路径重写和 frontmatter 注入。

**4. 校验层。** 每平台存 golden snapshot，发布前 diff 渲染结果，顺带检查图片可达性和死链。

图片处理单独一路：加一个 download-and-reupload 节点，远程图片先落地、再传到目标平台或自有图床，回写 URL。

## 踩坑点

- **正则改 Markdown 是最大的坑。** 代码围栏里的 `#`、`*` 会被误伤。所有转换必须走 AST，fenced code 天然是 leaf 节点，不会被打散。
- **LLM 偶尔输出 4 空格缩进的“代码块”**，在列表上下文里会被解析成嵌套列表。源头约束 + lint 双保险。
- **公众号会剥离 `<style>` 标签和 class**，SVG 也部分被清，只认内联 style。输出前必须再过一遍“内联化”步骤。
- **frontmatter 未被支持的平台会把 `---` 渲染成分割线。** adapter 的第一步永远是摘掉 frontmatter。
- **中文内容里的多余转义**（`\*`）在部分渲染器下直接显示反斜杠，肉眼很难在长文里发现，靠 lint 兜底。

## 可复用建议

- 每平台维护一份**能力矩阵**（表格/脚注/公式/mermaid 是否支持），adapter 据此自动降级，比如公式转图片、mermaid 转 PNG。
- 把"发布"封装成一个 **MCP tool**：入参 markdown + target，让 agent 不感知平台差异。比让模型自己临场调格式可靠得多。
- golden 文件进 CI，prompt 或转换规则改动后跑 snapshot 对比，避免“改了 A 平台坏了 B 平台”。
- **canonical 文件是唯一事实源**，禁止直接手改平台产物，否则下次构建全部回滚。

## 总结

Markdown 适合当中间表示（IR），不适合直接当发布格式。这套管线的关键就三件事：源头约束、AST 级转换、快照校验——把格式适配从每次发布的手工活，变成可测试的构建步骤。remark/unified 生态的积木都是现成的，自建一套大概一个下午，之后每接入新平台只是多写一个 adapter。与其在发布日手忙脚乱，不如把这条管线跑通一次。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-16/d6caee3854321e62.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-16/0119d86d4d29f12c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-16/cb88bac4ca5fee7a.png)

