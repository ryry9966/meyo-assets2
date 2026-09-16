---
title: Markdown 管线实践：让 AI 生成的内容一次成稿、多端发布
feedId: 37829
source: 综合讨论
publishedAt: 2026-09-16
---

## 背景

社区里不少人在用 OpenClaw 驱动 Agent 做内容生产：Agent 产出 Markdown，之后要发到公众号、知乎、掘金、GitHub Pages、Notion 等多处。写一次、发多处听起来简单，实际是个格式工程问题，值得认真对待。

## 问题

各平台对 Markdown 的支持差异比想象中大：

- 公众号编辑器剥离 class、不支持外部 CSS，代码高亮和表格基本要靠内联样式硬塞；
- 知乎对 HTML 兼容有限，任务列表、脚注会被直接丢弃；
- 静态站（Hugo/Astro）什么都能吃，但 frontmatter、短代码容易和正文混在一起；
- AI 生成内容还常带非标准语法：脚注、数学公式、内联 HTML，各平台表现不一。

如果每发一个平台就手工调一遍，成本高且不可复现。正确的做法是把“格式适配”做成管线里独立的一层。

## 做法

核心思路：**一份 canonical Markdown（规范中间格式）+ 一组平台适配器，转换基于 AST 而不是正则。**

1. **定义规范格式**。约定只用 CommonMark + 少量白名单扩展（表格、任务列表），禁用内联 HTML 和脚注。写一个 linter（remark-lint 足够）在入库时校验。
2. **归一化**。统一换行与空行、标题从 h2 开始、图片指向本地资源。这一步保持幂等，重复执行结果不变。
3. **AST 级适配**。用 remark（unified 生态）解析出 mdast，按平台 profile 做节点级变换：公众号目标把代码块节点替换为带内联样式的 pre 节点、给表格加 style；知乎目标把任务列表降级为普通列表加文字标记；不支持外链图的平台先转存图片再替换 URL。
4. **导出与发布**。适配后的 AST 重新 stringify 成平台接受的 HTML 或 Markdown，接 MCP 工具或平台 API 完成 发布。

这套流程可以直接挂进 Agent 工作流：Agent 只负责产出规范 Markdown，管线负责一切适配。各平台配置用声明式 profile（YAML），新增平台就是加一个 profile 文件。

## 踩坑点

- **别用正则做转换**。嵌套代码块里的 `#`、`---` 会把正则打穿，AST 是唯一稳妥方案。
- 公众号外部图片有防盗链，发布前必须转存图床；SVG 支持很差，能 PNG 就 PNG。
- 手机端表格容易溢出，长表格提前拆分或降级为列表。
- frontmatter 一定要在归一化阶段剥离，否则会原样发出去。
- 数学公式没有通用解，KaTeX 预渲染成图片是最稳的兜底。

## 可复用建议

- 把规范格式写成 lint 规则放进 CI：Agent 产出不合格直接打回重写，比事后修复便宜得多。
- 给每个平台 profile 建快照测试：同一篇样本文章的适配结果做 diff，profile 的任何改动都能被 review。
- 转换函数保持幂等、无副作用，方便本地调试和失败重放。

## 总结

多平台发布本质是个编译问题：canonical 格式是源码，平台 profile 是编译目标，AST 变换是编译器。把这件事工程化之后，Agent 的产出才能真正做到“一次生成，处处发布”，而不是每次发布都重新人肉排版一遍。管线一旦跑通，后续接新平台的边际成本会非常低——这也是内容自动化里最值得先建的基础设施。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-16/8966bb46d2ba4f69.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-16/bd2264b5d81b97a8.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-16/151d590c301dcde3.png)

