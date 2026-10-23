---
title: Markdown 管线实践：AI 生成内容的多平台格式适配
feedId: 42358
source: 综合讨论
publishedAt: 2026-10-23
---

## 背景

在 OpenClaw 的自动化场景里，Agent 的标准产出是 Markdown：文章、周报、发布说明，最终要落到多个平台——公众号、知乎、掘金、自建静态站。内容只有一份，格式的坑却有五份。我们的做法是把适配逻辑从 prompt 里挪出来，做成一条确定性的 Markdown 管线。

## 问题

格式不一致分三层：

1. **AI 输出不稳定**。同样的 prompt，模型这次用三级标题起头，下次直接加粗；frontmatter 时有时无，偶尔幻觉出一段裸 HTML。
2. **平台方言差异**。公众号只认内联样式的 HTML，`class` 全部失效；知乎对表格和脚本的容忍度不同；静态站则保留完整 HTML 能力。
3. **样式要求不同**。代码高亮、图片宽度、行距，每个平台一套。

如果靠 prompt 要求模型"输出适配公众号的格式"，等于把转换交给一个不稳定的生成器，重跑结果还会漂移。

## 做法

管线分四段，基于 unified/remark 生态实现，最后包成 MCP tool 暴露给 Agent 调用：

**1. 归一化。** Agent 产出先过 lint 和自动修复：标题层级强制从 `h2` 起（公众号对 `h1` 的渲染不可控）、列表项空行统一、代码块语言标签做白名单映射、frontmatter 按固定 schema 校验、裸 HTML 直接剥掉。这一步输出"规范 Markdown"，与平台无关。

**2. 元数据绑定。** 归一化后统一注入 frontmatter：标题、摘要、标签、目标平台列表。后续每段只读元数据，不再碰正文。

**3. 平台转换。** 以公众号为例：`remark-rehype` 转 AST 后遍历节点，把所有样式内联——`h2` 补字号行距，`table` 补边框和内边距，`pre` 换成带背景色的 `section`。知乎和静态站走同一入口、不同 renderer，静态站保留 `class` 交给主题 CSS。

**4. 发布适配。** 每个平台一个 adapter，输入规范 Markdown，输出草稿 ID。用正文 hash 做幂等去重，重跑管线不会堆出重复草稿。

整条管线对外只暴露两个 MCP tool：`markdown.normalize` 和 `publish.to`。Agent 只负责生成内容，格式适配完全确定性。

## 踩坑点

- **`rehype-raw` 会放行 AI 幻觉出的任意 HTML**，必须接 sanitize，否则发出去的就是安全隐患。
- **公众号会拦外链图片**（防盗链），转换阶段要把图片转存到自己的图床再替换 URL。
- **模型爱用加粗当小标题**。`**xxx**` 单独成段的情况，归一化时直接提升为 `h3`，比在 prompt 里反复强调有效得多。
- **AI 写的 YAML 不保证合法**：冒号后缺空格、值里带冒号都会让解析崩。校验前先做容错解析，失败就丢弃重建。
- **排查格式问题先分清来源**：保留 AI 原始输出做 diff，是生成时就错了，还是转换引入的？两类问题修法完全不同。

## 可复用建议

- 格式规范写成 lint 规则和 AST 转换，不要写进 prompt——规则可测试、可版本化，prompt 会漂移。
- 转换函数保持纯函数（字符串进、字符串出），配 snapshot 测试，升级依赖时一眼看出渲染差异。
- adapter 一个平台一个文件，新增平台的成本收敛到写一个 renderer。

## 总结

这条管线的核心思路：把"格式适配"从生成侧挪到管线侧。Agent 产出越规范越省事，但不必指望它完美——归一化兜底，转换确定性，发布幂等。内容一次生成，格式各按所需。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-23/61bf69c78538a66d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-23/9a5e20b6895af2c6.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-23/e27f3bbc8e277745.png)

